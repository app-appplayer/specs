# MCP Gateway Dispatch Protocol — 1.0

How full MCP crosses a mediated relay. A tool-providing node sits behind NAT and dials OUT; consumers reach it through a hub. The endpoints run a gateway layer; the hub relays opaque frames. This spec defines those frames, the verbs they carry, the routing/namespace rules, the error space, and the liveness and trust requirements — so every embedder (AppPlayer, Studio, headless nodes, the marketplace hub) speaks one protocol instead of re-deriving it from an implementation.

```
local mcp_server ── gateway(provider) ──▶ relay(hub) ◀── gateway(consumer) ── mcp_client
                      dials out             opaque         dials out
                                            pipe
```

- The **channel** (platform spec 15) carries the frames and owns authentication, session lifetime, and close semantics. It never opens a frame.
- The **gateway** (this spec) owns the frames: request/response mediation, aggregation, namespacing, notifications, and the reverse direction.
- Reference implementation: `mcp_gateway` (pub) — library semantics; `recipes/gateway_node` — the hub binding.

## 1. Frames

A frame is one JSON object. Keys are single-purpose and flat:

| key | meaning |
|---|---|
| `k` | frame kind: `req` · `res` · `event` · `rreq` · `rres` |
| `id` | correlation id — REQUIRED on `req`/`res`/`rreq`/`rres`, absent on `event` |
| `method` | MCP method name (`req`/`rreq` only) |
| `params` | method params object (`req`/`rreq` only) |
| `ok` | boolean success (`res`/`rres` only) |
| `result` | result object when `ok` (`res`/`rres` only) |
| `error` | `{code, message, data?}` when not `ok` |
| `providerId` · `eventType` · `payload` | `event` frames only |

- `req`/`res` flow consumer → provider (through the gateway); `rreq`/`rres` flow provider → consumer (the reverse direction: `sampling/createMessage`, `roots/list`, `elicitation/create`, `ping`). A frame kind is never inferred from the method name.
- `event` frames are one-way provider → consumer. `eventType` carries the **standard MCP notification method name** when one exists (`notifications/resources/updated`, `notifications/tools/list_changed`, `notifications/progress`, `notifications/cancelled`, …); non-standard events ride `notifications/gateway/event` with the original type in the payload. The canonical JSON-RPC projection is: `{jsonrpc:'2.0', method: eventType, params: payload}`.
- Correlation: a responder MUST echo `id` verbatim. A consumer MAY pipeline requests; ordering across distinct `id`s is not guaranteed. Streaming results ride multiple `res` frames with the same `id` plus `{stream: {seq, end?}}` in `result` (binding-level; see the reference hub binding).

## 2. Verb catalog

### 2.1 Pass-through MCP methods (`req`)

`initialize` · `ping` · `tools/list` · `tools/call` · `resources/list` · `resources/templates/list` · `resources/read` · `resources/subscribe` · `resources/unsubscribe` · `prompts/list` · `prompts/get`.

