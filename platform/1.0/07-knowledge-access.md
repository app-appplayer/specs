# 07 — Knowledge Access

The domain access model for knowledge tools (`bk.<facade>.<verb>` 41 standard). The original role of `BundleSessionBridge`. The core contract of scopeId + bundleId-prefixed alias.

## 8 Facades — the Source of Knowledge

| facade | content |
|---|---|
| `facts` | SVO triple (subject · predicate · object · confidence · source) |
| `skills` | Procedure · Step · Action (procedural logic) |
| `profiles` | Profile · sections · capabilities (evaluation elements) |
| `philosophies` | EthosRecord · Philosophy (values) |
| `workflows` | OpsRuntime's workflow registry |
| `pipelines` | OpsRuntime's pipeline registry |
| `runbooks` | OpsRuntime's runbook registry |
| `agents` | agent definition (id · role · model · systemPrompt) |

`KnowledgeSystem` (flowbrain_core) is the single source — shared by all instances.

## `kb://<facade>/<id>` URI Scheme

The framework standard resource URI:

```
kb://fact/<id>           ← FactRecord
kb://skill/<id>          ← Skill (Procedure group)
kb://profile/<id>        ← Profile
kb://philosophy/<id>     ← EthosRecord
kb://workflow/<id>       ← Ops workflow
kb://pipeline/<id>       ← Ops pipeline
kb://runbook/<id>        ← Ops runbook
kb://agent/<id>          ← agent definition
```

The UI's mcp_ui_dsl resource action · bridge.readResource · an external LLM's resources/list all use the same URI.

## scopeId — Per-Bundle Namespace

Behavior of `DispatchContext.scopeId(localId)`:

| session | input | output |
|---|---|---|
| master (host home) | `'foo'` | `'foo'` (pass-through) |
| non-master (`bundleId='recipe_demo'`) | `'foo'` | `'recipe_demo.foo'` (within own namespace) |
| non-master | `'recipe_demo.foo'` (already prefixed) | `'recipe_demo.foo'` (idempotent) |
| non-master | `'other_bundle.foo'` (explicit cross-bundle) | `'other_bundle.foo'` (pass-through) |

= a domain script calls `bk.fact.write({id: 'alice', ...})` → the bridge applies scopeId → actual storage = `recipe_demo.alice`. Isolated within own namespace.

## The Bridge's Two-Name Auto-Mapping

```
bridge.registerTool(
  name: 'bk.fact.write',                   # canonical
  handler: <kernel standardTools handler>,
  inputSchema: <strict>,
)
  → _tools['bk.fact.write'] = def          (dispatcher layer · own-context dispatch)
  → publish an alias per active non-master session:
    serverAdapter(BridgeToolDef(
      name: 'bk.<bundleId>.fact.write',    # alias (externally exposed)
      handler: (args) => callTool(session, 'bk.fact.write', args),
      inputSchema: <same strict>,
    ))
    → registered on the external endpoint
```

= **one registration → two positions activated automatically**:
- own bundle script = calls the standard name `bk.fact.write` (scopeId auto)
- external LLM (CLI subprocess · another host) = calls `bk.<bundleId>.fact.write` (isolated)

## bridge.readResource — kb:// URI Resolution

```dart
Future<Object?> readResource(String uri) async {
  // 1. custom (non-kb) URI first — handler registered by the host
  if (_resources.containsKey(uri)) return _resources[uri](uri);

  // 2. parse kb:// URI
  final ref = KbResourceRef.parse(uri);
  if (ref == null) return null;

  // 3. scopeId applied automatically
  final scopedId = context.scopeId(ref.id);

  // 4. per-facade lookup
  switch (ref.facade) {
    case KbFacade.fact: return system.facts.getFact(scopedId);
    case KbFacade.skill: return system.skillRuntime?.registry.getSkill(scopedId);
    case KbFacade.profile: return system.profile.get(scopedId);
    case KbFacade.philosophy: return system.ethosStore?.getEthos(scopedId);
    case KbFacade.workflow: ...
    case KbFacade.pipeline: ...
    case KbFacade.runbook: ...
    case KbFacade.agent: return system.agents.getAgent(scopedId);
  }
}
```

= a domain script reads `kb://fact/alice` → the bridge applies scopeId → actual lookup = `recipe_demo.alice`. For cross-bundle intent = specify `kb://fact/other_bundle.alice` explicitly.

## Domain Access Flows

### A domain script stores + retrieves its own fact

```
bundle.script:
  await bridge.runScoped(session, () async {
    // 1. store a fact within own namespace
    await bridge.callTool(session, 'bk.fact.write', {
      subject: 'alice',
      predicate: 'age',
      object: 30,
    })
    // → scopeId auto: subject='alice' → actual storage id = 'recipe_demo.alice'

    // 2. retrieve a fact within own namespace
    final fact = await bridge.readResource('kb://fact/alice')
    // → scopeId auto: actual lookup = 'recipe_demo.alice'
    // → same-namespace result
  })
```

= a domain freely accesses its own assets within its own namespace. No effect on other bundles.

### An external LLM accesses a fact of the same bundle

```
external LLM (Claude Code subprocess):
  → endpoint.callTool('bk.recipe_demo.fact.query', {subject: 'alice'})

endpoint dispatch:
  → the bridge's alias handler
  → bridge.callTool(session, 'bk.fact.query', args)
  → same scopeId path (already in bundleId context)
  → retrieves the fact of recipe_demo.alice
```

= an external LLM also accesses a fact of the same bundle in isolation. Calling another bundle's alias (`bk.other_bundle.fact.query`) = a different namespace.

### Explicit cross-bundle access

