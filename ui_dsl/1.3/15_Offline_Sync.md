# 15. Offline and Sync

**Status:** Normative.
This section defines how actions behave when the client is disconnected from the server, how queued work is reconciled on reconnect, and how the UI observes connectivity and sync state.

Offline and sync are optional capabilities. Unless an implementation claims the Client Profile, it is not required to implement the queue or connectivity APIs. See [`18_Conformance.md`](18_Conformance.md).

---

## 15.1 Connectivity State

The runtime exposes connectivity via the `runtime.*` and `sync.*` binding prefixes.

| Binding | Value | Description |
|---------|-------|-------------|
| `{{runtime.networkStatus}}` | `"online"` \| `"offline"` \| `"slow"` | Current connectivity |
| `{{runtime.networkType}}` | `"none"` \| `"wifi"` \| `"cellular"` \| `"ethernet"` \| `"vpn"` | Underlying network interface |
| `{{sync.status}}` | `"online"` \| `"offline"` \| `"syncing"` \| `"error"` | Aggregate sync state |
| `{{sync.pendingCount}}` | number | Queued actions awaiting dispatch |
| `{{sync.lastSyncAt}}` | ISO timestamp \| null | When the last successful sync completed |

Connectivity transitions re-evaluate all bindings that reference `runtime.networkStatus`, `runtime.networkType`, or `sync.*`.

---

## 15.2 Offline Action Queue

Actions that require network — `tool`, `resource` with remote URIs, `client.httpRequest`, and any action explicitly tagged — MAY be queued when offline and dispatched when connectivity returns.

```json
{
  "type": "tool",
  "tool": "saveDocument",
  "params": { "content": "{{editor.content}}" },
  "offline": {
    "queue": true,
    "priority": "high",
    "maxRetries": 3
  },
  "onSuccess": { "type": "notification", "message": "Saved" }
}
```

### 15.2.1 Queue Entry Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `queue` | boolean | `false` | Enable offline queuing for this action |
| `priority` | `"high"` \| `"normal"` \| `"low"` | `"normal"` | Dispatch order hint |
| `maxRetries` | number | 3 | Attempts before marking as failed |
| `ttl` | number (seconds) | unbounded | Discard after this age |
| `conflictPolicy` | string | application default | Override per-action conflict policy (§15.5) |

### 15.2.2 Queue Dispatch Order

1. Priority class (`high` > `normal` > `low`).
2. Insertion timestamp within a priority class (FIFO).

### 15.2.3 Application-Level Configuration

An application MAY declare queue behavior globally:

```json
{
  "type": "application",
  "sync": {
    "queue": {
      "maxSize": 100,
      "overflow": "dropOldest",
      "strategy": "sequential"
    },
    "onConflict": "lastWriteWins"
  }
}
```

---

## 15.3 Queue Overflow Strategies

When an incoming action would exceed the configured `maxSize`:

| Strategy | Behavior |
|----------|----------|
| `rejectNewest` | The new action is not enqueued; its `onError` fires with code `QUEUE_FULL` |
| `dropOldest` | The oldest queued action (regardless of priority) is evicted; its `onError` fires with code `QUEUE_EVICTED` |
| `error` | The queue operation raises an error immediately and the calling action fails with `QUEUE_FULL` |

Default: `dropOldest`.

---

## 15.4 Dispatch Strategies

`sync.strategy` controls how queued actions are dispatched on reconnect:

| Strategy | Behavior |
|----------|----------|
| `sequential` | Dispatch one at a time in queue order. Failures halt dispatch until resolved |
| `parallel` | Dispatch all pending actions simultaneously |
| `smart` | Group by target (same `tool` or same `resource`) and dispatch groups in parallel, within a group sequentially |

Default: `sequential`.

---

## 15.5 Conflict Resolution

A conflict arises when a queued action's target state has changed on the server since the action was enqueued (detected via `lastModified`, `etag`, or application-defined version fields).

| Strategy | Behavior |
|----------|----------|
| `clientWins` | The client-side change overrides the server's newer state |
| `serverWins` | The server state prevails; the client's pending change is discarded |
| `lastWriteWins` | The side with the newer timestamp prevails |
| `merge` | A structured field-level merge is attempted; unmergeable fields fall back to `lastWriteWins` |
| `manual` | The action is deferred; the application is expected to present a resolution UI |

### 15.5.1 Declaring Strategy

Declared at the application level in `application.sync.onConflict`, or per-action via `offline.conflictPolicy`. The per-action value wins.

### 15.5.2 Manual Resolution

When `manual` is in effect, the runtime emits a `conflict` event on the originating widget and exposes conflict metadata at `{{event.conflict}}` (server value, client value, field path). A typical pattern:

```json
{
  "type": "tool",
  "tool": "saveDocument",
  "offline": { "queue": true, "conflictPolicy": "manual" },
  "onConflict": {
    "type": "dialog",
    "dialog": {
      "type": "alertDialog",
      "title": "Sync Conflict",
      "content": "This document was modified elsewhere.",
      "actions": [
        { "label": "Keep Mine", "action": { "type": "tool", "tool": "resolveConflict", "params": { "strategy": "clientWins" } } },
        { "label": "Use Server", "action": { "type": "tool", "tool": "resolveConflict", "params": { "strategy": "serverWins" } } }
      ]
    }
  }
}
```

