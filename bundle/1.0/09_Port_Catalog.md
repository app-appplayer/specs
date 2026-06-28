# 09. Port Catalog

The `mcp_bundle` package ships a catalog of **capability-named
contracts** ("ports") that other packages and hosts implement. The
catalog is part of the package surface — bundle declarations indirectly
depend on these contracts because hosts back the JS host-bridge atoms
(see [`04_Tools.md`](04_Tools.md) §4.8) with port implementations.

This document is a **reference** — it lists every port defined under
`lib/src/ports/` in the `mcp_bundle` package. The detailed Dart API
lives in source; this section anchors the names so they appear once
in the spec and so adopters know which contracts are stable.

References, in the `mcp_bundle` package:
- `lib/src/ports/ports.dart` (barrel export)
- `lib/ports.dart` (top-level barrel)

## 9.1 Catalog Layout

Ports are grouped by capability domain. The grouping mirrors the
barrel file's section headers.

### 9.1.1 Core Capability Ports

| Port | File | Role |
|------|------|------|
| `LlmPort` | `llm_port.dart` | LLM inference. Used by skills, agents. |
| `StoragePort` | `storage_port.dart` | KV-style host storage. |
| `MetricPort` / `MetricsPort` | `metric_port.dart` / `metrics_port.dart` | Metric emission + structured aggregation. |
| `EventPort` | `event_port.dart` | Domain event bus. |
| `ChannelPort` | `channel_port.dart` | Bidirectional messaging (Slack / Telegram / Discord / HTTP / WS / ...). |
| `ApprovalPort` | `approval_port.dart` | Human-in-the-loop approval. |
| `NotificationPort` | `notification_port.dart` | User-facing notifications. |
| `IngestPorts` | `ingest_ports.dart` | Document ingestion pipeline. |

`ChannelPort` carries `ChannelIdentity`, `ConversationKey`,
`ChannelEvent`, `ChannelResponse`, `ChannelCapabilities` — see
`channel_port.dart` for the full type set.

### 9.1.2 Knowledge / Data Ports

| Port | File | Role |
|------|------|------|
| `FactsPort` | `facts_port.dart` | Read / write fact triples. |
| `EntitiesPort` | `entities_port.dart` | Typed entity store. |
| `ClaimsPort` | `claims_port.dart` | Claim assertions with confidence + provenance. |
| `EvidencePort` | `evidence_port.dart` | Source evidence backing claims. |
| `CandidatesPort` | `candidates_port.dart` | Candidate generation + ranking. |
| `PatternsPort` | `patterns_port.dart` | Reusable pattern matching. |
| `SummariesPort` | `summaries_port.dart` | Pre-computed summaries. |
| `IndexPort` | `index_port.dart` | Vector / keyword indices. |
| `RetrievalPort` | `retrieval_port.dart` | Query → top-k results. |
| `AssetPort` | `asset_port.dart` | Binary / large-blob storage. |
| `ContextBundlePort` | `context_bundle_port.dart` | Composing context bundles for LLM calls. |
| `CollectionStoragePort` | `knowledge_ports.dart` | Generic typed collection store. |
| `DatasourceAdapter` (+ `FsAdapter` / `DbAdapter`) | `datastore_port.dart` | `fs.*` / `db.*` data-interface adapter contracts (datastore capability — platform spec `13-datastores`; impls in the `mcp_datastore*` packages). |

### 9.1.3 Profile / Decision Ports

| Port | File | Role |
|------|------|------|
| `AppraisalPort` | `appraisal_port.dart` | Score / evaluate candidates. |
| `DecisionPort` | `decision_port.dart` | Make a decision under policy. |
| `ExpressionPort` | `expression_port.dart` | Evaluate expression-language strings against a context. See [`11_Expression.md`](11_Expression.md). |
| `ProfileSummariesPort` | `profile_summaries_port.dart` | Per-profile metric aggregation. |

### 9.1.4 Skill Execution Ports

| Port | File | Role |
|------|------|------|
| `SkillRegistryPort` | `skill_registry_port.dart` | Register / look up skills. |
| `SkillRuntimePort` | `skill_runtime_port.dart` | Execute a skill's procedure graph. |

### 9.1.5 Operations Ports

| Port | File | Role |
|------|------|------|
| `WorkflowPort` | `workflow_port.dart` | Run a `WorkflowsSection` entry. |
| `PipelinePort` | `pipeline_port.dart` | Run a `PipelinesSection` entry. |
| `RunbookPort` | `runbook_port.dart` | Run a `RunbooksSection` entry. |
| `RunsPort` | `runs_port.dart` | Persist + query execution runs. |
| `ScheduleTriggerPort` | `schedule_trigger_port.dart` | Cron / interval triggers. |
| `AuditPort` | `audit_port.dart` | Audit log emission. |

