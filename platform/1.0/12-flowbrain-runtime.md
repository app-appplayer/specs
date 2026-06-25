# 12 — FlowBrain Runtime (Operational Concept)

The operational-concept contract for FlowBrain. **This document is the definition (canon), and the consumers (flowbrain_core · brain_kernel · host=AppPlayer Standard · Studio · FlowBrain Server) follow it.** Per-host ad-hoc implementation is forbidden.

---

## 0. Identity — FlowBrain = One Name, Three Seats

1. **capability / operational concept** — defined by this spec. The substrate implemented by `flowbrain_core` + `brain_kernel`.
2. **standalone product** — FlowBrain Server (headless knowledge-OS). A complete product built from this capability alone.
3. **embedded feature (host function)** — the feature embedded into brain_kernel/brain-core consumers such as AppPlayer Standard · Studio.

The three seats **share the same operational concept** — this spec defines that concept, so it applies identically whether FlowBrain runs standalone or is embedded into a host.

Operational concept: a runtime where human-organization-style multi-agents (manager · team lead · worker) work via rule-based handoff, **accumulating and evolving 4-layer knowledge (FactGraph · Skill · Profile · Philosophy) across all 4 axes**. The workflow (role division + gates) is itself the deliverable and the asset. Two control planes strike the same surface — **external LLM via MCP** (`bk.agent.*`) + **internal chat** (host dialogue).

## 1. Placement Principle (Single Decision)

**The entire runtime = core (`flowbrain_core` / `brain_kernel`).** All the host provides is **a single confirm callback**.

- Rationale: `AgentRuntime.ask` and the workflow engine are **a single path shared by all hosts**. Placed in core, AppPlayer Standard · Studio · FlowBrain Server receive it automatically, and since no host re-implements it there is **no drift**. Pulling it into the host-layer = per-host re-implementation = drift (the very thing to avoid).
- The host's only responsibility = **injecting the irreversible-action confirm UI callback** + adopting the KnowledgeSystem at boot. core has no screen, so only the confirm UI is physically placed in the host (a constraint, not a placement choice).
- What ops (Studio ops) currently does host-local (the flowbrain agent mirroring of `MemberRegistry`, the skill-dispatch philosophy gate of `ToolDispatcher`) is **lifted into core** → only the confirm callback remains in ops. (A reference implementation, not the final seat.)

## 1.5 Project Knowledge — Scoping + Locality

Project knowledge is (a) **scoped** per project/workspace and (b) **co-located** in the project folder. Move the project and the knowledge follows (self-contained · movable = tenant 0 / token-locality principle).

- **core primitive (scoping)**: `KvStoragePortAdapter(rootDir, workspaceId)` — file KV, enforcing a `ws/<workspaceId>/` key prefix (workspace-scope enforcement, with only global keys excepted). agent registry `agent/<workspaceId>/<agentId>` + a per-agent owned 4-axis fork. That is, scoping is guaranteed by the core primitive.
- **locality (folder residency)**: the host binds `rootDir` to the **project folder** (workspacesRoot).
  - seed/authored knowledge (agents · skills · profiles · philosophy · processes definitions) = `project.mbd` (org-wide shared) + `<wsId>.mbd` (per-workspace) + `project.opsproj` (root marker).
  - runtime accumulation (facts · forks · snapshots) = persisted workspace-scoped under the same `rootDir`.
- **2-tier boundary**: `project.mbd` = organization-shared layer (reused across all workspaces) / `<wsId>.mbd` = workspace-isolated layer (team · project · individual). The shared vs isolated boundary is made explicit.
- The canonical KV/registry = core; the binding (rootDir=project folder) · 2-tier seeding = host. ops is the reference implementation (`KvStoragePortAdapter(rootDir: config.workspacesRoot, workspaceId: …)`).

## 2. 4-Axis ask-time Composition — core (`flowbrain_core` `AgentRuntime`)

At `ask` time the 4 axes assigned to the agent (facts · skills · profile · philosophy) are composed into the systemPrompt.

- **opt-in**: only assigned axes are composed, unassigned axes are skipped (0 new dependency · 0 forcing). An agent with no philosophy assigned = behaves like a pure LLM.
- **Profile confines an agent to its specialty** — this composition is that mechanism. (Currently `_composeAssignedFacts` is facts-only → extended to the 4 axes.)
- The canonical composer = `SystemPromptComposer` (brain_kernel). `AgentRuntime.ask` auto-loads the registry's assigned fork and passes it to the composer (so the host does not manually assemble the snapshot).

