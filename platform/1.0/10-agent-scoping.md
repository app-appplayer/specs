# 10 — Per-Agent Capability Scoping

This spec = the **true benefit of introducing a kernel** into the framework. The wiring contract by which each agent receives only its own tool / knowledge / asset catalog. Thesis:

> Each agent carries its own tools and knowledge separately — if the entire knowledge and toolset is exposed to an agent, the result is no different from a single entity doing everything without agents. Agent specialization + efficiency + token-usage optimization + performance improvement.

= per-agent capability scoping = the framework's raison d'être.

## The True Benefit of Introducing a Kernel

| Value | Without scoping | With scoping |
|---|---|---|
| **Specialization** | every agent = one jack-of-all-trades LLM = separation is meaningless | researcher = knowledge tools only / coder = editing tools only / manager = delegation tools only |
| **Token efficiency** | full catalog · full 4-axis on every call = token explosion | only own assets composed into the prompt = 1/N cost |
| **Performance (LLM accuracy)** | select from 100 catalog entries → higher LLM confusion | catalog of 5 = accurate selection |
| **Context isolation** | one agent's assets become noise in another agent's decisions | isolation per bundle / role |
| **Security / permissions** | every agent = root | per-role capability subset (worker cannot call host.*) |
| **multi-agent orchestration** | manager directly performs worker roles = delegation is meaningless | manager holds delegation tools only → forced delegation |

= without a kernel, every agent absorbs the host's entire asset set = monolithic LLM. Only with a kernel are per-agent catalog scope + role-based subset + token budget guaranteed = **true multi-agent**.

## 4-Dimensional Scoping

The assets each agent receives = a subset across 4 dimensions:

| Dimension | Assets | Scoping decision |
|---|---|---|
| **tools** | subset of the host KernelEndpoint catalog | per-role · per-bundle · manifest allowlist |
| **4 axis** (skills / profiles / facts / philosophies) | own bundle + cross-bundle inherit set | BundleActivation's scopeId + manifest inherit |
| **knowledge resources** (kb:// URI) | read access per facade · scope | bridge's scopeId + session mapping |
| **session** (master / non-master) | master = whole host · non-master = bundle scope | manager role = master / worker role = non-master |

= **all 4 dimensions are the framework's responsibility**. Applied automatically by manifest + KernelApp-helper wiring alone.

## Per-Role Default Subset

Per the agent's `role` (flowbrain_core `AgentRole` enum — `worker` · `manager` · `reviewer`), the default catalog:

| role | tool subset | session | intent |
|---|---|---|---|
| `manager` | **framework `bk.*` delegation + read subset only** (`bk.agent.*` delegation + `bk.fact/skill/profile/philosophy/knowledge.query/get/list` read + `bk.workflow/pipeline/runbook` status). **mutation excluded · host-specific tools (`studio.*`, `app.*`, arbitrary) also excluded** — host's free namespace is not hard-coded by the kernel. The host explicitly widens via manifest `agents[i].tools` or `explicitAllowlist` | master | sibling delegation + knowledge read only · prevents monolithic LLM · guarantees framework universality |
| `worker` | the non-master session subset of its own bundle (`bk.<bundleId>.*` alias + `<bundleId>.*` domain) | non-master | its own scope only |
| `reviewer` | the review-friendly subset of master (knowledge read + agent.history) + no delegation | master | reviews other agents' outputs |

= role decision = host-side wiring. The framework provides the default subset + the manifest's `agents[i].tools` allowlist overrides.

When a narrow capability such as a specialist is needed = **specify the manifest's `tools` allowlist** (overriding the role's default). Do not introduce a separate enum.

## `LlmTool` Wiring Path (framework source of truth)

The framework's tool → agent-prompt wiring path = mcp_llm standard + flowbrain_core wrapping:

