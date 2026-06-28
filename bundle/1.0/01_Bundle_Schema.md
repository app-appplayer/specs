# 01. Bundle Schema

The root of every bundle is an `McpBundle` JSON object. This document
defines its required and optional fields and how the host walks them.

Reference, in the `mcp_bundle` package: `lib/src/models/bundle.dart`.

## 1.1 The `McpBundle` Object

### 1.1.1 Schema

| Field | Type | Required | Default | Description |
|-------|------|:--------:|---------|-------------|
| `schemaVersion` | string | **MUST** | `"1.0.0"` | Bundle schema this document conforms to. Hosts MUST treat anything other than `"1.0.0"` as outside the contract of this spec version. |
| `manifest` | [`BundleManifest`](02_Manifest.md) | **MUST** | — | Identity, metadata, dependencies, wiring. |
| `ui` | [`UiSection`](03_Sections.md#33-uisection) | MAY | — | Inline UI data (deprecated round-trip; canonical UI lives in `ui/`). |
| `flow` | [`FlowSection`](03_Sections.md#34-flowsection) | MAY | — | Flow definitions (mcp_flow_dsl). |
| `skills` | [`SkillSection`](03_Sections.md#35-skillsection) | MAY | — | Skill modules (procedure graphs + rubrics). |
| `assets` | [`AssetSection`](03_Sections.md#36-assetsection) | MAY | — | Static asset declarations (images, fonts, files). |
| `knowledge` | [`KnowledgeSection`](03_Sections.md#37-knowledgesection) | MAY | — | RAG sources + retriever / index config. |
| `bindings` | `BindingSection` | MAY | — | Cross-section data binding (sources / computed / two-way). |
| `tests` | `TestSection` | MAY | — | Test suites and fixtures bundled for QA. |
| `policies` | `PolicySection` | MAY | — | Decision / validation policy rules. |
| `profiles` | [`ProfilesSection`](03_Sections.md#38-profilessection) | MAY | — | Profile definitions (system-prompt building blocks). |
| `philosophy` | [`PhilosophySection`](03_Sections.md#39-philosophysection) | MAY | — | Guiding principles + examples. |
| `agents` | [`AgentsSection`](03_Sections.md#310-agentssection) | MAY | — | Agent definitions (4-axis bindings + runtime config). |
| `behavior` | [`BehaviorSection`](03_Sections.md#318-behaviorsection) | MAY | — | Declarative behavior definitions — ordered `do`/`when`/`then` steps the host execution engine runs. |
| `facts` | [`FactsSection`](05_Knowledge_Sections.md#51-factssection) | MAY | — | Subject-predicate-object triples. |
| `workflows` | [`WorkflowsSection`](05_Knowledge_Sections.md#52-workflowssection) | MAY | — | Ordered step sequences. |
| `pipelines` | [`PipelinesSection`](05_Knowledge_Sections.md#53-pipelinessection) | MAY | — | Ordered stage sequences for data / build / deploy. |
| `runbooks` | [`RunbooksSection`](05_Knowledge_Sections.md#54-runbookssection) | MAY | — | Operational procedures. |
| `tools` | [`ToolsSection`](04_Tools.md) | MAY | — | Host-callable tool declarations. |
| `requires` | [`RequiresSection`](02_Manifest.md#27-portability-contract--requires) | MAY | — | Portability contract — atoms / tool names the bundle needs. |
| `factGraphSchema` | `FactGraphSchema` | MAY | — | Type definitions for entities / relations / facts. |
| `factGraphSection` | `FactGraphSection` | MAY | — | Instance data for the fact graph (embedded / referenced / hybrid). |
| `compatibility` | `CompatibilityConfig` | MAY | — | Version + feature requirements enforced at install. |
| `integrity` | `IntegrityConfig` | MAY | — | Content hash, file hashes, signatures. See [`10_Distribution.md`](10_Distribution.md). |
| `extensions` | object | MAY | `{}` | Free-form custom data — round-trips verbatim. Hosts MAY ignore. |

The `directory` field on the in-memory model points to the on-disk
`.mbd/` root when the bundle was loaded from a directory; it is **not
serialized** and never appears in `manifest.json`.

### 1.1.2 Minimal Bundle

```json
{
  "schemaVersion": "1.0.0",
  "manifest": {
    "id": "com.example.minimal",
    "name": "Minimal",
    "version": "1.0.0"
  }
}
```

This is a conformant bundle. It declares nothing and the host activates
it as an empty product. Hosts MUST accept it.

### 1.1.3 Typical Tool-Bearing Bundle

```json
{
  "schemaVersion": "1.0.0",
  "manifest": {
    "id": "com.example.notes",
    "name": "Notes",
    "version": "0.4.0",
    "wiring": {
      "domainActions": [
        {
          "kind": "button",
          "tool": "notes.export",
          "icon": "upload",
          "tooltip": "Export"
        }
      ],
      "lifecycle": [
        { "slot": "project.save", "tool": "notes.save" }
      ]
    }
  },
  "tools": {
    "tools": [
      {
        "name": "notes.export",
        "kind": "js",
        "target": { "entry": "tools/export.js", "fn": "exportAll" }
      },
      {
        "name": "notes.save",
        "kind": "host",
        "target": {}
      }
    ]
  },
  "requires": {
    "builtinAtoms": ["fs", "ui", "workspace"]
  }
}
```

Inside the `.mbd/` working tree this typically pairs with
`ui/app.json` (mcp_ui_dsl) and `tools/export.js`.

## 1.2 Section Independence

Sections are **independent**. Adding `tools` does not require any
other section. Adding `agents` does not require `skills` to be present
even though an agent's `skillIds[]` may reference skill ids — the
validator emits a warning for unresolved cross-references but does not
refuse the bundle (see [`08_Validation.md`](08_Validation.md) §8.3).

A bundle MUST NOT make one section's parse depend on another's. The
loader walks sections in declaration order with no fix-up pass; cross
references are resolved by the host at activation time.

## 1.3 `presentSections` Walk

Hosts that need to enumerate what a bundle declares MUST iterate the
canonical order:

```
ui, flow, skills, assets, knowledge, bindings, tests, policies,
profiles, philosophy, agents, facts, workflows, pipelines, runbooks,
tools, requires, factGraphSchema, factGraphSection, compatibility,
integrity, extensions
```

Reference: `bundle.dart` `presentSections` getter (~line 336).

This order is fixed so different consumers (installers, signers,
loggers) report sections identically.

## 1.4 Round-Trip Guarantee

The reference loader / writer guarantees that:

```
McpBundle.fromJson(input).toJson() ≡ input
```

modulo:

- omitted optional fields with empty / null / default values;
- `schemaVersion` defaulting to `"1.0.0"`;
- enum values that round-trip as `'unknown'` when the input string did
  not match a known variant.

`extensions` is preserved verbatim. Section objects with raw-keep
fields (currently `UiSection.raw`) preserve unknown keys inside that
section.

## 1.5 Manifest vs Folder Equivalence

Most content sections may be expressed **either** inline in
`manifest.json` (under the section key) **or** as on-disk files inside
the matching reserved folder. The two are equivalent — a host that
reads both folders MUST merge with folder content taking precedence
when the same id is declared in both, and SHOULD warn.

| Section | Inline key | Folder | Typed accessor |
|---------|-----------|--------|----------------|
| `ui` | `manifest.ui` (deprecated) | `ui/<rel>.json` | `bundle.uiResources` |
| `assets` | `manifest.assets` | `assets/...` | `bundle.assetResources` |
| `skills` | `manifest.skills` | `skills/...` | `bundle.skillResources` |
| `knowledge` | `manifest.knowledge` | `knowledge/...` | `bundle.knowledgeResources` |
| `facts` | `manifest.facts` | `facts/...` | `bundle.factsResources` |
| `workflows` | `manifest.workflows` | `workflows/...` | `bundle.workflowsResources` |
| `pipelines` | `manifest.pipelines` | `pipelines/...` | `bundle.pipelinesResources` |
| `runbooks` | `manifest.runbooks` | `runbooks/...` | `bundle.runbooksResources` |
| `tools` | `manifest.tools` | `tools/...` | `bundle.toolsResources` |
| `profiles` | `manifest.profiles` | `profiles/...` | `bundle.profileResources` |
| `philosophy` | `manifest.philosophy` | `philosophy/...` | `bundle.philosophyResources` |
| `agents` | `manifest.agents` | `agents/...` | `bundle.agentResources` |

The portability contract (`requires`) and the housekeeping slots
(`bindings`, `tests`, `policies`, `factGraphSchema`,
`factGraphSection`, `compatibility`, `integrity`, `extensions`) are
**inline-only** in v1.0 — they have no reserved folder. See
[`07_Resources.md`](07_Resources.md) §7.1.2.

## 1.6 `extensions`

`extensions` is a free-form object — the host MUST round-trip it but
MAY ignore its contents. Use it for host-specific data that does not
belong in the standardized sections (e.g. analytics tags, marketplace
listing copy that has no spec slot yet). Hosts MUST NOT treat
`extensions` as authoritative for any spec-defined behavior.

```json
{
  "schemaVersion": "1.0.0",
  "manifest": { "id": "com.example.x", "name": "X", "version": "1.0.0" },
  "extensions": {
    "marketplace.tagline": "Yet another notes app.",
    "analytics.cohort": "beta"
  }
}
```

## 1.7 Conformance Summary

A loader implementing this spec MUST:

1. Read `manifest.json` as the canonical bundle root.
2. Default `schemaVersion` to `"1.0.0"` when omitted.
3. Accept any subset of the listed optional sections.
4. Preserve unknown keys in `extensions` and in `UiSection.raw` across
   load → save.
5. Accept `'unknown'` round-trip for enum strings it does not
   recognize.

A loader MAY:

- Parse on-disk reserved folders eagerly (default) or lazily.
- Apply schema validation at load time or defer until activation.
- Warn on inline + folder duplicates.
- Reject bundles whose `compatibility.minRuntimeVersion` exceeds the
  host's runtime. See [`10_Distribution.md`](10_Distribution.md) §10.5.
