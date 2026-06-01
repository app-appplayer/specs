# 17. Naming Conventions and Legacy Alias Registry

**Status:** Normative. Every canonical name in this specification is authoritative; every alias here is non-preferred but accepted by conformant runtimes.

This document is the **single source** for canonical names, naming conventions, and legacy aliases. No other section in this specification defines aliases; they all reference this file.

---

## 17.1 Naming Conventions

### 17.1.1 JSON Keys — camelCase

All JSON keys use camelCase.

```json
{ "type": "box", "backgroundColor": "#F5F5F5", "borderRadius": 8 }
```

Rationale: dominant convention in the JSON ecosystem (JavaScript/TypeScript, Java, Dart, Swift, Kotlin, C#, Go).

### 17.1.2 Widget Type Names — Short lowercase, multi-word camelCase

- Single-word: `box`, `text`, `button`, `image`, `icon`, `card`, `divider`, `badge`, `chip`, `avatar`, `tooltip`, `spacer`
- Multi-word: `textInput`, `listItem`, `progressBar`, `iconButton`, `singleChildScrollView`

### 17.1.3 Enum Values — lowercase single-word or camelCase multi-word

- Single-word: `horizontal`, `vertical`, `center`, `start`, `end`, `elevated`, `outlined`, `filled`
- Multi-word: `spaceBetween`, `spaceAround`, `spaceEvenly`, `scaleDown`, `topStart`, `bottomEnd`

### 17.1.4 Event Names and Callback Properties

Two distinct forms:

| Form | Where | Style | Examples |
|------|-------|-------|----------|
| **Event-name string** | value in event firing / subscription | camelCase | `"change"`, `"tap"`, `"longPress"`, `"submit"` |
| **Callback property** | JSON key on a widget | `on` + PascalCase | `"onChange"`, `"onTap"`, `"onLongPress"`, `"onSubmit"` |

Rationale: matches React's convention (the dominant component-based UI grammar). HTML DOM uses lowercase (`onchange`); React's `onChange` is the scaled-up form for multi-word events.

### 17.1.5 Binding Expressions — `{{...}}`

- Syntax: `{{expression}}`
- Prefixes use dot notation: `{{state.count}}`, `{{route.params.id}}`, `{{theme.colorScheme.primary}}`
- Nested access: `{{user.profile.name}}`
- See [`03_Data_Binding.md`](03_Data_Binding.md) for the full prefix list and resolution order.

### 17.1.6 Action Types — Single word or dotted namespace

- Core: `state`, `navigation`, `tool`, `resource`, `dialog`, `batch`, `conditional`, `notification`, `parallel`, `sequence`, `cancel`, `animation`
- Namespaced (dotted): `client.selectFile`, `client.readFile`, `channel.start`, `channel.stop`, `permission.revoke`

The dot separator indicates a family of related operations on one subsystem.

---

## 17.2 Canonical Vocabulary

### 17.2.1 Widget Types

Full widget catalog in [`02_Widgets.md`](02_Widgets.md). Names below are normative; aliases in §17.3 are accepted.

#### Core widgets (Core Profile)

| Canonical | Purpose |
|-----------|---------|
| `box` | Rectangular region with padding, margin, border, decoration (replaces CSS `<div>`) |
| `linear` | Linear layout with `direction: "horizontal" \| "vertical"` |
| `stack` | Overlapping children with alignment |
| `center`, `align`, `padding` | Single-child layout helpers |
| `expanded`, `flexible`, `spacer` | Flex children |
| `text` | Text display with optional `style` |
| `richText` | Styled text with inline spans |
| `image`, `icon` | Visual elements |
| `button`, `iconButton` | Interactive buttons |
| `textInput` | Single-line text entry |
| `toggle` | Binary on/off control |
| `select` | Dropdown selection |
| `checkbox`, `radio`, `radioGroup`, `checkboxGroup` | Boolean/choice controls |
| `slider`, `rangeSlider` | Continuous value selection |
| `listItem` | Item within a list |
| `list`, `grid` | Scrollable collections |
| `card` | Elevated container |
| `divider`, `verticalDivider` | Visual separators |

#### Navigation widgets (Core Profile)

`headerBar`, `bottomNavigation`, `tabBar`, `drawer`, `navigationRail`, `floatingActionButton`, `popupMenuButton`

#### Scroll and layout (Core Profile)

`scrollView`, `singleChildScrollView`, `pageView`, `sizedBox`, `aspectRatio`, `fractionallySized`, `intrinsicHeight`, `intrinsicWidth`, `wrap`, `positioned`, `safeArea`, `margin`, `visibility`, `conditional`

#### Form controls (Core Profile)

`form`, `numberField`, `dateField`, `timeField`, `datePicker`, `timePicker`, `dateRangePicker`, `colorPicker`, `segmentedControl`, `stepper`, `numberStepper`, `rating`

#### Dialogs (Core Profile)

`alertDialog`, `simpleDialog`, `customDialog`, `snackBar`, `bottomSheet`

#### Interaction (Core Profile)

`gestureDetector`, `inkWell`, `draggable`, `dragTarget`

#### Advanced widgets (Advanced Profile — see [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md))

`chart`, `table`, `dataTable`, `map`, `mediaPlayer`, `calendar`, `timeline`, `gauge`, `heatmap`, `tree`, `graph`, `networkGraph`, `codeEditor`, `terminal`, `fileExplorer`, `markdown`, `webView`, `signature`, `canvas` *(since v1.3)*

#### Animation widgets (Core Profile / v1.3)

`animatedContainer`, `opacity` *(since v1.3)*, `transform` *(since v1.3)*, `lottieAnimation`

#### Template widgets (Template Profile — see [`09_Templates.md`](09_Templates.md))

`use`, `template` (special; template invocation and definition)

#### Utility

`placeholder`, `banner`, `accessibleWrapper`, `lazy`, `decoration`, `fittedBox`, `clipOval`, `clipRRect`

### 17.2.2 Action Types

Full catalog in [`04_Actions.md`](04_Actions.md).

#### Core Profile

`state`, `navigation`, `tool`, `resource`, `dialog`, `batch`, `conditional`, `notification`, `parallel`, `sequence`, `cancel`, `animation`

#### Client Profile

`client.selectFile`, `client.readFile`, `client.writeFile`, `client.saveFile`, `client.listFiles`, `client.httpRequest`, `client.getSystemInfo`, `client.clipboard`, `client.exec`, `client.notification`, `client.storage.get`, `client.storage.set`, `client.storage.remove`, `permission.revoke`, `channel.start`, `channel.stop`, `channel.restart`, `channel.toggle`, `channel.send`

### 17.2.3 Navigation Sub-Actions

Used inside `{"type": "navigation", "action": "..."}`.

| Action | Purpose |
|--------|---------|
| `push` | Push a new route onto the stack |
| `replace` | Replace the current route |
| `pop` | Pop the current route |
| `popToRoot` | Pop all routes to the initial route |
| `pushAndClear` | Clear the stack and push a new route |
| `setIndex` | Set the active index for tab-based navigation |
| `openApp` *(since v1.3)* | Transition from dashboard mode to full application rendering |
| `exitApp` *(since v1.3)* | Signal the host to exit; invokes host-registered `onExit` callback |

### 17.2.4 State Sub-Actions

Used inside `{"type": "state", "action": "..."}`.

`set`, `increment`, `decrement`, `toggle`, `append`, `remove`, `push`, `pop`, `removeAt`

### 17.2.5 Binding Prefixes

Full resolution order in [`03_Data_Binding.md`](03_Data_Binding.md).

| Prefix | Scope |
|--------|-------|
| *(bare variable)* | Local scope (list context, template scope, page state) |
| `state.` | Current scope state (synonym for bare when unambiguous) |
| `page.` | Explicit page state |
| `app.` | Application-wide state |
| `local.` | Component-scoped local state (template instance, explicit) |
| `route.params.` | Current route parameters |
| `theme.` | Theme tokens (colors, typography, spacing, radius, elevation) |
| `event.` | Event data (within an action callback) |
| `item`, `index`, `isFirst`, `isLast`, `isEven`, `isOdd` | List iteration context |
| `client.` | Client capabilities (v1.1 — Client Profile) |
| `client.theme.` | Host environment theme colors |
| `client.env.` | Environment variables (allowlist-restricted) |
| `client.file.` | Client file metadata |
| `client.system.` | Client system info |
| `permissions.` | Permission grant states |
| `channels.` | Bidirectional channel data streams |
| `resources.` | Loaded MCP resource data |
| `sync.` | Sync operation status |
| `runtime.` | Runtime capability flags |
| `i18n.` | Internationalization keys (see [`12_Internationalization.md`](12_Internationalization.md)) |

### 17.2.6 Callback Property Names

`onTap`, `onDoubleTap`, `onLongPress`, `onChange`, `onSubmit`, `onSelect`, `onOpen`, `onClose`, `onFocus`, `onBlur`, `onHover`, `onScroll`, `onRefresh`, `onError`, `onSuccess`, `onInit`, `onDestroy`, `onMount`, `onUnmount`, `onReady`, `onPause`, `onResume`, `onMessage`, `onConnect`, `onDisconnect`, `onPanStart`, `onPanUpdate`, `onPanEnd`, `onCommand`, `onDrop`, `onExit`

---

## 17.3 Legacy Alias Registry

Every entry is accepted by conformant runtimes. Emitters SHOULD prefer the canonical form.

### 17.3.1 Widget Type Aliases

| Canonical | Legacy aliases |
|-----------|----------------|
| `box` | `container`, `decoratedBox` |
| `linear` + `direction` | `row`, `column` |
| `textInput` | `textField`, `textfield`, `textFormField` (Flutter/Material; form-aware semantics folded into `textInput` + `validation`) |
| `toggle` | `switch` |
| `listItem` | `listTile`, `list-tile` |
| `select` | `dropdown` |
| `progressBar` | `linearProgressIndicator`, `loadingIndicator`, `loading-indicator`, `progress-bar`, `progress` |
| `bottomNavigation` | `bottomNav`, `bottomnavigationbar` |
| `headerBar` | `appbar` |
| `list` | `listView`, `listview` (Flutter-style casing) |
| `grid` | `gridview` (Flutter-style casing) |

### 17.3.2 Property Aliases

| Widget | Canonical property | Legacy aliases |
|--------|--------------------|----------------|
| `text` | `text` (the content) | `content` (was canonical in v1.0) |
| `button` | `label` | `text` (when applied to a button; v1.1 drift) |
| `textInput`, `textField` | `placeholder` | `hint` |
| `bottomNavigation` item, `drawer` item | `label` | `title` |
| `tabBar` tab | `label` | `text` |
| Any widget | `accessibility.label` | `semanticLabel` (shorthand), `ariaLabel`, `aria-label` |
| Any widget | `accessibility.hint` | `ariaHint`, `aria-hint` |
| Any widget | `accessibility.role` | `ariaRole`, `aria-role`, `role` |
| Any widget | `accessibility.live` | `ariaLive`, `aria-live` |
| `linear` | `spacing` | `gap`, `itemSpacing` |
| `linear` | `direction` | `orientation` (in some layouts), `scrollDirection` (in scrollable lists) |
| `linear` | `distribution` | `mainAxisAlignment` |
| `linear` | `alignment` | `crossAxisAlignment` |
| `button` | `variant` | `style` |
| `image` | `src` | `backgroundImage`, `source` |
| `grid` | `columns` | `crossAxisCount` |
| `select`, `radioGroup`, `checkboxGroup` | `options` | `items` |
| `slider` | `value` | `values` (when single-value) |
| `avatar` | `label` | `text` |
| `avatar`, `card`, `box` | `color` | `backgroundColor` |
| Template invocation (`use`) | `itemTemplate` | `template` |

### 17.3.3 Callback Aliases

| Canonical | Legacy aliases |
|-----------|----------------|
| `onTap` | `click`, `onPressed` |
| `onDoubleTap` | `double-click` |
| `onLongPress` | `long-press`, `longPress` |
| `onChange` | `onChanged`, `change` |
| `onSubmit` | `submit` |
| `onSelect` | `select`, `onDestinationSelected` (navigation rail), `onSelected` (popup menu) |
| `onOpen` | `open` |
| `onClose` | `close` |
| `onBlur` | `blur` |
| `onFocus` | `focus` |
| `onCommand` | `command` |
| `onDrop` | `drop` |
| `onPanStart` | `panStart` |
| `onPanEnd` | `panEnd` |

### 17.3.4 Action Shape Aliases

| Canonical | Legacy aliases |
|-----------|----------------|
| `{"type": "channel", "action": "start", ...}` | `{"type": "channel", "action": "channel.start", ...}` (dotted sub-action) · `{"type": "channel.start", ...}` (v1.1 flat) |
| `{"type": "channel", "action": "stop", ...}` | `{"type": "channel", "action": "channel.stop", ...}` · `{"type": "channel.stop", ...}` |
| `{"type": "channel", "action": "restart", ...}` | `{"type": "channel", "action": "channel.restart", ...}` · `{"type": "channel.restart", ...}` |
| `{"type": "channel", "action": "toggle", ...}` | `{"type": "channel", "action": "channel.toggle", ...}` · `{"type": "channel.toggle", ...}` |
| `{"type": "channel", "action": "send", ...}` | `{"type": "channel", "action": "channel.send", ...}` · `{"type": "channel.send", ...}` |
| `{"type": "permission", "action": "revoke", ...}` | `{"type": "permission.revoke", ...}` (v1.1 flat) |

### 17.3.5 Type Name Aliases (page/screen)

| Canonical | Legacy alias |
|-----------|--------------|
| `"type": "page"` (page definition root) | `"type": "screen"` |

---

## 17.4 Ghost References (Removed)

The following terms appeared in prior naming documents but are not part of the canonical vocabulary. They are removed; implementations do not need to recognize them.

| Ghost term | Where it appeared | Replacement |
|-----------|-------------------|-------------|
| `conditionalWidget` | Legacy Naming_Conventions §3.1 | `conditional` |
| `dashboard` as a widget type | Legacy Naming_Conventions §3.1 | `dashboard` is an ApplicationDefinition field, not a widget. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md). |
| `args.*` binding prefix | Legacy Naming_Conventions §4 | `route.params.*` |

