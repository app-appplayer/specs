# MCP Bundle Specification — v1.0

This directory holds the canonical specification for **MCP Bundle** — the
packaging format that lets a single artifact carry UI definitions, host-callable
tools, knowledge assets, agents, and the wiring that binds them to host chrome.

The spec is implemented by the Dart reference in the `mcp_bundle` package
(`lib/src/`). Released package versions trail the spec
(current Dart implementation: `0.3.3`); the spec itself is `1.0` because the
shape it documents is stable enough to publish as a contract.

## Contents

| File | Topic |
|------|-------|
| [`00_Overview.md`](00_Overview.md) | What an MCP Bundle is, who consumes it, scope of this spec, terminology, requirement levels. |
| [`01_Bundle_Schema.md`](01_Bundle_Schema.md) | The `McpBundle` root object — `schemaVersion`, `manifest`, the optional content sections. |
| [`02_Manifest.md`](02_Manifest.md) | `BundleManifest` — identity, metadata, dependencies, platform requirements. |
| [`03_Sections.md`](03_Sections.md) | The 14 content sections (`ui`, `flow`, `skills`, `assets`, `knowledge`, `profiles`, `philosophy`, `agents`, `facts`, `workflows`, `pipelines`, `runbooks`, `tools`, `factGraphSchema` / `factGraphSection`) and how the host walks them. |
| [`04_Tools.md`](04_Tools.md) | `ToolsSection` — the 4 tool kinds (`host` / `mcp` / `cloud` / `js`), `target` shapes, JS host bridge. |
| [`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) | `FactsSection` / `WorkflowsSection` / `PipelinesSection` / `RunbooksSection` — data shapes for the knowledge-ops slots. |
| [`06_Wiring.md`](06_Wiring.md) | `manifest.wiring.{domainActions, lifecycle, settings}` + `chat.slashCommands[]`. The Studio-host surface that maps tools to UI slots. |
| [`07_Resources.md`](07_Resources.md) | The 12 reserved folders, the `.mbd/` layout, the `.mcpb` archive, manifest-vs-folder precedence. |
| [`08_Validation.md`](08_Validation.md) | `McpBundleValidator` — error codes, schema / reference / integrity passes, partial-entry pre-insertion validation (§8.6.1), authoring transaction guards (§8.8). |
| [`09_Port_Catalog.md`](09_Port_Catalog.md) | The 50+ Port contracts that bundles indirectly depend on through host capabilities. |
| [`10_Distribution.md`](10_Distribution.md) | Pack / sign / install pipeline — `BundleSigner`, `BundlePacker`, `BundleInstaller`, `TrustStore`, `RuntimeDescriptor`. |
| [`11_Expression.md`](11_Expression.md) | The Expression Language used inside bundle conditions, transforms, and template interpolation. |

## Reading order

Implementers wiring a bundle host to render, dispatch, and install bundles
should read in numeric order: 00 → 11.

Bundle authors writing `.mbd/` directories by hand can start at `01` and
`07`, then jump to whichever section matches the asset they are adding
(`04` for tools, `05` for knowledge, `06` for wiring).

Reviewers sanity-checking a third-party bundle should read `08` and `10`
first, then walk back into the relevant content section.

## Cross-references

- UI definitions inside the `ui/` folder follow
  [`mcp_ui_dsl 1.3`](../../ui_dsl/1.3/).
- Flow definitions inside the optional `flow` section follow the MCP Flow
  DSL spec.
- Code references throughout the spec point at `lib/src/<file>:<line>` in
  the `mcp_bundle` package.
