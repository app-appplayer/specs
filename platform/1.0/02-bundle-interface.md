# 02 — Bundle Manifest Interface

Bundle (`.mbd`) = layer 2 (conformance interface). This spec is the framework-level bundle composition interface — the contract every host (studio · player · user-arbitrary) MUST follow. It is aligned with the [`../bundle/`](../bundle/) schema spec.

## Structure — Category Matrix

```
.mbd/                                       (directory or .mcpb archive)
├── manifest.json                           ← manifest
├── sources/                                ← original materials (optional)
├── ui/                                     ← UI definition (DSL JSON)
├── tools/                                  ← domain tool source (JS / wasm / etc.)
├── agents/                                 ← agent definitions
├── facts/ skills/ profiles/ philosophies/  ← 4 axis + knowledge
├── flows/ pipelines/ runbooks/             ← operational logic
└── _meta/                                  ← free (cache · index · etc.)
```

Manifest categories (aligned with the actual `McpBundle` schema sections):

| Category | spec | host wiring path |
|---|---|---|
| **id / name / version / requires** | bundle identity + compatibility gate | `BundleActivation.activate(bundle, bundleIdOverride?)` |
| **tools.tools[]** | domain-owned tools (kind: js · external · mcp · cloud) | host → `HostToolRegistry.registerExposed` (`<bundleId>.<rawName>` prefix) |
| **agents** | agent definition (id · role · model · systemPrompt) | `BundleActivation` calls `KnowledgeSystem.agents.createAgent` |
| **skills / profiles / philosophy / facts** | 4 axis + knowledge assets | `BundleActivation` registers 4 facades (automatic) |
| **flow (workflows)** | OpsRuntime workflow registry | `BundleActivation` registers `FlowDefinitionWorkflow` factory (automatic) |
| **pipelines / runbooks** | OpsRuntime pipeline / runbook registry | host wires itself (not auto-handled by BundleActivation · separate round) |
| **ui** | UI definition (mcp_ui_dsl) | host UI runtime mounts |
| **wiring** | chrome slot wiring (lifecycle · domainActions · titlebar · statusbar · settings · chat) | host chrome wires |
| **settings.sections** | settings dialog spec | host settings dialog mounts |
| **chat.slashCommands** | slash commands | host chat panel mounts |
| **assets** | arbitrary data assets (host domain) | host free use |
| **knowledge** | KnowledgeSection (`mcp_knowledge` integration) | host decides (OpsFacade.loadBundle · etc.) |
| **bindings** | data binding definitions | host UI / runtime interprets per case |
| **tests** | bundle-owned test definitions | host verification path |
| **policies** | policy definitions (RBAC · access · security · etc.) | host policy engine |
| **factGraphSection** | fact graph additional assets | `mcp_fact_graph` processing path |

## Per-Category Contract

### 1. tools.tools[]

Domain-owned logic tools. Name = unique within the bundle (the same name MAY exist in another bundle — the host adds a `<bundleId>.<rawName>` prefix).

```json
{
  "name": "editor.open",
  "description": "Open the editor for a given asset.",
  "kind": "js",
  "target": { "entry": "tools/editor.js", "fn": "openEditor" },
  "inputSchema": {
    "type": "object",
    "properties": { "assetId": {"type": "string"} },
    "required": ["assetId"]
  }
}
```

