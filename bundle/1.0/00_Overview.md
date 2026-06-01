# 00. Overview

## 0.1 What is an MCP Bundle?

An **MCP Bundle** is a self-contained package of declarative assets — UI
definitions, host-callable tools, knowledge sources, agent definitions,
and the wiring that binds them to host chrome — distributed as either a
`.mbd/` directory (working tree) or a `.mcpb` archive (deterministic
ZIP). One bundle is one product unit: a host that knows how to read this
spec installs it once and gains the entire surface the bundle declares
without any per-bundle code.

Three properties are definitional:

1. **Declarative.** Every section is JSON (or JSON-shaped files on disk).
   Behavior is owned by the host runtime; the bundle only describes
   *what* the product is.
2. **Section-composable.** A bundle includes only the sections it needs.
   A UI-only bundle carries `ui/`. A knowledge-only bundle carries
   `knowledge` and `facts`. A full product carries everything.
3. **Manifest-authoritative.** `manifest.json` is the single source the
   host trusts for identity, declarations, dependencies, and wiring.
   Reserved folders may carry the same content (e.g. `ui/app.json`
   instead of inline `ui` data) — the manifest sees them as equivalent.

## 0.2 Who Reads a Bundle?

| Consumer | Reads |
|----------|-------|
| **AppPlayer / Studio host** | `manifest.json`, `ui/`, `tools[]`, `wiring`, `agents`, `requires` — to install and run the bundle. |
| **mcp_ui_runtime** | `ui/` files (mcp_ui_dsl JSON) — to render screens. |
| **mcp_skill** | `skills` section — to dispatch procedures. |
| **mcp_knowledge / mcp_fact_graph** | `knowledge`, `facts`, `factGraphSchema`, `factGraphSection` — to populate retrieval and triple stores. |
| **mcp_knowledge_ops** | `workflows`, `pipelines`, `runbooks` — to run operational procedures. |
| **Bundle installer** | `integrity`, `compatibility`, `requires` — to gate installation. |

The bundle itself does nothing. It is data — a package format whose
contents are activated by the host that loads it.

## 0.3 Scope of this Specification

This specification defines:

- The shape of `McpBundle` (root) and `BundleManifest` (identity).
- The 14 content sections and their JSON schemas.
- The `tools[]` declaration with its 4 dispatch kinds.
- The knowledge-ops sections (`facts`, `workflows`, `pipelines`,
  `runbooks`).
- The `wiring` surface that binds tools to UI slots and lifecycle
  hooks.
- The 12 reserved folders, the `.mbd/` directory layout, and the `.mcpb`
  archive format.
- The validation pipeline (schema, reference, integrity).
- The Port catalog the host exposes for tools to call back into.
- The pack / sign / install lifecycle.
- The Expression Language used in conditions, transforms, and
  template interpolation.

This specification does **not** define:

- The internal structure of any specific host runtime.
- How a host renders a particular UI widget — see
  [`mcp_ui_dsl 1.3`](../../ui_dsl/1.3/).
- The flow execution semantics — see the MCP Flow DSL spec.
- Marketplace policy (signing roots, fee model, listing rules) — those
  belong to the marketplace product, not the package format.

## 0.4 Relation to Other Specs

| Spec | Relation |
|------|----------|
| **mcp_ui_dsl 1.3** | Files under `ui/` MUST be valid mcp_ui_dsl JSON. The bundle is the on-disk container; the runtime contract for rendering is owned by the UI DSL spec. |
| **mcp_flow_dsl** | The optional `flow` section follows the flow DSL contract. A bundle MAY omit `flow` entirely; UI bundles often do. |
| **MCP base protocol** | `tools[]` of `kind: mcp` reference external MCP servers reached via `transport: 'http'` or `transport: 'stdio'`. The MCP wire protocol is unchanged by this spec — the bundle only declares the connection. |

## 0.5 Terminology

