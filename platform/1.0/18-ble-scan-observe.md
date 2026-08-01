# 18 — BLE advertisement observation (scan/observe capability)

> status: **draft** (2026-07-19 new · realized + verified by the `ble_scan` recipe reference)
> peer: [`16-ble-transport.md`](16-ble-transport.md) (byte pipe to ONE server) · [`17-device-discovery.md`](17-device-discovery.md) (MCP-UUID-filtered probe) — same physical radio, different consumption
> binding: [`../../ui_dsl/1.3/08_Client_Extensions.md`](../../ui_dsl/1.3/08_Client_Extensions.md) §8.6.2 (`client.mcpStream` channel — how a bundle observes)
> reference impl: `os/core/brain_kernel/recipes/ble_scan/` (`publish_to: none`, vendored)

A capability that lets a bundle app **sense** nearby BLE advertisements — the connection-less, session-less broadcasts every BLE device emits — and render them live (a list, a signal chart, a beacon monitor). The motivating case: a bundle wants to draw a graph of incoming Bluetooth advertisements. That is not a transport (no server to talk to) — it is a **sensing** capability the host exposes and the MCP UI DSL binds. This standard fixes the data model, the multiplex model over one shared radio, and the DSL binding contract so any Flutter host and any bundle interoperate without per-host dialects.

---

## 1. Principle — observation is sensing, not transport

BLE advertisements are broadcast by every nearby device with no connection, no session, and no peer. Consuming them is fundamentally different from the two connection-oriented BLE standards. All three ride **one** physical radio; they differ only in what they do with it.

| | consumption | shape | this standard |
|---|---|---|---|
| **transport** (16) | connect to **one** MCP server | bidirectional byte pipe (GATT), a session | — |
| **discovery** (17) | find **MCP boards** among nearby devices | scan filtered by the MCP service UUID → probe-confirm | one privileged subscriber (§3) |
| **observation** (this, 18) | sense **many** devices' raw broadcasts | connection-less advertisement stream, a general data source | **normative here** |

Consequences that this standard commits to:

- Observation carries **no UUID filter requirement and no probe** — a subscription may observe every advertisement, or narrow by its own filter (§2), but it never "confirms" a device as an MCP node. That framing belongs to discovery.
- Discovery (17) is therefore **just one subscriber** of this capability whose filter pins the MCP service UUID and whose Stage-2 pipeline runs on top. It does not own the radio; it multiplexes over it like any other subscriber (§3).
- Observation is a **read/sense** surface. It never writes to a device, never opens a session, and never negotiates GATT. A bundle that needs to talk to a board uses 16, not this.

## 2. Data model

Two value types. Field names, types, and match semantics are normative — a host on any platform MUST expose exactly these shapes so a bundle written once renders everywhere.

### 2.1 `BleAdvertisement` — one observed broadcast

| field | type | meaning |
|---|---|---|
| `deviceId` | string | platform device identifier (stable per session; not the MAC on Apple platforms) |
| `name` | string | advertised local name; MAY be empty (the name is optional in the PDU) |
| `rssi` | int | signal strength in dBm; `0` when the platform does not report it |
| `serviceUuids` | string[] | advertised service UUIDs, **lowercased 128-bit form** |
| `manufacturerData` | int[] | raw manufacturer-specific data bytes (empty when none) |
| `receivedAtMs` | int | ingest timestamp (epoch ms), stamped by the radio at receipt so a chart can plot RSSI over time; `0` = unstamped |

A single device re-advertises continuously, so the **same `deviceId` arrives repeatedly** with a fresh `rssi`. This standard does not dedup at the source — consumers window/dedup as they see fit (the reference capability keeps latest-per-device; §4).

### 2.2 `BleScanFilter` — a subscriber's own constraint

| field | type | match rule |
|---|---|---|
| `serviceUuids` | string[] | match when the ad advertises **ANY** of these UUIDs; empty = any. UUIDs are compared **lowercased** |
| `deviceIds` | string[] | match only these device ids; empty = any |
| `minRssi` | int? | match when `rssi >= minRssi`; null = any |

All present constraints are **AND-combined**. An empty/unset constraint matches all, so a default (empty) filter observes every advertisement. The filter is the "different thing each bundle app registers" (§3): app A pins beacon UUIDs, app B sets an RSSI floor, app C names specific device ids — each an independent view of the same radio.

## 3. Multiplex model — one radio, N isolated subscriptions

The core requirement. A single physical BLE radio is multiplexed across **many concurrent subscriptions**, each with its **own** filter and its **own** event stream, fully isolated from the others.

### 3.1 The hub fan-out

