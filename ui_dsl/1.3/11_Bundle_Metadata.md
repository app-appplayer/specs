# 11. Bundle Metadata

**Profile:** Bundle Profile. See [`18_Conformance.md`](18_Conformance.md) §18.4.
This section defines the metadata fields on `ApplicationDefinition` that support app discovery, listing, packaging, and multi-app rendering contexts. All fields defined here are optional — a v1.0 `ApplicationDefinition` without any of them remains valid.

## 11.1 ApplicationDefinition Metadata Fields *(since v1.2)*

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Reverse-domain app identifier (e.g., `com.example.myapp`). Stable across versions. |
| `description` | string | Short human-readable description for listings. |
| `icon` | string | Icon reference. Accepts `bundle://` URI, HTTP(S) URL, or `data:` URI. |
| `splash` | SplashConfig | Splash screen configuration. See §11.4. |
| `category` | string | Category tag. Recommended values in §11.1.1. |
| `publisher` | PublisherInfo | Publisher metadata. See §11.2. |
| `timestamps` | TimestampInfo | Creation / update timestamps. See §11.3. |
| `screenshots` | string[] | Array of screenshot references (`bundle://`, URL, or data URI). |
| `dashboard` | DashboardConfig | Compact rendering entry point. See §11.9. *(since v1.3)* |

### 11.1.1 Recommended `category` Values

Free-form string. The following values are recommended for interoperability:

`productivity`, `education`, `entertainment`, `social`, `business`, `utilities`, `health`, `finance`, `lifestyle`, `news`, `travel`, `food`, `sports`, `music`, `photo`, `video`, `communication`, `developer`, `reference`, `other`.

### 11.1.2 Example

```json
{
  "type": "application",
  "id": "com.example.myapp",
  "title": "My App",
  "version": "1.0.0",
  "description": "An example productivity application.",
  "icon": "bundle://assets/icon.png",
  "splash": {
    "image": "bundle://assets/splash.png",
    "backgroundColor": "#FFFFFF",
    "duration": 2000
  },
  "category": "productivity",
  "publisher": {
    "name": "Example Corp",
    "website": "https://example.com",
    "email": "contact@example.com"
  },
  "timestamps": {
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-04-18T00:00:00Z"
  },
  "screenshots": [
    "bundle://assets/screenshot-1.png",
    "bundle://assets/screenshot-2.png"
  ],
  "initialRoute": "/home",
  "routes": { },
  "state": { },
  "theme": { }
}
```

## 11.2 PublisherInfo