### 9.1.6 Philosophy Ports

| Port | File | Role |
|------|------|------|
| `PhilosophyPort` | `philosophy_port.dart` | Read / apply philosophy entries. |
| `EthosStorePort` | `ethos_store_port.dart` | Persisted ethos records (philosophy + provenance). |

### 9.1.7 IO Ports

| Port | File | Role |
|------|------|------|
| `IoDevicePort` | `io_device_port.dart` | Device I/O abstraction. |
| `IoPolicyPort` | `io_policy_port.dart` | Per-device policy enforcement. |
| `IoRegistryPort` | `io_registry_port.dart` | Device discovery / registration. |
| `IoAuditPort` | `io_audit_port.dart` | I/O-layer audit events. |
| `IoStreamPort` | `io_stream_port.dart` | Streamed I/O channels. |

### 9.1.8 Form Ports

| Port | File | Role |
|------|------|------|
| `FormPort` | `form_port.dart` | Generic form state. |
| `FormRendererPort` | `form_renderer_port.dart` | Render a form definition. |
| `FormTemplatePort` | `form_template_port.dart` | Reusable form templates. |

### 9.1.9 Analysis Ports

| Port | File | Role |
|------|------|------|
| `AnalysisPort` | `analysis_port.dart` | Analytical computation entry point. |
| `AnalysisDatasourcePort` | `analysis_datasource_port.dart` | Source for analytical inputs. |
| `AnalysisFunctionPort` | `analysis_function_port.dart` | Named analytical function. |

### 9.1.10 UI / Flow / MCP Ports

| Port | File | Role |
|------|------|------|
| `UiPort` | `ui_port.dart` | UI activation entry point. |
| `FlowPort` | `flow_port.dart` | Flow execution entry point. |
| `McpPort` | `mcp_port.dart` | Bridge to the MCP wire protocol. |

## 9.2 Port-vs-Atom Relationship

Ports live in **Dart** — typed contracts implemented by host packages.
Atoms live in **JS** — runtime globals exposed to bundle scripts.

A typical host implements:

- one or more ports (e.g. `FactsPort`, `LlmPort`),
- a host MCP server registering builtin tools that delegate to those
  ports,
- a JS host-bridge that exposes the same surface under `host.<atom>`.

A bundle's JS does not import port classes — it talks to the atom
bridge. Hosts internally route atoms through ports. The two layers
share names by convention but are independently evolvable.

## 9.3 Port Stability

Ports under `dart/lib/src/ports/` are part of the **mcp_bundle public
surface**. Adding a port is additive; renaming or breaking a port's
method shape is a major-version event for the package.

This spec does not freeze each port's method signature — that is owned
by the Dart source. The spec freezes:

1. Port **names** as they appear in the catalog.
2. The package layout (`lib/src/ports/` in the `mcp_bundle` package).
3. The barrel file (`ports.dart`) as the single import surface.

Adopters may bind to ports by name knowing the catalog will grow but
not silently shed entries.

## 9.4 Where Ports Are Implemented

Ports are implemented by:

| Package | Implements |
|---------|------------|
| `mcp_llm` | `LlmPort`. |
| `mcp_knowledge` | `FactsPort`, `EntitiesPort`, `ClaimsPort`, `EvidencePort`, `CandidatesPort`, `SummariesPort`, `RetrievalPort`, `IndexPort`. |
| `mcp_knowledge_ops` | `WorkflowPort`, `PipelinePort`, `RunbookPort`, `RunsPort`, `ScheduleTriggerPort`. |
| `mcp_profile` | `AppraisalPort`, `DecisionPort`, `ExpressionPort`, `ProfileSummariesPort`. |
| `mcp_skill` | `SkillRegistryPort`, `SkillRuntimePort`. |
| Host (AppPlayer / Studio) | `StoragePort`, `EventPort`, `NotificationPort`, `ApprovalPort`, `UiPort`, `FlowPort`, `McpPort`, `IoDevicePort` family. |

This mapping is informational — implementers can re-shuffle ports
across packages as needed.

## 9.5 Conformance

A bundle MUST NOT depend on a specific port being implemented. The
correct dependency surface is `manifest.requires.builtinAtoms[]` and
`builtinTools[]` (see [`02_Manifest.md`](02_Manifest.md) §2.7). Ports
are an internal detail of how hosts back those declarations.

A host MAY implement any subset of the catalog; bundles that require
an atom whose backing port is missing MUST fail activation per
[`02_Manifest.md`](02_Manifest.md) §2.7 — regardless of whether the
port itself is "implemented" in some other sense.