- `kind` = `host` · `mcp` · `cloud` · `js`
- On host wiring = `HostToolRegistry.registerExposed(bundleId, rawName: 'editor.open', ...)` → external name `<bundleId>.editor.open`
- A tool name starting with `bk.` = **forbidden** (knowledge tool domain — the bridge's responsibility)

### 2. agents

Agent definition. On activation, `KnowledgeSystem.agents.createAgent` is called → the bundle's agent pool.

```json
{
  "id": "researcher",
  "name": "Researcher",
  "role": "worker",
  "model": { "provider": "anthropic", "model": "claude-opus-4-7" },
  "systemPrompt": "You are a domain researcher...",
  "tools": ["bk.fact.*", "bk.skill.execute", "editor.open"]
}
```

- agent id = unique within the bundle → exposed as `<bundleId>.<rawId>` on activation (BundleActivation applies the prefix)
- ModelSpec.id is the key into the LLM pool — each agent MAY use a different model
- **`tools` (optional)** = capability allowlist (glob patterns). When specified = overrides the role's default subset. When unspecified = the role's default applies. Canonical specification = [`10-agent-scoping.md`](10-agent-scoping.md).
- Per-role default subset · framework wiring = [`10-agent-scoping.md`](10-agent-scoping.md) + [`06-tool-registry.md`](06-tool-registry.md)

### 3. skills / profiles / philosophies / facts

4 axis assets. On activation, `BundleActivation.activate(bundle)` registers 6 facades.

- skills = `mb.SkillModule` → `KnowledgeSystem.skill.execute`
- profiles = `mb.ProfileDefinition` → `KnowledgeSystem.profile.register`
- philosophies = `mb.Philosophy` → `KnowledgeSystem.ethosStore.putEthos`
- facts = `mb.Fact` (SVO triple) → `KnowledgeSystem.facts.writeFacts`

Each id is unique within the bundle → exposed as `<bundleId>.<id>` on activation.

### 4. Behavior Definition (behavior definition · former flows / pipelines / runbooks)

Among the knowledge categories, **"behavior definition"** = what is executed, under what condition, and how. It was formerly split into three kinds — workflow / pipeline / runbook — but its essence is **a single universal execution engine** — owned by the framework (platform) and consumed **in common by all domains** such as app_builder · Ops · FlowBrain · AppPlayer (no single domain gates it).

Execution = stepping over `state` (a map).

- **step** = `{ id, do(tool/skill call), when(arbitrary expression over state), then(expression result → outcome map), dependsOn?, onFailure? }`.
- **outcome = an open registry** (proceed · skip · wait · stop · goto …) — each a registered handler. A new control = a registered handler, core unchanged. Not an enum/catalog.
- **gate / approval / quality / philosophy = not core concepts.** Expressed by a `when` expression + a tool mutating `state`. "Approval" = an ordinary tool sets `state` → `when` passes → `wait` releases.
- **flow vs runbook vs pipeline = the same engine, differing only in state store** — flow=ephemeral (in-memory) / runbook=durable (persistent, suspend·resume). Not separate models · not separate activation paths.

`BundleActivation` registers these step assets into OpsRuntime via a **single path** (no flow/runbook/pipeline asymmetry).

> Implementation status (2026-05): end-to-end landed — bundle public schema `BehaviorSection` (`mcp_bundle`, `manifest.behavior`) → `BundleActivation.registerBehavior` (brain_kernel) → universal engine `mcp_knowledge_ops/src/behavior/` (state+step+guard+open outcome registry+StateStore, run/resume) → `OpsRuntime.behaviorRegistry`. The full guard/wait/resume path passes tests, with existing flow/runbook unchanged (0 regressions). Remaining (deferred · incremental) = absorbing existing `FlowDefinition`/`Runbook` into engine profiles — not forced, since it is legacy preservation. Canonical (architecture) = a single engine.

### 5. ui

mcp_ui_dsl JSON definition. The host UI runtime (mcp_ui_runtime · custom widgets · etc.) mounts it.

```json
{
  "ui": {
    "page": {
      "type": "page",
      "lifecycle": { "onInit": { "type": "tool", "tool": "refresh" } },
      "content": { ... }
    }
  }
}
```

Compatibility spec = [`../ui_dsl/`](../ui_dsl/). This spec does not define the detail — UI host free.

### 6. wiring

chrome slot wiring. The host chrome reads the manifest and wires:

```json
{
  "wiring": {
    "lifecycle": [
      { "slot": "project.new", "tool": "myCustomNew" }
    ],
    "domainActions": [
      { "tool": "editor.open", "icon": "edit", "tooltip": "Open editor" }
    ],
    "titlebar": "build {{status}} · {{commit}}",
    "statusbar": "recordings: {{count}}",
    "settings": [ ... ]
  }
}
```

- `lifecycle` = row 1 system buttons (standard slots: project.new/open/save/edit.undo/redo/history/revert/close/export.bundle · etc.)
- `domainActions` = row 2 chrome domain icons (tool call auto-prefixed as `<bundleId>.<tool>`)
- `titlebar` / `statusbar` = text payloads (state binding `{{key}}` supported)
- `settings` = settings dialog action entries

### 7. settings.sections

settings spec for inline editor mode. `manifest.settings.sections[].fields[]` of mcp_bundle 0.3.3+.

```json
{
  "settings": {
    "sections": [
      {
        "id": "auth",
        "title": "Authentication",
        "fields": [
          { "id": "apiKey", "type": "secret", "label": "API Key" }
        ]
      }
    ]
  }
}
```

- Each field has type · label · default · validation
- autosave (on change, immediately persisted to host-side secure storage)
- inline editor mode mounts it (mode 5)

### 8. chat.slashCommands

Adds slash chips to the host chat panel.

```json
{
  "chat": {
    "slashCommands": [
      { "command": "/refresh", "tool": "refresh" }
    ]
  }
}
```

- When a user types `/refresh` in chat = calls `<bundleId>.refresh`

## Compatibility Matrix (requires)

```json
{
  "requires": {
    "builtinAtoms": ["text@1.3", "button@1.3", ...],
    "kernelVersion": "^0.1.0",
    "uiSpec": "mcp_ui_dsl@1.3"
  }
}
```

= the host verifies the compatibility gate before activating the manifest. On incompatibility, activation is rejected + the user is notified.

## Change Contract

When changing the categories / wiring paths of this spec:
- Synchronize the schema change in the `mcp_bundle` package
- Verify BundleActivation cascade across all hosts (vibe_studio · AppPlayer · user host)
- Forward compatibility of existing `.mbd` (unknown enum / partial load policy)

## Non-Goals

- Detailed definition of the manifest JSON schema = [`../bundle/`](../bundle/)
- Widget spec of the UI DSL = [`../ui_dsl/`](../ui_dsl/)
- This spec = **only the bundle composition interface matrix + per-category wiring paths**
