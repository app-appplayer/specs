# 01. Core Concepts

This document defines the three definition types that form every MCP UI DSL program, the scopes under which state lives, the lifecycle hooks that bound that state, and the version field that gates features.

Full canonical-name and legacy-alias references live in [`17_Naming.md`](17_Naming.md). Normative requirements (MUST / SHOULD / MAY) are anchored in [`18_Conformance.md`](18_Conformance.md).

## 1.1 The Definition Tree

Every program the runtime consumes is one of three definition shapes:

| Definition | `type` value | Role |
|------------|--------------|------|
| **ApplicationDefinition** | `"application"` | Top-level container — owns routes, theme, app-wide state, navigation chrome |
| **PageDefinition** | `"page"` | Full-screen view referenced by a route |
| **WidgetDefinition** | widget type string (e.g. `"box"`, `"text"`) | UI node within a page's content tree |

The program forms a tree:

```
ApplicationDefinition
 ├── theme, navigation, state (app scope), lifecycle (app hooks), services
 └── routes: { "/path": PageDefinition | ui-resource-uri }
              └── PageDefinition
                   ├── state (page scope), lifecycle (page hooks)
                   └── content: WidgetDefinition
                                 └── child | children: WidgetDefinition[...]
```

Page trees are resolved lazily: a route value may be an inline `PageDefinition` or a URI (`"ui://pages/dashboard"`) that the runtime reads via the MCP resource protocol. See [`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.1 for the resolution contract.

### 1.1.1 Canonical vs Legacy Root Types

The root `type` for a page definition is canonically `"page"`. The legacy value `"screen"` is accepted by conformant runtimes (see [`17_Naming.md`](17_Naming.md) §17.3.5). Emitters MUST emit `"page"`.

## 1.2 ApplicationDefinition

An ApplicationDefinition is the entry point. The runtime loads it first and derives all other rendering from its routes, theme, and state.

### 1.2.1 Schema

| Field | Type | Required | Since | Description |
|-------|------|:--------:|:-----:|-------------|
| `type` | string (`"application"`) | yes | v1.0 | Discriminator. |
| `routes` | `{ [path: string]: RouteValue }` | yes | v1.0 | Map of route path to a `RouteValue`: inline `PageDefinition`, a `ui://` URI string, or a `{ page, transition }` wrapper that attaches a per-route `RouteTransition` (since v1.3). Path syntax supports parameters: `"/users/:id"`. |
| `title` | string | no | v1.0 | Human-readable name shown by hosts and app launchers. |
| `version` | string | no | v1.0 | DSL version the definition targets. Defaults to `"1.0"`. See §1.7. |
| `initialRoute` | string | no | v1.0 | Route entered at launch. If omitted the runtime picks the first route in declaration order. |
| `theme` | ThemeDefinition | no | v1.0 | Application-wide theme. See [`05_Theme.md`](05_Theme.md). |
| `navigation` | NavigationConfig | no | v1.0 | Global navigation chrome (drawer, bottom bar, rail, tabs). |
| `state` | `{ "initial": object }` | no | v1.0 | Initial application-scope state. Accessible via `{{app.*}}`. See §1.6. |
| `lifecycle` | LifecycleHooks | no | v1.0 | Application-lifetime hooks (see §1.5). |
| `services` | `{ [name: string]: ServiceDefinition }` | no | v1.0 | Background services (polling, subscriptions) launched at app init. |
| `id` | string | no | v1.2 | Stable bundle identifier (reverse-DNS, e.g. `"com.example.myapp"`). See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.1. |
| `description` | string | no | v1.2 | Short human-readable description. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.1. |
| `icon` | IconReference | no | v1.2 | App icon reference. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.1. |
| `splash` | SplashConfig | no | v1.2 | Splash-screen configuration. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.4. |
| `category` | string | no | v1.2 | Classification tag (e.g. `"productivity"`, `"utility"`). |
| `publisher` | PublisherInfo | no | v1.2 | Publisher metadata. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.2. |
| `timestamps` | TimestampInfo | no | v1.2 | Creation / update timestamps. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.3. |
| `screenshots` | string[] | no | v1.2 | Reference URIs for store/listing screenshots. |
| `dashboard` | DashboardDefinition | no | v1.3 | Compact rendering entry point. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.9. |
| `templates` | `{ [name: string]: TemplateDefinition }` | no | v1.3 | Application-scope reusable templates. See [`09_Templates.md`](09_Templates.md) §9.2. |
| `templateLibraries` | TemplateLibraryRef[] | no | v1.3 | References to external template collections. See [`09_Templates.md`](09_Templates.md) §9.9. |
| `i18n` | I18nConfig | no | v1.3 | Locale bundles and default locale. See [`12_Internationalization.md`](12_Internationalization.md). |

### 1.2.2 Example

```json
{
  "type": "application",
  "version": "1.3",
  "id": "com.example.taskflow",
  "title": "TaskFlow",
  "description": "A lightweight task tracker.",
  "initialRoute": "/dashboard",
  "theme": {
    "mode": "system",
    "colors": { "primary": "#2196F3", "surface": "#F5F5F5" }
  },
  "routes": {
    "/dashboard": "ui://pages/dashboard",
    "/tasks/:id": "ui://pages/task-detail"
  },
  "state": {
    "initial": {
      "user": { "name": "Guest", "isAuthenticated": false },
      "filter": "all"
    }
  },
  "lifecycle": {
    "onInit": { "type": "tool", "tool": "loadSession" },
    "onDestroy": { "type": "tool", "tool": "persistSession" }
  },
  "navigation": {
    "type": "drawer",
    "items": [
      { "label": "Dashboard", "route": "/dashboard", "icon": "dashboard" }
    ]
  }
}
```

### 1.2.3 NavigationConfig

The `navigation` field on ApplicationDefinition declares the global navigation chrome.

| Field | Type | Required | Since | Description |
|-------|------|:--------:|:-----:|-------------|
| `type` | enum (`drawer` / `bottomBar` / `rail` / `tabs`) | yes | v1.0 | Chrome style. `drawer` = side drawer; `bottomBar` = bottom navigation; `rail` = vertical rail; `tabs` = tab strip. |
| `items` | NavItem[] | no | v1.0 | Navigation entries. Each entry: `{ label, icon?, route, badge?, children?, onTap?, style? }`. |
| `header` | Widget | no | v1.0 | Optional header widget rendered above `items` (used by `drawer` / `rail`). |
| `footer` | Widget | no | v1.0 | Optional footer widget rendered below `items` (used by `drawer` / `rail`). |
| `style` | `NavigationStyle` | no | v1.3 | Visual styling for the navigation surface (backgroundColor / backgroundImage / indicatorColor / indicatorShape / dividerColor+thickness+indent / labelStyle / iconStyle / selectedColor / unselectedColor / elevation). Per-item overrides via `NavItem.style`. |

```json
{
  "navigation": {
    "type": "drawer",
    "items": [
      { "label": "Home", "route": "/", "icon": "home" },
      { "label": "Settings", "route": "/settings", "icon": "settings" }
    ],
    "header": { "type": "text", "text": "Acme Corp" },
    "footer": { "type": "button", "label": "Sign out", "onTap": { "type": "tool", "tool": "signOut" } }
  }
}
```

## 1.3 PageDefinition

A PageDefinition describes one full-screen view. It may be embedded inline inside `routes` or served as an MCP resource.

### 1.3.1 Schema

| Field | Type | Required | Since | Description |
|-------|------|:--------:|:-----:|-------------|
| `type` | string (`"page"`) | yes | v1.0 | Discriminator. `"screen"` is a legacy alias; see [`17_Naming.md`](17_Naming.md) §17.3.5. |
| `content` | WidgetDefinition | yes | v1.0 | Root of the widget tree for the page. |
| `title` | string | no | v1.0 | Page title; hosts MAY display this in the header. |
| `state` | `{ "initial": object }` | no | v1.0 | Initial page-scope state. Accessible via `{{page.*}}` (and, when unambiguous, bare identifiers). See §1.6. |
| `lifecycle` | LifecycleHooks | no | v1.0 | Page-lifetime hooks (see §1.5). |
| `themeOverride` | Partial ThemeDefinition | no | v1.0 | Page-scoped theme overrides merged onto the application theme. See [`05_Theme.md`](05_Theme.md) §5.7. |
| `validation` | ValidationConfig | no | v1.0 | Form validation rules attached to the page. |
| `permissions` | PermissionDeclaration | no | v1.1 | Per-page permission requirements. See [`08_Client_Extensions.md`](08_Client_Extensions.md). |
| `version` | string | no | v1.0 | Overrides the application `version` for this page; rarely used. |

### 1.3.2 Example

```json
{
  "type": "page",
  "title": "Dashboard",
  "state": {
    "initial": { "filter": "active", "selectedId": null }
  },
  "lifecycle": {
    "onReady": {
      "type": "tool",
      "tool": "loadTasks",
      "params": { "filter": "{{page.filter}}" }
    }
  },
  "content": {
    "type": "linear",
    "direction": "vertical",
    "children": [
      { "type": "text", "text": "Welcome, {{app.user.name}}" },
      { "type": "button", "label": "Refresh", "onTap": { "type": "tool", "tool": "loadTasks" } }
    ]
  }
}
```

### 1.3.3 Single-Widget vs List Content

`content` is canonically a single widget. When the root needs multiple siblings, wrap them in a layout widget (`linear`, `stack`, `box`). Historical `children` at the page root is accepted for backward compatibility (v1.1 shape) and is normalized by the runtime to `content: { type: "linear", direction: "vertical", children: [...] }` equivalent semantics; emitters MUST use `content`.

## 1.4 WidgetDefinition

A WidgetDefinition is any JSON object with a `type` field that names a widget from the catalog in [`02_Widgets.md`](02_Widgets.md).

### 1.4.1 Common Shape

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Widget type name. Required. |
| `child` | WidgetDefinition | Sole child (single-child widgets). |
| `children` | WidgetDefinition[] | Children list (multi-child widgets). Single-child widgets accept a one-element `children` array for authoring convenience. |
| `visible` | boolean | Default `true`. When `false` the subtree is not rendered. |
| `tooltip` | string | Hover / long-press tooltip wrapper. See [`02_Widgets.md`](02_Widgets.md) §2.2. |
| `click` | Action | Tap action fired by a gesture-surface wrap. See [`02_Widgets.md`](02_Widgets.md) §2.2 and [`04_Actions.md`](04_Actions.md). |
| `accessibility` | AccessibilityBlock | Grouped a11y fields (`label`, `hint`, `role`, `live`, ...). See [`13_Accessibility.md`](13_Accessibility.md). |
| `style`, `padding`, `margin`, `width`, `height`, ... | widget-specific | Per-widget properties. See [`02_Widgets.md`](02_Widgets.md). |
| `id` | string | Optional stable handle for test hooks and action targeting. |

All widget-specific properties (e.g. `src` on `image`, `label` on `button`, `text` on `text`) are collected in the same JSON object alongside `type`. There is no separate property bag.

### 1.4.2 Composition Rules

- Single-child widgets (`box`, `center`, `padding`, `card`, ...) expect `child`.
- Multi-child widgets (`linear`, `stack`, `wrap`, `list`, `grid`, ...) expect `children` (an array).
- When a single-child widget receives `children` with exactly one element, the runtime treats it as equivalent to `child`.
- When both `child` and `children` are provided, `child` wins and the runtime MAY emit a warning.

### 1.4.3 Example

```json
{
  "type": "box",
  "padding": { "all": 16 },
  "decoration": { "color": "{{theme.colors.surface}}", "borderRadius": 8 },
  "child": {
    "type": "linear",
    "direction": "vertical",
    "spacing": 8,
    "children": [
      { "type": "text", "text": "{{page.title}}", "style": { "fontWeight": "bold" } },
      { "type": "button", "label": "Continue", "onTap": { "type": "navigation", "action": "push", "route": "/next" } }
    ]
  }
}
```

## 1.5 Lifecycle Hooks

Lifecycle hooks bind actions to well-defined moments in the lifetime of an Application, a Page, or a template instance.

### 1.5.1 Hook Names

| Hook | Fires |
|------|-------|
| `onInit` | Immediately after the definition is parsed and state is initialized; before any render. |
| `onReady` | After the first successful render / mount. Safe to call tools that depend on bound state. |
| `onMount` | When the definition becomes part of the live tree (page shown, template instance inserted). |
| `onPause` | When the definition loses active focus but is not destroyed (e.g. navigated away from on a stack). |
| `onResume` | When a paused definition regains active focus. |
| `onUnmount` | When the definition is removed from the live tree. |
| `onDestroy` | When the runtime releases the definition; state is disposed after this hook returns. |

Each hook value is a single Action or an Action array. Arrays execute sequentially. See [`04_Actions.md`](04_Actions.md).

### 1.5.2 Firing Order

For a given definition instance:

```
onInit → onMount → onReady → (onPause ↔ onResume)* → onUnmount → onDestroy
```

`onReady` always fires after `onMount`. Paused instances may cycle `onPause`/`onResume` any number of times before unmount. Runtimes MUST guarantee `onUnmount` runs even when removal is caused by conditional rendering. See [`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.8 for the normative contract.

### 1.5.3 Placement: Definition-Level vs Instance-Level

Lifecycle hooks appear in two distinct positions, and the placement is not interchangeable:

**Definition-level (top-level fields on the definition object).** Applies to ApplicationDefinition, PageDefinition, and TemplateDefinition. The hook names appear as sibling fields alongside `type`, `content`, `state`, etc. These run for the definition as a whole.

```json
{
  "type": "page",
  "onInit": { "type": "tool", "tool": "loadData" },
  "onReady": { "type": "state", "action": "set", "binding": "page.ready", "value": true },
  "onDestroy": { "type": "tool", "tool": "flushCache" },
  "content": { "type": "box", "child": { "type": "text", "text": "Hello" } }
}
```

Equivalently (and preferred for a large set of hooks), a definition-level `lifecycle` object may group them:

```json
{
  "type": "page",
  "lifecycle": {
    "onInit": { "type": "tool", "tool": "loadData" },
    "onReady": { "type": "state", "action": "set", "binding": "page.ready", "value": true },
    "onDestroy": { "type": "tool", "tool": "flushCache" }
  },
  "content": { "type": "box", "child": { "type": "text", "text": "Hello" } }
}
```

When both top-level hook fields and a `lifecycle` object are present on the same definition, the two sets are merged; a name appearing in both positions is an error and the runtime MUST reject the definition.

**Instance-level (on a template invocation).** A `use` widget that invokes a template may declare instance-specific hooks. Instance hooks are always wrapped in a `lifecycle:` block — never placed as top-level fields on the `use` widget:

```json
{
  "type": "use",
  "template": "expandablePanel",
  "params": { "title": "Advanced" },
  "lifecycle": {
    "onMount": { "type": "tool", "tool": "trackImpression", "params": { "component": "advanced-panel" } },
    "onUnmount": { "type": "tool", "tool": "flushMetrics" }
  }
}
```

Instance hooks run for that particular template instance only and see the instance's `local.*` scope. They do not replace the template definition's own hooks — both fire, with the definition's hook first.

### 1.5.4 Hook Errors

If a hook action fails, the runtime logs the error and continues. It MUST NOT cancel the remaining lifecycle sequence. `onError` callbacks declared on individual actions (see [`04_Actions.md`](04_Actions.md) §4.7) remain the correct recovery mechanism.

## 1.6 State Scopes

State in MCP UI DSL is partitioned into four scopes. Each scope has a distinct lifetime and a distinct binding prefix. The resolution order for unprefixed identifiers is specified in [`03_Data_Binding.md`](03_Data_Binding.md) §3.4.

| Scope | Prefix | Lifetime | Mutability | Declared in |
|-------|--------|----------|------------|-------------|
| Application | `app.*` | From application load until `onDestroy` | Read-write via `state` actions | ApplicationDefinition `state.initial` |
| Page | `page.*` | From page mount until unmount | Read-write via `state` actions | PageDefinition `state.initial` |
| Local (template instance) | `local.*` | From template instance mount until unmount | Read-write via `state` actions | TemplateDefinition `stateDefaults` *(since v1.3)* |
| Route parameters | `route.params.*` | The current route's active lifetime | Read-only | Route path segments (`/users/:id` → `route.params.id`) |

### 1.6.1 Isolation

- Sibling pages do not share `page.*` state; unmounting a page disposes its state.
- Sibling template instances do not share `local.*` state; each `use` gets its own copy of `stateDefaults`. See [`09_Templates.md`](09_Templates.md) §9.7.
- `app.*` state persists across page transitions and is the only scope suitable for cross-page communication.

### 1.6.2 Read-only Scopes

`route.params.*` is read-only. Any `state` action targeting `route.params.*` MUST be rejected by the runtime. To change route parameters, emit a `navigation` action that pushes or replaces the route with new parameter values (see [`04_Actions.md`](04_Actions.md) §4.2).

Additional read-only binding prefixes (`theme.*`, `client.*`, `permissions.*`, `channels.*`, `resources.*`, `sync.*`, `runtime.*`, `i18n.*`, `event.*`) are not state scopes — they expose runtime and context data. The full prefix table is in [`17_Naming.md`](17_Naming.md) §17.2.5.

### 1.6.3 State Actions

State is mutated only through `state` actions:

```json
{ "type": "state", "action": "set", "binding": "app.filter", "value": "active" }
```

Supported sub-actions: `set`, `increment`, `decrement`, `toggle`, `append`, `remove`, `push`, `pop`, `removeAt`. See [`04_Actions.md`](04_Actions.md) §4.2 and [`17_Naming.md`](17_Naming.md) §17.2.4.

## 1.7 Version Field

The `version` field on ApplicationDefinition declares which DSL version the definition targets.

### 1.7.1 Shape

```json
{ "type": "application", "version": "1.3", ... }
```

Accepted string forms: `"1.0"`, `"1.1"`, `"1.2"`, `"1.3"`. A three-part form (`"1.3.0"`) is also accepted; comparison uses `major.minor` only. If `version` is omitted the runtime MUST treat the definition as `"1.0"`.

### 1.7.2 Effect on Parsing

A runtime MAY use `version` to decide whether to accept version-gated fields (e.g. `dashboard` on v1.3, `templates` on v1.3, `permissions` on v1.1). Fields newer than the declared version SHOULD be accepted if the runtime supports them (forward-compatibility) but the runtime MAY warn.

### 1.7.3 Relationship to Profiles

The `version` field is informational for feature introduction. It is **not** the conformance dimension. A runtime's support surface is declared in terms of Profiles (Core, Client, Bundle, Advanced, Template), defined in [`18_Conformance.md`](18_Conformance.md). A document carrying `version: "1.3"` may be served to a runtime that claims Core Profile only; the runtime accepts the parts inside its profile and rejects or ignores the rest per its documented policy.

### 1.7.4 `since:` Markers

Features introduced after v1.0 carry a `since: vX.Y` marker on their heading or schema row throughout this specification. The marker tells implementers which DSL version first introduced the feature, independently of which profile governs it.

## 1.8 Required Runtime Behavior

Normative requirements for parsing, rendering, lifecycle execution, and state isolation are in [`18_Conformance.md`](18_Conformance.md) §18.2 (Core Profile) with cross-cutting requirements in §18.2.8 (security) and §18.2.10 (legacy alias acceptance).

Specifically:

- ApplicationDefinition parsing and MCP protocol read: [`18_Conformance.md`](18_Conformance.md) §18.2.6
- Widget rendering and tree composition: [`18_Conformance.md`](18_Conformance.md) §18.2.1
- Lifecycle execution order: [`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.8 (normative behavior) anchored by [`18_Conformance.md`](18_Conformance.md) §18.2.6
- State isolation across scopes: [`18_Conformance.md`](18_Conformance.md) §18.2.8 (security §7.4)
- Legacy alias acceptance for `screen`/`page`, `container`/`box`, `content`/`text`, etc.: [`18_Conformance.md`](18_Conformance.md) §18.2.10 referencing [`17_Naming.md`](17_Naming.md) §17.3
