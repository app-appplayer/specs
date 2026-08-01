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
- **Ordering is TCP's.** There is no replay buffer, and **a reconnect is fresh**. A use that needs durability solves it above the channel in its own way (server-side storage, for instance) — the channel is a live pipe.
- The protocol carried inside a frame (gateway dispatch: request/result + event) is §8. The channel only passes it.

## 4. Transport = the relay ws, and it belongs to the host

- The host attaches to the relay ws with a `HubRelayConnection` — self-contained given `relayUrl` / `sessionId` / `relayToken`. The mediating package takes no direct dependency on a websocket or a database client.
- **relay-close**: ending a session **is closing the socket**. The relay signals the control-plane (`POST /hub/relay-close`, server-to-server), which sets `status: closed` and settles. **A host closes its socket normally and does nothing else.** A one-shot session ends the moment the host finishes its turn and closes. Because the relay never opens a frame, it observes closure as a socket close and not by parsing an MCP response.
- Metering is relay frame bytes → `POST /hub/relay-usage` (one path).

## 5. The control / data boundary

- **control (REST)**: node registration (`/hub/nodes`), access scope, session open/close, and session metadata (participants, status, node id, client identity). It never sees a frame.
- **data (relay ws)**: opaque frame relaying only. No database hop.
- This separation is the point — the mediator is an http policy decision-maker, and the relay is transport substrate.

## 6. Authentication · lifetime

- **Authentication**: a `relayToken` — an opaque bearer signed by the control-plane and bound to the session. The relay stores no secret; on every connection it delegates with `POST /hub/relay-verify` and **fails closed**: the stored session token must match and the session must be `open`, after which the control-plane resolves the node id.
- **Lifetime**: `open → closed/swept`. Closure is the host closing its socket (which triggers relay-close) or a policy-expiry sweep. A closed session's `relayToken` is refused at relay-verify.

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

---

## Consumer alignment

- **An embedding marketplace**: control-plane REST plus the relay grant (a reference) — pure http.
- **Hosts (AppPlayer · Studio · a local gateway app)**: the §4 data-plane over relay ws. Consume with `connectHubViaGateway` (the relay seam) + `HubConsumerTransport`, which surfaces as a host server-open. Expose with `exposeThroughGateway` + `HubGatewayProvider`.
- **The `gateway_node` recipe**: the §8 binding, built from the public surface with no core modification.
