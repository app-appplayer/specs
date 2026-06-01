# 05. Knowledge Sections

This document specifies the four knowledge-ops sections added in
spec 1.0 (round A): `facts`, `workflows`, `pipelines`, `runbooks`.
They share a uniform shape — top-level section with `schemaVersion`
+ entries, entries with stable `id`, optional metadata, and free-form
step / stage / procedure payloads pending stricter typing in a future
spec patch.

These sections live at the **bundle root** (`McpBundle.facts`,
`McpBundle.workflows`, …) and serialize as inline keys in
`manifest.json`. They each have a matching reserved folder
(`facts/`, `workflows/`, `pipelines/`, `runbooks/`) under `.mbd/` —
authors may carry the entries inline, on disk, or split across both,
with folder content taking precedence on id conflict. The same
equivalence rule that applies to `ui/`, `assets/`, etc. governs
these slots — see [`07_Resources.md`](07_Resources.md) §7.1.

References, in the `mcp_bundle` package:
- `lib/src/models/facts_section.dart`
- `lib/src/models/workflows_section.dart`
- `lib/src/models/pipelines_section.dart`
- `lib/src/models/runbooks_section.dart`

## 5.1 `FactsSection`

Atomic factual statements as **subject-predicate-object** triples
with optional confidence and source provenance. Distinct from
`KnowledgeSection` (which carries retrievable document content) —
facts are typed assertions; knowledge is searchable text.

Distinct from `factGraphSchema` (type definitions) and
`factGraphSection` (typed graph instance data) — `FactsSection`
carries un-typed SPO triples for lightweight graph use.

### 5.1.1 Schema

```json
"facts": {
  "schemaVersion": "1.0.0",
  "facts": [
    {
      "id": "f1",
      "subject": "alice",
      "predicate": "owns",
      "object": "laptop-001",
      "confidence": 0.95,
      "source": "inventory-2026-04",
      "metadata": {}
    }
  ]
}
```

| `Fact` field | Type | Required | Description |
|--------------|------|:--------:|-------------|
| `id` | string? | no | Optional identifier for cross-reference. |
| `subject` | string | **MUST** | Entity / topic. |
| `predicate` | string | **MUST** | Relation / property. |
| `object` | any | **MUST** | String, number, bool, Map, or List. Free-form so hosts encode scalar / structured values uniformly. |
| `confidence` | number? (0.0-1.0) | no | Clamped to range on parse — values < 0 → 0, > 1 → 1. `null` = unknown. |
| `source` | string? | no | Citation / source pointer (knowledge source id, citation, URL). |
| `metadata` | object | no (default `{}`) | Free-form. |

### 5.1.2 Object Encoding

`object` is intentionally `dynamic`:

```json
{ "subject": "x", "predicate": "value", "object": 42 }
{ "subject": "x", "predicate": "tags", "object": ["a", "b", "c"] }
{ "subject": "x", "predicate": "config", "object": { "k": "v" } }
```

Hosts that consume facts MUST be type-aware when reading `object`.
The `_clampConfidence` factory rule applies on every parse — it
silently coerces out-of-range floats; it does not warn.

### 5.1.3 Lookup

The reference model exposes `findById(id)` for O(n) lookup.
Indexed lookup is host responsibility.

## 5.2 `WorkflowsSection`

Ordered **step sequences** describing how to accomplish a task.

```json
"workflows": {
  "schemaVersion": "1.0.0",
  "workflows": [
    {
      "id": "publish",
      "name": "Publish Release",
      "version": "1.0.0",
      "description": "Build, sign, push.",
      "steps": [
        { "id": "build", "tool": "build.run" },
        { "id": "sign",  "tool": "sign.detached", "after": "build" },
        { "id": "push",  "tool": "registry.push", "after": "sign" }
      ],
      "metadata": {}
    }
  ]
}
```

| `WorkflowEntry` field | Type | Required | Description |
|----------------------|------|:--------:|-------------|
| `id` | string | **MUST** | Unique within the section. |
| `name` | string | **MUST** | Display name. |
| `version` | string | no (default `"1.0.0"`) | Semver. |
| `description` | string? | no | — |
| `steps` | object[] | no (default `[]`) | Free-form maps. The host (typically `mcp_knowledge_ops`) defines step shape. |
| `metadata` | object | no | Free-form. |