---

## 17.5 Policies

### 17.5.1 Alias Acceptance

A conformant runtime MUST accept canonical names. It SHOULD accept the aliases listed above. A runtime MAY log a deprecation warning when encountering an alias.

### 17.5.2 Canonical-Only Emission

Generators (e.g., `MCPUIJsonGenerator`, AI-agent-authored JSON) MUST emit the canonical form. Aliases exist only to accept existing legacy JSON, not to produce new JSON.

### 17.5.3 Collision Rule

If a JSON object contains both a canonical key and an alias key (e.g., both `label` and `title`), the **canonical key wins**. Implementations MAY emit a warning.

### 17.5.4 Adding New Aliases

New aliases are added only when a renaming proposal is approved. The approval record lives in [`CHANGELOG.md`](CHANGELOG.md). Aliases once added are never removed (only their use is discouraged).

### 17.5.5 Adding New Canonical Names

New canonical names are additive: they do not replace existing canonicals. If a concept needs a new name, the old name becomes an alias and both are accepted indefinitely.

---

## 17.6 Language Neutrality

Canonical names avoid terms specific to one UI framework:

- `container` (Flutter/Bootstrap-specific) → `box` (CSS/web)
- `linearProgressIndicator` (Flutter) → `progressBar` (HTML)
- `listTile` (Material) → `listItem` (HTML)
- `textField` (Material/Flutter) → `textInput` (HTML)

Where multiple frameworks agree, the shared term is used (`button`, `image`, `icon`, `text`, `select`, `form`).
