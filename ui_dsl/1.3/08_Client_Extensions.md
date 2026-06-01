# 08. Client Extensions

**Profile:** Client Profile. Core-only runtimes MAY skip this section; a runtime that advertises the Client Profile MUST satisfy [`18_Conformance.md`](18_Conformance.md) §18.3.
**Since:** v1.1.

## 8.1 Overview

The Client Profile extends the Core DSL with capabilities that execute in the client's local environment rather than on the MCP server:

- File system operations (read, write, pick, enumerate)
- HTTP requests issued from the client
- System information, clipboard, shell execution, and native notifications
- Client-side local storage
- The `client://` resource protocol
- Bidirectional real-time channels
- Client-side data bindings (`client.*`, `channels.*`, `permissions.*`)
- A user-consented permission model with runtime revocation

Every operation in this section is permission-gated. A runtime MUST reject a `client.*` action or `channel.*` lifecycle call whose permission has not been granted.

## 8.2 Client Action Catalog

Canonical action type names are registered in [`17_Naming.md`](17_Naming.md) §17.2.2 (Client Profile). The count below matches that registry.

### 8.2.1 File Actions (5)

| Action | Permission | Purpose |
|--------|------------|---------|
| `client.selectFile` | `file.read` | Open a native file picker dialog |
| `client.readFile` | `file.read` | Read the contents of a file |
| `client.writeFile` | `file.write` | Write content to a file path |
| `client.saveFile` | `file.write` | Save-as dialog with content |
| `client.listFiles` | `file.read` | Enumerate a directory |

Example — open picker, forward result to a server tool:

```json
{
  "type": "button",
  "label": "Open File",
  "action": {
    "type": "client.selectFile",
    "params": {
      "title": "Select a configuration file",
      "filters": [
        {"name": "JSON Files", "extensions": ["json"]},
        {"name": "All Files", "extensions": ["*"]}
      ],
      "defaultPath": "{{client.workingDirectory}}",
      "multiple": false
    },
    "onSuccess": {
      "type": "tool",
      "tool": "process_file",
      "params": {
        "path": "{{event.path}}",
        "content": "{{event.content}}"
      }
    }
  }
}
```

Example — read a file with error handling:

```json
{
  "type": "button",
  "label": "Load Config",
  "action": {
    "type": "client.readFile",
    "params": { "path": "./config.json", "encoding": "utf-8" },
    "onSuccess": {
      "type": "tool",
      "tool": "parse_config",
      "params": { "content": "{{event.content}}" }
    },
    "onError": {
      "type": "notification",
      "params": {
        "message": "Failed to load config: {{event.error.message}}",
        "severity": "error"
      }
    }
  }
}
```

### 8.2.2 Network Actions (1)

| Action | Permission | Purpose |
|--------|------------|---------|
| `client.httpRequest` | `network.http` | Issue an HTTP request from the client |

```json
{
  "type": "button",
  "label": "Check for Updates",
  "action": {
    "type": "client.httpRequest",
    "params": {
      "url": "https://api.example.com/version",
      "method": "GET",
      "headers": {"Accept": "application/json"}
    },
    "onSuccess": {
      "type": "tool",
      "tool": "process_version",
      "params": {"response": "{{event.data}}"}
    }
  }
}
```

### 8.2.3 System Actions (4)

| Action | Permission | Purpose |
|--------|------------|---------|
| `client.getSystemInfo` | `system.info` | Return platform, architecture, memory, CPUs, hostname, etc. |
| `client.clipboard` | `system.info` (read) / `system.notification`-class grant (write) per runtime policy | Read or write the system clipboard |
| `client.exec` | `system.exec` | Execute a shell command (highest trust; MAY be refused) |
| `client.notification` | `system.notification` | Display a system-level (OS) notification, distinct from in-app `notification` |

```json
{
  "type": "button",
  "label": "Get System Info",
  "action": {
    "type": "client.getSystemInfo",
    "params": {"properties": ["platform", "arch", "memory", "cpus"]},
    "onSuccess": {
      "type": "tool",
      "tool": "analyze_system",
      "params": {"info": "{{event.data}}"}
    }
  }
}
```

