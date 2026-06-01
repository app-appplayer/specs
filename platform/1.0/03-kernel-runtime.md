# 03 — Kernel Operational Logic

The kernel layer's responsibility = the framework's common operational logic. A single source for all instances (studio · player · FlowBrain · arbitrary host). **headless** — 0 UI dependency.

Implementation source of truth = the `brain_kernel` package. This spec is the framework-level operational logic contract.

## Core Components

### KernelApp — Single Boot Entry Point

Every host boots the operational layer with a single `KernelApp.boot(...)` call. Input = host-determined arguments:

```dart
final app = await KernelApp.boot(
  workspaceId: '...',
  kvStorage: ...,             // KvStoragePort implementation
  llmProviders: {...},        // modelId → LlmPortAdapter pool
  fallbackLlm: ...,           // optional — default LlmPort when provider unspecified
  opsRuntime: ...,            // optional — OpsRuntime
  config: ConfigPort,         // 4 port 1
  uiResource: UiResourcePort, // 4 port 2
  observability: ObservabilityPort, // 4 port 3
  bundleSource: BundleSourcePort,   // 4 port 4
  serverHostFactory: ...,     // (optional) MCP server-backed endpoint factory
  clientHost: ...,            // (optional) outbound MCP client
  chatLogDir: '...',
  bundleRegistryStorageDir: '...',
);
```

boot result:
- `KnowledgeSystem` (8 facades)
- `AgentLlmSessions` (pool keyed by modelId)
- `BundleActivationRegistry` (process singleton)
- 4 ports active
- endpoint pool (wired when the host calls addEndpoint)

### Knowledge Asset Surface — Two-Dimension Framing

**Object dimension** (`KnowledgeSystem` getters):

| Object | Kind | Responsibility |
|---|---|---|
| `facts` | FactFacade | SVO triple write / query / delete / extract |
| `skill` | SkillFacade | procedure (Procedure · Step · Action) execute |
| `profile` | ProfileFacade | profile (Profile · sections · capabilities) register / list / get |
| `ops` | OpsFacade | unified facade over workflow / pipeline / runbook |
| `agents` | AgentFacade | agent (id · role · model · systemPrompt) createAgent / ask / list / delete |
| `ethosStore` | EthosStorePort (port) | values (EthosRecord · Philosophy) put / get / activate |

= **5 facades + 1 ethosStore port** + optional runtimes (`skillRuntime` · `opsRuntime`).