```
KernelEndpoint catalog (KernelToolHandler list)
  ↓ KernelApp.toolsForAgent(agentId, {role})        ← new helper
List<LlmTool>
  ↓ AgentChatController(tools: ...)                 ← brain_kernel supported by default
List<LlmTool>?
  ↓ AgentFacade.ask(agentId, message, tools: ...)   ← flowbrain_core supported by default
LlmRequest.parameters['tools'] = [...]              ← mcp_llm standard
  ↓
LlmProvider.complete(request)
  ├── anthropic / openai / gemini = catalog forward → tool_use response → host loop
  └── claude_code = ignores catalog by design + absorbs the host MCP server from --mcp-config (mode A)
```

= flowbrain_core's `AgentFacade.ask(..., tools)` + brain_kernel's `AgentChatController(tools)` = **already supported by the framework**. The missing piece = the host-side `KernelApp.toolsForAgent` helper.

## Natural Branching by Provider — Mode A vs Mode B

Per the provider's design constraints, two modes branch naturally. Zero host-code change:

| provider | mode | tool-catalog wiring | host standardManagerSend loop |
|---|---|---|---|
| anthropic / openai / gemini | **mode B** | inject `LlmRequest.parameters['tools']` → LLM responds with tool_use → host dispatch | multi iteration |
| claude_code | **mode A** | host MCP server spec → CLI integration (A.1 or A.2) | natural termination at first iteration (`reply.toolCalls` always null) |

= both modes are naturally absorbed by the standardManagerSend loop. Consistent with the framing "not two separate things — both must work."

### The Two Sub-Modes of Mode A

A CLI-process-based provider such as Claude Code has its own tool-use protocol. It does not forward the host's mcp_llm-standard `LlmRequest.parameters['tools']` natively. Hence two sub-modes branch naturally:

#### Mode A.1 — `--mcp-config` wiring (real dispatch)

- the provider automatically forwards the host MCP server endpoint spec (`KernelApp.hostMcpServerSpec`) as `--mcp-config <json>`
- Claude Code's own MCP client connects to the server → absorbs the tools catalog → invokes them itself
- = real dispatch (the CLI calls the host endpoint) — the bridge handler receives args directly
- Constraint: the endpoint requires a binding over streamable HTTP / SSE / stdio transport. File-system / CLI-registration race possible.

#### Mode A.2 — prompt-prepended catalog (catalog awareness)

- when the provider receives `LlmRequest.parameters['tools']` → it prepends a prose catalog to the system prompt ("Available tools (caller-supplied catalog): - name: description, Parameters: {...}")
- (optional) the host MCP server URL is also stated in the same prose ("Connected MCP servers: - name (http): url")
- the CLI's in-process LLM is aware of the in-prompt catalog + uses it in answers (e.g. for the question "what agents exist?", it accurately names the sibling agents in the catalog)
- = catalog awareness only · no real host dispatch (`reply.toolCalls` null) · however **manager's inventory answer + sibling awareness + role consistency are all satisfied by mode A.2**
- consistent with the mcp_llm-standard `LlmRequest.tools` path — same interface as existing providers. The provider wires within its design intent (no separate module / package introduced).

### The Two Sub-Modes Are Orthogonal

- mode A.1 only (URL spec exposed · no tools arg) = real dispatch possible, no prompt catalog
- mode A.2 only (tools arg + optional URL) = catalog awareness, no dispatch
- mode A.1 + A.2 together (URL spec + tools arg) = both effects (the CLI calls via mcp-config + the LLM is aware of the prompt catalog)

= the provider decides the two sub-modes. The host only passes `KernelApp.toolsForAgent` (mcp_llm-standard path) + `hostMcpServerSpec` (mode A.1 option).

### Scoping Wiring of Mode A

Both of Claude Code's sub-modes **can themselves be subset per agent**. The session-based catalog (master / non-master) mapping of KernelEndpoint applies as-is:

- manager agent call = master session catalog (mode A.2 prompt) + master endpoint URL (mode A.1)
- worker agent call = its own bundle's non-master session catalog (mode A.2 prompt) + its own endpoint URL (mode A.1)

