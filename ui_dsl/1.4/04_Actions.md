# 04. Actions

An action is a JSON object describing an operation performed in response to a user gesture, a lifecycle event, or a watcher. Actions are the only way for a DSL page to cause side effects.

Normative conformance requirements: [`18_Conformance.md`](18_Conformance.md) §18.2.2 (required actions), §18.2.3 (navigation sub-actions), §18.2.4 (state sub-actions), §18.2.5 (binding integration), §18.3 (Client Profile), §18.11 (Payment Profile).

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
| `openUrl` | v1.4 | Open a URL outside the application |

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

### 4.3.3 `openUrl` *(since v1.4)*

Opens a URL **outside** the application — the host's browser, mail client, dialer, or whatever it maps the scheme to. Every other `navigation` sub-action addresses a route *inside* the app; this is the only one that leaves it.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"navigation"` |
| `action` | string | Yes | `"openUrl"` |
| `url` | string \| binding | Yes | Absolute URL. A relative value is an error, not a route. |
| `target` | string | No | `"new"` (default) or `"same"`. A **hint** — a host with no notion of tabs MAY ignore it. |

```json
{ "type": "navigation", "action": "openUrl", "url": "https://example.com/terms", "target": "new" }
```

**Runtime behavior:**

- The runtime MUST NOT treat `openUrl` as a route change: no navigation stack entry, no page lifecycle, no `route.params`.
- A host that cannot open the URL — no handler for the scheme, blocked by policy, or user-denied — MUST report the failure through the action's `onError` and MUST NOT silently no-op. "Pressed it and nothing happened" is indistinguishable from a broken document, and the author has no way to find out.
- Opening a URL leaves the application's trust boundary. A runtime SHOULD apply the scheme and transport policy of [`07_Security.md`](07_Security.md) §7.3.4 and MAY require confirmation for schemes other than `https`.
- `openUrl` is Core: a document that links outward must not have to claim the Client Profile to do it.

**Not the same as `webView`.** `webView` (also Core) renders a URL *inside* the application, under the app's own frame and lifecycle. `openUrl` hands the URL to the host and the app keeps rendering what it was rendering. Choose `webView` to keep the user in the app, `openUrl` to send them out of it — an external link rendered in a `webView` still traps the user inside a frame the app controls, which is the wrong answer for terms-of-service links, mail addresses, and dialer URLs.

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
| `subscribe` | Start a subscription; bind updates **and the initial read** to `binding` |
| `unsubscribe` | End a subscription |
| `read` | One-time read; store result at `binding`. Holds no subscription |
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

**Where the payload lands.** A resource's payload is stored **at `binding`, as it arrives**, by
every path that delivers it: the subscription's first read, each `notifications/resources/updated`,
a one-shot `read`, and the re-read after a reconnect. A document therefore reads
`{{binding.field}}`, and a page shows the current state on the frame it opens rather than on the
first notification after it. A host that populates only some of those paths produces a page that
looks like a stale cache: correct after an update, empty or one-update-behind before one.

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

## 4.9a Sound Action *(since v1.4)*

Plays a short sound: a button click, an alarm, a confirmation chime. **Core
Profile** — a served application and a browser-rendered one need to be heard as
much as an installed one, and a sound carries none of the user's data, so there
is nothing here to confine to the Client Profile.

Distinct from `mediaPlayer` (§10.6) on purpose. That widget is a *transport with
a surface*: it occupies layout, shows position, and is seeked. A sound effect has
no surface, overlaps other sounds, and is over before anyone would look at it.
Expressing it as a widget would force an author to place an invisible element to
make a beep.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `type` | string | Yes | — | `"sound.play"` |
| `source` | `AssetRef` | Yes | — | The sound to play (§6.12 — a bundled file, a served one, an inline `data:`) |
| `volume` | number | No | 1.0 | 0.0–1.0, relative to the host's own volume |
| `id` | string | No | — | Names this playback so `sound.stop` can end it |
| `loop` | boolean | No | `false` | Repeat until stopped. Requires `id` — an unnamed loop cannot be stopped |
| `onError` | Action | No | — | Fired when the sound cannot be played, with the standard action-error shape (`event.code`, `event.message`) — no new spelling for the same event |