```json
{
  "type": "client.notification",
  "params": {
    "title": "Download Complete",
    "body": "File has been saved to {{outputPath}}",
    "icon": "check_circle"
  }
}
```

`client.exec` is the most sensitive action in this profile. Runtimes MAY refuse it unconditionally and still claim Client Profile conformance (see [`18_Conformance.md`](18_Conformance.md) §18.3.2).

### 8.2.4 Storage Actions (3)

| Action | Permission | Purpose |
|--------|------------|---------|
| `client.storage.get` | implicit (app-scoped) | Read a key from client-local storage |
| `client.storage.set` | implicit (app-scoped) | Write a key to client-local storage |
| `client.storage.remove` | implicit (app-scoped) | Delete a key from client-local storage |

Storage is scoped per MCP server identity; cross-server reads are prohibited.

### 8.2.5 Unified Result Envelope

Every `client.*` action returns one of the two shapes below. Callbacks see the inner `data` fields promoted to `event.*`.

```json
{ "success": true, "data": { }, "timestamp": "2026-04-18T10:30:00Z" }
```

```json
{
  "success": false,
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "Human-readable description",
    "details": {"permission": "file.read", "path": "/etc/shadow"}
  },
  "timestamp": "2026-04-18T10:30:00Z"
}
```

Common error codes: `PERMISSION_DENIED`, `CANCELED` (user-cancelled dialog), `NOT_FOUND`, `TIMEOUT`, `UNSUPPORTED`, `INVALID_PATH`.

## 8.3 `client://` Resource Protocol

The `client://` URI scheme exposes client-side resources declaratively (e.g., as `image.src` or as a `resource` action target).

### 8.3.1 URI Format

```
client://{type}/{path}[?{query}]
```

### 8.3.2 Resource Types

| Scheme | Root | Description |
|--------|------|-------------|
| `client://file/{path}` | System absolute path | Direct filesystem access (requires `file.read` / `file.write`) |
| `client://workspace/{path}` | Current working directory | Workspace-relative paths |
| `client://temp/{path}` | OS temp directory | Short-lived scratch files |
| `client://cache/{path}` | App cache directory | Cached data keyed by path |
| `client://asset/{path}` | App asset directory | Bundled assets shipped with the app |

### 8.3.3 Path Rules

- Forward slashes on all platforms; runtimes convert to native separators
- Path traversal segments (`..`) are rejected before permission evaluation
- Paths are normalized (`.` removed, `//` collapsed) before permission matching
- Symlinks are resolved to real paths and re-validated against the permission scope
- Case sensitivity follows the host filesystem (sensitive on Linux; insensitive on macOS/Windows)

### 8.3.4 Image and Fallback Integration

```json
{
  "type": "image",
  "src": "client://workspace/images/photo.jpg",
  "fallback": "client://asset/placeholder.png",
  "width": 200,
  "height": 200,
  "fit": "cover"
}
```

When a resource is unavailable, the runtime uses `fallback` if provided, otherwise renders a placeholder.

## 8.4 Permission System

### 8.4.1 Permission Categories (7)

| Category | Grants |
|----------|--------|
| `file.read` | `client.selectFile`, `client.readFile`, `client.listFiles`, reads via `client://file` / `client://workspace` |
| `file.write` | `client.writeFile`, `client.saveFile`, writes via `client://file` / `client://workspace` |
| `network.http` | `client.httpRequest`, HTTP-backed resources |
| `network.websocket` | WebSocket channels (`client.websocket`) |
| `system.info` | `client.getSystemInfo`, `client.env.*` binding reads |
| `system.exec` | `client.exec` |
| `system.notification` | `client.notification` |

### 8.4.2 Page-Level Permission Declaration

```json
{
  "type": "page",
  "permissions": {
    "file": {
      "read": {"paths": ["./config", "./data"], "extensions": ["json", "txt"]},
      "write": {"paths": ["./output"], "maxSize": "10MB"}
    },
    "network": {
      "http": {"domains": ["api.example.com", "*.mydomain.com"]}
    },
    "system": {
      "info": ["platform", "memory"],
      "exec": {"commands": ["ls", "cat", "echo"]}
    }
  }
}
```

