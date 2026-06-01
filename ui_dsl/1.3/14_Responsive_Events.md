# 14. Responsive Layout and Events

**Status:** Normative.
This section defines how UI definitions adapt to viewport size, orientation, and host platform, and how user-interaction events propagate through the widget tree.

Responsive properties and event propagation are part of the Core Profile. See [`18_Conformance.md`](18_Conformance.md).

---

## 14.1 Form Factor Classes

A form factor is a named viewport-width class. The runtime classifies the
current logical-pixel window width into exactly one of the six classes
below and exposes it through `FormFactorScope`. Responsive property
values (§14.2) and the `mediaQuery` widget (§14.3) both branch on this
classification.

The classes mirror Material 3's window-size guidance (3 standard +
extraLarge for ultra-wide hardware) plus an out-of-band `embedded`
class hosts can pin for kiosk / industrial chrome.

### 14.1.1 Default Classes

| Label | Width (dp) | Typical device |
|-------|------------|----------------|
| `compact` | `< 600` | Phone, narrow split pane |
| `medium` | `600 – 839` | Tablet portrait, foldable |
| `expanded` | `840 – 1199` | Tablet landscape, small desktop |
| `large` | `1200 – 1599` | Desktop |
| `extraLarge` | `≥ 1600` | Wide / external monitor, dashboard wall |
| `embedded` | n/a (host-pinned) | Kiosk / industrial / vehicle chrome — set by the host, not derived from width |

A viewport of 1024 dp resolves to `expanded` (≥ 840 and < 1200). The
boundary values are inclusive on the low end and exclusive on the high
end so each pixel maps to exactly one class.

### 14.1.2 Custom Boundaries

An application MAY override the four numeric boundaries via
`theme.breakpoints` (§5.7 of the Theme spec) or an application-level
`breakpoints` field. The class names themselves are fixed — only the dp
thresholds may move.

```json
{
  "type": "application",
  "breakpoints": {
    "compact": 0,
    "medium": 600,
    "expanded": 840,
    "large": 1280,
    "extraLarge": 1920
  }
}
```

`embedded` has no width threshold; it is selected only when the host
explicitly pins it (see §14.1.3).

### 14.1.3 Resolved Class Sources

The runtime picks the active class from this priority chain — the first
non-`auto` rung wins:

1. Per-page DSL pin (`page.responsive.formFactor`).
2. Application pin (`application.responsive.formFactor`).
3. Host-supplied `FormFactorScope` (e.g. an embedded shell pinning
   `embedded`).
4. Width derived from `MediaQuery.sizeOf(context)` and the boundaries
   above.

`{{runtime.formFactor}}` resolves to the active class label
(`"compact"` / `"medium"` / `"expanded"` / `"large"` / `"extraLarge"` /
`"embedded"`). The legacy `{{runtime.breakpoint}}` binding name is
retired in 1.3 — no alias is provided.

---

## 14.2 Responsive Property Values

Any property MAY accept a **responsive object** — a JSON object whose
keys are form-factor labels (§14.1.1) and whose values are the
property's normal type:

```json
{
  "type": "text",
  "text": "Hello",
  "style": {
    "fontSize": {
      "compact": 14,
      "medium": 16,
      "expanded": 18,
      "large": 20,
      "extraLarge": 24
    }
  }
}
```

### 14.2.1 Resolution Rule

The runtime resolves a responsive value by walking from the active
class toward the smaller classes and picking the first key that is
present:

1. Try the active class directly.
2. If absent, fall back through `extraLarge → large → expanded →
   medium → compact`.
3. If still no key matches, use a `default` key when present.
4. `embedded` is standalone — it tries `embedded`, then `compact`, then
   `default`. (The width-based fallback chain does not reach it.)

```json
{ "columns": { "default": 1, "medium": 2, "expanded": 3, "large": 4 } }
```

This means authors can define only the breakpoints that matter — e.g.
`{ default: 8, expanded: 24 }` keeps 8 dp on phone+tablet and 24 dp on
desktop+ without enumerating every class.

### 14.2.2 Where Responsive Objects Are Accepted