The `steps[]` shape is intentionally un-typed at v1.0. A future spec
patch will introduce a typed step schema that aligns with
`mcp_knowledge_ops`. Until then, the host owns the step contract — a
host MAY validate steps against its own schema and refuse activation
if shape does not match.

## 5.3 `PipelinesSection`

Ordered **stage sequences** for data / build / deploy flows.

```json
"pipelines": {
  "schemaVersion": "1.0.0",
  "pipelines": [
    {
      "id": "etl-daily",
      "name": "Daily ETL",
      "version": "1.0.0",
      "description": "Pull → transform → load.",
      "stages": [
        { "name": "extract",   "source": "warehouse" },
        { "name": "transform", "transform": "stage1.sql" },
        { "name": "load",      "target": "olap-cube" }
      ]
    }
  ]
}
```

| `PipelineEntry` field | Type | Required | Description |
|----------------------|------|:--------:|-------------|
| `id` | string | **MUST** | Unique within the section. |
| `name` | string | **MUST** | Display name. |
| `version` | string | no (default `"1.0.0"`) | Semver. |
| `description` | string? | no | — |
| `stages` | object[] | no (default `[]`) | Free-form maps; typed schema deferred. |
| `metadata` | object | no | Free-form. |

Same unrestricted-shape rule as `workflows.steps[]`.

The conceptual difference workflows ↔ pipelines: workflows describe
**human-decided procedures** (do A, then B); pipelines describe
**data-flow stages** (data passes through ordered transforms).
Hosts MAY treat both with the same execution engine — the
distinction is for human authoring intent.

## 5.4 `RunbooksSection`

Ordered **operational procedures** — incident response, recovery,
routine maintenance.

```json
"runbooks": {
  "schemaVersion": "1.0.0",
  "runbooks": [
    {
      "id": "db-failover",
      "name": "Database Failover",
      "version": "1.0.0",
      "description": "Promote replica + drain primary.",
      "procedure": [
        { "step": "verify replica lag", "tool": "db.replication.status" },
        { "step": "promote replica",    "tool": "db.replica.promote" },
        { "step": "drain primary",      "tool": "db.primary.drain" }
      ]
    }
  ]
}
```

| `RunbookEntry` field | Type | Required | Description |
|---------------------|------|:--------:|-------------|
| `id` | string | **MUST** | Unique within the section. |
| `name` | string | **MUST** | Display name. |
| `version` | string | no (default `"1.0.0"`) | Semver. |
| `description` | string? | no | — |
| `procedure` | object[] | no (default `[]`) | Ordered procedure entries. Free-form; typed schema deferred. |
| `metadata` | object | no | Free-form. |

## 5.5 Lookup, Validation, Round-Trip

For all four sections:

- The reference model exposes `isEmpty` / `isNotEmpty` getters.
- `findById(id)` returns the first entry matching `id` or `null`.
- Round-trip is total: `Section.fromJson(j).toJson() == j` for every
  field defined here.

The validator's checks are minimal (lenient policy at v1.0):

- `id` (where required) must be non-empty.
- `id` must be unique within the section.
- No cross-section reference checks — hosts validate references
  against their own runtime registries.

## 5.6 Distinguishing the Knowledge-Adjacent Sections

The bundle has several adjacent slots — clarification:

| Slot | Carries | Use when |
|------|---------|----------|
| `knowledge` | RAG document collections + retriever / index config | You want LLM-side retrieval over text. |
| `facts` (v1.0) | Atomic SPO triples | You have small structured assertions to ship inline. |
| `factGraphSchema` | Type definitions for entities / relations / facts | You want hosts to validate typed graph structure. |
| `factGraphSection` | Instance data for the typed graph | You ship typed entities / relations / summaries / policies. |
| `workflows`, `pipelines`, `runbooks` (v1.0) | Procedural / stage / operational sequences | The host is `mcp_knowledge_ops` (or compatible) and consumes these as executable assets. |
| `skills.modules[*].procedures[]` | Skill-internal step graph | The procedure belongs to a single skill module. |

These are independent — a bundle MAY carry any subset.

## 5.7 Conformance

A bundle MUST:

1. Use the listed required fields for every entry (`subject` /
   `predicate` / `object` for facts; `id` / `name` for the other
   three sections).
2. Maintain `id` uniqueness within each section.

A host MAY:

- Apply its own typed-schema validation to `steps` / `stages` /
  `procedure` arrays and refuse activation when the shape does not
  match.
- Index facts by predicate / subject for query performance.
- Cache compiled workflows / pipelines / runbooks across activations.
