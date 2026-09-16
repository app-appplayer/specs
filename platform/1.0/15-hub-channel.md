# 15 — Hub (a single relay data-plane · consume = connect to a server / expose = publish one)

The contract for a remote MCP connection made through a hub. **This document is the definition (canon), and its consumers — hosts (AppPlayer · Studio · a local gateway app), an embedding marketplace, and [`mcp_gateway`](https://pub.dev/packages/mcp_gateway) — follow it.** Per-host improvisation is forbidden.

> **Connection model.** A hub *mediates* discovery and connection and then **stays out of the data path**. A host only ever "opens an MCP surface over a transport"; it never talks to a mediator's database. There are exactly **two directions**:
>
> - **consume**: the host's `mcp_client` **always connects to a server**. That server is either a cloud endpoint (direct HTTP) or a **hub-exposed local device** (relay ws). The consumption contract is **single** — "connect to a server". A hub is not itself something you consume.
> - **expose**: a host **connects to the hub and publishes its own local surface as a server** (an `mcp_gateway` *provider*). What it exposed looks to everyone else like **one server**. A host doing both at once is the interesting case: it consumes and is consumed over the same substrate.

---

## 0. Identity

A hub exposes a **bidirectional frame pipe** between two participants under one contract that says nothing about how the transport is implemented.

- **control-plane** = session open/close/access. Owned by the mediating backend over REST. That package stays **pure-http** — it is not on the frame path.
- **data-plane = the relay, and only the relay.** Real frames flow over the **ws relay** (`hub/relay`) alone. A mediator does not invent its own transport and route frames through its own database.
- The relay **does not open frames** (opaque pass-through). What a frame *means* — MCP dispatch, an event — is the business of the gateway at each end.

## 1. Structure (layers · fixed)

```
endpoint ── gateway ── relay(hub) ── gateway ── endpoint
 (app)    (mcp_gateway)  (ws pipe)  (mcp_gateway)  (exposed node = server)
```

- **endpoint** = an app (Studio · AppPlayer · a local gateway app · a machine on a factory floor).
- **gateway (`mcp_gateway`)** = the MCP routing/adapter layer at an endpoint (§8). An exposed node runs the provider side; a consuming endpoint runs the consumer side.
- **relay (hub)** = the cloud ws transport server. Both endpoints dial out as `role=client`, and the relay passes frames through opaquely.
- **server** = the authoritative node on the far side (a local surface that exposed itself). To the consuming side it is **one server**.

## 2. Session · relay grant

- **A channel instance is a session.** The control-plane opens one (`POST /hub/sessions`) and **always** returns a relay grant: `{ sessionId, status, policy, relayToken, relayUrl }`. There is no `transport` field — the relay is the only one.
- **The address is the `sessionId`.** The host connects to `relayUrl`: `wss://<relayUrl>/?role=client&sessionId=<sid>&token=<relayToken>`. It **does not pass a `nodeId`** — the relay resolves that from the control-plane.
- **Who the participants are is a control-plane fact** (session metadata). The host receives only the grant, which is a reference.

## 3. Frames (the relay is an opaque pipe)

- A relay wire frame **is the gateway dispatch message itself**, as an opaque payload. The relay does not wrap it in a seq/sender envelope — ws already gives TCP ordering, so this is a **raw frame pipe**.
- **One exception, toward the node: an address envelope.** A single node socket carries **many consumer sessions**, so that direction alone is enveloped: relay → node `{type:'open'|'close', sessionId}` (a session beginning or ending) and `{type:'frame', sessionId, frame}` (where `frame` is the opaque payload, unchanged); node → relay `{sessionId, frame}`. **Toward the consumer there is no envelope** — a frame goes out exactly as it came in. What the envelope carries is an address and nothing else, so §0's "the relay does not open frames" still holds. The node-side endpoint reads this shape, which is why it is written here: it is a contract between two implementations.
- **Ordering is TCP's, and there is no replay buffer.** When a transport drops, the frames that were on it do not carry over — **the session does** (§6).
- **Delivery is at-most-once.** A frame the relay has handed over is never sent again. The channel cannot tell "received it and then died" from "never received it" (the `id` response of §8 is the only confirmation), and **a resend can run a tool twice** — MCP `tools/call` is not idempotent. An unanswered request is completed by the calling gateway's `dispatchTtl` as `-33004`, so the caller does learn of it. A use that needs durability solves it above the channel in its own way (server-side storage, for instance) — the channel is a live pipe.
- The protocol carried inside a frame (gateway dispatch: request/result + event) is §8. The channel only passes it.

## 4. Transport = the relay ws, and it belongs to the host

- The host attaches to the relay ws with a `HubRelayConnection` — self-contained given `relayUrl` / `sessionId` / `relayToken`. The mediating package takes no direct dependency on a websocket or a database client.
- **A closed socket is not a closed session.** The relay reports that the transport detached (`POST /hub/relay-detach`, server-to-server) and the control-plane keeps the session **open**, recording only that nothing is attached right now. **The one exception is `oneshot`** — in that mode alone a socket close is the end signal, because the relay never opens a frame and therefore cannot observe the response that completes the turn. Settlement happens once, when the session actually ends, however many times the transport dropped in between.
- **Re-attaching**: an open session is re-entered **with the same `sessionId`** (relay-verify re-checks that it is still `open`). Two sockets cannot hold one session at once — while a live one is attached a second attempt is refused, so session hijacking stays closed off.
- **A host only has to reconnect.** Redial with the same `sessionId` and `relayToken`; whatever was in flight on the old socket is cleaned up by `dispatchTtl` (§3 — there is no resend).
- Metering is relay frame bytes → `POST /hub/relay-usage` (one path).

## 4-1. Waiting and waking (when the exposed node is not attached)

**An exposed node does not have to be attached all the time.** Not holding a physical connection open while there is no data *is* this channel's cost model, which makes "the node is not attached" a **normal state** rather than an error.

| Step | What |
|---|---|
| 1 | A consumer request reaches the relay. That session's node is not attached |
| 2 | The relay holds the request in **its own memory**, per session (count bounded by the same number as the gateway's `maxQueueDepth`; a byte bound is the relay's own guard). Anything past the bound is **not accepted** |
| 3 | The relay asks the control-plane to wake the node — a push to the devices registered for that node's owner |
| 4 | When the node attaches on the same `sessionId`, what was waiting is handed over **once** and cleared (§3, at-most-once) |
| 5 | If the node does not arrive within `dispatchTtl`, the caller gets `-33004` and the waiting frames are dropped |

- **The waiting lives only in the relay's memory.** Frames are never written into the mediator's database (§0 and §5 stand unchanged). It exists only while a caller is waiting for an answer, so when the relay dies the consumer's socket dies with it and **the caller redials on its own**.
- **Waking is not a guarantee.** A device that cannot receive a push (a browser, say) is reachable only while already attached; that case ends at step 5.
- **The relay does not manufacture errors.** It never opens a frame, so it does not know a JSON-RPC `id`, and without one it cannot invent a response. `-33004` and `-33011` are codes the **calling gateway** completes with, out of its own deadline and its own queue (§3). The relay's job is narrower — **hold, bound, hand over once, and drop when the deadline passes.** Blurring that line would force the relay to open frames, and at that moment §0's "the mediator stays out of the data path" is gone.
- **A node that stays attached is equally valid** (devices that need immediacy). Steps 1–3 simply do not occur; the contract is the same.

## 5. The control / data boundary

- **control (REST)**: node registration (`/hub/nodes`), access scope, session open/close, and session metadata (participants, status, node id, client identity). It never sees a frame.
- **data (relay ws)**: opaque frame relaying only. No database hop.
- This separation is the point — the mediator is an http policy decision-maker, and the relay is transport substrate.

## 6. Authentication · lifetime

- **Authentication**: a `relayToken` — an opaque bearer signed by the control-plane and bound to the session. The relay stores no secret; on every connection it delegates with `POST /hub/relay-verify` and **fails closed**: the stored session token must match and the session must be `open`, after which the control-plane resolves the node id.
- **Lifetime is decided by policy, not by the transport**: `unlimited` (tidied only on inactivity) · `ttl` · `idle` · `oneshot`. The only other endings are an explicit `DELETE /hub/sessions/:id` and removal of the node. A closed session's `relayToken` is refused at relay-verify.
- **The session document is the reconnection anchor.** It stays alive independently of any transport or server instance, and that is what makes "dropped, then re-attached, is the same session" true.

## 7. consume vs expose (the canonical statement of direction)

- **consume**: `connectHub` is not "connect to a hub" — it is **a variant of `connectServer` whose transport is the relay**. What the consumer does is identical whether the server is a cloud endpoint or a hub-exposed local one: connect to a server. The consumer does not pick a node id or a transport; deployment decides, and the contract stays single.
- **expose**: the word "hub" is used for the **publishing direction** only — a host making its local surface available as a server (`mcp_gateway` provider: register + poll/serve). It is a publish mechanism, not something you consume. First-class as `exposeThroughGateway` / `HubGatewayProvider`.
- Left to the application and **not** part of this canon: collaboration, presence and rooms; payment and settlement; the per-app schema carried inside frames; and the policy linking a catalog entry to a hub node.

## 8. `mcp_gateway` binding (structure fixed · adoption per app)

`mcp_gateway` — in-process MCP routing and dispatch plus an event bus — is the gateway layer of §1. Gateway dispatch and event frames cross the relay as payload.

> **The canon for frames, verbs, the error space and the trust boundary is [`../../gateway/1.0/`](../../gateway/1.0/README.md)** (the Gateway Dispatch Protocol). This section covers only the coupling with the channel — the reverse tunnel, the poll model, and where the reference implementation sits. The frame schema (req/res/event/**rreq/rres**), the verb catalog (full MCP: resources, prompts, notifications, the reverse direction), the vendor error band (-33xxx), and the liveness and trust requirements all come from that spec. The target profile for a hub binding is **full**, because UI serving presumes `resources/read`.

- **The topology is a reverse tunnel.** The relay is a cloud transport server and both ends dial out as `role=client`. **The node that exposes tools — the MCP server — is a dial-out client**, so there is no inbound path: it **pulls dispatch with a long poll** and returns results. That is why the protocol above the wire is gateway dispatch (request/result + event) rather than raw MCP JSON-RPC. Note the naming inversion this creates: `GatewayClientAdapter` says "Client" but is the provider (server role), and `GatewayServerAdapter` says "Server" but is the consumer (client role).
- **Per app**: a node exposing a local surface (Studio, a factory machine, a local gateway app) runs the provider bridge. A pure consuming endpoint runs only the consumer bridge. Anything not listed is a subset.
- **Implementation: no core modification, assembled from the public surface.** Ingress is `GatewayRuntime.handleConsumerRequest`; egress is `runtime.eventBus.onEventRelay` (the same hook the package itself uses in `GatewayServerAdapter`); provider registration and polling is `GatewayClientAdapter`. The relay transport is injected by the host. The reference implementation is the `gateway_node` recipe: **`HubGatewayProvider`** (expose), **`HubGatewayConsumer`** (consume), **`HubConsumerTransport`** (surfacing a consumer as an `mcp_client.ClientTransport`), and **`HubRelayConnection`** (the relay ws).

## 9. A hub is the **market door** (and where the account door begins)

What a hub opens is the **direction in which you publish to others** — which is exactly why listings, access lists, tenancy and wallets attach here. Devices signed in to the **same account** lending one another a connection is a **different door**, and it is not layered on top of a hub's nodes and sessions: dragging publisher concepts into a place that has exactly one owner drags in everything that place will never use.

| | Hub (the market door) | Account peer (the account door) |
|---|---|---|
| To whom | Someone else | My own devices |
| Introduction | Node registration · session grant | A directory and an offer, held by the account service |
| Data | Through the relay | **Device to device**; where that is blocked, a **shared TURN** — not a hub session. The relay is borrowed as a temporary detour only until TURN is standing |
| Frames | Gateway dispatch (§8) | **Plain MCP** |
| Rule for what you expose | One re-exposure rule governs **both** |

What the account door borrows from this document is **the relay, and only that** — a last resort on networks where a direct path is blocked. Even then §0 holds: a direct path only moves the mediator further out of the data path.

---

## Consumer alignment

- **An embedding marketplace**: control-plane REST plus the relay grant (a reference) — pure http.
- **Hosts (AppPlayer · Studio · a local gateway app)**: the §4 data-plane over relay ws. Consume with `connectHubViaGateway` (the relay seam) + `HubConsumerTransport`, which surfaces as a host server-open. Expose with `exposeThroughGateway` + `HubGatewayProvider`.
- **The `gateway_node` recipe**: the §8 binding, built from the public surface with no core modification.