## 3. Philosophy's Dual Role — One Evolving Axis + Disciplinarian (Anti-Drift Anchor)

Philosophy **evolves as one** of the 4 axes (§4) while at the same time **uniquely disciplining the other 3 axes (facts · skill · profile)**. This disciplinary role is the anti-drift anchor, which is why it gets its own section (the other axes are not disciplinarians).

- Intervenes at the `ask` moment + the fork-evolution boundary (when philosophy is assigned to an agent).
- Invokes `intervene` (3-stage: pre knowledge/skill/posture · during candidates/expression · post prohibition-block/sufficient-evidence/tone) on the ask path.
- Invokes `detectTensions` (Profile · Knowledge · State ↔ constitution tension detection) on fork evolution → so the evolution of the other 3 axes cannot drift outside the constitution.
- That is, "the company rule blocks or corrects at the moment the coder works" operates within an agent turn. The engines (`intervention_engine` · `tension_detector`) exist, so **call-site connection** is the job of this contract.

### 3.1 Prohibition Enforcement Model — Deterministic First, Semantic Judgement = LLM seam

A headless rule engine cannot itself judge a **semantic** violation of a natural-language `statement` (e.g. "do not leak secrets") — that is the LLM's job. Therefore enforcement is honestly specified in 3 layers:

1. **deterministic (sound) — `Prohibition.forbiddenPatterns`**: the author/LLM declares concrete strings that "must never appear" → the engine blocks them definitively, case-insensitive. **The only sound path that actually blocks via the `checkProhibitions` · post-gen `intervene` gate.** (mcp_bundle 0.4.4 `Prohibition.forbiddenPatterns` + mcp_philosophy 0.1.2 `_detectViolation`.)
2. **best-effort heuristic**: the engine's built-in 2-pattern (uncertainty↔confidence · limitation-concealment) — auxiliary only, powerless against general prohibitions.
3. **semantic judgement = LLM seam (unimplemented, explicit follow-up)**: a semantic violation of an arbitrary NL `statement` requires LLM judgement. Since `ask` already has an LLM, attaching a hard-prohibition LLM judge on that path is the future seat. **Until then, NL-only prohibitions (those without forbiddenPatterns) are structurally not blocked** — this contract states it, rather than failing silent-false.

⚠️ Past defect (discovered · corrected in 2026-06-23 dogfood): `_detectViolation` returned a silent `false` for every prohibition outside the 2-pattern → the `check`·§3 gate failed to block arbitrary prohibitions while reporting "blocked". Honesty restored by the forbiddenPatterns deterministic path + this 3-layer specification.

## 4. All 4 Layers Evolve — outcome → knowledge Loop (core)

An agent's work result feeds back into **all 4 axes**. It is not Philosophy alone that evolves — facts accumulation, skill refinement, profile divergence, and philosophy deepening all occur from outcome.

| Axis | Evolution form | Mechanism |
|---|---|---|
| **facts** | new fact accumulation · correction (supersede) | outcome → FactGraph write (evidence/version) |
| **skill** | refinement of procedure/company rule | outcome → skill improvement proposal |
| **profile** | per-agent divergence (two coders, different personalities) | OwnedFork evolution + growth counter (GrowthKind) |
| **philosophy** | constitution deepening | reinforcement (`FeedbackEvent` pattern analysis) |

- Common loop: `agentInvoked` / reviewer result → `FeedbackEvent` emit → the relevant axis's `EvolutionProposal` → **human-gate** (no auto-apply, self-design safety NFR).
- All axes **FactGraph-traced**: every evolution is left as timestamp · evidence · version · provenance, so "was this evolution correct" can be traced back (drift audit).
- Accumulation (facts loading · growth recording) already operates — this contract specifies the **4-axis feedback (outcome→feedback→proposal) call-site**.

## 5. Role Orchestration — core (`brain_kernel`)

Rule-based role handoff (worker → team lead → manager judgement chain).