- **One radio, one platform scan.** The radio is a raw source: it scans with **no service filter** (raw observation, not discovery) and emits every advertisement it hears. It is isolated behind an abstract seam so the multiplex logic is pure Dart (§5).
- **Subscribe → own stream.** A consumer calls `subscribe(filter)` and receives a **subscription** keyed by a stable `subscriptionId`, carrying that subscriber's filter and its **own** advertisement stream. The reference handle is `BleScanSubscription{ id, filter, events, cancel() }`.
- **Fan-out is per-subscription filtering.** For each advertisement the radio emits, the hub delivers it to **every** subscription whose filter matches — and only those. One radio feeds N different views; filtering happens per subscription, not on the radio.
- **Isolation is mandatory.** A subscriber MUST NOT observe another subscriber's existence, filter, or events. There is no shared advertisement bus visible across subscriptions; each sees only its own matched stream.

### 3.2 Radio lifecycle = subscription ref-count

The physical radio's power state is driven entirely by subscription count:

- The scan **starts** when the first subscription opens (`0 → 1`).
- The scan **continues** while **≥ 1** subscription is open.
- The scan **stops** and the radio powers down when the **last** subscription is cancelled (`1 → 0`).

`subscription.cancel()` (equivalently `cancelById(subscriptionId)`) decrements the ref-count. No consumer starts or stops the radio directly — the hub owns that transition. This makes an idle app cost nothing and guarantees the radio is never left scanning with no observer.

### 3.3 Discovery as a subscriber

Device discovery (17) is one such subscriber: its filter pins the MCP service UUID and its probe-confirm pipeline runs over the delivered stream. It obtains no special radio access — it ref-counts in and out like every other subscription. This is how discovery and observation share one radio without contention: they are two subscriptions, not two radios.

## 4. DSL exposure via `client.mcpStream`

A bundle observes advertisements declaratively through the `client.mcpStream` channel type (DSL 1.3, [`08_Client_Extensions.md`](../../ui_dsl/1.3/08_Client_Extensions.md) §8.6.2) — the binding contract between this capability and the UI. No new widget is required; observation is a stream that accumulates into state that an existing `list`/chart binds.

### 4.1 Declaring the observation channel

A bundle declares a `client.mcpStream` channel whose `uri` selects this capability by its **scheme** and whose `params` carry the §2.2 filter:

```json
{
  "type": "page",
  "channels": {
    "advertisements": {
      "type": "client.mcpStream",
      "params": { "uri": "ble://scan", "params": { "minRssi": -70 } }
    }
  }
}
```

- `uri` scheme = **`ble`** (`ble://scan`) selects the host-registered BLE scan source. The remainder is source-specific.
- inner `params` = the `BleScanFilter` (§2.2), forwarded verbatim to `hub.subscribe(filter)`.

### 4.2 Push → accumulate → render

Each observed advertisement arrives as one `onMessage` push (the payload binds
as the `data` context variable — the runtime's uniform channel onData contract —
and is a `BleAdvertisement`, §2.1). The `onMessage` handler is declared on the
channel itself; buttons only start / stop it. The bundle appends each push to
state and binds a list/chart to that state:

```json
"channels": {
  "advertisements": {
    "type": "client.mcpStream",
    "params": { "uri": "ble://scan", "params": { "minRssi": -70 } },
    "onMessage": {
      "type": "state", "action": "append",
      "binding": "advertisements", "value": "{{data}}"
    }
  }
}
```

A `list`/chart bound to `{{advertisements}}` then renders each row live (e.g. `{{item.name}}` + `{{item.rssi}} dBm`). The capability supplies data; the DSL renders. Accumulation policy (window / TTL / max-N) is the bundle's choice — the channel delivers raw pushes.

### 4.3 Lifecycle

- `channel.start` opens the subscription (ref-count `+1`, §3.2).
- `channel.stop` unsubscribes (ref-count `−1`); when it was the last subscriber the radio powers down.
- The channel machinery (backpressure, `onError`/`onConnect`/`onDisconnect`, restart policy) is the standard §8.6 behavior — observation adds no new lifecycle semantics.

### 4.4 The host seam

The runtime stays transport-agnostic. A host registers the `ble` scheme once, backed by the capability's `BleScanStreamSource`:

```
runtime.registerStreamSource("ble", BleScanStreamSource(hub).open)
```

`registerStreamSource(scheme, open)` (Flutter runtime) maps each channel subscribe to a hub subscription; the runtime never learns what `ble` streams. The source MUST tie the stream's cancellation back to the hub subscription: on `channel.stop` the runtime cancels its stream listener, and `BleScanStreamSource` forwards that to `subscription.cancel()`, decrementing the hub ref-count (§3.2). Returning the bare `hub.subscribe(filter).events` broadcast stream would **leak** — cancelling a broadcast listener does not cancel the subscription, so the radio would never power down. `BleScanStreamSource.open` wraps the subscription in a controller whose `onCancel` calls `subscription.cancel()`, which is why it, not the raw `.events`, is the seam. This is the whole binding: `client.mcpStream` ⟶ `registerStreamSource("ble", BleScanStreamSource(hub).open)` ⟶ one hub subscription per channel.

