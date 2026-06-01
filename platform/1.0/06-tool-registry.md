# 06 — Tool Registration Paths

A core contract of the framework. Three registration paths · two naming models · code-level blocking (throw on violation).

## Three Registration Paths

| Tool kind | Registration path | Externally exposed name | Position |
|---|---|---|---|
| **kernel standard knowledge tools** (`bk.<facade>.<verb>` 41) — host home context | `KernelEndpoint.addStandardTools(app)` | `bk.<facade>.<verb>` (canonical as-is · master) | simple path · own tools within the same process |
| **domain access to knowledge tools** (a domain uses the standard 41 or its own facade within its own context) | `BundleSessionBridge.registerTool` (**name MUST start with `bk.`**) | `bk.<bundleId>.<rest>` (alias auto-published · external endpoint) | wrapping path · per-domain isolation + scopeId applied |
| **general tools** (host's own logic · a bundle's `manifest.tools.tools[]`) | `HostToolRegistry.registerExposed` | `<bundleId>.<rawName>` (prefix auto · dispatcher + endpoint simultaneously) | utility · per-bundle collision avoidance |

## Two Naming Models (knowledge tools only)

Two bundles may use the same tool name (e.g. `bk.fact.write`) at the same time → isolation is required. The bridge auto-maps two names:

| Calling site | Name |
|---|---|
| Within own bundle (script · own context) | **standard name** `bk.<facade>.<verb>` (scopeId auto — dispatched into the bundleId namespace) |
| From outside (another bundle · CLI subprocess · external MCP client) | **bundleId-prefixed name** `bk.<bundleId>.<rest>` (bridge alias) |

= **own tools are called simply by their own name · external calls carry an explicit bundleId for isolation**.

**General tools** (host + domain logic) have no per-bundle name collision → no two-name model needed. `HostToolRegistry` auto-prefixes a single name `<bundleId>.<rawName>`.

## Code-Level Blocking — Prevent Incorrect Usage

### Validator of bridge.registerTool

```dart
void registerTool({required String name, ...}) {
  if (!name.startsWith(_aliasablePrefix /* 'bk.' */)) {
    throw ArgumentError(
      'BundleSessionBridge.registerTool only accepts knowledge tools '
      '(name must start with "bk."). General domain or host tools go '
      'through HostToolRegistry or direct dispatcher + endpoint wiring.',
    );
  }
  ...
}
```

= attempting to register a general tool via the bridge throws ArgumentError. Violations of the principle are blocked automatically.

### Scope of `addStandardTools`

The inputSchema of `KernelEndpoint.addStandardTools(app)` = placeholder `{type: object, additionalProperties: true}`. **Intent** = host home context (no schema validation needed when the host calls its own tools within its own process). Not an external exposure path.

→ When an external client (CLI subprocess · another host) calls, it MUST go through the bridge or HostToolRegistry (strict schema · alias).

### Prefix Enforcement of HostToolRegistry

`HostToolRegistry.registerExposed(bundleId, rawName, ...)` = exposedName `<bundleId>.<rawName>` automatic. The host MUST NOT add an explicit prefix.

= two bundles registering the same raw name (`editor.open`) → exposedName `recipe_a.editor.open` · `recipe_b.editor.open` auto-isolated.

## Selection Criteria (host responsibility)

| Use case | path |
|---|---|
| host uses its own standard 45 tools within its own process (host home) | `addStandardTools(app)` |
| a domain (bundle script) calls the standard 41 within its own context + exposes a bundleId-isolated alias on the external endpoint | `BundleSessionBridge.registerTool` (name = `bk.<facade>.<verb>` enforced) |
| host's own tools · a bundle's domain logic tools (manifest.tools.tools[]) | `HostToolRegistry.registerExposed` |

## HostToolRegistry — Signature

The signature = the **standard utility** for general tool registration:

```dart
class HostToolRegistry {
  HostToolRegistry({
    required this.endpoint,                  // KernelServerHost
    required this.attachToDispatcher,        // attach to host's in-process dispatcher
    required this.detachFromDispatcher,
  });

  String registerExposed({
    required String bundleId,
    required String rawName,
    required String description,
    required KernelToolHandler handler,
    Map<String, dynamic>? inputSchema,
  });
  // → returns exposedName '<bundleId>.<rawName>'
  // → registers at both the dispatcher + endpoint simultaneously

  String? unregisterExposed({
    required String bundleId,
    required String rawName,
  });
}
```

- `endpoint` = `KernelServerHost` (various implementations — InProcessKernelServerHost · ServerBootstrap · USB · user-defined)
- `attachToDispatcher` / `detachFromDispatcher` = host's dispatcher wiring callbacks (AppPlayer's `ToolDispatcher` · vibe_studio's runtime toolExecutor · headless in-memory map, etc.)
- prefix auto = per-bundleId isolation
- dispatcher + endpoint simultaneous registration = registration at two positions automatically

## BundleSessionBridge.registerTool — Signature

```dart
void registerTool({
  required String name,                      // 'bk.' enforced
  required BridgeToolHandler handler,
  String? description,
  Map<String, dynamic>? inputSchema,
});
// → bridge._tools[name] = def (script layer · own-context dispatch source)
// → publishes an alias per active non-master session (serverAdapter callback · external endpoint mirror)
```

- name = `bk.<facade>.<verb>` or `bk.<domain>.<verb>` form
- alias name = `bk.<bundleId>.<rest>` (`rest` = the portion of canonical after `bk.`)
- inputSchema preserved — the host-specified strict schema is identical on the alias
- for a master session = alias publication skipped (host home context · canonical exposed as-is)