### 8.4.3 Trust Levels (4)

| Level | Meaning |
|-------|---------|
| `untrusted` | No client actions permitted; display-only rendering |
| `basic` | Standard read capabilities (`file.read`, `network.http`, `system.info`) |
| `elevated` | Write capabilities (`file.write`, unrestricted-path file access) |
| `full` | All permissions including `system.exec` (development only) |

An application MAY declare a required `trustLevel`; escalating from a lower level requires an explicit user-consent prompt.

### 8.4.4 Permission Request Flow

1. **Declaration** — the page declares permissions it MAY use.
2. **Timing** — each declaration specifies `requestTiming`:

   | Timing | Description |
   |--------|-------------|
   | `upfront` | Requested on page load before user interaction |
   | `justInTime` | Requested when the first dependent action fires |
   | `contextual` | Requested with user-visible rationale at the moment the UI calls for it |

3. **Prompt** — the client presents a consent dialog with title, rationale, scope, and a persistence choice (`false` / `"session"` / `"permanent"` / `"duration"`).
4. **Decision** — the result is persisted per MCP server identity.
5. **Enforcement** — every `client.*` invocation re-checks the live grant; revoked or expired grants are treated as `PERMISSION_DENIED`.

### 8.4.5 Permission Status Binding

```json
{
  "type": "text",
  "text": "File access: {{permissions.file.read.status}}"
}
```

| Status | Meaning |
|--------|---------|
| `granted` | Active grant |
| `denied` | User explicitly denied |
| `pending` | Not yet requested |
| `revoked` | Previously granted, then revoked |
| `unavailable` | Feature not supported by the host |

### 8.4.6 Permission Revocation

Canonical shape:

```json
{
  "type": "permission",
  "action": "revoke",
  "permissions": ["file.read", "file.write"]
}
```

The legacy v1.1 shape `{"type": "permission.revoke", "permissions": [...]}` is accepted as an alias; see [`17_Naming.md`](17_Naming.md) §17.3.4. Revocation is immediate: in-flight operations depending on the revoked permission abort with `PERMISSION_DENIED`, and every `permissions.<name>.status` binding transitions to `revoked` in the same tick.

## 8.5 Client Data Bindings

All prefixes below appear in [`17_Naming.md`](17_Naming.md) §17.2.5 and resolve through the binding engine defined in [`03_Data_Binding.md`](03_Data_Binding.md).

| Binding | Content |
|---------|---------|
| `client.workingDirectory` | Current working directory (absolute path) |
| `client.userName` | Logged-in user name |
| `client.platform` | One of `macos`, `linux`, `windows`, `ios`, `android`, `web` |
| `client.locale` | IETF BCP 47 locale tag (e.g., `en-US`) |
| `client.theme.*` | Host environment theme tokens — 11 slots matching the theme schema in [`05_Theme.md`](05_Theme.md) §5.3: `background`, `foreground`, `primary`, `secondary`, `surface`, `onSurface`, `error`, `success`, `warning`, `info`, `muted` |
| `client.env.*` | Environment variables — allowlist-restricted and requires `system.info` permission |
| `client.file.*` | File metadata populated by file actions (e.g., `client.file.lastSelected.path`) |
| `client.system.*` | System info populated by `client.getSystemInfo` results |
| `permissions.<category>.status` | Live permission grant status per §8.4.5 |
| `channels.<name>.*` | Channel state and latest payload (see §8.6.6) |

Resolution follows the order in [`03_Data_Binding.md`](03_Data_Binding.md); `client.*` bindings read through to the runtime capability layer and are inert on non-Client-Profile runtimes (they resolve to `null`, never throwing).

## 8.6 Bidirectional Channels

Channels are long-lived bidirectional streams declared at the page or application level, controlled by lifecycle actions, and bound via `channels.*`.

### 8.6.1 Channel Declaration