### 4.5 Request/response fallback (non-normative)

A runtime or host without live push MAY expose the same capability as a **subscribe + buffer + poll** tool surface — `ble.scan.start(filter) → {subscriptionId}`, `ble.scan.poll(id) → {advertisements}`, `ble.scan.stop(id)`, `ble.scan.list` — where each subscription accumulates matching advertisements into a bounded ring (latest-per-`deviceId`) and a timer re-polls. This is the same multiplex from the hub, adapted to a request/response executor; the reference capability ships it for agent-surface parity. The push binding (§4.1–4.4) is the primary path.

## 5. Placement

- **Host capability.** Observation needs the physical radio and OS Bluetooth permission, so it lives at the **host** layer, injected through the capability seam — not in a core package. **No kernel/core package carries a radio dependency** (`brain_kernel`, `mcp_client` untouched).
- **Realized by the `ble_scan` recipe.** The reference implementation is the `ble_scan` recipe (`publish_to: none`) — vendored by Flutter hosts (AppPlayer, Studio) exactly like the BLE transport/discovery scanners (16/17 §7). It is a sibling of the `capability_tools` recipe family.
- **Abstract radio seam.** The concrete radio (`universal_ble`, BSD-3-Clause; mobile/desktop/web) sits behind an **abstract `BleScanRadio` seam** (`start` / `stop` / `advertisements` stream). All multiplex logic (the hub fan-out and ref-count) is therefore **pure Dart, testable without hardware** — the reference passes its multiplex suite against a fake radio.
- **Dual surface.** The capability reaches both consumer surfaces: an **agent** via an MCP tool binding and a **bundle app** via the `client.mcpStream` channel (§4). Both drive the one hub; neither carries logic.

## 6. Permission & lifecycle

- **OS permission gate.** Observation is gated on the host's OS Bluetooth permission. A runtime MUST NOT open a `ble://scan` subscription before that grant; a denied grant surfaces as a channel `onError` (permission model per DSL §8.4).
- **Explicit scan consent.** Continuous scanning is **battery-relevant**. Starting a scan is an explicit, user-consented action ("start scanning") — a bundle SHOULD gate `channel.start` behind user intent, not auto-start silently, and the ref-count (§3.2) guarantees the radio idles the moment the last observer stops.
- **Platform support.** BLE radios exist on mobile and desktop. On **web / unsupported platforms** the host degrades gracefully: the `ble` scheme resolves to an unavailable source (the channel reports an unregistered/unsupported-source error rather than hanging), and `client.*` bindings on a non-capable runtime resolve inert rather than throwing.

## 7. Conformance

A host claiming this capability MUST:

1. Expose exactly the §2 `BleAdvertisement` and `BleScanFilter` shapes (field names, types, lowercased-UUID and AND/empty-matches-all semantics).
2. Multiplex N concurrent subscriptions over **one** radio, each with its own filter and its own stream, each isolated from the others (§3.1).
3. Drive the radio lifecycle by subscription ref-count — scan while ≥ 1 open, stop at 0 (§3.2).
4. Register the `ble` scheme via `registerStreamSource` so a `client.mcpStream` `uri: "ble://scan"` channel subscribes with its `params` filter and delivers one `onMessage` push per advertisement; `channel.stop` unsubscribes (§4).
5. Gate on OS Bluetooth permission and degrade gracefully where the radio is unavailable (§6).

The reference `ble_scan` recipe verifies filter isolation, single-radio fan-out across concurrent subscriptions, ref-count lifecycle, re-subscribe restart, and cancel isolation against a fake radio, plus a live DSL round-trip (subscribe → state → `list` render).

## 8. Open items / non-goals

- **Multi-device CONNECT is out of scope.** Extending the same multiplex model beyond observation to concurrently **connecting** many devices couples to transport (16) multi-session semantics — a separate standard, not this one. This standard covers connection-less sensing only.
- **Filter renegotiation** via `channel.send` (adjusting a live subscription's filter without stop/start) is a follow-on; today a filter change is stop + start with new `params`.
- **Real chart widgets** — a first-class time-series/RSSI chart bound to the accumulated state — are a UI follow-on; the current binding renders via `list` and generic widgets.
- **Radio-sharing coordination detail** between discovery (17) and observation under concurrent scan sessions is settled in principle here (discovery = one subscriber, §3.3); host-side scan-session tuning is left to the reference impl.
