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