## Behavioral Flows

### path 1 — addStandardTools

```
at host boot:
  app = KernelApp.boot(...)
  ep = app.addEndpoint(label: 'main')
  ep.addStandardTools(app)
    → 45 tools registered on the endpoint with canonical names (bk.fact.write etc.)
    → callable within the host's same process
```

### path 2 — BundleSessionBridge.registerTool

```
at host boot:
  bridge = BundleSessionBridge(
    systemResolver: () => app.system,
    serverAdapter: (def) => ep.addTool(...)       # external endpoint mirror
    serverAdapterRemove: ep.removeTool,
  )

at bundle activation:
  activation = BundleActivation(system: app.system, bundleId: 'recipe_demo')
  session = bridge.openSession(activation)   # non-master

(optional) wrapping of standardTools:
  for entry in standardTools(app).entries:
    bridge.registerTool(
      name: entry.key,                       # 'bk.fact.write' etc.
      handler: wrapInProcess(entry.value),
      inputSchema: <strict>,
    )
    → bridge._tools['bk.fact.write'] = def   (script layer)
    → 'bk.recipe_demo.fact.write' auto-published on the endpoint (alias)

domain script call:
  await bridge.callTool(session, 'bk.fact.write', args)
    → scopeId auto (namespace within bundleId 'recipe_demo')
    → executes the standardTools handler

external LLM call:
  → endpoint's 'bk.recipe_demo.fact.write' (alias)
  → bridge alias handler → callTool(session, 'bk.fact.write', args)
  → same dispatch path
```

### path 3 — HostToolRegistry.registerExposed

```
at host boot:
  registry = HostToolRegistry(
    endpoint: ep.server,
    attachToDispatcher: (name, handler) => dispatcher.register(name, handler),
    detachFromDispatcher: dispatcher.unregister,
  )

at bundle activation (per manifest.tools.tools[]):
  for tool in bundle.manifest.tools.tools:
    handler = _buildHandlerForToolEntry(tool)
    registry.registerExposed(
      bundleId: bundle.id,
      rawName: tool.name,               # 'editor.open' etc.
      description: tool.description,
      inputSchema: tool.inputSchema,
      handler: handler,
    )
    → registers '<bundle.id>.editor.open' on the dispatcher
    → registers '<bundle.id>.editor.open' on the endpoint
    → no collision even if two bundles use the same raw name

domain script call:
  → dispatcher.callInProcess('<bundle.id>.editor.open', args)

external LLM call:
  → endpoint's '<bundle.id>.editor.open'
  → executes the same handler
```

## Tool → Agent Catalog Wiring

The framework's source-of-truth wiring:

```
KernelEndpoint catalog (sum of registration path 3)
  ↓ KernelApp.toolsForAgent(agentId, {role})         ← helper
List<LlmTool> (per-role · per-bundle subset)
  ↓
AgentChatController(tools: ...) / AgentFacade.ask(...tools: ...)
  ↓
LlmRequest.parameters['tools'] (mcp_llm standard)
  ↓ natural branching per provider
  ├── mode B (anthropic / openai / gemini) — catalog forward → tool_use response → host loop
  └── mode A (claude_code) — catalog ignored + --mcp-config wiring (host MCP server absorbed)
```

### `KernelApp.toolsForAgent` Contract

```dart
List<LlmTool> toolsForAgent(
  String agentId, {
  required AgentRole role,
  List<String>? explicitAllowlist,  // manifest agents[i].tools
  String? bundleId,                  // scope of worker / specialist
});
```

**Per-role default subset** (flowbrain_core `AgentRole` 3 kinds — framework default provided):

| role | default subset | session |
|---|---|---|
| `manager` | framework `bk.*` delegation (`bk.agent.*`) + read (`bk.*.query/get/list` · `bk.workflow/pipeline/runbook` status). **mutation excluded · host-specific namespace (`studio.*` · `app.*` · arbitrary) also excluded** (kernel does not hard-code host's prefix · widened only by explicit manifest allowlist or `explicitAllowlist`) | master |
| `worker` | own `<bundleId>.*` domain + `bk.<bundleId>.*` alias | non-master |
| `reviewer` | master's review-friendly subset (read tools + agent.history) | master |

When `explicitAllowlist` is specified = override the default subset (glob patterns supported: `bk.fact.*` · `<bundle>.editor.*`). Narrow capabilities such as specialist are specified by allowlist alone, without a separate enum.

Detailed principle + anti-patterns = [`10-agent-scoping.md`](10-agent-scoping.md).

## Per-Host Implementation Matrix

| Host | dispatcher implementation | endpoint implementation |
|---|---|---|
| **vibe_studio** | runtime toolExecutor (or own map) | `ServerBootstrap` (server-backed) |
| **AppPlayer** | `ToolDispatcher` (`runtime/tool_dispatcher.dart`) | `InProcessKernelServerHost` (client default) or `ServerBootstrap` (Pro's server option) |
| **Headless CLI** | own in-memory map | `InProcessKernelServerHost` or `ServerBootstrap` |
| **Arbitrary host** | free (callback path) | free (KernelServerHost abstract implementation) |

## Non-Goals

- Logic definition of a tool handler = bundle / host domain
- Canonical specification of an agent's catalog scope principle = [`10-agent-scoping.md`](10-agent-scoping.md)
- This spec = **the interface of the three registration paths + the two-name model + code-level blocking + agent catalog wiring helper** only