- **List verbs** are answered by the GATEWAY (aggregation across providers, policy-filtered per consumer). They support spec pagination: `params.cursor` in, `result.nextCursor` out (opaque cursor).
- **`initialize`** is answered by the gateway facade: it negotiates `protocolVersion` across the revisions it supports (echo the client's when supported, else answer with the latest supported) and declares capabilities `{tools:{listChanged}, resources:{subscribe, listChanged}, prompts:{listChanged}}`.
- **Item verbs** (`tools/call`, `resources/read|subscribe|unsubscribe`, `prompts/get`) are routed to one provider (§3) and forwarded with ORIGINAL names/URIs.
- Methods outside this catalog MUST be rejected with `-32601` unless the request carries an explicit target hint (§3.4), in which case the gateway forwards them verbatim (future-proof pass-through).

### 2.2 Reverse methods (`rreq`)

`sampling/createMessage` · `roots/list` · `elicitation/create` · `ping` — issued by a provider against the consumer that sent it a dispatch (the dispatch carries the consumer identity) or, when exactly one consumer is attached, against that consumer. A consumer that does not implement a reverse method answers `ok:false` with `-32601`.

### 2.3 Notifications (`event`)

| eventType | emitted when |
|---|---|
| `notifications/tools/list_changed` · `notifications/resources/list_changed` · `notifications/prompts/list_changed` | a provider registers, updates its schema, or unregisters — fanned out to every attached consumer |
| `notifications/resources/updated` | a subscribed resource changed — delivered ONLY to consumers whose `resources/subscribe` for that URI was acknowledged |
| `notifications/progress` | a provider reports progress for an in-flight dispatch whose request carried `_meta.progressToken`; payload = `{progressToken, progress, total?, message?}` |
| `notifications/cancelled` | delivered TO the provider (as a queued one-way dispatch) when the consumer cancelled; params = `{requestId, reason?}` |

## 3. Routing and namespaces

1. **Tools / prompts** are namespaced `{providerId}.{name}` in aggregated lists; item verbs parse the prefix and forward the ORIGINAL name to the provider.
2. **Resource URIs are never rewritten.** URIs are absolute identifiers; renaming them breaks consumers (UI renderers resolve `ui://…` literally). Ownership is resolved from the registry: exact registered URI first, then RFC 6570 level-1 template match (`{var}` = one path segment). Aggregated list items carry provenance in `_meta.gateway` (`providerId`, `providerVersion`).
3. **Collisions**: exact-URI ownership is first-registered-wins; consumers needing a specific provider use the target hint.
4. **Target hint**: `_meta.gateway.target = providerId` inside `params` overrides routing for any method. (Top-level `_gateway` is accepted as a deprecated alias through 1.x.) The gateway strips the hint — and only the hint — before forwarding; the rest of `_meta` (e.g. `progressToken`) passes through untouched.

## 4. Error space

- Standard JSON-RPC / MCP error codes pass through the relay UNTOUCHED — the gateway never remaps a provider's error. The gateway itself answers with standard codes where the spec defines them: `-32601` method not found, `-32002` resource not found.
- Gateway-ORIGINATED conditions use the vendor block **-33001..-33099** (outside the JSON-RPC reserved band, so they can never shadow a standard code):

| code | condition |
|---|---|
| -33001 | provider not found |
| -33002 | schema update conflict |
| -33003 | provider offline |
| -33004 | dispatch timeout (provider never answered within TTL) |
| -33005 | batch limit exceeded |
| -33006 | invalid dispatch id |
| -33007 | schema version mismatch |
| -33008 | unauthorized |
| -33009 | forbidden (policy) |
| -33010 | rate limit exceeded |
| -33011 | queue overflow (backpressure) |

## 5. Liveness (normative)

- A gateway MUST bound the time a consumer request can stay unanswered (`dispatchTtl`; reference default 120 s) and complete it with `-33004` when exceeded.
- A gateway MUST bound the per-provider pending queue (`maxQueueDepth`; reference default 1000) and fail fast with `-33011` instead of growing memory without bound.
- Completed dispatch records MUST NOT be retained past a bounded horizon.

## 6. Trust boundary (normative)

- The consumer identity a gateway attaches to a request is trusted only as far as the CHANNEL authenticated it (platform 15 §6 — hub key gate, relay session). A gateway MUST NOT accept identity claims from frame content.
- Policy (visibility, per-consumer filtering, rate limits) is enforced AT the gateway, before dispatch; providers may assume policy-passed traffic but MUST still validate params.
- `_meta.gateway` is a routing hint, never an authority: a target hint cannot bypass policy.

## 7. Conformance profiles

| profile | required |
|---|---|
| **core** | `initialize` · `ping` · `tools/list` · `tools/call` · list_changed events · §4 error space · §5 liveness |
| **full** | core + resources family (incl. subscribe/updated) + prompts family + pagination + progress/cancelled + reverse methods |

A binding (e.g. the hub wire) states its profile. The marketplace hub binding targets **full** — UI serving requires `resources/read`.

## 8. Versioning

This document is versioned as `mcp_gateway/spec/1.0`. Frame keys and verb semantics are frozen within a major version; new verbs and event types are additive within 1.x. The `initialize` negotiation carries the MCP protocol revision (2024-11-05 / 2025-03-26 / 2025-06-18 at time of writing) — MCP revision and gateway spec version are independent axes.