```json
{
  "publisher": {
    "name": "Example Corp",
    "logo": "https://example.com/logo.png",
    "website": "https://example.com",
    "email": "contact@example.com"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Publisher display name. |
| `logo` | string | No | Publisher logo. URL or data URI. |
| `website` | string | No | Publisher website URL. |
| `email` | string | No | Contact email. |

## 11.3 TimestampInfo

```json
{
  "timestamps": {
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-04-18T00:00:00Z"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `createdAt` | string (ISO 8601 UTC) | No | Creation timestamp. |
| `updatedAt` | string (ISO 8601 UTC) | No | Last update timestamp. |

Runtimes MUST accept ISO 8601 UTC form (trailing `Z`). Runtimes MAY accept zoned forms but are not required to.

## 11.4 SplashConfig

```json
{
  "splash": {
    "image": "bundle://assets/splash.png",
    "backgroundColor": "#FFFFFF",
    "duration": 2000
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | string | No | Splash image (`bundle://`, URL, or data URI). |
| `backgroundColor` | string | No | Background hex color (`#RRGGBB` or `#RRGGBBAA`). |
| `duration` | number | No | Display duration in milliseconds. |

## 11.5 `bundle://` URI Scheme

`bundle://` references assets within the currently loaded bundle. It is used on `icon`, `splash.image`, `screenshots`, and anywhere else an image or asset reference is accepted.

### 11.5.1 Syntax

```
bundle://<path>
```

`<path>` is a slash-separated relative path within the bundle root.

### 11.5.2 Resolution

Given bundle root `<bundle-root>`:

- `bundle://assets/icon.png` resolves to `<bundle-root>/assets/icon.png`.
- Leading `/` in `<path>` is ignored (`bundle:///assets/icon.png` → `<bundle-root>/assets/icon.png`).
- Resolution MUST stay within `<bundle-root>`. Paths containing `..` segments or otherwise escaping the root (via symlinks or absolute-path encoding) MUST be rejected. Runtimes MUST NOT follow resolution outside the loaded bundle.
- Case sensitivity follows the underlying asset store (host OS filesystem or in-memory archive). Emitters SHOULD match the canonical casing of the stored asset.

### 11.5.3 Serving Transforms

When a server serves a definition that originated from a bundle, it MAY rewrite `bundle://` references into client-consumable forms (HTTP URL, data URI). Runtimes MUST accept definitions where `bundle://` has been rewritten to HTTP(S) URLs or `data:` URIs transparently — the DSL contract is the asset reference, not the scheme.

## 11.6 `ui://app/info` Well-Known Resource

`ui://app/info` is a lightweight app-metadata resource. A server exposing it returns a summary suitable for app listings and discovery, without requiring the client to parse the full `ApplicationDefinition`.

### 11.6.1 Response Shape

```json
{
  "id": "com.example.myapp",
  "name": "My App",
  "version": "1.0.0",
  "description": "An example productivity application.",
  "icon": "data:image/png;base64,...",
  "category": "productivity",
  "publisher": {
    "name": "Example Corp",
    "website": "https://example.com"
  },
  "timestamps": {
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-04-18T00:00:00Z"
  },
  "screenshots": [
    "https://cdn.example.com/apps/myapp/shot-1.png"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | App identifier. |
| `name` | string | Yes | Display name. See §11.6.2 on `name` ↔ `title` mapping. |
| `version` | string | Yes | App version string. |
| `description` | string | No | Short description. |
| `icon` | string | No | Icon reference. Server-resolved to URL or data URI — `bundle://` SHOULD NOT appear in this response. |
| `category` | string | No | Category tag. |
| `publisher` | PublisherInfo | No | Publisher summary. |
| `timestamps` | TimestampInfo | No | Timestamps (as in §11.3). |
| `screenshots` | string[] | No | Screenshot references, server-resolved. |

### 11.6.2 `name` ↔ `title` Mapping

`ui://app/info` uses `name` for the display title. `ApplicationDefinition` uses `title`. The two names refer to the same concept and are both canonical in their respective contexts.

Runtimes MUST map `title` ↔ `name` transparently:

- When constructing a `ui://app/info` response from an `ApplicationDefinition`, `ApplicationDefinition.title` populates `name`.
- When a client derives `ApplicationDefinition` metadata from `ui://app/info`, `name` populates `title`.
- A server MAY include both fields on either side; when both are present, the canonical for the container wins (`name` in `ui://app/info`, `title` in `ApplicationDefinition`), matching the general collision rule in [`17_Naming.md`](17_Naming.md) §17.5.3.

### 11.6.3 Fallback

If a server does not expose `ui://app/info`:

1. The server returns a resource-not-found error for the URI.
2. The client falls back to loading the full `ApplicationDefinition` via `ui://app` and extracts the metadata fields listed in §11.1.
3. The client MAY cache the derived summary.

## 11.7 Serving Paths

Two serving paths use the same `ApplicationDefinition` schema:

### 11.7.1 Online Serving

An MCP server loads the source (bundle, database, code) and exposes the derived definition over the MCP protocol:

- `ui://app` — full `ApplicationDefinition`.
- `ui://page/<route>` — per-page definitions.
- `ui://app/info` — lightweight summary (§11.6).
- Asset references (`bundle://` or resolved equivalents) are readable by the client through the server's normal resource access.

The client sees only `ui://` resources; the originating data source is transparent.

### 11.7.2 Local Serving

A host application (e.g., AppPlayer) loads a bundle file directly without a network server. It constructs `ApplicationDefinition` from the bundle via a bundle adapter (§11.8) and renders in-process. `bundle://` references resolve to local asset paths within the unpacked bundle.

## 11.8 Bundle Adapters

Runtimes supporting the Bundle Profile SHOULD implement the `UiPort` contract (defined in the `mcp_bundle` contract package) to convert between bundle representations and DSL definitions:

| Adapter | Direction | Methods |
|---------|-----------|---------|
| `BundleUiReadAdapter` | Bundle → DSL | `toDefinition()` returns full `ApplicationDefinition` JSON; `toAppInfo()` returns the `ui://app/info` response shape. |
| `BundleUiWriteAdapter` | DSL → Bundle | `fromDefinition(json)` returns a `UiWriteOutput` describing the bundle artifacts to emit. |

A single-direction adapter MAY throw `UnsupportedError` for the unsupported direction. Runtimes embedding a bundle adapter MUST invoke `toAppInfo()` when answering `ui://app/info` and MUST invoke `toDefinition()` when answering `ui://app`. The `bundle://` resolution rules in §11.5.2 apply equally to both adapters.

## 11.9 Dashboard *(since v1.3)*

`dashboard` is a field on `ApplicationDefinition`, not a widget type. It defines how the app presents itself in compact / embedded contexts (app-launcher grids, monitoring panels, multi-app dashboards).

### 11.9.1 Rendering Modes

A runtime exposes two rendering entry points:

| Mode | Entry Point (informative) | Renders |
|------|---------------------------|---------|
| Full | e.g., `runtime.renderApp(definition)` | The full application (routes, navigation, pages). |
| Dashboard | e.g., `runtime.renderDashboard(definition)` | The `dashboard.content` widget tree only. |

If `dashboard` is not defined, runtimes rendering in dashboard mode MAY display a default card derived from `title` and `icon`, or skip the entry.

### 11.9.2 Schema

```json
{
  "type": "application",
  "id": "com.example.sensor-a",
  "title": "Sensor A",
  "templates": {
    "statusBadge": {
      "params": { "status": { "type": "string", "required": true } },
      "content": {
        "type": "chip",
        "label": "{{status}}",
        "variant": "filled",
        "style": {
          "backgroundColor": "{{status == 'online' ? '#4CAF50' : '#F44336'}}",
          "color": "#FFFFFF"
        }
      }
    }
  },
  "dashboard": {
    "content": {
      "type": "linear",
      "direction": "vertical",
      "spacing": 8,
      "children": [
        { "type": "text", "text": "{{app.title}}", "style": { "fontSize": 16, "fontWeight": "bold" } },
        { "type": "use", "template": "statusBadge", "params": { "status": "{{app.status}}" } },
        { "type": "text", "text": "CPU: {{app.cpu}}%  MEM: {{app.mem}}%" }
      ]
    },
    "refreshInterval": 5000,
    "onTap": { "type": "navigation", "action": "openApp", "route": "/home" }
  },
  "routes": { },
  "state": { }
}
```

### 11.9.3 `DashboardConfig` Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | Widget | Yes | Widget tree rendered in dashboard mode. Subject to the same validation as any page widget tree. |
| `refreshInterval` | number | No | Re-evaluation interval in milliseconds for bindings and resources referenced by `content`. Applies only in dashboard mode. |
| `onTap` | Action | No | Action invoked when the dashboard card is tapped. Typically a `navigation` action with `openApp`. |

### 11.9.4 Dashboard Capabilities

The `dashboard.content` tree is a live, interactive subtree:

- Full access to application state and bindings (`{{app.*}}`, `{{state.*}}`, `{{resources.*}}`, `{{channels.*}}`).
- Interactive widgets (`button`, `toggle`, `iconButton`) fire actions normally.
- Templates declared under the enclosing `ApplicationDefinition.templates` are available.
- Bindings re-evaluate on state changes and on the `refreshInterval` schedule.

### 11.9.5 `openApp` Linkage

`onTap` is conventionally set to a `navigation` action with `action: "openApp"`, transitioning the runtime from dashboard mode to full rendering for the same application.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"navigation"`. |
| `action` | string | Yes | `"openApp"`. |
| `appId` | string | No | Target app identifier. If omitted, targets the enclosing application. Used when a dashboard card opens a different app's full rendering. |
| `route` | string | No | Initial route within the target app. If omitted, the target app's `initialRoute` is used. |

See [`04_Actions.md`](04_Actions.md) §4.2 for `openApp` and `exitApp` action details.

## 11.10 Conformance

See [`18_Conformance.md`](18_Conformance.md) §18.4 for normative Bundle Profile requirements.