Responsive objects are accepted on any numeric, string, enum, or token
shorthand property. Non-scalar properties (children, actions, widget
subtrees) are resolved via the `mediaQuery` widget (§14.3), not via
responsive objects.

---

## 14.3 MediaQuery Widget

`mediaQuery` chooses between widget subtrees based on viewport characteristics.

```json
{
  "type": "mediaQuery",
  "condition": { "minWidth": 900 },
  "then": {
    "type": "linear",
    "direction": "horizontal",
    "children": [
      { "type": "box", "flex": 1, "child": { "type": "text", "text": "Sidebar" } },
      { "type": "box", "flex": 3, "child": { "type": "text", "text": "Main" } }
    ]
  },
  "else": {
    "type": "linear",
    "direction": "vertical",
    "children": [
      { "type": "text", "text": "Main" },
      { "type": "text", "text": "Sidebar" }
    ]
  }
}
```

### 14.3.1 Condition Properties

| Property | Type | Description |
|----------|------|-------------|
| `minWidth` | number (dp) | Matches when viewport width ≥ value |
| `maxWidth` | number (dp) | Matches when viewport width ≤ value |
| `minHeight` | number (dp) | Matches when viewport height ≥ value |
| `maxHeight` | number (dp) | Matches when viewport height ≤ value |
| `orientation` | `"portrait"` \| `"landscape"` | Matches current device orientation |
| `platform` | platform string | Matches the host platform (see §14.5) |
| `breakpoint` | breakpoint label | Matches when the current breakpoint equals the value |

Multiple conditions combine with AND. Pass `then` and `else` for the branching subtrees.

### 14.3.2 Lifecycle

`mediaQuery` re-evaluates its condition on every viewport change (resize, rotation). The subtree is swapped atomically; local state in the discarded subtree is disposed per state-isolation rules.

---

## 14.4 Orientation

`{{runtime.orientation}}` resolves to `"portrait"` or `"landscape"`.

### 14.4.1 Page-Level Hooks

A page MAY declare an `onOrientationChange` action, fired whenever orientation changes:

```json
{
  "type": "page",
  "onOrientationChange": {
    "type": "state",
    "action": "set",
    "binding": "layoutMode",
    "value": "{{runtime.orientation}}"
  }
}
```

### 14.4.2 Orientation Lock

A page MAY request an orientation lock via `lockOrientation`:

| Value | Meaning |
|-------|---------|
| `portrait` | Lock to portrait |
| `landscape` | Lock to landscape |
| `any` | Allow all orientations (default) |

Runtimes MAY ignore the lock on platforms that do not support programmatic orientation control (desktop, web).

### 14.4.3 Re-Evaluation

On orientation change, all responsive properties and `mediaQuery` conditions that reference orientation re-evaluate and their subtrees re-render.

---

## 14.5 Platform Detection

`{{runtime.platform}}` resolves to one of:

| Value | Host |
|-------|------|
| `macos` | macOS |
| `linux` | Linux |
| `windows` | Windows |
| `ios` | iOS |
| `android` | Android |
| `web` | Browser / web runtime |

Grouped aliases accepted in `mediaQuery.condition.platform`:

| Alias | Expands to |
|-------|------------|
| `mobile` | `ios`, `android` |
| `desktop` | `macos`, `linux`, `windows` |
| `apple` | `macos`, `ios` |

---

## 14.6 Event Propagation Phases

Events propagate through three phases, modeled on the DOM event model:

| Phase | Direction | Purpose |
|-------|-----------|---------|
| `capture` | Root → target | Ancestors observe the event before it reaches the target |
| `target` | At target | The target widget's handler fires |
| `bubble` | Target → root | Ancestors observe the event after the target handles it |

Default behavior: events fire in the `target` phase, then bubble up. Capture handlers fire only if explicitly registered.

### 14.6.1 Stop Propagation

A handler MAY halt further traversal:

```json
{
  "type": "box",
  "onTap": {
    "type": "state",
    "action": "set",
    "binding": "selected",
    "value": "{{item.id}}",
    "propagation": "stop"
  }
}
```

