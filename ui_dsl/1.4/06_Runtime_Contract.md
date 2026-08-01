# 06. Runtime Contract

This section defines the contract between a DSL runtime and its MCP server: protocol facilities, initialization flow, resource subscription modes, host integration points, lifecycle hook placement, error handling, and performance expectations.

Normative requirements live in [`18_Conformance.md`](18_Conformance.md) §18.2.6 (MCP protocol), §18.2.11 (performance).

## 6.1 MCP Protocol Integration

A conformant runtime communicates with a server over the Model Context Protocol (JSON-RPC 2.0 — https://spec.modelcontextprotocol.io). The DSL relies on three protocol facilities:

| Facility | Purpose |
|----------|---------|
| `resources/read` | Fetch UI definitions and bundle assets (`ui://app`, `ui://page/*`, `ui://app/info`, `bundle://...`) |
| `tools/call` | Execute server-side tools invoked by `{"type": "tool", ...}` actions |
| `notifications/*` | Receive server-pushed events, including `notifications/resources/updated` for subscribed resources |

All DSL definitions MUST be valid JSON. The runtime performs no speculative parsing: a malformed document is rejected whole (see §6.9).

## 6.2 Well-Known Resources

| URI | Returns | Since |
|-----|---------|-------|
| `ui://app` | ApplicationDefinition | v1.0 |
| `ui://page/{name}` | PageDefinition for a lazy-loaded route | v1.0 |
| `ui://page/info/{name}` | Optional lightweight page metadata | v1.0 |
| `ui://app/info` | Lightweight app metadata — see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.6 | v1.2 |
| `bundle://...` | Bundle-internal asset — see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.5 | v1.2 |

Runtimes claiming the Bundle Profile MUST resolve `bundle://` URIs inside the currently loaded bundle only.

## 6.3 Initialization Flow

1. Runtime connects to the MCP server.
2. Runtime issues `resources/read` for `ui://app`.
3. Runtime parses `ApplicationDefinition` and resolves its `version` (defaults to `"1.0"` when absent — see [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.7).
4. Runtime evaluates `theme`, `state`, and `navigation` on the ApplicationDefinition.
5. Runtime navigates to `initialRoute` (or the first declared route when absent).
6. Runtime fetches the initial page definition via `resources/read` on the mapped `ui://page/*` URI.
7. Runtime renders page content.
8. Runtime fires definition-level lifecycle hooks in order: `onInit` → `onMount` → `onReady` (see §6.8).

Subsequent route changes repeat steps 6–8 for the new page.

## 6.4 Resource Subscription

A runtime MAY subscribe to any resource URI it has read. Servers notify changes via `notifications/resources/updated`.

Two subscription modes:

| Mode | Notification payload | Runtime behavior |
|------|----------------------|------------------|
| **Standard** | URI only | Re-issue `resources/read` to fetch updated content |
| **Extended** | URI plus the new content | Apply the content directly without re-fetching |

Core Profile requires Standard mode; Extended mode is SHOULD (see [`18_Conformance.md`](18_Conformance.md) §18.2.6). A runtime MAY negotiate Extended mode at connect time; when unavailable it MUST fall back to Standard.

Subscriptions are released when the subscribing scope unmounts (e.g., a page subscription is released on `onDestroy`).

## 6.5 Tool Calls

A `{"type": "tool", ...}` action maps to `tools/call` on the server:

```json
{
  "type": "tool",
  "tool": "loadDashboardData",
  "params": { "userId": "{{app.user.id}}" },
  "onSuccess": { "type": "state", "action": "set", "binding": "page.data", "value": "{{event.result}}" },
  "onError":   { "type": "notification", "message": "{{event.error.message}}", "severity": "error" }
}
```

Tool responses populate the `event.*` binding scope for `onSuccess` / `onError` callbacks (see [`04_Actions.md`](04_Actions.md) §4.7–§4.8). A long-running tool call carries an implicit cancellation handle; the `cancel` action can target it by id (see [`04_Actions.md`](04_Actions.md) §4.9).

## 6.6 Host Integration

The runtime exposes a small surface to its embedding host:

### 6.6.1 `onExit` Registration

The host registers an exit callback through the runtime's single entry point — e.g., `MCPUIRuntime.buildUI(onExit: cb)` in the Dart implementation. `buildUI(onExit:)` is the one canonical registration path; there is no separate `registerOnExit()` or post-build registration API.

When `onExit` is registered:

- `{"type": "navigation", "action": "exitApp"}` invokes the callback (see [`04_Actions.md`](04_Actions.md) §4.2).
- The runtime automatically appends a **host close button** to `headerBar.actions` on the **root route only**, positioned at the trailing (rightmost) edge after any app-defined actions. Tapping it invokes `exitApp`. The button uses the `close` icon by default; see [`02_Widgets.md`](02_Widgets.md) §2.8.1 for the `headerBar.exitButton` customization hook.
- On inner routes, the AppBar `leading` is the automatic back button and the host close button is not rendered.

When `onExit` is not registered, `exitApp` is a no-op and the host close button is not rendered.

### 6.6.2 Tool Executor Hooks

The runtime MAY expose optional pre/post hooks around `tools/call` for logging, authentication injection, or parameter rewriting. Hook failure MUST NOT corrupt the DSL-visible tool result.

## 6.7 Page Transitions

Transition animations between routes are declared on the `navigation` action (`pageTransition` field) or at page level. See [`16_Animations.md`](16_Animations.md).

## 6.8 Lifecycle Hook Placement

The DSL uses a dual placement rule for lifecycle hooks:

| Placement | Applies to | Shape |
|-----------|------------|-------|
| **Definition-level** | `ApplicationDefinition`, `PageDefinition` | Hooks are **top-level properties** of the definition (e.g., `"onInit": [...]`, `"onDestroy": [...]`) |
| **Instance-level** | Any widget (including template instances) | Hooks are wrapped in a `"lifecycle": {...}` object property on the widget |

Definition-level placement reflects that the definition itself is the lifecycle-aware entity. Instance-level placement keeps lifecycle concerns explicitly separated from the widget's own properties and prevents collision with them.

### 6.8.1 Definition-Level Example

```json
{
  "type": "page",
  "title": "Dashboard",
  "onInit":    [ { "type": "tool", "tool": "loadDashboardData" } ],
  "onReady":   [ { "type": "resource", "action": "subscribe", "uri": "ui://metrics/live", "binding": "page.metrics" } ],
  "onPause":   [ { "type": "state", "action": "set", "binding": "page.isPaused", "value": true } ],
  "onResume":  [ { "type": "state", "action": "set", "binding": "page.isPaused", "value": false } ],
  "onDestroy": [ { "type": "resource", "action": "unsubscribe", "uri": "ui://metrics/live" } ],
  "content":   { "...": "..." }
}
```

### 6.8.2 Instance-Level Example

```json
{
  "type": "box",
  "lifecycle": {
    "onMount":   { "type": "state", "action": "set", "binding": "page.ready", "value": true },
    "onUnmount": { "type": "state", "action": "set", "binding": "page.ready", "value": false }
  },
  "child": { "...": "..." }
}
```

### 6.8.3 Hook Firing Order

For a page mount: `onInit` → `onMount` → `onReady`.
For a page unmount: `onPause` → `onUnmount` → `onDestroy`.
For navigation A → B: A fires `onPause` → `onUnmount` → `onDestroy`, then B fires `onInit` → `onMount` → `onReady`.
On back navigation: B fires `onUnmount` → `onDestroy`, then A fires `onMount` → `onResume`.

Hooks within the same stage execute in definition order. A failing hook logs its error; subsequent hooks MUST still run. `onInit` hooks complete before `onReady` begins. `onDestroy` completes before the runtime releases page-scoped resources (subscriptions, channels, local state).

See [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.5 for the canonical hook list.

## 6.9 Error Handling

| Condition | Runtime behavior |
|-----------|------------------|
| Invalid JSON for a definition | Reject the whole definition; MAY render a runtime error widget in its place |
| Unknown widget type | Render an error widget at that position; log; continue rendering siblings |
| Binding resolution failure | Render empty (or declared fallback); log |
| Tool or resource error | Invoke the action's `onError` callback when present; otherwise log |
| Subscription failure | Log; attempt reconnection per the runtime's reconnect policy |

A runtime MUST NOT crash the host application on any of the above conditions. It MAY expose aggregate error state via `runtime.*` bindings or a diagnostic panel.

## 6.10 Performance Contract

Targets (SHOULD, see [`18_Conformance.md`](18_Conformance.md) §18.2.11):

- Widget tree render < 100 ms for 1000 widgets
- Binding resolution < 10 ms for 1000 expressions
- State update propagation < 16 ms (single 60 fps frame)

A runtime MAY publish measurements via a diagnostic channel; the contract applies to steady-state rendering after initialization.

## 6.11 Multi-Origin Resolution *(since v1.4, Composition Profile)*

§6.1 describes the runtime's relationship with **one** server. A document that composes several origins (see [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.9) needs one more contract: how a `DefinitionSource` naming a `from` origin is resolved, and what the resolved subtree is attached to.

### 6.11.1 Connections are a host capability, not a DSL feature

The DSL does **not** define how a connection is opened, authenticated, or torn down. It defines only how a definition is addressed once a connection exists. Establishing outbound connections is the host's job, exposed to the document as ordinary tools.

The canonical host surface is the platform's outbound MCP client tool set (`mcp.connect` / `call_tool` / `read_resource` / `list_tools` / `list_resources` / `disconnect`) — an application calls `mcp.connect` in `lifecycle.onInit`, keeps the returned id in state, and refers to it as `{ "connection": "{{conn.temp}}" }`. Hosts that expose a different naming for the same capability substitute their own; the `Origin` shape is what this specification fixes.

Consequently a composing document needs no new declaration to say "I open connections" — it already declares its use of the host's tool surface through the host's existing capability-gate mechanism.

### 6.11.2 Resolution

For a `DefinitionSource` of the qualified form `{ "$ref": <uri>, "from": <origin> }`:

1. Resolve `from` to a live connection. Any binding inside it resolves first, in the scope containing the source.
2. If no such connection exists, or the origin key is unrecognised, resolution **fails** — see §6.11.4. The runtime MUST NOT fall back to the current origin.
3. Read `<uri>` from that connection via `resources/read`.
4. Parse the result as a definition. A malformed document fails resolution (§6.9 — rejected whole).
5. Attach the parsed definition to a subtree whose ambient origin is that connection.

For the binding form, steps 1–3 are skipped: the definition is already in state, and its ambient origin is the origin of the scope that holds it.

### 6.11.3 One runtime scope per origin

The resolved subtree runs in its **own scope**: its own state tree, its own subscription registry, its own permission and storage identity. It is not a sub-tree of the embedder's state.

`notifications/resources/updated` arriving on a connection dispatch to the scope(s) bound to that connection, never to the embedder's scope. Two `view`s bound to the same connection share that connection but hold separate scopes.

### 6.11.4 Failure and lifecycle

- **Resolution failure is local.** A route whose source fails to resolve reports a navigation error; a `view` whose source fails renders its `fallback` and fires `onError` while its siblings and the embedding page render normally.
- **Disconnect.** When a connection drops, scopes bound to it enter the failed state and render `fallback`. Their subscriptions are dropped.
- **Reconnect.** On reconnect the runtime MUST re-resolve the source and remount the scope. Local state inside the embedded scope does not survive; a definition that must survive reconnection persists through its own origin. (Remount rather than resume is chosen because the origin may have changed what it serves.)
- **Depth and cycles.** Runtimes MUST enforce a maximum nesting depth for embedded definitions and MUST detect cycles in the origin/URI pair chain. On either, the offending source fails resolution and renders `fallback` — it MUST NOT recurse.