```json
{
  "type": "page",
  "channels": {
    "fileWatch": {
      "type": "client.watchFile",
      "params": { "path": "./config.json", "events": ["change", "rename", "delete"] }
    },
    "systemMonitor": {
      "type": "client.systemMonitor",
      "params": { "metrics": ["cpu", "memory"], "interval": 1000 }
    }
  }
}
```

ChannelDefinition fields:

| Field | Type | Required | Since | Description |
|-------|------|:--------:|:-----:|-------------|
| `type` | string | yes | v1.0 | Channel type — one of the values listed in §8.6.2 (e.g. `client.watchFile`, `client.poll`). |
| `params` | object | no | v1.0 | Type-specific parameters per §8.6.2 (e.g. `{ path, events }` for `client.watchFile`). |
| `url` | string | no | v1.0 | Endpoint URL for transport-style channels (`client.websocket` etc.). |
| `tool` | string | no | v1.0 | MCP tool name to invoke when the channel kind dispatches via tool call (e.g. `client.poll` driving a server tool). |
| `binding` | string | no | v1.0 | Override for where the latest payload binds. Default is `channels.<name>`; setting `binding` redirects to a custom path. |
| `lifecycle` | ChannelLifecycle | no | v1.0 | Lifecycle config — see §8.6.5. |
| `onMessage` / `onError` / `onConnect` / `onDisconnect` | Action | no | v1.0 | Callbacks — see §8.6.4. |

### 8.6.2 Channel Types

| Type | Purpose | Key Parameters |
|------|---------|----------------|
| `client.watchFile` | Watch a file for changes | `path`, `events[]` |
| `client.watchDirectory` | Watch a directory tree | `path`, `pattern`, `recursive`, `events[]` |
| `client.systemMonitor` | Stream system metrics | `metrics[]`, `interval` |
| `client.poll` | Periodic invocation of an action | `action`, `interval` |
| `client.websocket` | Raw WebSocket connection | `url`, `protocols[]`, `headers` |

### 8.6.3 Channel Lifecycle Actions

Canonical shape — `type: "channel"`, `action: "<lifecycle>"`:

```json
{
  "type": "button",
  "label": "Start Monitoring",
  "action": {
    "type": "channel",
    "action": "channel.start",
    "channel": "systemMonitor"
  }
}
```

```json
{
  "type": "button",
  "label": "Stop Monitoring",
  "action": {
    "type": "channel",
    "action": "channel.stop",
    "channel": "systemMonitor"
  }
}
```

```json
{
  "type": "button",
  "label": "Toggle",
  "action": {
    "type": "channel",
    "action": "channel.toggle",
    "channel": "systemMonitor"
  }
}
```

```json
{
  "type": "channel",
  "action": "channel.send",
  "channel": "dataStream",
  "payload": {"op": "subscribe", "topic": "price-updates"}
}
```

Sub-actions: `channel.start`, `channel.stop`, `channel.restart`, `channel.toggle`, `channel.send` (5). The legacy v1.1 shape `{"type": "channel.start", ...}` is accepted as an alias; see [`17_Naming.md`](17_Naming.md) §17.3.4 for the full alias table.

### 8.6.4 Channel Callbacks

Channel lifecycle actions accept four callback properties. All four MUST be supported by a Client-Profile runtime ([`18_Conformance.md`](18_Conformance.md) §18.3.4):

| Callback | Fires when |
|----------|-----------|
| `onMessage` | A payload arrives from the channel (primary data path) |
| `onError` | The channel reports an error (transport or protocol) |
| `onConnect` | The channel transitions to `connected` |
| `onDisconnect` | The channel transitions to `disconnected` (graceful or error-driven) |

```json
{
  "type": "channel",
  "action": "channel.start",
  "channel": "liveData",
  "onConnect": {
    "type": "state", "action": "set",
    "binding": "status", "value": "live"
  },
  "onMessage": {
    "type": "state", "action": "set",
    "binding": "latestTick", "value": "{{event.payload}}"
  },
  "onDisconnect": {
    "type": "state", "action": "set",
    "binding": "status", "value": "offline"
  },
  "onError": {
    "type": "notification",
    "params": {"message": "{{event.error.message}}", "severity": "warning"}
  }
}
```