| `propagation` value | Effect |
|---------------------|--------|
| `continue` | Default. Event continues to bubble |
| `stop` | Halts traversal after this handler |
| `stopImmediate` | Halts traversal and prevents other handlers on the same widget from firing |

---

## 14.7 Event Delegation

A parent widget MAY register a delegated handler that fires for descendants matching a selector:

```json
{
  "type": "list",
  "items": "{{items}}",
  "delegate": true,
  "onTap": {
    "type": "state",
    "action": "set",
    "binding": "selectedItem",
    "value": "{{event.target.id}}"
  },
  "itemTemplate": { "type": "listItem", "label": "{{item.name}}" }
}
```

With `delegate: true`, the parent's `onTap` fires whenever any descendant emits a tap event. The original target is available at `{{event.target}}`.

### 14.7.1 Selector Form

Delegation MAY narrow by selector:

```json
{ "onTap": { "delegate": "button[data-action]", "action": { } } }
```

Selectors follow the subset defined by the runtime (at minimum: widget type, `id`, and data-attribute presence).

---

## 14.8 Custom Events

Widgets MAY emit custom events via the `event` action:

```json
{
  "type": "button",
  "label": "Select",
  "onTap": {
    "type": "event",
    "action": "emit",
    "name": "itemSelected",
    "data": "{{item}}",
    "bubble": true
  }
}
```

Ancestors subscribe via the `on` property:

```json
{
  "type": "box",
  "on": {
    "itemSelected": {
      "type": "state",
      "action": "set",
      "binding": "selected",
      "value": "{{event.data}}"
    }
  }
}
```

### 14.8.1 Event Filters

A subscription MAY include a `when` condition. The handler fires only when the condition resolves truthy:

```json
{
  "on": {
    "itemSelected": {
      "when": "{{event.data.category === 'electronics'}}",
      "action": {
        "type": "state",
        "action": "set",
        "binding": "selectedElectronics",
        "value": "{{event.data}}"
      }
    }
  }
}
```

### 14.8.2 Application Event Bus

Emitting with `bubble: true` and no matching ancestor handler propagates to the application event bus. Any subscriber anywhere in the tree receives the event.

---

## 14.9 Event Naming

See [`17_Naming.md`](17_Naming.md) §17.1.4 — event-name strings use camelCase (`change`, `tap`, `longPress`, `itemSelected`); callback properties use `on` + PascalCase (`onChange`, `onTap`, `onLongPress`).

Legacy kebab-case event-name strings (`long-press`, `double-click`, `value-change`) are accepted per [`17_Naming.md`](17_Naming.md) §17.3.3.

---

## 14.10 Event Object Shape

Inside any event handler, `{{event}}` resolves to:

| Field | Type | Description |
|-------|------|-------------|
| `event.type` | string | Event-name string (e.g., `"tap"`, `"change"`) |
| `event.target` | object | The widget that originated the event (id, type, data attributes) |
| `event.currentTarget` | object | The widget whose handler is currently executing |
| `event.phase` | string | `"capture"`, `"target"`, or `"bubble"` |
| `event.data` | any | Handler-specific payload (new value for `onChange`, dragged item for `onDrop`, custom data for emitted events) |
| `event.timestamp` | number | Milliseconds since the runtime started |

---

## 14.11 Conformance

Core Profile runtimes MUST:

- Resolve responsive property values per §14.2.1.
- Render the `mediaQuery` widget with all condition properties listed in §14.3.1.
- Expose `{{runtime.orientation}}`, `{{runtime.platform}}`, and `{{runtime.breakpoint}}` bindings.
- Dispatch events through capture → target → bubble phases and honor `propagation: "stop"` / `"stopImmediate"`.
- Fire delegated handlers per §14.7 when `delegate: true` is set.
- Accept both canonical and legacy event-name spellings per [`17_Naming.md`](17_Naming.md) §17.3.3.

Core Profile runtimes SHOULD:

- Honor `lockOrientation` where the host platform supports orientation control.
- Implement the application event bus (§14.8.2).
- Accept custom breakpoint labels defined in `application.breakpoints` or `theme.breakpoints`.