= the host can expose a different endpoint URL / auth per agent role → a `KernelApp.hostMcpServerSpecForAgent(agentId, {role})` form is possible (an extension point).

= **the agent-scoping thesis applies consistently across all modes**.

## Manifest Wiring

When the bundle manifest's `agents[i]` specifies a `tools` allowlist = applied automatically:

```json
{
  "agents": [
    {
      "id": "researcher",
      "role": "worker",
      "tools": ["bk.fact.*", "bk.skill.execute", "<bundleId>.editor.open"],
      "systemPrompt": "..."
    }
  ]
}
```

- pattern = glob (`bk.*` · `bk.fact.*` · `<bundleId>.editor.*`, etc.)
- empty / unspecified allowlist = the role's default subset applies
- explicit list = subset enforced (independent of the role default)

= reinforces the `agents` category of `02-bundle-interface.md`.

## Anti-Patterns (Forbidden)

Host-side ad-hoc overrides outside framework-automatic / manifest wiring = forbidden. Cases that violate the thesis canon:

- ❌ **forcing the full catalog onto the manager** — the "manager must see everything" rationale. Equivalent to monolithic LLM. Token explosion + LLM confusion + thesis violation. The framework's initial wiring applied this very anti-pattern and was corrected on 2026-05-27 — `KernelApp.toolsForAgent`'s `manager` default is delegation + inventory + read subset only. Mutation tools = only when specified in `explicitAllowlist`
- ❌ **worker able to call manager tools** — breaks per-role isolation. Capability escalation
- ❌ **every agent on a master session** — defeats the isolation purpose of non-master sessions. Neutralizes BundleSessionBridge's scopeId wiring
- ❌ **fallback catalog** — auto full exposure when an agent profile's tools wiring is absent. Violates `feedback_no_fallback_anywhere`. Unspecified → throw + diagnosable

= when the above anti-patterns are found = immediately correct the wiring + spec audit.

## Consistent Wiring Patterns

| Decision | Wiring |
|---|---|
| agent aware of its siblings + delegation | include `bk.agent.list` · `bk.agent.ask` in the manager profile's tools + automatic sibling slot in `SystemPromptComposer` (a separate-round candidate) |
| agent aware of the host's builtin-app inventory | register a host-side `host.builtin.list` tool on KernelEndpoint + include it in the manager profile's tools |
| domain worker sees only its own bundle assets | role=worker + non-master session + manifest tools allowlist (own bundle prefix) |
| specialist sees only specialized tools (e.g. a code-review agent) | role=specialist + manifest explicit allowlist (`['bk.skill.execute', '<bundle>.lint.run']`, etc.) |
| Claude Code agent calls host tools | auto-inject `KernelApp.hostMcpServerSpec` → `--mcp-config` wiring → the CLI integrates its own catalog |

## Compatibility Wiring — Existing-Spec Cross-Refs

- `03-kernel-runtime.md` — `AgentFacade.ask`'s tools-arg path + `KernelApp.toolsForAgent` helper (Phase 3 reinforcement)
- `06-tool-registry.md` — per-agent-role catalog-decision algorithm (Phase 3 reinforcement)
- `02-bundle-interface.md` — the `agents[i].tools` allowlist contract of the manifest (Phase 3 reinforcement)
- `07-knowledge-access.md` — knowledge-resources scoping (already scopeId-wired)
- `05-composition-patterns.md` — agent scope auto-applied in multi-bundle / multi-server patterns

## Non-Goals

- This spec = **framework-level contract + thesis canon**. Implementation detail = the code spec of brain_kernel · flowbrain_core · recipes.
- Extending the agent-role enum = the domain of flowbrain_core `AgentRole` (outside the framework spec)
- The exact JSON shape of the manifest schema = the `mcp_bundle` spec
- Per-LLM-provider wire format = the `mcp_llm` spec
