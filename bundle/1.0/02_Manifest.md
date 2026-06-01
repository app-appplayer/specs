# 02. Manifest

`BundleManifest` is the only required field on the bundle root. It
identifies the bundle, carries product metadata, declares dependencies,
and (through optional sub-objects) wires the bundle into host chrome.

Reference, in the `mcp_bundle` package: `lib/src/models/manifest.dart`.

## 2.1 Identity (Required)

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | **MUST** | Reverse-DNS unique identifier (`com.example.notes`). MUST match `^[a-z0-9.\-_]+$`. The host uses this as the install key. |
| `name` | string | **MUST** | Human-readable display name. |
| `version` | string | **MUST** | Bundle's own product version. SHOULD follow semver (`MAJOR.MINOR.PATCH`). Independent of `schemaVersion`. |

`id` is the **install identity**. Two bundles with the same `id` are
considered the same product across versions; installing a higher
`version` over an existing install replaces it (subject to
[`InstallPolicy.onConflict`](10_Distribution.md#103-installpolicy)).

```json
{
  "manifest": {
    "id": "com.example.notes",
    "name": "Notes",
    "version": "0.4.0"
  }
}
```

## 2.2 Provenance and Description

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `provider` | string? | — | Author / publisher identifier. Distinct from `publisher.name` — `provider` is a free-form short tag, `publisher` is the structured profile. |
| `description` | string? | — | Short product description (one paragraph). |
| `schemaVersion` | string | `"1.0.0"` | Bundle schema this manifest conforms to. Mirrors the bundle root field. |
| `type` | enum [`BundleType`](#221-bundletype) | `application` | Bundle role hint (application / library / skill / profile / extension / unknown). |
| `entryPoint` | string? | — | Optional entry hint for hosts that need a "default" view (e.g. a route path). The host decides what the value means. |
| `license` | string? | — | SPDX license identifier (`MIT`, `Apache-2.0`, ...). |
| `homepage` | string? | — | Marketing URL. |
| `repository` | string? | — | Source repository URL. |
| `tags` | string[] | `[]` | Free-form tags for marketplace categorisation. |
| `metadata` | object | `{}` | Free-form bag for host-specific metadata. |

### 2.2.1 `BundleType`

```
application | library | skill | profile | extension | unknown
```

- `application` — full UI-bearing product (default).
- `library` — reusable assets / templates intended to be referenced by
  other bundles via `dependencies[]`.
- `skill` / `profile` — single-domain bundles that carry only that
  section.
- `extension` — augments an existing host or another bundle.
- `unknown` — round-trip slot for forward-compat values.

The host MAY use `type` for filtering / categorisation but SHOULD NOT
gate behavior on it — capability is determined by what sections the
bundle actually carries, not by the type label.

## 2.3 Capabilities

| Field | Type | Description |
|-------|------|-------------|
| `capabilities` | string[] | Coarse capability tags the bundle advertises (free-form). Distinct from [`requires.builtinAtoms`](#27-portability-contract--requires) which gates host capabilities the bundle *needs*. |

`capabilities` is informational. It is **not** the gate that
determines whether a bundle runs — that is `requires`.

## 2.4 Dependencies

```json
"dependencies": [
  { "id": "com.example.lib.charts", "version": "^1.2.0" },
  { "id": "com.example.lib.icons", "version": "*", "optional": true }
]
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | **MUST** | The depended-on bundle's `manifest.id`. |
| `version` | string | **MUST** | Semver constraint (`^1.2.0`, `>=1.0.0 <2.0.0`, `*`). |
| `optional` | bool | no | When `true`, install proceeds even if the dependency is missing. |
| `features` | string[] | no | Required features the dependency must advertise via `capabilities`. |

The bundle format does not implement dependency resolution — that is
the host's responsibility. The spec only fixes the declaration shape.

## 2.5 Platform Requirements

```json
"platform": {
  "dartSdk": ">=3.0.0",
  "flutterSdk": ">=3.10.0",
  "os": ["macos", "linux", "windows"],
  "envVars": ["OPENAI_API_KEY"]
}
```

Hosts MAY refuse activation when `platform` is incompatible. SHOULD
report the offending field (`os`, `dartSdk`, ...) when refusing.

## 2.6 App Metadata (UI Hosts)

For bundles surfaced in app launchers / marketplaces:

| Field | Type | Description |
|-------|------|-------------|
| `icon` | string? | Asset id or URL. |
| `splash` | `SplashConfig` | `{ image?, backgroundColor?, duration? }`. |
| `screenshots` | string[] | Asset ids or URLs for store listings. |
| `category` | enum [`AppCategory`](#261-appcategory) | Marketplace category. |
| `publisher` | `PublisherInfo` | `{ name (required), logo?, url?, email? }`. |
| `createdAt` / `updatedAt` / `publishedAt` | ISO-8601 string | Lifecycle timestamps. |
| `minRuntimeVersion` | string? | Minimum AppPlayer / runtime version. Distinct from `compatibility.minRuntimeVersion` which is the install-gate version. |
| `localization` | `{[locale]: LocalizedInfo}` | Per-locale `name` / `description` / `icon` overrides. |
| `ageRating` | string? | `"everyone"` / `"teen"` / `"mature"` etc. |
| `privacyPolicy` | string? | URL to privacy policy. |

### 2.6.1 `AppCategory`

```
productivity | education | entertainment | social | business | utilities |
health | finance | lifestyle | news | travel | food | sports | music |
photo | video | communication | developer | reference | other
```

Unknown values round-trip as `other`.

## 2.7 Portability Contract — `requires`

A bundle declares the host capabilities it **needs** through the
`requires` section (carried inline as `manifest.requires` and modeled
as a separate top-level section on `McpBundle.requires`).

Reference: `models/requires_section.dart`.

```json
"requires": {
  "schemaVersion": "1.0.0",
  "builtinAtoms": ["mcp", "fs", "ui", "workspace"],
  "builtinTools": ["studio.workspace.save", "studio.fs.read"]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per the common section pattern (see [`03_Sections.md`](03_Sections.md) §3.2). |
| `builtinAtoms` | string[] | Coarse capability categories the bundle's JS / DSL surface may call. Examples: `mcp`, `fs`, `ui`, `workspace`, `kb`, `agent`, `bus`, `bundle`. |
| `builtinTools` | string[] | Specific host tool names the bundle requires (e.g. `studio.workspace.save`). |

Both lists are **strict gates**. The host MUST refuse to activate a
bundle when:

- any `builtinAtoms` entry is not in the host's advertised set, or
- any `builtinTools` entry is not in the host's tool registry.

Authoring convention: declare only what the bundle actually uses.
Over-declaring reduces portability (a bundle that lists `kb` will fail
on a host that doesn't expose knowledge atoms even when the bundle
never calls them).

A bundle with `requires` absent OR with both lists empty is **fully
portable** — it runs on any host implementing this spec.

The host's atom-vs-tool distinction:

- **Atoms** map to JS host-bridge globals (`host.<atom>.<verb>` —
  see [`04_Tools.md`](04_Tools.md#43-js-host-bridge)). Declaring `fs`
  unlocks `host.fs.read`, `host.fs.write`, etc. If the JS code calls
  an atom not declared, the host MUST emit `host.X is undefined`.
- **Tools** map to MCP tool names (`host.mcp.callTool('studio.fs.read',
  …)`). Declaring `studio.fs.read` asserts the host's MCP registry has
  that tool.

The host's actual atom / tool catalog is host-specific. This spec
defines the declaration shape, not the catalog.

## 2.8 Wiring (Studio Host Surface)

The `wiring` and `chat` sub-objects on the manifest declare how
bundle-defined tools bind into host chrome. This is the
**Studio-host-defined surface** — defined here so any host that wants
to host bundle UI can implement the same slots.

```json
"wiring": {
  "domainActions": [...],
  "lifecycle":     [...],
  "settings":      [...]
},
"chat": {
  "slashCommands": [...]
}
```

The full schema for these arrays lives in [`06_Wiring.md`](06_Wiring.md).
This document only fixes the surface keys on the manifest.

Hosts that do not implement Studio chrome MAY ignore `wiring` and
`chat` — the bundle remains conformant. A bundle MAY carry both
`wiring` for chrome hosts and arbitrary `extensions` for host-specific
chrome.

## 2.9 Backward Compatibility

- Unknown fields on the manifest are preserved verbatim through
  `metadata` is **not** the round-trip channel; future spec patches
  that add manifest fields will appear as named optional fields.
  Loaders SHOULD ignore unknown top-level manifest keys without
  failing.
- `BundleType.unknown` and `AppCategory.other` are the
  forward-compat targets for unrecognized enum strings.

## 2.10 Conformance

A manifest MUST:

1. Carry `id`, `name`, `version` as non-empty strings.
2. Round-trip `schemaVersion` (default `"1.0.0"` when absent).
3. Round-trip `extensions` and `metadata` verbatim.

A manifest SHOULD:

- Use reverse-DNS for `id`.
- Use semver for `version`.
- Declare `requires` truthfully (under-declaring causes activation-time
  failures).

A manifest MAY:

- Omit every optional field.
- Carry `wiring` / `chat` even when no Studio host is the deployment
  target — those hosts simply ignore them.