| Term | Definition |
|------|------------|
| **Bundle** | The whole package — a `McpBundle` root with manifest and zero or more content sections. |
| **Manifest** | The `BundleManifest` object in `manifest.json` — identity and metadata. |
| **Section** | One of the 14 named top-level fields under the bundle root (`ui`, `tools`, `knowledge`, …). |
| **Reserved folder** | A directory name the bundle format owns (`ui/`, `assets/`, `skills/`, `knowledge/`, `profiles/`, `philosophy/`, `agents/`). See [`07_Resources.md`](07_Resources.md). |
| **`.mbd/`** | Working-tree directory layout — the bundle exploded as files. Authoring and dev workflows operate here. |
| **`.mcpb`** | Deterministic ZIP archive of an `.mbd/`. Distribution form. |
| **Tool** | A leaf invocable declared in `tools[]` — host-builtin / external MCP / cloud HTTPS / bundled JS. See [`04_Tools.md`](04_Tools.md). |
| **Wiring** | The `manifest.wiring.*` and `manifest.chat.slashCommands[]` surface that maps tools to host chrome slots. See [`06_Wiring.md`](06_Wiring.md). |
| **Host** | The runtime that loads, installs, and activates the bundle (e.g. AppPlayer, Studio). |
| **Slot** | A named position in the host chrome that the host owns; bundles bind tools into slots through `wiring.lifecycle[]` or `wiring.domainActions[]`. |
| **Atom** | A coarse-grained host capability category declared via `manifest.requires.builtinAtoms[]` (e.g. `mcp`, `fs`, `ui`). JS tools call host functionality through `host.<atom>.<verb>`. |
| **Port** | A capability-named contract defined under `package:mcp_bundle/ports.dart`. Hosts implement ports; bundles declare requirements but do not bind to port classes directly. See [`09_Port_Catalog.md`](09_Port_Catalog.md). |

## 0.6 Requirement Levels

This spec uses **MUST / SHOULD / MAY** per RFC 2119 / RFC 8174:

- **MUST** — absolute requirement. A bundle / host that violates this
  is non-conformant.
- **SHOULD** — strong recommendation. Divergence is allowed when
  documented; consumers should still accept the canonical form.
- **MAY** — optional. Either choice conforms.

Each section calls out the level explicitly when a rule is normative.
Prose without an explicit level is descriptive.

## 0.7 Versioning

Two version axes are tracked separately:

| Axis | Field | Owned by |
|------|-------|----------|
| **Spec version** | This document — `1.0`. | This `bundle/1.0/` spec directory. |
| **Bundle schemaVersion** | `McpBundle.schemaVersion` (default `"1.0.0"`). | The bundle author. The reference implementation's CHANGELOG anchors what additive shapes are accepted under each schemaVersion. |

A bundle with `schemaVersion: "1.0.0"` MUST be parseable by any host
that implements this spec. New sections added in spec patches (`1.0.X`)
remain backward-compatible — `1.0.0` bundles continue to load. Breaking
changes bump the spec major version (`2.0`) and live under a new
sibling directory.

The `BundleManifest.version` field is the **bundle's own product
version** (semver) — independent of both the spec version and the
schemaVersion.

## 0.8 JSON Conventions

- **Key style:** camelCase.
- **Enum value style:** lowercase single-word or camelCase multi-word.
  All Dart enums in the reference implementation include an `unknown`
  variant; loaders MUST accept unknown enum strings without throwing
  (round-trip as `unknown`) so additive enum changes are
  forward-compatible.
- **Comments:** JSON does not support comments. Inline `//` in this
  document's examples are informative only.
- **Required vs optional:** Required fields are marked
  `required` in schema tables. Optional fields show their default
  inline.
- **Empty collections:** A section that exists but is empty (`{ "tools":
  { "tools": [] } }`) is valid — it just exposes nothing. The host
  treats absent and empty as equivalent.

## 0.9 Where the Spec Lives in Code

The reference implementation is the `mcp_bundle` package; the paths below
are relative to that package's root.

| Concern | Reference file in the `mcp_bundle` package |
|---------|----------------|
| Root model | `lib/src/models/bundle.dart` |
| Manifest | `lib/src/models/manifest.dart` |
| Section models | `lib/src/models/*_section.dart` |
| Reserved folders | `lib/src/io/bundle_resources.dart` |
| Validator | `lib/src/validator/mcp_bundle_validator.dart` |
| Loader | `lib/src/io/mcp_bundle_loader.dart` |
| Pack / sign / install | `lib/src/install/` |
| Expression engine | `lib/src/expression/` |
| Port catalog | `lib/src/ports/` |

Code citations throughout this spec use the form
`<file>:<line-or-symbol>`.