---

## 15.6 Optimistic Updates

An action MAY apply a local state change immediately and roll back on server failure:

```json
{
  "type": "tool",
  "tool": "likePost",
  "params": { "postId": "{{post.id}}" },
  "optimistic": {
    "apply": {
      "type": "state",
      "action": "increment",
      "binding": "post.likes"
    },
    "rollbackOn": "error"
  }
}
```

### 15.6.1 Optimistic Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `apply` | action | State mutation to perform immediately |
| `rollbackOn` | `"error"` \| `"conflict"` \| `"both"` | When to revert the optimistic change |
| `rollback` | action (optional) | Explicit inverse action; if absent, runtime auto-inverts when possible |

### 15.6.2 Rollback Semantics

When rollback triggers, the runtime:

1. Reverts the optimistic mutation (explicit inverse if provided; otherwise automatic inverse for known operations: `increment`/`decrement`, `append`/`remove`, `push`/`pop`).
2. Fires the action's `onError` handler with the original error context.

---

## 15.7 Offline-Aware Widgets

Widgets MAY declare connectivity-sensitive behavior:

```json
{
  "type": "button",
  "label": "Upload",
  "offlineBehavior": "disable",
  "offlineTooltip": "Upload requires internet connection"
}
```

| `offlineBehavior` value | Effect while `runtime.networkStatus === "offline"` |
|-------------------------|----------------------------------------------------|
| `hide` | Widget is removed from the tree |
| `disable` | Widget is rendered non-interactive (grayed) |
| `cached` | Widget reads from cached state only |
| `fallback` | Widget renders its `offlineFallback` subtree |

---

## 15.8 Offline Widget Wrapper

`offlineWrapper` (alias: `OfflineWidgetWrapper`) selects between subtrees based on connectivity:

```json
{
  "type": "offlineWrapper",
  "mode": "both",
  "online": { "type": "text", "text": "Connected" },
  "offline": { "type": "text", "text": "You are offline" }
}
```

| `mode` | Behavior |
|--------|----------|
| `onlineOnly` | Renders `online` subtree or nothing |
| `offlineOnly` | Renders `offline` subtree or nothing |
| `both` | Renders `online` when connected, `offline` otherwise |

---

## 15.9 Client Storage

Local persistence is exposed via `client.storage.*` actions (Client Profile):

```json
{
  "type": "button",
  "label": "Save Draft",
  "onTap": {
    "type": "client.storage.set",
    "key": "draft-{{document.id}}",
    "value": "{{editor.content}}",
    "onSuccess": { "type": "notification", "message": "Draft saved locally" }
  }
}
```

| Action | Purpose |
|--------|---------|
| `client.storage.get` | Read a stored value |
| `client.storage.set` | Write a value; overwrites existing |
| `client.storage.remove` | Delete a stored key |

Storage is namespaced per application. Quota enforcement raises `QUOTA_EXCEEDED` per [`08_Client_Extensions.md`](08_Client_Extensions.md) error registry.

---

## 15.10 Connectivity Monitoring

Runtimes implementing the Client Profile MUST expose `ConnectivityManager` semantics:

- Detect transitions between `none`, `wifi`, `cellular`, `ethernet`, `vpn`.
- Emit a `connectivityChange` event on the application event bus.
- Drive `{{runtime.networkStatus}}`, `{{runtime.networkType}}`, and `{{sync.status}}` reactively.

```json
{
  "type": "page",
  "on": {
    "connectivityChange": {
      "type": "conditional",
      "condition": "{{event.data.status === 'offline'}}",
      "then": { "type": "notification", "message": "Offline — changes will sync later" }
    }
  }
}
```

---

## 15.11 Conformance

Implementations claiming the Client Profile MUST:

- Expose `{{runtime.networkStatus}}`, `{{runtime.networkType}}`, `{{sync.status}}`, `{{sync.pendingCount}}`, `{{sync.lastSyncAt}}`.
- Queue actions tagged `offline.queue = true` when offline.
- Honor `offline.priority`, `offline.maxRetries`, and the application-level `sync.queue.maxSize` / `sync.queue.overflow`.
- Implement all conflict strategies in §15.5.
- Fire optimistic `apply` and perform rollback per §15.6.
- Render `offlineBehavior` per §15.7 and the `offlineWrapper` widget per §15.8.

Implementations claiming the Client Profile SHOULD:

- Implement `client.storage.*` actions (§15.9).
- Implement all dispatch strategies in §15.4 (at minimum `sequential`).
- Emit the `connectivityChange` event on transitions (§15.10).

Core Profile runtimes MAY omit offline and sync support entirely. In that case they MUST render `offlineWrapper` as if always online, and ignore `offline.*` fields on actions (the action dispatches immediately or fails per normal error handling).