Callback bodies receive `event.*` populated from the channel message envelope (§8.6.7).

### 8.6.5 Channel Lifecycle Configuration

```json
{
  "channels": {
    "dataStream": {
      "type": "client.websocket",
      "url": "wss://stream.example.com/data",
      "lifecycle": {
        "autoStart": true,
        "autoDispose": false,
        "restartOnError": true,
        "maxRestarts": 5,
        "restartDelay": 1000,
        "restartBackoff": "exponential"
      }
    }
  }
}
```

| Property | Default | Description |
|----------|---------|-------------|
| `autoStart` | `false` | Start when the page loads |
| `autoDispose` | `false` | Opt-in: stop and release the channel when the declaring page unmounts. When `false` (default) the channel persists for the connection's lifetime and can only be ended with an explicit `channel.stop`, `channel.restart` cycle, or MCP connection close. |
| `restartOnError` | `false` | Auto-restart on error |
| `maxRestarts` | `3` | Cap on automatic restarts |
| `restartDelay` | `1000` | Initial delay between restarts (ms) |
| `restartBackoff` | `"fixed"` | One of `fixed`, `linear`, `exponential` |

Connection states a channel transitions through: `connecting`, `connected`, `disconnected`, `reconnecting`, `failed`, `stopped`. Each is observable via `channels.<name>.state`.

Default lifecycle summary: a channel is connection-scoped — it survives page navigation and is torn down only by explicit stop or connection close. Set `autoDispose: true` when the channel should be bound to a single page and cleaned up when the page leaves the tree.

### 8.6.6 Channel Bindings

| Binding | Content |
|---------|---------|
| `channels.<name>.state` | Connection state (`connecting` / `connected` / ...) |
| `channels.<name>.active` | `true` when state is `connected` or `reconnecting` |
| `channels.<name>.<field>` | Latest message payload fields (e.g., `channels.systemMonitor.cpu`) |
| `channels.<name>.lastMessage` | The full latest message envelope |
| `channels.<name>.error` | The last error object, if any |

```json
{
  "type": "text",
  "text": "CPU Usage: {{channels.systemMonitor.cpu}}%"
}
```

### 8.6.7 Message Envelope

Messages exchanged over a channel use a structured envelope:

```json
{
  "type": "channel.message",
  "id": "msg-uuid-001",
  "channel": "dataStream",
  "direction": "client-to-server",
  "payload": {"op": "subscribe", "topic": "price-updates"},
  "timestamp": "2026-04-18T10:30:00.000Z",
  "sequence": 42
}
```

| Field | Description |
|-------|-------------|
| `type` | `channel.message`, `channel.control`, or `channel.error` |
| `id` | Unique message identifier |
| `channel` | Declared channel name |
| `direction` | `client-to-server` or `server-to-client` |
| `payload` | Arbitrary application data |
| `timestamp` | ISO 8601 send time |
| `sequence` | Monotonic per-channel sequence number |

## 8.7 Permission Revocation at Runtime

Revocation is a first-class action so servers and user interfaces can offer an explicit "disconnect" affordance:

```json
{
  "type": "button",
  "label": "Revoke File Access",
  "action": {
    "type": "permission",
    "action": "revoke",
    "permissions": ["file.read", "file.write"],
    "onComplete": {
      "type": "notification",
      "params": {
        "message": "File access permissions revoked",
        "severity": "info"
      }
    }
  }
}
```

Revocation semantics:

1. Stored grants for the listed permissions are cleared for the current MCP server identity.
2. In-flight `client.*` operations bound to those permissions abort with `PERMISSION_DENIED`.
3. Active channels whose declared type requires a revoked permission (e.g., `client.watchFile` under `file.read`) transition to `stopped`; their `onDisconnect` callback fires.
4. `permissions.<name>.status` bindings update to `revoked` in the same state tick that the action resolves.
5. Re-requesting a revoked permission follows the normal request flow (§8.4.4); previous persistence choices do not survive a revoke.