```
bundle_A.script (within bundle_A context):
  // explicit access to bundle_B's fact
  final fact = await bridge.readResource('kb://fact/bundle_B.alice')
  // → bundleId explicit → scopeId pass-through → actual lookup = 'bundle_B.alice'
```

= cross-bundle communication = the caller specifies it explicitly (`<other_bundle>.<id>` form). The bridge does not auto-prefix (explicit intent preserved).

## Bridge Composition Freedom

| host composition | result |
|---|---|
| bridge + serverAdapter null + master session | no scopeId · canonical name in-process only |
| bridge + serverAdapter wired + master session | no scopeId · canonical as-is on external endpoint (host home) |
| bridge + serverAdapter null + non-master session | scopeId auto · in-process only (no external exposure) |
| bridge + serverAdapter wired + non-master session | scopeId auto · in-process canonical + external bundleId-prefixed alias |

= 4 composition modes free — the host decides whether to expose externally (a wiring concept).

## ethos Governance — scope · provenance · charter Inheritance

The contract that extends the scopeId model above into **governance** of ethos (philosophy). The absence of this contract is the origin of the "philosophy globally active" drift; writing it down aligns the implementation (isomorphic to the `keys(prefix:)` contract gap). (§5's "single source — shared by all instances" is the host home default; governance divides per scope as below.)

### scope: an active ethos is scope-bound

- Governance decisions (the process gate `checkProhibitions`, generation intervention `intervene(postGeneration)`) are made against **that scope's active ethos**. Not one global active. (Both judge **by the active ethos**, not by an agent's forked copy. An agent's owned philosophy ref is only an opt-in flag for generation intervention.)
- `EthosStorePort.activateEthos` / `getActiveEthosId` = the active pointer of that store instance, i.e. of that scope (`<ns>:__active__`). A per-workspace (per-project) store therefore means a per-workspace active.
- ⚠️ Implementation alignment: if the standard tools (`bk.philosophy.*`) are bound to a global KernelApp they bypass the per-scope store and share one global active → no per-scope governance. A host MUST route the governance paths (gate, generation intervention) through **the active scope's system, i.e. that scope's active ethos**.

### provenance: source grade of an ethos + its lifecycle

An ethos payload may carry an optional provenance:

```
payload.provenance = { kind: 'anchor'|'derived'|'workaround', serves?: <parent ethos id>, validWhile?: <condition> }
```

- `kind` defaults to `anchor` (unspecified = a principle; existing data is unconstrained). `derived` / `workaround` (a derived judgement, a temporary detour) **require `serves`** — the higher principle they serve.
- Lifecycle (mirroring fact candidate→confirm): `put` = candidate (`derived` / `workaround` forced inactive), `activate` = confirm (promotion to a principle). This blocks a derivation or a detour from **quietly hardening into a principle**.
- Enforcement point = the kernel's `bk.philosophy.put` / `activate`. Data model = `Ethos` in `mcp_bundle` (payload preserved as-is).

### charter Inheritance — a recursive org unit (top-down)

- **A workspace is a scale-invariant organizational unit** (team · department · company · country; nestable). Each level holds its own **charter** = that scope's active anchor ethos (reusing `valuePriorities` · `prohibitions`, plus mission / northStar).
- Governance resolves as a **top-down chain**: member or child scope → own scope's active → parent scope's active → … A child inherits the parent charter; only an explicit override is its own.
- An override is that member's or scope's `derived` / `workaround` ethos (provenance `serves` = the parent charter). **An exception someone answers for — not a copy of the charter.**
- A charter change is live. Non-overriding children realign automatically, because this is inheritance and not a copy.
- ⚠️ Implement one level first (org→member) and keep resolution **chain-shaped**; nesting then costs zero additional contract. Nesting itself lands on demand.
- **Implemented (2026-07-01, ops host)**: charter and knowledge inheritance resolve along the workspace's **own ancestor chain** (`WorkspaceRegistry.ancestors` = the parentId line, **that line only** — siblings and other branches excluded). Charter prohibitions accumulate (company ∘ department ∘ self); mission / northStar / values take the nearest (self-first). `workspace_get_charter` (effective value + `inheritedFrom`) · the `philosophy_check` gate (prohibitions across the whole chain, forbiddenPatterns as the decision seam) · `knowledge_file_list` / `read` (self + ancestors, self shadows, `inheritedFrom`). This replaces the old per-project single active. ⚠️ Open: unifying the charter active and the `bk.philosophy` active into one (kernel).
- **The head of a unit (team lead, department head) = `Workspace.leadMemberId`** (a member id within it, 2026-06-30). Each organizational unit may hold its own *head* alongside its charter — the apex of the org hierarchy and the default target for approval and escalation. The runtime model for role division = [`12-flowbrain-runtime.md`](12-flowbrain-runtime.md) §5.

> The ethos governance model of "a workspace is a recursive organizational unit" **is canonical in this platform spec**; it needs no separate promotion.

## Compatibility

- `KnowledgeSystem` API = the spec of `flowbrain_core` (a separate position)
- `kb://` URI scheme = framework standard (this spec)
- `BundleSessionBridge` API = brain_kernel's `system/bridge/`
- the bridge's automatic behavior (scopeId · alias publication) = the precise logic of the `BundleSessionBridge` code

## Non-Goals

- Access of domain logic tools (manifest.tools' own logic) = the HostToolRegistry path of `06-tool-registry.md`
- An agent's LLM call = the agent layer of `03-kernel-runtime.md`
- 4-axis prompt composition = flowbrain_core spec
- This spec = **the standard for knowledge tool access + scopeId + kb URI + alias model** only