**URI dimension** (kb:// 8 categories — `KbFacade` enum):

```
kb://fact/<id>      kb://skill/<id>      kb://profile/<id>     kb://philosophy/<id>
kb://workflow/<id>  kb://pipeline/<id>   kb://runbook/<id>     kb://agent/<id>
```

= the 3 sub-domains of ops (workflow · pipeline · runbook) are separated at the URI level — an external client MAY reference each sub-domain. The mapping between the object-dimension `ops` (unified facade) and the URI-dimension 3 sub-domains = handled automatically by `bridge.readResource`.

> Architecture: the workflow / pipeline / runbook 3-way split = fragments of a single **"behavior definition"** primitive (the state+step+outcome universal engine, §02-4), being unified into one platform-owned category. The URI 3-sub surface is retained for compatibility, but execution is a single engine. flow=ephemeral / runbook=durable profiles.

The exact API of each facade / port = the spec of `flowbrain_core`.

### 4 Axis — Profile · Skill · Fact · Philosophy

The 4 axis combine at the moment an agent's system prompt is composed:

```
SystemPromptComposer.compose({
  agentSystemPrompt,
  snapshot: FourAxisSnapshot(
    profiles: [...] (priority desc),
    skills:   [...] (bullet),
    facts:    [...] (truncated),
    philosophies: [...] (statement + rationale),
  ),
})
→ final system prompt
```

= the agent automatically holds its own 4 axis assets. The host only wires.

### standardTools — bk.<facade>.<verb> 45 Tools

`standardTools(KernelApp app)` returns a standard wrapper per facade:

| facade | tools |
|---|---|
| fact (9) | `bk.fact.write` · `query` · `get` · `delete` · `extract` · `candidates.list` · `candidates.confirm` · `candidates.reject` · `entity.get` |
| skill (3) | `bk.skill.list` · `get` · `execute` |
| profile (4) | `bk.profile.register` · `unregister` · `list` · `get` |
| philosophy (6) | `bk.philosophy.put` · `list` · `get` · `activate` · `get_active_id` · `check` (prohibition check) |
| ops (10) | `bk.workflow.run` · `list` · `get_run` · `bk.pipeline.run` · `get_run` · `bk.runbook.run` · `list` · `bk.behavior.run` · `resume` (+statePatch) · `list` (behavior definition engine) |
| agent (11) | `bk.agent.list` · `get` · `ask` · `create` · `delete` · `history` · `assign_skill` · `assign_profile` · `assign_philosophy` · `assign_facts` · `materialize` |
| knowledge (2) | `bk.knowledge.query` · `test` |

- typedef `InProcessToolHandler = Future<Object?> Function(Map<String, dynamic>)`
- `wrapInProcess(handler)` → `KernelToolHandler` (envelope)
- `scopeIdFor(localId)` applied automatically (master = pass-through · domain = `<bundleId>.<localId>` prefix)
- Error envelope = `stdErr(msg)` → `{ok: false, error: msg}`

### Three Tool Registration Paths

A core framework contract — see `06-tool-registry.md`. Summary:

| path | use |
|---|---|
| `KernelEndpoint.addStandardTools(app)` | host home context — host uses the standard 45 within the same process (placeholder schema) |
| `BundleSessionBridge.registerTool` | domain wrapping of knowledge tools — `bk.` enforced · alias auto-published |
| `HostToolRegistry.registerExposed` | general tools (host + domain logic) — `<bundleId>.<rawName>` prefix automatic |

### MCP Prompts Surface — `KernelServerHost.addPrompt`

The framework provides a host-side wrapper for MCP `prompts/list` · `prompts/get`. A builtin / bundle app uses the prompt registration path without directly importing `package:mcp_server` — aligned with the "OS api" framing (`vibe_studio/builtin_api.dart` · wiring through a host wrapper layer).

```dart
host.addPrompt(
  name: 'drive_workspace',
  description: 'Prompt preset for performing work in a specific workspace.',
  arguments: <KernelPromptArgument>[
    KernelPromptArgument(name: 'workspaceId', description: '...', required: true),
    KernelPromptArgument(name: 'goal', description: '...', required: true),
  ],
  handler: (args) async => KernelGetPromptResult(
    description: 'workspace operation brief',
    messages: <KernelPromptMessage>[
      KernelPromptMessage(
        role: 'user',
        content: KernelTextContent(text: '...'),
      ),
    ],
  ),
);
```

- wrapper symbols = `KernelPromptArgument` · `KernelPromptMessage` · `KernelGetPromptResult` · `KernelPromptHandler` typedef · `KernelPromptDef`
- `removePrompt(name)` · `promptDefinitions` getter
- Two reference impls: `InProcessKernelServerHost` (in-memory map) · `ServerBootstrap` (mcp_server forward + `KernelGetPromptResult` ↔ `mcp.GetPromptResult` conversion)

### BundleSessionBridge — 4 Standard Channels

- `callTool(session, name, args)` — tool dispatch + automatic scopeId
- `readResource('kb://<facade>/<id>')` — read assets of the 8 facades + automatic scopeId
- `listTools()` / `listResources()` — catalog
- `attach(session, SessionHandle)` — binds the UI mount / subscription lifecycle

session = master · non-master. non-master = the alias publication path of bundle activation.

Detail = `07-knowledge-access.md`.

### BundleActivation — Automatic Registration of 7 Categories

`BundleActivation.activate(McpBundle bundle)` registers the manifest's 7 asset categories into the KnowledgeSystem in bulk + a `<bundleId>.<rawId>` namespace prefix:

| Category | Manifest location | Wiring |
|---|---|---|
| skills | `bundle.skills.modules[]` | SkillRuntime · `registerSkill` |
| profiles | `bundle.profiles.profiles[]` | ProfileFacade · `register` |
| philosophy | `bundle.philosophy.philosophies[]` | EthosStorePort · `putEthos` |
| facts | `bundle.facts.facts[]` | FactFacade · `writeFacts` |
| **behavior definition (behavior)** | `bundle.behavior.definitions[]` | OpsRuntime · `behaviorRegistry` (state+step+outcome single engine · §02-4). Executed via `bk.behavior.run`/`resume` |
| flow (legacy) | `bundle.flow.flows[]` | OpsRuntime · `workflowRegistry` (profile to be absorbed into the engine) |
| agents | `bundle.agents.agents[]` | AgentFacade · `createAgent` |

**Manifest categories NOT auto-handled by BundleActivation** (host domain):

- `manifest.tools.tools[]` — the host calls `HostToolRegistry.registerExposed` directly (composing the domain tool logic handler is also the host's responsibility)
- Operational logic wiring = single-engine registration (table above). Implementation status: `bundle.behavior` (unified behavior definition) + `bundle.flow` (legacy) are auto-registered. Absorbing the legacy `flow`/`runbook` executors into engine profiles is an incremental, non-forced cleanup that preserves the compatibility surface.
- `manifest.ui` — the host UI runtime mounts it
- `manifest.wiring` — the host chrome wires it
- `manifest.settings.sections[]` — the host settings dialog wires it
- `manifest.chat.slashCommands[]` — the host chat panel wires it
- `manifest.assets / knowledge / bindings / tests / policies / factGraphSection` — host domain (free use or separate processing path)

### 4 Ports — Host Wiring

| port | Responsibility | default |
|---|---|---|
| `ConfigPort` | config load · watch · patch | `NullConfig.instance` |
| `UiResourcePort` | UI resource list · read · register · events | `NullUiResource.instance` |
| `ObservabilityPort` | event · cost · metric · stream | `NullObservability.instance` |
| `BundleSourcePort` | bundle fetch · list | `InMemoryBundleSource()` |

The host implements only the ports it needs · unused ports = `Null*` default.

### LLM Pool — AgentLlmSessions

- `LlmPortAdapter` map keyed by modelId
- 1:1 mapping with the agent's `ModelSpec.id`
- No global single ChatSession (per-agent isolation)
- On key/model/endpoint change, only the affected agent's adapter is rebuilt
- **No automatic fallback** — the `boot(fallbackLlm: ...)` argument is not auto-absorbed (when the user does not specify → throw + UI banner). Aligned with [`feedback_no_fallback_anywhere`]. A fallback hides debugging (the user does not know which provider is actually operating)

`LlmPortAdapter`:
- default ctor — ClaudeProvider automatic
- `.fromInterface(modelId, provider)` — injects an arbitrary `mcp_llm.LlmProvider` (9 official + CustomLlmProvider extension)
- `providerName` getter — the explicit name of the active provider (e.g. `'anthropic'` · `'openai'` · `'claude_code'`). For wiring UI banner / chat header / observability events.

### Agent → Tool Catalog Wiring (per-agent scoping)

A core framework wiring — each agent receives only its own tool · knowledge catalog. Canonical specification = [`10-agent-scoping.md`](10-agent-scoping.md). Source-of-truth path:

```
KernelEndpoint catalog (the union of the 3 registration paths)
  ↓ KernelApp.toolsForAgent(agentId, {role, explicitAllowlist?, bundleId?})
List<LlmTool>
  ↓
AgentChatController(tools: ...)                       ← supported by default in brain_kernel
AgentFacade.ask(agentId, message, tools: ...)         ← supported by default in flowbrain_core (`agent_facade.dart` line 251-258)
  ↓
LlmRequest.parameters['tools'] = [...]                ← mcp_llm standard
  ↓ natural branch per provider
  ├── Mode B (anthropic / openai / gemini) — catalog forward → tool_use response → host loop
  └── Mode A (claude_code) — catalog ignored + --mcp-config wiring (host MCP server absorbed)
```

#### `KernelApp.toolsForAgent` Contract

```dart
List<LlmTool> toolsForAgent(
  String agentId, {
  required AgentRole role,
  List<String>? explicitAllowlist,  // manifest agents[i].tools (glob)
  String? bundleId,                  // scope of worker / specialist
});
```

Per-role default subset · anti-patterns · manifest wiring = [`10-agent-scoping.md`](10-agent-scoping.md). This spec covers only the helper's signature.

#### `KernelApp.hostMcpServerSpec` Contract

```dart
McpServerSpec? hostMcpServerSpec({String? endpointLabel});
```

- `endpointLabel` specified → returns only that endpoint's `server.externalSpec` (null if absent or in-process)
- `endpointLabel` unspecified → returns the spec of the first transport-binding endpoint. **Safe only for a single-endpoint host (AppPlayer / headless / probe default).**
- All endpoints in-process = null

**A multi-endpoint host (vibe_studio's multi-domain pool · a user-arbitrary host's multi-transport wiring)** = MUST specify `endpointLabel`. Using the first-found spec = a silent choice = risks violating alignment with [`feedback_no_fallback_anywhere`].

= the `--mcp-config` automatic wiring path of the Claude Code recipe. The `ClaudeCodeProvider.forKernel(app, endpointLabel: ...)` factory absorbs it automatically + is multi-endpoint safe.

### Host adapters — mcp Dependency Abstraction

| Abstraction | Responsibility | reference impl |
|---|---|---|
| `KernelServerHost` | server-side endpoint (addTool · removeTool · addResource · removeResource · **addPrompt · removePrompt · promptDefinitions** · callTool · start · shutdown · dispatchLog · externalSpec · etc.) | `ServerBootstrap` (`package:brain_kernel/mcp_host.dart`) |
| `KernelClientHost` | outbound MCP client (connect · ...) | `McpClientKernelHost` |
| `InProcessKernelServerHost` | 0 mcp-dependency default | brain_kernel itself |

= the host freely chooses its own transport (mcp_server · mcp_client · USB · IPC · custom).

## 4 Per-Host Composition Modes

| host | serverHostFactory | clientHost |
|---|---|---|
| vibe_studio | `ServerBootstrap.factory` | `McpClientKernelHost()` |
| AppPlayer (client default) | null (in-process) | `McpClientKernelHost()` |
| headless probe / test | null | null |
| arbitrary host (USB / IPC) | own KernelServerHost impl | own KernelClientHost impl |

Detail = `05-composition-patterns.md`.

## Non-Goals

- This spec = **framework-level contract**. All detail (method sig · class signature · test) = the `brain_kernel` package.
- 4 axis prompt composition detail = `flowbrain_core` spec.
- Per-provider LLM wire format = `mcp_llm` spec.