- Manager `route` (→ targetAgent) + reviewer `review` (→ verdict) are **exposed as `bk.agent.*` tools** (`bk.agent.route`/`review`/the existing `ask`). The declarative agent→agent pipeline = expressed as a workflow `action` step's `config.tool: bk.agent.route`/`ask`/`review` — the engine (`FlowDefinitionWorkflow`) dispatches `action`/`api` steps via `toolDispatcher`→host `callTool`, so it **works without an additional enum**. A dedicated `agent` StepType is merely syntactic sugar to add to the mcp_bundle core (the general-purpose primitive already provides the capability) — under the no-core-modification principle it is not added.
- Roles = `worker` · `manager` · `reviewer` (runtime-enforced). The team lead (intermediate layer) is expressed via manager nesting or a role extension (made explicit upon revision of this contract).
- Handoff = not loose communication but a defined flow (planning→coder→QA→human). Not caller-driven — the engine receives the route decision and dispatches to the next agent.

## 6. Irreversible-Action Gate — core logic + host confirm callback

Irreversible actions such as git push · mail send · settlement · external publish are a **default gate** (not opt-in).

- **core**: `readOnly` / `destructive` classification on tools + a dispatch interceptor — if destructive, an **approval token** is required (blocked if absent). The policy · interceptor live in core (brain_kernel `HostToolRegistry` / dispatcher).
- **host**: injects the confirm UI callback — raises a confirmation dialog to a person and issues the approval token. core has no UI.
- Even an autonomous/scheduled task, if destructive, cannot run without the callback. (The advisory system-prompt wording is not enforcement — the interceptor is canonical.)

## 7. Host Responsibilities (This Is All)

1. boot the kernel at startup → **adopt the KnowledgeSystem** (no parallel system, the ops `_doBoot` pattern).
2. **inject the confirm callback** (§6).
3. (UI) visualization/navigation — tools are core, expression is host.

→ No re-implementing the org-runtime. The host does only the above three. The confirm callback · boot pattern are specified by a shared host recipe + this spec (isomorphic to the io_drivers model — copy-in).

## 8. Non-Goals

- Per-host re-implementation of the org-runtime (drift).
- Auto-applying evolution (human-gate mandatory).
- LLM-generated handoff interfaces (net-new design, outside this spec).

## 9. Current State — Cross-Verification (2026-06-23)

The engines/primitives exist and are unit-tested. What is missing is **not capability but the call-site (wiring)** — this spec specifies that call-site.

| Contract | Engine | Call-site connection |
|---|---|---|
| §2 4-axis composition | exists | ✅ **connected** (flowbrain_core `AgentRuntime.ask`, 2026-06-23 — facts→4 axes) |
| §3 Philosophy intervention | exists (intervene·tension) | ✅ **ask connected** (post-gen gate, opt-in) + **prohibition enforcement made sound** (2026-06-23 — `forbiddenPatterns` deterministic block verified, §3.1 3-layer model; past silent-false defect corrected) + **fork-evolution `detectTensions` connected** (`AgentForkTensionDetectedEvent` emit, advisory). NL-semantic judgement is an LLM seam (follow-up) |
| §4 4-axis evolution loop | facts accumulation · profile growth · philosophy reinforcement exist | ✅ **all axes connected** (2026-06-23): philosophy=review→FeedbackEvent→proposeFeedback (human-gate) · skill=deficient verdict→`trackVariation` (`skillCandidateCount` accumulator, human-gated promotion) · facts accumulation + profile growth already operate |
| §5 orchestration | exists (route·review) | ✅ **`bk.agent.route`/`review` tools exposed** (2026-06-23). **declarative agent→agent pipeline=exists**: expressed as a workflow `action` step's `config.tool: bk.agent.route`/`ask`/`review` (the engine dispatches toolDispatcher→host callTool). The dedicated `agent` StepType enum is merely mcp_bundle core syntactic sugar — the general-purpose primitive (action→tool) provides the capability, so the core is unmodified |
| §6 destructive gate | — | ✅ **connected** (brain_kernel `HostToolRegistry`: `registerExposed(destructive:)` + `confirmDestructive` callback, deny-by-default, 2026-06-23). The host injects the callback |

multi-agent · roles · 4-axis fork · FactGraph (evidence/version/trace) · Profile divergence (OwnedFork) · skill · behavior · Studio Process+gate · 2-tier seeding = real operating behavior.

---

Related: `10-agent-scoping.md` (per-agent capability scoping) · `06-tool-registry.md` (tool registration · capability) · `03-kernel-runtime.md` · the `flowbrain-server-knowledge-os` track (headless operational entity). Implementation seats = `flowbrain_core` (AgentRuntime · ForkEngine) · `brain_kernel` (workflow engine · HostToolRegistry · SystemPromptComposer) · shared host recipe (confirm/boot).
