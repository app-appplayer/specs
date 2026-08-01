# 04. Actions

An action is a JSON object describing an operation performed in response to a user gesture, a lifecycle event, or a watcher. Actions are the only way for a DSL page to cause side effects.

Normative conformance requirements: [`18_Conformance.md`](18_Conformance.md) §18.2.2 (required actions), §18.2.3 (navigation sub-actions), §18.2.4 (state sub-actions), §18.2.5 (binding integration), §18.3 (Client Profile).

The full list of action type names is in [`17_Naming.md`](17_Naming.md) §17.2.2.

## 4.1 Action Shape

Every action has a required `type`. Most types take a sub-operation in an `action` field and type-specific payload fields:

```json
{ "type": "state", "action": "set", "binding": "count", "value": 1 }
{ "type": "navigation", "action": "push", "route": "/profile" }
{ "type": "tool", "tool": "increment", "params": {} }
```

An action MAY nest callbacks (`onSuccess`, `onError`, `onMessage`, etc.). Callbacks receive the triggering event data through the `event.*` binding scope (see [`03_Data_Binding.md`](03_Data_Binding.md) §3.5.2).

Actions are dispatched from three kinds of carriers: lifecycle hooks (see [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.5), widget-local activation slots (e.g. `button.onTap`, `richText.spans[].onTap`), and the universal `click` common property (see [`02_Widgets.md`](02_Widgets.md) §2.2) which wraps **any** widget in a gesture surface. The action payload shape is identical in all three positions — `type` plus type-specific fields, optionally nesting `onSuccess` / `onError` callbacks.

## 4.2 State Actions

Mutates state. Required fields: `type: "state"`, `action`, `binding`; `value` is required for most sub-actions.

| Sub-action | Purpose | Value |
|------------|---------|-------|
| `set` | Assign the binding to `value` | Required |
| `increment` | Increase numeric value | Optional (default 1) |
| `decrement` | Decrease numeric value | Optional (default 1) |
| `toggle` | Flip a boolean | — |
| `append` | Push to end of array | Required |
| `push` | Alias for `append` | Required |
| `pop` | Remove last array element | — |
| `remove` | Remove first element equal to value | Required |
| `removeAt` | Remove element at `index` | Requires `index` |

```json
{ "type": "state", "action": "set", "binding": "user.name", "value": "John" }
{ "type": "state", "action": "set", "binding": "app.theme", "value": "dark" }
{ "type": "state", "action": "increment", "binding": "count", "value": 5 }
{ "type": "state", "action": "removeAt", "binding": "items", "index": 2 }
```

## 4.3 Navigation Actions

Required fields: `type: "navigation"`, `action`. Route-based sub-actions require `route`; `setIndex` requires `index`.

| Sub-action | Since | Purpose |
|------------|-------|---------|
| `push` | v1.0 | Push a new route onto the stack |
| `replace` | v1.0 | Replace the current route |
| `pop` | v1.0 | Pop the current route |
| `popToRoot` | v1.0 | Pop all routes to the initial route |
| `pushAndClear` | v1.0 | Clear stack and push a new route |
| `setIndex` | v1.0 | Set active index for tab-based navigation |
| `openApp` | v1.3 | Transition from dashboard rendering mode to full application |
| `exitApp` | v1.3 | Signal the host to exit the application |

```json
{
  "type": "navigation",
  "action": "push",
  "route": "/profile",
  "params": {
    "userId": "{{user.id}}",
    "from": "dashboard"
  }
}
```

### 4.3.1 `openApp` *(since v1.3)*

Transitions from a dashboard rendering context to full application rendering. Typically bound to `dashboard.onTap`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"navigation"` |
| `action` | string | Yes | `"openApp"` |
| `route` | string | No | Initial route to open. Defaults to the app's `initialRoute`. |

```json
{ "type": "navigation", "action": "openApp", "route": "/home" }
```

### 4.3.2 `exitApp` *(since v1.3)*

Signals the host environment to close the application. The runtime invokes the host-registered `onExit` callback.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"navigation"` |
| `action` | string | Yes | `"exitApp"` |

```json
{ "type": "navigation", "action": "exitApp" }
```

**Runtime behavior:**

- When the host registers `onExit`, the runtime automatically appends a **host close button** to the `headerBar.actions` slot on the application's **root route**. The button appears at the trailing (rightmost) edge, after any app-defined actions. Tapping it invokes `exitApp`.
- The close button is hidden on inner routes (where `leading` is the automatic back button) and hidden entirely when no `onExit` callback is registered.
- `exitApp` MAY also be triggered explicitly from any DSL handler (independent of the host button).
- The host button uses the `close` icon by default. Apps MAY customize it via `headerBar.exitButton` (see [`02_Widgets.md`](02_Widgets.md) §2.8.1).
- Inner-page back navigation is unaffected by `exitApp`.

## 4.4 Tool Actions

Calls an MCP server tool.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"tool"` |
| `tool` | string | Yes | Tool name to invoke |
| `params` | object | No | Tool parameters (default `{}`) |
| `bindResult` | string | No | State path to store the raw result. When set, auto-merge is skipped. |
| `onSuccess` | Action | No | Action on successful response |
| `onError` | Action | No | Action on failure |
| `timeout` | number | No | Timeout in milliseconds (default 30000) |
| `onTimeout` | Action | No | Action triggered on timeout |
| `cancellable` | boolean | No | Allow cancellation via `cancel` |
| `onCancel` | Action | No | Action triggered on cancel |
| `loading` | object | No | `{ binding, indicator }` — auto-sets a boolean loading flag |

```json
{
  "type": "tool",
  "tool": "validateAndSave",
  "params": { "data": "{{formData}}" },
  "onSuccess": {
    "type": "batch",
    "actions": [
      { "type": "notification", "message": "Saved", "severity": "success" },
      { "type": "navigation", "action": "push", "route": "/success" }
    ]
  },
  "onError": {
    "type": "notification",
    "message": "Error: {{event.message}}",
    "severity": "error"
  }
}
```

### 4.4.1 Auto-Merge Behavior

On success, the response's parsed JSON text is auto-merged into page state — each top-level key becomes a state variable. See [`03_Data_Binding.md`](03_Data_Binding.md) §3.10. Set `bindResult` to suppress auto-merge.

### 4.4.2 Response and Error Context

Inside `onSuccess`: the parsed response is exposed through the `event.*` scope. Inside `onError`: `event.code`, `event.message`, `event.details`.

`event.*` resolution depends on the response shape:

| Response shape | `event.*` resolution |
|----------------|----------------------|
| Object (Map) | Each top-level key resolves as `event.<key>` (e.g. `event.name`, `event.value`). `event` itself resolves to the full Map. |
| List, scalar (string, number, boolean), or `null` | The full response is exposed as `event.value`. `event.message`, `event.code`, and other nested keys resolve to `null`. |

For `onError`, the runtime always exposes `event.code`, `event.message`, and `event.details` from the error object regardless of response shape (these fields are part of the error structure, not the success payload).

## 4.5 Resource Actions

Operates on MCP resources.

| Sub-action | Purpose |
|------------|---------|
| `subscribe` | Start a subscription; bind updates to `binding` |
| `unsubscribe` | End a subscription |
| `read` | One-time read; store result at `binding` |
| `list` | List resources matching a URI pattern |

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"resource"` |
| `action` | string | Yes | Sub-action name |
| `uri` | string | Yes | Resource URI |
| `binding` | string | For `subscribe`, `read`, `list` | State path to receive the data |
| `autoUnsubscribe` | boolean | No | Opt-in: release the subscription when the declaring page unmounts. Default `false` (subscription lives until explicit `unsubscribe` or the MCP connection closes). |
| `onSubscriptionError` | Action | No | Action on subscription failure |

```json
{
  "type": "resource",
  "action": "subscribe",
  "uri": "ui://sensors/temperature",
  "binding": "temperature"
}
```

Lifecycle: a subscription persists until the author fires an explicit `unsubscribe`, the MCP connection closes, or — when `autoUnsubscribe: true` was set on the subscribing page — the page unmounts. A subscription is therefore connection-scoped by default; `autoUnsubscribe` is the only knob that ties it to page scope.

## 4.6 Dialog Actions

Opens a dialog widget.

```json
{
  "type": "dialog",
  "dialog": {
    "type": "alertDialog",
    "title": "Delete Item",
    "text": "Are you sure?",
    "dismissible": false,
    "actions": [
      { "label": "Cancel", "onTap": "close" },
      {
        "label": "Delete",
        "primary": true,
        "onTap": {
          "type": "tool",
          "tool": "deleteItem",
          "params": { "id": "{{item.id}}" }
        }
      }
    ]
  }
}
```

The special handler value `"close"` dismisses the current dialog.

## 4.7 Batch, Parallel, Sequence

### 4.7.1 batch

Executes a group of actions treated as one logical operation.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `actions` | Action[] | required | Actions to execute |
| `sequential` | boolean | `true` | Run sequentially when `true`; in parallel when `false` |
| `stopOnError` | boolean | `false` | Stop at first error (only meaningful with `sequential: true`) |

```json
{
  "type": "batch",
  "sequential": true,
  "stopOnError": true,
  "actions": [
    { "type": "state", "action": "set", "binding": "loading", "value": true },
    { "type": "tool", "tool": "saveData" },
    { "type": "state", "action": "set", "binding": "loading", "value": false }
  ]
}
```

### 4.7.2 parallel

Explicitly concurrent execution. All child actions start simultaneously.

| Property | Type | Description |
|----------|------|-------------|
| `actions` | Action[] | Actions to run concurrently |
| `onAllComplete` | Action | Runs after all children complete |
| `onAnyError` | Action | Runs if any child fails |

```json
{
  "type": "parallel",
  "actions": [
    { "type": "tool", "tool": "saveLocal" },
    { "type": "tool", "tool": "syncServer" }
  ],
  "onAllComplete": { "type": "notification", "message": "Done" },
  "onAnyError": { "type": "notification", "message": "Some failed", "severity": "warning" }
}
```

### 4.7.3 sequence

Explicitly ordered execution with error control.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `actions` | Action[] | required | Actions to run in order |
| `stopOnError` | boolean | `true` | Stop at first error |
| `onComplete` | Action | — | Runs after all complete |

### 4.7.4 Choosing between them

| Type | Execution | Error behavior | Use case |
|------|-----------|----------------|----------|
| `batch` | Sequential by default; configurable | Continues by default | General multi-step operations |
| `parallel` | Concurrent | Fires `onAnyError`, continues others | Independent operations |
| `sequence` | Sequential | Stops on error (configurable) | Dependent operations |

## 4.8 Conditional Action

Executes one of two branches based on a binding expression.

```json
{
  "type": "conditional",
  "condition": "{{isValid}}",
  "then": { "type": "tool", "tool": "submit" },
  "else": { "type": "notification", "message": "Please fix errors" }
}
```

## 4.9 Notification Action

Shows an in-app toast or snackbar.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `type` | string | Yes | — | `"notification"` |
| `message` | string | Yes | — | Message text (supports bindings) |
| `severity` | string | No | `"info"` | `info`, `success`, `warning`, `error` |
| `duration` | number | No | 3000 | Display duration in ms |
| `position` | string | No | `"bottom"` | `top`, `bottom`, `center` |
| `action` | object | No | — | Optional action button with `label` and `onTap` |

```json
{
  "type": "notification",
  "message": "Item saved",
  "severity": "success",
  "duration": 3000,
  "action": {
    "label": "Undo",
    "onTap": { "type": "tool", "tool": "undoSave" }
  }
}
```

System-level notifications (outside the app UI) use `client.notification` — see §4.12.

## 4.10 Animation Action

Triggers an imperative animation on a target widget.

```json
{
  "type": "animation",
  "target": "cardId",
  "animation": "fadeIn",
  "duration": 300
}
```

## 4.11 Cancel Action

Cancels a running action by target id.

```json
{ "type": "cancel", "target": "uploadFile" }
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `type` | string | Yes | `"cancel"` |
| `target` | string | Yes | Id of the running action to cancel |

See §4.15 for cancellation semantics.

## 4.12 Client Actions *(since v1.1, Client Profile)*

Client actions operate on the client's environment. Each requires an explicit permission grant — see [`08_Client_Extensions.md`](08_Client_Extensions.md). Required Client Profile actions are listed in [`18_Conformance.md`](18_Conformance.md) §18.3.1.

| Action | Permission |
|--------|------------|
| `client.selectFile` | `file.read` |
| `client.readFile` | `file.read` |
| `client.writeFile` | `file.write` |
| `client.saveFile` | `file.write` |
| `client.listFiles` | `file.read` |
| `client.httpRequest` | `network.http` |
| `client.getSystemInfo` | `system.info` |
| `client.clipboard` | `system.clipboard` |
| `client.exec` | `system.exec` |
| `client.notification` | `system.notification` |
| `client.storage.get` / `set` / `remove` | `system.storage` |

Every client action shares a unified result envelope:

```json
{ "success": true,  "data": { }, "timestamp": "2026-04-18T10:30:00Z" }
{ "success": false, "error": { "code": "...", "message": "...", "details": { } }, "timestamp": "..." }
```

### 4.12.1 client.selectFile

```json
{
  "type": "client.selectFile",
  "params": {
    "title": "Select a file",
    "filters": [
      { "name": "JSON Files", "extensions": ["json"] },
      { "name": "All Files", "extensions": ["*"] }
    ],
    "defaultPath": "{{client.workingDirectory}}",
    "multiple": false
  },
  "onSuccess": {
    "type": "tool",
    "tool": "processFile",
    "params": { "path": "{{event.path}}" }
  }
}
```

### 4.12.2 client.readFile

```json
{
  "type": "client.readFile",
  "params": { "path": "./config.json", "encoding": "utf-8" },
  "onSuccess": {
    "type": "state",
    "action": "set",
    "binding": "configText",
    "value": "{{event.content}}"
  }
}
```

### 4.12.3 client.writeFile / client.saveFile

```json
{
  "type": "client.writeFile",
  "params": {
    "path": "./config.json",
    "content": "{{configData}}",
    "encoding": "utf-8"
  },
  "confirmMessage": "Save configuration?",
  "onSuccess": { "type": "notification", "message": "Saved", "severity": "success" }
}
```

`client.saveFile` opens a save-as dialog and returns the user-selected path.

### 4.12.4 client.listFiles

```json
{
  "type": "client.listFiles",
  "params": {
    "path": "{{selectedDirectory}}",
    "pattern": "*.{json,yaml}",
    "recursive": false,
    "includeHidden": false
  },
  "onSuccess": {
    "type": "state",
    "action": "set",
    "binding": "fileList",
    "value": "{{event.files}}"
  }
}
```

### 4.12.5 client.httpRequest

```json
{
  "type": "client.httpRequest",
  "params": {
    "url": "https://api.example.com/version",
    "method": "GET",
    "headers": { "Accept": "application/json" }
  },
  "onSuccess": {
    "type": "state",
    "action": "set",
    "binding": "version",
    "value": "{{event.data.version}}"
  }
}
```

### 4.12.6 client.getSystemInfo

```json
{
  "type": "client.getSystemInfo",
  "params": { "properties": ["platform", "arch", "memory", "cpus"] },
  "onSuccess": {
    "type": "state",
    "action": "set",
    "binding": "sysInfo",
    "value": "{{event}}"
  }
}
```

### 4.12.7 client.clipboard

```json
{
  "type": "client.clipboard",
  "params": { "action": "write", "format": "text", "content": "{{code}}" },
  "onSuccess": { "type": "notification", "message": "Copied", "duration": 1500 }
}
```

### 4.12.8 client.exec

```json
{
  "type": "client.exec",
  "params": { "command": "ls", "args": ["-la"], "cwd": "{{selectedDirectory}}" },
  "requireConfirmation": true,
  "onSuccess": {
    "type": "tool",
    "tool": "processFileList",
    "params": { "output": "{{event.stdout}}" }
  }
}
```

### 4.12.9 client.notification

A system-level notification displayed outside the app UI. Distinct from §4.9 `notification`.

```json
{
  "type": "client.notification",
  "params": {
    "title": "Download Complete",
    "body": "File saved to {{outputPath}}",
    "icon": "check_circle"
  }
}
```

### 4.12.10 client.storage

```json
{ "type": "client.storage.set", "params": { "key": "prefs", "value": "{{prefs}}" } }
{ "type": "client.storage.get", "params": { "key": "prefs" },
  "onSuccess": { "type": "state", "action": "set", "binding": "prefs", "value": "{{event.value}}" } }
{ "type": "client.storage.remove", "params": { "key": "prefs" } }
```

## 4.13 Channel Actions *(since v1.1, Client Profile)*

Channels are bidirectional real-time streams between client and server. Canonical shape separates the subsystem (`type: "channel"`) from the bare operation (`action: "start"`), matching the JSON-RPC pattern used by `state` and `navigation` actions.

| Sub-action | Purpose |
|------------|---------|
| `start` | Open a channel |
| `stop` | Close a channel |
| `restart` | Stop and reopen |
| `toggle` | Start if stopped, stop if started |
| `send` | Send a message to the server side of the channel |

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"channel"` |
| `action` | string | Yes | One of the sub-actions above |
| `channel` | string | Yes | Channel identifier |
| `data` | any | For `send` | Payload to send |
| `onMessage` | Action | No (start) | Fires for each message received |
| `onConnect` | Action | No (start) | Fires when the channel connects |
| `onDisconnect` | Action | No (start) | Fires when the channel disconnects |
| `onError` | Action | No | Fires on channel error |

```json
{
  "type": "channel",
  "action": "start",
  "channel": "device.{{deviceId}}.status",
  "onMessage": {
    "type": "state",
    "action": "set",
    "binding": "local.status",
    "value": "{{event.message.status}}"
  },
  "onDisconnect": {
    "type": "notification",
    "message": "Device disconnected",
    "severity": "warning"
  }
}
```

```json
{ "type": "channel", "action": "stop", "channel": "device.{{deviceId}}.status" }
```

Legacy forms — accepted but MUST NOT be emitted by generators (see [`17_Naming.md`](17_Naming.md) §17.3.4):

- Dotted sub-action: `{"type": "channel", "action": "channel.start", ...}`
- v1.1 flat shape: `{"type": "channel.start", ...}`

## 4.14 Permission Actions *(since v1.1, Client Profile)*

Canonical shape: `{ "type": "permission", "action": "revoke", "permissions": [...] }`.

```json
{
  "type": "permission",
  "action": "revoke",
  "permissions": ["file.read", "file.write"],
  "onComplete": {
    "type": "notification",
    "message": "File permissions revoked",
    "severity": "info"
  }
}
```

Legacy v1.1 flat shape `{ "type": "permission.revoke", ... }` is accepted but MUST NOT be emitted (see [`17_Naming.md`](17_Naming.md) §17.3.4).

## 4.15 Action Cancellation

Long-running actions — `tool`, `client.httpRequest`, `client.exec`, `parallel`, `sequence`, and `channel.*` subscriptions — carry an implicit cancellation handle. The `cancel` action targets them by id.

Implicit cancel triggers:

- The user navigates away from the hosting page.
- The parent action group is cancelled.
- An explicit `cancel` action targets the id.

```json
{
  "type": "tool",
  "tool": "uploadFile",
  "cancellable": true,
  "onCancel": { "type": "notification", "message": "Upload cancelled" }
}
```

Cancelling a `parallel` or `sequence` cancels all in-flight child actions.

## 4.16 Action Timeout

```json
{
  "type": "tool",
  "tool": "longRunningTask",
  "timeout": 30000,
  "onTimeout": {
    "type": "notification",
    "message": "Operation timed out",
    "severity": "warning"
  }
}
```

Default timeout is 30000 ms. Timed-out actions produce error code `"TIMEOUT"` in their error envelope.

## 4.17 Action Result Envelope

All actions produce a unified result.

**Success:**

```json
{ "success": true, "data": { }, "error": null }
```

**Error:**

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "NETWORK_ERROR",
    "message": "Failed to connect to server",
    "details": { "url": "...", "status": 0 }
  }
}
```

## 4.18 Event Data in Callbacks

Inside a callback, `event.*` resolves to the result `data` (for `onSuccess`, channel `onMessage`) or the `error` object (for `onError`). See [`03_Data_Binding.md`](03_Data_Binding.md) §3.5.2.

| Binding | Available in |
|---------|--------------|
| `event.*` | `onSuccess`, `onMessage` — fields of `data` |
| `event.code` | `onError` |
| `event.message` | `onError`, channel `onMessage` |
| `event.details` | `onError` |

## 4.19 Race Conditions and Concurrency

- When multiple actions update the same state path, the default policy is last-write-wins.
- State updates within a single frame are batched; only the final value triggers re-render.
- For strict ordering, use `sequence`.
- The runtime enforces a default concurrency cap of 10 simultaneous tool calls. Excess calls are queued or rejected depending on `runtime.actions.queueOverflow` (`"queue"` or `"reject"`).

## 4.20 Loading State Pattern

```json
{
  "type": "tool",
  "tool": "fetchData",
  "loading": { "binding": "isLoading", "indicator": true }
}
```

The runtime sets `isLoading` to `true` before execution and `false` after completion (success, error, timeout, or cancel).

## 4.21 HTTP Action *(optional extension)*

A standalone `http` action is defined as an optional extension. It is not required by any profile. Runtimes that do not implement it SHOULD treat it as an unknown action type and apply standard error handling.

```json
{
  "type": "http",
  "method": "POST",
  "target": "https://api.example.com/items",
  "data": { "name": "{{form.name}}" },
  "bindResult": "createResult"
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `type` | string | required | `"http"` |
| `method` | string | `"GET"` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `target` | string | required | Target URL |
| `data` | object | null | Request body |
| `bindResult` | string | null | State path for the response |
| `onSuccess` | Action | null | On success |
| `onError` | Action | null | On failure |

Preferred paths are `tool` (server-owned operations) and `client.httpRequest` (client-owned operations under `network.http` permission).