```json
{
  "type": "sound.play",
  "source": "bundle://assets/alarm.mp3",
  "volume": 0.8,
  "id": "alarm",
  "loop": true
}
```

`sound.stop` ends a playback started with `id`; omitting `id` stops every sound
this document started.

```json
{ "type": "sound.stop", "id": "alarm" }
```

**Rules**

- Sounds **overlap**. Playing a second sound while one is sounding MUST NOT cut
  the first off — a click during an alarm is both, not the last one.
- A runtime that cannot play sound MUST NOT swallow the action: it fires
  `onError` and declares the capability absent
  ([`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.13.2). Silence that
  looks like success is the failure this rule exists to prevent.
- A host MAY refuse playback (a browser before the first user gesture, a muted
  device, a policy). That refusal is an error the document can see, through
  `onError`, and never a rendered message.
- A document MUST NOT rely on a sound as the only carrier of information. It is
  an accompaniment: something audible must also be visible, or the app is unusable
  where audio is refused, muted, or unheard.

## 4.9b Media Actions *(since v1.4)*

Drives a `mediaPlayer` (§10.6) from the document: `media.play`, `media.pause`,
`media.toggle`, `media.seek`. **Core Profile**, for the same reason as §4.9a.

Exists because `controls: false` was otherwise a dead end. An author who wants
their own transport — a play button that matches their design, a scrubber built
from a `slider` — could hide the built-in one and then had no way to start
playback. Turning something off must not remove the ability to do it yourself.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `type` | string | Yes | — | `"media.play"` · `"media.pause"` · `"media.toggle"` · `"media.seek"` |
| `id` | string | Yes | — | The `id` of the `mediaPlayer` this acts on |
| `position` | number | `media.seek` only | — | Target position in seconds |
| `onError` | Action | No | — | Fired when the action cannot be carried out (standard `event.code` / `event.message`) |

```json
{
  "type": "mediaPlayer", "id": "lecture", "controls": false,
  "source": "bundle://assets/lecture.mp3", "mediaType": "audio",
  "onTimeUpdate": {
    "type": "state", "action": "set", "binding": "at", "value": "{{event.currentTime}}"
  }
}
```
```json
{ "type": "button", "label": "Play", "click": { "type": "media.toggle", "id": "lecture" } }
```

**Rules**

- The target is addressed by the widget's `id`. An action naming an `id` that is
  not mounted MUST report through `onError` — silently doing nothing would be
  indistinguishable from a player that is present but refusing.
- Reading playback state uses the events the widget already declares
  (`onPlay`, `onPause`, `onTimeUpdate`, `onEnded`); no separate binding surface
  is introduced. **A runtime MUST fire `onTimeUpdate` as the position advances**
  — an author building their own scrubber has no other way to know where
  playback is, and a transport built on a value that never changes is the
  facsimile [`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.13.1 forbids.
- These actions never *create* playback: they act on a mounted `mediaPlayer`.
  Playing a sound with no surface is `sound.play` (§4.9a).

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

## 4.22 Submit Action *(form-scoped)*

Triggers validation and submission of the enclosing `form` (§2.6.23). It is the
only action whose meaning depends on where it sits: it is resolved by the widget
that carries it, against the nearest `form` ancestor, and never reaches the
action dispatcher.

```json
{ "type": "button", "label": "Submit", "onTap": { "type": "submit" } }
```

The widget validates every field in that form. **If validation fails, nothing
else happens** — the form's `onSubmit` does not fire and no error action is
raised; the fields show their own messages according to `showErrorsOn`. On
success the form saves its fields and then fires its `onSubmit`.

A `submit` with no `form` ancestor is a no-op. Runtimes SHOULD report it —
a submit button that silently does nothing is indistinguishable from a broken
document.

## 4.23 Event Action

Emits a named in-document event. Publishes to state rather than to the server,
so any binding can observe it without a subscription.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"event"` |
| `action` | string | no | `"emit"` (default; the only sub-action defined) |
| `event` | string | yes | Event name. Empty or missing is an error. |
| `data` | any | no | Payload. |

```json
{ "type": "event", "event": "cart.changed", "data": { "count": "{{cart.items.length}}" } }
```

The runtime writes the payload to `_events.<event>.data` and an ISO-8601 stamp
to `_events.<event>.timestamp`. Listeners read those paths like any other state:

```json
{ "type": "text", "content": "Last change: {{_events.cart.changed.timestamp}}" }
```

`_events` is runtime-owned. Documents SHOULD treat it as read-only outside this
action — writing it directly bypasses the timestamp and makes an emit
indistinguishable from a stale value.

## 4.24 Payment Action *(since v1.4.2, Payment Profile)*

Asks the host to take payment for a declared item. The document names **what** and, where it can, **whose**; the merchant, the provider, the credentials, the price and the session all belong to the payment surface the host opens. There is deliberately no field for a provider or a credential — a document that could name the provider would need that provider's keys to be reachable from inside a rendered page.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"payment"` |
| `action` | string | no | `"checkout"` — the default and the only sub-action defined |
| `seller` | string \| binding | no | Opaque identifier of the receiving party, as issued by the payment surface. Not a merchant id and not an account number. Omitted where the receiving party is not the document's to name — see §4.24.2. |
| `itemId` | string \| binding | yes | Identifier of the item **as the payment surface knows it**. A preselection, not a definition — the description and, for most items, the price live on the other side. |
| `amount` | number \| binding | no | **Only for an item the payment surface prices as customer-entered** (a tip, a donation, a counter total). Major units, greater than zero. See §4.24.3. |
| `onSuccess` | action | no | Fires on a `success` return. Read §4.24.4 before binding anything of value here. |
| `onError` | action | no | Fires on cancel, on an unreadable return, and on every host-side failure. |

```json
{
  "type": "payment",
  "seller": "{{app.seller}}",
  "itemId": "wash-premium",
  "onSuccess": { "type": "navigation", "action": "push", "route": "/receipt" },
  "onError": { "type": "state", "action": "set", "binding": "msg", "value": "{{event.message}}" }
}
```

### 4.24.1 Dispatch

1. The runtime resolves the declared fields from bindings.
2. It hands them to the host's payment port. The host resolves the receiving party where the document did not name one (§4.24.2), assembles the payment address and mints the return address ([`07_Security.md`](07_Security.md) §7.3.5); **the document supplies none of those.**
3. The host presents the payment surface. A runtime with no payment port, and a host that cannot present the surface, MUST report through `onError` and MUST NOT no-op. A payment button that does nothing is indistinguishable from a broken document.
4. The outcome returns to the same action, which produces the §4.17 envelope.

**Where the surface appears is the host's answer, not the document's.** It may be outside the application — the host's browser or a custom tab, leaving the trust boundary exactly as `openUrl` does (§4.3.3) — or it may be a payment surface the host renders itself. A host that can run the payment provider's own front end is not required to leave the application to reach it, and a document MUST NOT be written to depend on either answer.

This is **not** the same as rendering a provider's payment page in a `webView`: card entry ends on the provider's own domain in every case, and a runtime MUST NOT frame a provider page inside the application to simulate an in-app surface.

**Where the seller offers more than one provider, choosing between them is the host's screen.** The document is not told the candidates and does not pick one — a document that picked the provider would be naming where the money goes, which is what §7.3.5 takes away from it. A choice the person abandons is a `cancel`.

| Outcome | Envelope |
|---------|----------|
| the payment surface's flow was completed | `{"success": true, "data": {"status": "success"}}` |
| the person dismissed it | `error.code = "PAYMENT_CANCELLED"` |
| the return carried no readable outcome, or no return arrived | `error.code = "PAYMENT_UNKNOWN"` |
| no payment port, or the host refused to open the surface | `error.code = "PAYMENT_UNAVAILABLE"` |

`{{event.status}}` reads the success branch; `{{event.code}}` and `{{event.message}}` read the error branch (§4.18). A runtime MUST NOT route `PAYMENT_CANCELLED` or `PAYMENT_UNKNOWN` to `onSuccess`. Cancelling is a normal outcome, not a failure to hide, and an unknown outcome is the one case where guessing is most expensive.

### 4.24.2 Who is being paid

A document names `seller` when it knows it — a shop's own application, a page selling its own items.

It omits `seller` when the receiving party is **not the document's to name**: a document served by a physical device, where who gets the money follows from which device this is, and the host establishes that by verifying the device's identity rather than by reading a field. A document that could name the receiving party in that situation could also name a different one.

- Where `seller` is absent the host MUST resolve the receiving party from the **verified identity of the origin that served the document**, and MUST fail with `PAYMENT_UNAVAILABLE` if it has no way to do so. It MUST NOT fall back to a default party.
- Where `seller` is present the host MUST use it, and MUST NOT substitute another party.
- Either way the host, not the document, decides whether that party can take payment at all. A receiving party with no usable provider is a refusal, not an empty surface.

### 4.24.3 Amounts

Most items are priced by the payment surface, and a document that sent a price would be quoting one side of a sale to the other. `amount` exists for the items where the person paying decides — a tip, a donation, a counter total.

- A runtime MUST send `amount` only for an item the payment surface prices as customer-entered. For any other item the surface derives the price and **ignores a supplied amount**; a runtime MUST NOT treat that as an error of its own, and MUST NOT show the document's number as the price.
- An amount outside the bounds the item declares is **refused, not clamped**. Silently charging a corrected figure is worse than refusing: the person agreed to what they typed.
- `amount` is in major units and MUST be greater than zero. The currency is never the document's — it comes from the item.

### 4.24.4 The return is a hint, not a settlement

`status: "success"` means **the person came back from the payment surface**, and nothing more. It arrives on a link that anything on the device can send — another application, a page in a browser, a scanned code — so a document that opens a door on it opens the door for whoever sends the link.

- A runtime MUST NOT describe the return as verified payment in any surface it renders itself.
- A document SHOULD treat `onSuccess` as a display transition: show a receipt view, start a poll, re-render a state the server owns. Capture is frequently asynchronous, so the authoritative record may still read as pending at the moment the person returns.
- Where the callback releases something of value — starts a machine, unlocks content, ships goods — the **server side of that operation** MUST confirm the payment against the payment surface before acting. `{"type": "tool"}` in `onSuccess` is the normal shape, and the tool's implementation is where that check belongs. A tool that acts on being called has no way to tell a paid caller from any other.

This is not a property of one payment surface. It holds for any hosted flow that reports its outcome through a link, because the link is a message from the device, not from the party that took the money.

### 4.24.5 Runtimes that do not take payment

`payment` is its own Profile ([`18_Conformance.md`](18_Conformance.md) §18.11) and Core does not include it. A runtime that does not claim the Payment Profile MUST fail the action through `onError` with `PAYMENT_UNAVAILABLE`, exactly as §18.2.2 requires for an action type it does not handle: logged, graceful, and visible to the document. Silence would teach an author that their document is wrong when it is the runtime that has nothing to open.

## 4.25 Location Action *(since v1.4.3, Location Profile)*

Asks the host **where this device is, once, now**. The document says how precise an answer it needs; whether it gets one, and whether the person is asked first, belong to the host.

There is deliberately no continuous form. A document that could follow someone is a different thing from one that can ask where they are, and the second is what this action is for — a report that says where it was filed from, a form that fills in an address, a screen that shows what is nearby.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"location"` |
| `action` | string | no | `"current"` — the default and the only sub-action defined |
| `precision` | string | no | `"coarse"` (default) or `"fine"`. What the document **needs**, not what it would like. See §4.25.2. |
| `onSuccess` | action | no | Fires with the position in the §4.17 envelope. |
| `onError` | action | no | Fires on refusal and on every host-side failure. |

```json
{
  "type": "location",
  "precision": "fine",
  "onSuccess": {
    "type": "state", "action": "set", "binding": "where",
    "value": "{{event.latitude}},{{event.longitude}}"
  },
  "onError": {
    "type": "state", "action": "set", "binding": "msg", "value": "{{event.message}}"
  }
}
```

### 4.25.1 Dispatch

1. The runtime resolves the declared fields from bindings.
2. It hands them to the host's location port. **The host owns the prompt** — whether the person is asked, in what words, and how the platform records their answer. A document MUST NOT draw a prompt of its own and MUST NOT be written to assume one appeared.
3. The host answers with a position, or refuses.
4. The outcome returns to the same action, which produces the §4.17 envelope.

| Outcome | Envelope |
|---------|----------|
| a position was obtained | `{"success": true, "data": {"latitude": …, "longitude": …, "accuracyMeters": …, "precision": "coarse"\|"fine", "at": "<ISO 8601>"}}` |
| the person refused, now or previously | `error.code = "LOCATION_DENIED"` |
| no location port, the platform has it turned off, or no position could be obtained | `error.code = "LOCATION_UNAVAILABLE"` |

`{{event.latitude}}` and the rest read the success branch; `{{event.code}}` and `{{event.message}}` read the error branch (§4.18).

**A refusal is a normal outcome, not a failure to hide.** A runtime MUST route `LOCATION_DENIED` to `onError` and MUST NOT retry on its own. A document SHOULD stay usable without a position — a form that cannot be submitted because someone declined is a form that asked for consent it had already decided to require.

### 4.25.2 Precision is a ceiling, not a preference

`precision` says what the document needs. The host MAY answer with something coarser, and **MUST NOT answer with something finer than was asked**.

- A document that asks `coarse` and is handed a street address has been given something it did not justify, and it now holds it. The ceiling is what keeps a document from collecting precision by asking softly.
- `accuracyMeters` and the returned `precision` say what actually arrived. A document MUST NOT assume the value it asked for is the value it got.
- The host decides how `coarse` is coarsened. Rounding at the client is not coarsening if the fine value was read to produce it — where the platform can request a reduced-accuracy fix, a host claiming this Profile SHOULD ask for one rather than degrading a precise reading.

### 4.25.3 Asking is an act, not a state

A position is read **in response to something a person did**, while the document is on screen.

- A runtime MUST NOT dispatch `location` from a lifecycle hook, a timer, or a binding evaluation. It runs from an action a person triggered.
- A runtime MUST NOT read a position while the document is not being rendered. There is no background form of this action.
- Repeating the action is how a document gets a newer answer. A runtime MUST NOT cache a position across dispatches to avoid asking again — a stale position presented as current is a wrong answer that looks like a fast one.

Where a document needs the person to understand that filing something will attach where they are, **the act that attaches it should be the one that says so** — the label on the button they press. That is the document's to write, and it is more honest than a prompt appearing after the fact.

### 4.25.4 A position is not an identity

A position says where a device was, once. It does not say who is holding it, and a runtime MUST NOT let it stand in for that.

- A host MUST NOT derive a principal from a position, and MUST NOT use one to satisfy an identity requirement (see [`19-scan-entry-identity.md`](../../../platform/19-scan-entry-identity.md) §5).
- A document that records where something was filed from is recording a circumstance. A document that decides *who* filed it from the same value is guessing, and the guess is worst exactly where it matters — two people at one address.

### 4.25.5 Runtimes that do not answer

`location` is its own Profile ([`18_Conformance.md`](18_Conformance.md) §18.12) and Core does not include it. A runtime that does not claim the Location Profile MUST fail the action through `onError` with `LOCATION_UNAVAILABLE`, exactly as §18.2.2 requires for an action type it does not handle: logged, graceful, and visible to the document.

A host that *can* answer and chooses not to — a policy that this build never reports position — reports the same code. The document is told it cannot have one, never told that it asked wrongly.
