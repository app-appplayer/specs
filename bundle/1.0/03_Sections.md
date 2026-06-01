# 03. Sections

This document defines the 19 content sections of an `McpBundle`. Each
is **optional** at the bundle root. Their JSON shapes are listed here;
the five newest knowledge / tool sections (`tools`, `facts`,
`workflows`, `pipelines`, `runbooks`) get their own documents — see
[`04_Tools.md`](04_Tools.md) and
[`05_Knowledge_Sections.md`](05_Knowledge_Sections.md). The portability
contract (`requires`) is defined in [`02_Manifest.md`](02_Manifest.md)
§2.7 and surfaces here as a typed top-level section accessor.

Reference, in the `mcp_bundle` package: `lib/src/models/`.

## 3.1 Section Roster

| # | Section | Detailed in | Inline key |
|---|---------|-------------|-----------|
| 1 | `ui` | [§3.3](#33-uisection) + [`mcp_ui_dsl 1.3`](../../ui_dsl/1.3/) | `ui` |
| 2 | `flow` | [§3.4](#34-flowsection) + the MCP Flow DSL spec | `flow` |
| 3 | `skills` | [§3.5](#35-skillsection) | `skills` |
| 4 | `assets` | [§3.6](#36-assetsection) | `assets` |
| 5 | `knowledge` | [§3.7](#37-knowledgesection) | `knowledge` |
| 6 | `bindings` | [§3.11](#311-bindingsection) | `bindings` |
| 7 | `tests` | [§3.12](#312-testsection) | `tests` |
| 8 | `policies` | [§3.13](#313-policysection) | `policies` |
| 9 | `profiles` | [§3.8](#38-profilessection) | `profiles` |
| 10 | `philosophy` | [§3.9](#39-philosophysection) | `philosophy` |
| 11 | `agents` | [§3.10](#310-agentssection) | `agents` |
| 12 | `facts` | [`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) §5.1 | `facts` |
| 13 | `workflows` | [`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) §5.2 | `workflows` |
| 14 | `pipelines` | [`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) §5.3 | `pipelines` |
| 15 | `runbooks` | [`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) §5.4 | `runbooks` |
| 16 | `tools` | [`04_Tools.md`](04_Tools.md) | `tools` |
| 17 | `requires` | [`02_Manifest.md`](02_Manifest.md) §2.7 + [§3.14](#314-requiressection) | `requires` |
| 18 | `factGraphSchema` | [§3.15](#315-factgraphschema) | `factGraphSchema` |
| 19 | `factGraphSection` | [§3.16](#316-factgraphsection) | `factGraphSection` |

(Note: `requires` is also documented as a manifest field in §2.7 of
the manifest doc — both views describe the same shape.)

(Numbering is for navigation; section presence order in
`presentSections` is fixed by the bundle root — see
[§1.3](01_Bundle_Schema.md#13-presentsections-walk).)

## 3.2 Common Section Pattern

Every typed section MUST:

- expose a `schemaVersion` field defaulting to `"1.0.0"`;
- be safe to omit entirely from the bundle root (absent = "no
  declarations of this kind");
- round-trip through `fromJson(json).toJson() == json` for every key
  the spec defines.

A section SHOULD:

- expose an `isEmpty` / `isNotEmpty` accessor in its model class;
- expose `findById(id)` / `findByName(name)` lookups when its entries
  carry stable identifiers.

## 3.3 `UiSection`

Reference: `models/ui_section.dart`.

The canonical UI representation is the on-disk `ui/` reserved folder
(see [`07_Resources.md`](07_Resources.md)). Each `ui/<rel>.json` file
maps to a `ui://<rel>` MCP resource URI.

The typed `pages` / `widgets` / `theme` / `navigation` / `state` fields
on `UiSection` are **deprecated** (round-trip-only forward-compat for
legacy bundles) and slated for removal in a future spec patch.
Authoring tools MUST write under `ui/` instead of inline. The
`UiSection.raw` field (in the `mcp_bundle` package) preserves the raw
inline map so unknown keys survive a load-save cycle.

```json
"ui": {
  "schemaVersion": "1.0.0",
  "pages": [...],         // deprecated
  "theme": { ... },       // deprecated
  "navigation": { ... }   // deprecated
}
```

The matching folder layout:

```
<bundle>.mbd/
├── manifest.json
└── ui/
    └── app.json    ← mcp_ui_dsl 1.3 ApplicationDefinition
```

## 3.4 `FlowSection`

Reference: `models/flow_section.dart`.

Carries flow definitions that follow the MCP Flow DSL contract.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. |
| `flows` | `FlowDefinition[]` | Flow definitions (id, name, trigger, steps, inputs, output, retry). |
| `sharedState` | object | Variables accessible across flows. |
| `errorHandlers` | `ErrorHandler[]` | Pattern-matched error handlers. |

Flow execution semantics belong to the flow DSL spec; the bundle only
carries the definitions.

## 3.5 `SkillSection`

Reference: `models/skill_section.dart`.

Skill module definitions for `mcp_skill`. Each module is a procedure
graph with rubrics.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. |
| `modules` | `SkillModule[]` | Skill modules. |
| `config` | `SkillConfig?` | Shared LLM defaults / timeout / retry. |

`SkillModule` exposes 11 fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | **MUST** be unique within the section. |
| `name` | string | Display name. |
| `version` | string | Default `"1.0.0"`. |
| `description` | string? | — |
| `provider` | string? | — |
| `inputs` | `SkillParameter[]` | Typed input parameters. |
| `output` | `SkillOutput?` | Output schema + optional `claims[]`. |
| `procedures` | `SkillProcedure[]` | Procedure graph (steps, conditions, error handlers). |
| `triggers` | `SkillTrigger[]` | Activation triggers (`explicit` / `intent` / `pattern` / `event`). |
| `capabilities` | string[] | Required capabilities. |
| `mcpTools` | `McpToolRef[]` | Required external MCP tools. |
| `knowledgeSources` | `KnowledgeSourceRef[]` | RAG source references. |
| `rubric` | `SkillRubric?` | Evaluation criteria. |

```json
"skills": {
  "modules": [
    {
      "id": "summarize",
      "name": "Summarize Document",
      "inputs": [
        { "name": "text", "type": "string", "required": true }
      ],
      "procedures": [
        {
          "id": "main",
          "name": "summarize",
          "steps": [
            { "id": "p", "action": { "type": "prompt", "template": "Summarize: ${inputs.text}" } }
          ]
        }
      ]
    }
  ]
}
```

## 3.6 `AssetSection`

Reference: `models/asset.dart`.

Static asset declarations (images, fonts, files). Distinct from the
`assets/` reserved folder which carries the binary data. The section
declares **what** assets exist; the folder carries the bytes.

| Field | Type | Description |
|-------|------|-------------|
| `assets` | `Asset[]` | Individual asset declarations. |
| `directories` | `AssetDirectory[]` | Glob-pattern directory inclusions. |
| `bundles` | `AssetBundle[]` | Grouped asset bundles with load strategies. |

Each `Asset`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string? | Optional stable identifier. |
| `path` | string | **MUST** — relative path under `assets/`. |
| `type` | enum [`AssetType`](#361-assettype) | Asset kind. |
| `mimeType`, `encoding`, `content`, `contentRef`, `hash`, `size`, `metadata` | various | Descriptive fields. |

### 3.6.1 `AssetType`

```
image | icon | font | audio | video | json | text | binary |
template | style | file | unknown
```

Each variant has a `commonMimeTypes` accessor in the reference
implementation.

### 3.6.2 `LoadStrategy`

```
eager | lazy | preload | unknown
```

For `AssetBundle.loadStrategy`. Defaults to `lazy`.

## 3.7 `KnowledgeSection`

Reference: `models/knowledge.dart`.

RAG sources + retriever + index configuration.

| Field | Type | Description |
|-------|------|-------------|
| `sources` | `KnowledgeSource[]` | Document collections / web / API / DB / directory / git sources. |
| `retriever` | `RetrieverConfig?` | Retrieval mode (`similarity` / `keyword` / `hybrid` / `mmr`), topK, min score, rerank, hybrid weights. |
| `index` | `IndexConfig?` | Index type (`memory` / `faiss` / `pinecone` / `qdrant` / `weaviate` / `chroma`). |

Each `KnowledgeSource` carries `chunking` (`fixedSize` / `sentence` /
`paragraph` / `semantic` / `recursive`) and `embedding` configuration.

```json
"knowledge": {
  "sources": [
    {
      "id": "docs",
      "name": "Product Docs",
      "type": "documents",
      "documents": [
        { "id": "intro", "title": "Intro", "content": "...", "format": "markdown" }
      ],
      "chunking": { "strategy": "paragraph", "chunkSize": 512 },
      "embedding": { "model": "text-embedding-3-small" }
    }
  ],
  "retriever": { "mode": "hybrid", "topK": 5 },
  "index": { "type": "faiss" }
}
```

## 3.8 `ProfilesSection`

Reference: `models/profile_section.dart`.

System-prompt building blocks. A `ProfileDefinition` collects
`ProfileContentSection[]` — named content blocks with priority and
optional condition expressions.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). |
| `profiles` | `ProfileDefinition[]` | Profile definitions. |

```json
"profiles": {
  "schemaVersion": "1.0.0",
  "profiles": [
    {
      "id": "engineer",
      "name": "Engineer",
      "version": "1.0.0",
      "sections": [
        { "name": "role", "content": "You are a careful engineer.", "priority": 100 },
        { "name": "style", "content": "Prefer terse answers.", "priority": 50 }
      ]
    }
  ]
}
```

## 3.9 `PhilosophySection`

Reference: `models/philosophy_section.dart`.

Guiding principles paired with applied examples and counter-examples.
**Distinct from `policies`** — philosophy captures *why* and *what to
learn from*, policy captures *what to enforce*.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). |
| `philosophies` | `Philosophy[]` | Philosophy definitions. |

| Field on `Philosophy` | Description |
|-----------------------|-------------|
| `id`, `name`, `statement` | **Required** identity + the one-or-two-sentence principle. |
| `rationale` | Optional — *why* the principle exists. |
| `examples`, `counterexamples` | `PhilosophyExample[]` — `description / context? / source?`. |
| `school` | Optional category (`engineering`, `design`, ...). |
| `confidence` | 0.0-1.0 confidence in the principle. |
| `metadata` | Free-form. |

```json
"philosophy": {
  "schemaVersion": "1.0.0",
  "philosophies": [
    {
      "id": "no-magic-numbers",
      "name": "No Magic Numbers",
      "statement": "Every numeric constant deserves a name.",
      "examples": [{ "description": "RETRY_LIMIT = 3" }],
      "counterexamples": [{ "description": "for (i = 0; i < 3; i++) {...}" }]
    }
  ]
}
```

## 3.10 `AgentsSection`

Reference: `models/agent_section.dart`.

Agent **definitions** — static descriptions of agents the host runtime
can instantiate. Each agent binds a role to four-axis knowledge assets
(profiles / skills / facts / philosophies) plus runtime config (model,
tools, behavior).

The bundle carries the definition only; the host owns instantiation.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). |
| `agents` | `AgentDefinition[]` | Agent definitions. |

| Field on `AgentDefinition` | Description |
|----------------------------|-------------|
| `id`, `name`, `role` | **Required**. |
| `description` | Optional. |
| `profileIds`, `skillIds`, `factSourceIds`, `philosophyIds` | The four axes — string id arrays referencing the corresponding sections. Unresolved ids produce validator warnings, not errors. |
| `systemPrompt` | Additional fragment appended after profile content. |
| `model` | `AgentModelConfig` — `provider`, `model`, `temperature`, `topP`, `maxTokens`, `options`. |
| `tools` | string[]? — tool names this agent may invoke. `null` = host default. |
| `behavior` | `AgentBehaviorConfig` — `turnLimit`, `timeoutMs`, `maxRetries`, `parallelTools`, `options`. |
| `metadata` | Free-form. |

```json
"agents": {
  "schemaVersion": "1.0.0",
  "agents": [
    {
      "id": "reviewer",
      "name": "Code Reviewer",
      "role": "reviewer",
      "profileIds": ["engineer"],
      "skillIds": ["review-pr"],
      "philosophyIds": ["no-magic-numbers"],
      "model": { "provider": "anthropic", "model": "claude-opus-4-7" },
      "behavior": { "turnLimit": 10, "timeoutMs": 30000 }
    }
  ]
}
```

## 3.11 `BindingSection`

Reference: `models/binding.dart`.

Cross-section data binding declarations.

| Field | Type | Description |
|-------|------|-------------|
| `bindings` | `DataBinding[]` | `{ id, source, target, direction, transform?, condition?, debounceMs? }`. |
| `sources` | `DataSource[]` | Source descriptors. |
| `computed` | `{[name]: ComputedValue}` | Computed value definitions. |

`BindingDirection`: `oneWay` (default) / `twoWay` / `oneTime` / `unknown`.

## 3.12 `TestSection`

Reference: `models/test_section.dart`.

Bundled test suites + fixtures + global test config. Authoring-time
QA — hosts MAY ignore at install. Used by editors / CI integrations
that want to run bundled tests against the host's tool surface.

## 3.13 `PolicySection`

Reference: `models/policy.dart`.

Decision / validation policy rules. Each `Policy` carries `rules[]`
plus a `priority` (0-100, higher first) and `enabled` flag.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). |
| `policies` | `Policy[]` | Policy definitions. |

## 3.14 `RequiresSection`

Reference: `models/requires_section.dart`.

Portability contract — the host capabilities the bundle needs to
function. Two parallel lists. The complete authoring guidance lives in
[`02_Manifest.md`](02_Manifest.md) §2.7; this section is the typed
top-level mirror reachable through `bundle.requires`.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). |
| `builtinAtoms` | string[] | Coarse atom-category requirements (`mcp`, `fs`, `ui`, `workspace`, `kb`, `agent`, `bus`, `bundle`, ...). |
| `builtinTools` | string[] | Specific host-tool name requirements (`studio.workspace.save`, ...). |

Both lists are **strict gates** — see
[`02_Manifest.md`](02_Manifest.md) §2.7 for activation refusal rules.

```json
"requires": {
  "schemaVersion": "1.0.0",
  "builtinAtoms": ["mcp", "fs", "ui"],
  "builtinTools": ["studio.workspace.save"]
}
```

## 3.15 `factGraphSchema`

Reference: `models/fact_graph_schema.dart`.

Type definitions for entities / relations / facts. Used by validators
when bundle data references typed graph elements.

| Field | Type | Description |
|-------|------|-------------|
| `entityTypes` | `EntityTypeDefinition[]` | `{ name, properties[], requiredProperties[], extendsType?, isAbstract }`. |
| `relationTypes` | `RelationTypeDefinition[]` | Relation type defs. |
| `factTypes` | `FactTypeDefinition[]` | Fact type defs. |

Distinct from `factGraphSection` (instance data) and from the
top-level `facts` section (atomic SPO triples — see
[`05_Knowledge_Sections.md`](05_Knowledge_Sections.md) §5.1).

## 3.16 `factGraphSection`

Reference: `models/fact_graph_section.dart`.

Instance data for the fact graph — entities, facts, relations,
summaries, policies — embedded inline or referenced externally.

| Field | Type | Description |
|-------|------|-------------|
| `schemaVersion` | string | Default `"1.0.0"`. Per common pattern (§3.2). Additive to the legacy `version` field — both round-trip independently. |
| `version` | string | Default `"1.0.0"`. Legacy version field preserved for backward compatibility with bundles authored before `schemaVersion` was added. |
| `mode` | enum `FactGraphMode` | `embedded` / `referenced` / `hybrid`. |
| `embedded` | `EmbeddedFactGraphData?` | `entities[]`, `facts[]`, `relations[]`, `summaries[]`, `policies[]`. |
| `external` | `ExternalFactGraphRef?` | Reference to an external fact graph. |
| `extraction` | `ExtractionConfig?` | Configuration for extracting facts from documents. |

## 3.17 Cross-Section References

A section MAY reference ids declared in another section:

| Reference | From | To |
|-----------|------|-----|
| `AgentDefinition.profileIds[]` | `agents` | `profiles[*].id` |
| `AgentDefinition.skillIds[]` | `agents` | `skills.modules[*].id` |
| `AgentDefinition.factSourceIds[]` | `agents` | `knowledge.sources[*].id` (or `facts[*].id`, host-defined) |
| `AgentDefinition.philosophyIds[]` | `agents` | `philosophy.philosophies[*].id` |
| `SkillModule.knowledgeSources[*].sourceId` | `skills` | `knowledge.sources[*].id` |
| `wiring.*[*].tool` | `manifest.wiring` | `tools.tools[*].name` (or host builtin) |
| `chat.slashCommands[*].tool` | `manifest.chat` | `tools.tools[*].name` (or host builtin) |

Unresolved references produce **warnings** at validation time. The
host MAY refuse activation when a wiring slot references a missing
tool — see [`06_Wiring.md`](06_Wiring.md) §6.7.
