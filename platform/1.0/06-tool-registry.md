# 06 — Tool Registration Paths

The framework's core contract. Three registration paths · two naming models · code-level guarding (throws on violation).

## Three Registration Paths

| Tool type | Registration path | External exposed name | Place |
|---|---|---|---|
| **kernel standard knowledge tools** (`bk.<facade>.<verb>` 41) — host home context | `KernelEndpoint.addStandardTools(app)` | `bk.<facade>.<verb>` (canonical as-is · master) | simple path · own tools within the same process |
| **domain access of knowledge tools** (a domain uses the standard 41 within its own context, or its own facade) | `BundleSessionBridge.registerTool` (**name forced to start with `bk.`**) | `bk.<bundleId>.<rest>` (alias auto-published · external endpoint) | wrapping path · per-domain isolation + scopeId applied |
| **general tools** (host's own logic · a bundle's `manifest.tools.tools[]`) | `HostToolRegistry.registerExposed` | `<bundleId>.<rawName>` (prefix automatic · dispatcher + endpoint simultaneously) | utility · per-bundle collision avoidance |

## Two Naming Models (knowledge tools only)

Knowledge tools have two bundles using the same tool name (`bk.fact.write`, etc.) simultaneously → isolation required. The bridge auto-maps two names:

| Calling place | Name |
|---|---|
| within its own bundle (script · own context) | **standard name** `bk.<facade>.<verb>` (scopeId automatic — dispatched in the bundleId namespace) |
| from outside (another bundle · CLI subprocess · external MCP client) | **bundleId-prefixed name** `bk.<bundleId>.<rest>` (bridge alias) |

= **own tools are called simply by their own name · external calls are isolated with bundleId made explicit**.

**General tools** (host + domain logic) = no same name across bundles → the two-naming model is unnecessary. `HostToolRegistry` auto-prefixes a single `<bundleId>.<rawName>` name.

## Code-Level Guarding — Preventing Misuse

### The validator of bridge.registerTool

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

= attempting to register a general tool through the bridge throws ArgumentError. Canon violation is auto-blocked.

### The scope of `addStandardTools`

The inputSchema of `KernelEndpoint.addStandardTools(app)` = placeholder `{type: object, additionalProperties: true}`. **Intent** = host home context (when the host calls its own tools within its own process, schema validation is unnecessary). Not an external-exposure path.

→ when an external client (CLI subprocess · another host) calls = it must go through the bridge or HostToolRegistry (strict schema · alias).

### The prefix enforcement of HostToolRegistry

`HostToolRegistry.registerExposed(bundleId, rawName, ...)` = exposedName `<bundleId>.<rawName>` automatic. The host does not add an explicit prefix.

= two bundles register the same raw name (`editor.open`) → exposedNames `recipe_a.editor.open` · `recipe_b.editor.open` are auto-isolated.

## Selection Criteria (host responsibility)

| Use case | path |
|---|---|
| host uses its own standard 45 tools within its own process (host home) | `addStandardTools(app)` |
| a domain (bundle script) calls the standard 41 within its own context + exposes a bundleId-isolated alias on an external endpoint | `BundleSessionBridge.registerTool` (name = `bk.<facade>.<verb>` enforced) |
| host's own tools · a bundle's domain-logic tools (manifest.tools.tools[]) | `HostToolRegistry.registerExposed` |

## HostToolRegistry — Signature

The signature = the **standard utility** for general-tool registration:

```dart
class HostToolRegistry {
  HostToolRegistry({
    required this.endpoint,                  // KernelServerHost
    required this.attachToDispatcher,        // attach to the host's in-process dispatcher
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
  // → registered at both the dispatcher and the endpoint simultaneously

  String? unregisterExposed({
    required String bundleId,
    required String rawName,
  });
}
```

- `endpoint` = `KernelServerHost` (various implementations — InProcessKernelServerHost · ServerBootstrap · USB · user-arbitrary)
- `attachToDispatcher` / `detachFromDispatcher` = the host's dispatcher-wiring callbacks (AppPlayer's `ToolDispatcher` · vibe_studio's runtime toolExecutor · headless in-memory map, etc.)
- prefix automatic = per-bundleId isolation
- dispatcher + endpoint registered simultaneously = two-place registration automatic

## BundleSessionBridge.registerTool — Signature

```dart
void registerTool({
  required String name,                      // 'bk.' enforced
  required BridgeToolHandler handler,
  String? description,
  Map<String, dynamic>? inputSchema,
});
// → bridge._tools[name] = def (script layer · own-context dispatch source)
// → an alias published per active non-master session (serverAdapter callback · external endpoint mirror)
```

- name = `bk.<facade>.<verb>` or `bk.<domain>.<verb>` form
- alias name = `bk.<bundleId>.<rest>` (`rest` = the part after `bk.` in the canonical)
- inputSchema preserved — the host-specified strict schema applies identically to the alias
- on a master session = alias publication is skipped (host home context · canonical exposed as-is)

## Behavioral Flow

### path 1 — addStandardTools

```
at host boot:
  app = KernelApp.boot(...)
  ep = app.addEndpoint(label: 'main')
  ep.addStandardTools(app)
    → 45 tools registered on the endpoint with canonical names (bk.fact.write, etc.)
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
      name: entry.key,                       # 'bk.fact.write', etc.
      handler: wrapInProcess(entry.value),
      inputSchema: <strict>,
    )
    → bridge._tools['bk.fact.write'] = def   (script layer)
    → 'bk.recipe_demo.fact.write' auto-published on the endpoint (alias)

domain script call:
  await bridge.callTool(session, 'bk.fact.write', args)
    → scopeId automatic (namespace within bundleId 'recipe_demo')
    → handler of standardTools executed

external LLM call:
  → the endpoint's 'bk.recipe_demo.fact.write' (alias)
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
      rawName: tool.name,               # 'editor.open', etc.
      description: tool.description,
      inputSchema: tool.inputSchema,
      handler: handler,
    )
    → '<bundle.id>.editor.open' registered on the dispatcher
    → '<bundle.id>.editor.open' registered on the endpoint
    → no collision even if two bundles use the same raw name

domain script call:
  → dispatcher.callInProcess('<bundle.id>.editor.open', args)

external LLM call:
  → the endpoint's '<bundle.id>.editor.open'
  → same handler executed
```

## Capability Tool Packs — Exposing Native Capability Packages

The contract for exposing **native Dart capability packages** such as `mcp_browser` · `mcp_form` · `mcp_ingest` · `appplayer_secure` as tools. Neither JS tools (bundle `manifest.tools.tools[]`) nor knowledge facades (`bk.*`), these are infrastructure capabilities the host adopts optionally. Registration uses **path 3 (`HostToolRegistry.registerExposed`)** as-is — no new path.

### Two layers — port contract + implementation

The capability already stands as layers:

| layer | place | content |
|---|---|---|
| **port contract** | `mcp_bundle` port catalog (`package:mcp_bundle/ports.dart`) | per-capability contracts such as `llm_port` · `channel_port` · `storage_port` · `ingest_ports` · `workflow_port` · `pipeline_port` |
| **implementation** | capability package | `mcp_browser` (`BrowserOperations` 9 kinds) · `mcp_form` (`form_tool_handler` + 5 renderers) · `mcp_io` (`IoTools` + `IoRuntime` 4-primitive) · `mcp_ingest` (`IngestPipeline`) · `mcp_channel` (9 connectors · `ChannelPort` bidirectional messaging) · `mcp_canvas` (pure-Dart `Canvas` + CDL) · `mcp_analysis` (`AnalysisPort` + spec/execution engine) · `mcp_datastore` (`DatastoreTools` — `fs.*`/`db.*` data interface layer, separate from io. Canon `13-datastores.md`) · `appplayer_secure` (`facade/appplayer_secure` — merged single package: domain-neutral primitive modules + AppPlayer domain layer) |

→ a capability tool = **the implementation package's operation wrapped as a `KernelToolHandler` and registered via path 3**. The wrapped target is an implementation satisfying the port, so the capability is replaceable.

### Contract

1. **namespace = the capability id as a synthetic bundleId.** `registerExposed(bundleId: 'browser', rawName: 'submit_form')` → `browser.submit_form`. The existing `<bundleId>.<rawName>` model as-is, with a stable capability id (`browser` · `form` · `ingest` · `io` · `fs` · `db` · `secure`) in the bundleId slot. Not a real bundle — a fixed host-capability namespace. (One package may expose multiple capability ids — `mcp_datastore` → `fs`·`db`.)

2. **opt-in declaration place = host-side. The kernel · `standardTools()` NEVER depend on capability packages.** The host (vibe_studio · AppPlayer Pro · Ops runtime) adds the capability package to its own pubspec, boots that package's runtime, then converts operation→handler with a thin tool-pack adapter and calls `registerExposed`. A headless · mobile host simply does not add it → it does not drag in a heavy dependency such as Chromium. (Contrast with the knowledge kernel being shared by every host — a capability is only the adopted portion.)

3. **tool-pack place = brain_kernel's single reference (the method) + the host core (real adoption).** `mcp_browser`, etc., must not know `KernelToolHandler` (a brain_kernel type) (staying kernel-agnostic). The canonical ***method*** of the conversion wiring **is presented as one unified reference `recipes/capability_tools/`** — not a separate package per capability but **one universal registration pattern + capability examples**:
   - `registerCapabilityTools(registry, capabilityId, tools)` = a capability-agnostic registration pattern. The single function the host (vibe_studio · AppPlayer) calls.
   - capability examples = two shapes. **shape A** (the package holds its own MCP surface — `mcp_form` · `mcp_io`[`IoTools.tools`+`call`]): map declared tools as-is. **shape B** (runtime only — `mcp_ingest` pipeline · `mcp_browser` 9 ops · `mcp_channel` `ChannelPort`(`channel.send`) · `mcp_canvas` CDL↔JSON(`canvas.*`) · `mcp_analysis` `AnalysisPort`(`analysis.*`) · kernel-canonical `KvStoragePortAdapter`(`kv.*`) · **`mcp_client` itself**): wrap operations per verb. External MCP connection is separated by its own contract (see "External MCP Servers — Two Modes" below).
   - same self-contained contract as `recipes/claude_code/` (`publish_to: none` · the brain_kernel main lib·barrel·pubspec unmodified · the capability dependency only in the reference pubspec).
   - the real host brings this *pattern* into its own core wiring. A bespoke tool (not a package) is also fed to the same function by directly constructing a `CapabilityTool`.
   - a Flutter-bound capability (`appplayer_secure` — secure storage) is a **separate recipe `recipes/secure_capability/`** (flutter dep, separate from the pure-Dart `capability_tools`) — `secure.*` (at-rest seal/open, facade wrap) · `secret.*` (keyed credential vault, `SecureStorage` wrap) · the credential migration core (`CredentialMigrator`, passphrase). Same `registerCapabilityTools` pattern, Flutter house. Detail = "secret capability — credential vault + asset convention" below.

   → brain_kernel core = capability-agnostic (the reference is a sibling folder). host/AppPlayer core = free to adopt.

4. **secure = expose only the `appplayer_secure` facade, not the internal primitive modules directly.** The capability tool pack wraps the `appplayer_secure` facade. The internal primitive modules (`package:appplayer_secure/src/...`) are reachable only through the facade — preserving the facade/primitive separation. (2026-06: the former `secure` primitive package + the `appplayer_secure` wrapper were merged into a single `appplayer_secure` — the separation shifted from *inter-package* → *intra-package module*. The facade-first principle is identical.)

5. **policy · consent gates remain inside the capability package.** browser's `PolicyEngine` (URL allow/deny · robots · resource cap) · `AuditTrail`, secure's encryption = the package's responsibility. The tool pack only exposes, never bypassing policy.

### per-host adoption matrix (host decision · illustrative)

| capability | vibe_studio | AppPlayer | headless CLI | Ops runtime |
|---|---|---|---|---|
| llm | host-owned (kernel) — adoption unnecessary | 〃 | 〃 | uses host pool (no own boot) |
| browser | adoptable | Pro option | usually not | adopted (member AuthProfile) |
| form | adoptable | adoptable | adoptable | adopted |
| ingest | adoptable | adoptable | adoptable | adopted |
| secure | adoptable | adoptable | optional | adopted |
| secret | adoptable | adoptable | optional | adopted (asset credential vault) |
| channel | adoptable | adoptable | adoptable | adopted (mcp_channel) |
| io | adoptable | adoptable | adoptable | adopted (mcp_io) |
| canvas | adoptable | adoptable | adoptable | optional (mcp_canvas) |
| analysis | adoptable | adoptable | adoptable | adopted (mcp_analysis) |
| kv | adoptable | adoptable | adoptable | adopted (kernel `KvStoragePortAdapter` — `workspaceId` injected) |

adoption = the host adds the capability package dependency + tool-pack registration. A non-adopting host does not have that capability's tools in its catalog.

#### The device adapter of the io capability

The `io` capability surface (`io.*`) is fixed, and device behavior enters as a **driver (adapter) = sibling `mcp_io_*` packages** (core unmodified · 0 new verbs). The **complete definition of** the 2 registration lifecycles (boot vs on-connect) · per-driver platform gating · Studio↔AppPlayer shared wiring · per-driver security (process sandbox / connection ACL) **= `11-io-devices.md`** (canon). Consumers follow it. Shared-wiring implementation = `recipes/io_drivers/`.

#### secret capability — credential vault + asset convention

`secret.*` (keyed credential vault, no plaintext get) · the `asset` fact convention (`category:"asset"` + `credentialRef` indirection, secret body never in a bundle/fact) · passphrase credential migration (`CredentialMigrator`) — the **complete definition = `14-asset-credentials.md`** (canon). Consumers follow it. Registration is path 3 + the single reference `recipes/secure_capability/`, the same as io · datastore.

### Anti-Patterns

- ❌ adding a capability to `standardTools()` (the `bk.*` knowledge facade surface) — pollutes the knowledge domain (`KnowledgeSystem`) + drags a capability dependency onto every kernel host.
- ❌ registering via `BundleSessionBridge.registerTool` — the `bk.` enforcement validator throws ArgumentError. A capability is not a knowledge tool.
- ❌ a capability package depending on `brain_kernel` — breaks kernel-agnosticism. The conversion is the host layer.
- ❌ exposing the `appplayer_secure` internal primitive modules (`src/...`) directly, bypassing the facade — violates the facade/primitive separation.

### Incorporation Gate — What Goes in the Package / What Goes in the Host

The goal is to incrementally incorporate ecosystem packages as capabilities (Studio = a superset oriented toward the whole · AppPlayer = a per-instance partial assembly). Not "all at once" but **deciding the place first via the gate below at each incorporation point** — since a package is shared · universal, a change to it is debt, so the owner does not uncritically accept a request and judges as a gatekeeper.

**3 questions before incorporation:**

1. Is this a **true primitive of its own domain**, or **caller convenience**? If the capability already works (if the optional field already exists), it is the convenience side.
2. Is the missing piece **owned by this package layer/domain**, or can **the consumer (host) do it on its own side**?
3. If this is added, will **every consumer · every capability come to require the same thing**? (then the package bloats into a junk drawer of per-consumer convenience knobs)

**Placement layer (the gate's conclusion):**

| What | place | rationale |
|---|---|---|
| capability surface (`<id>.<verb>` wrapper) | **host wiring** (`recipes/capability_tools/` pattern → host core) | neither a shared package nor a builtin (ops, etc.) |
| capability runtime / connector | **shared package** (`mcp_browser`·`mcp_channel`·…) | the package grows only when its *own domain's true primitive* is lacking (e.g. `mcp_browser`'s base IO `RobotsFetcher` = web-access domain ✓) |
| host-global config·identity·default | **host dispatch seam** — merged into op args | no `defaults:` knob in the package registry (violates question 3 → every capability requires it) |
| cross-cutting infra the kernel consumes (LLM provider pool · inbound MCP) | **host-level service → injected into the kernel** | not a capability surface · not builtin-private (the provider is booted by the host and injected via `InfraPorts.llmProviders`) |
| a state-port the kernel consumes (KvStoragePort, etc.) | **single canonical implementation (where the consumer lives) + injected by the host** | no per-builtin fork — since the kernel already provides file KV + workspace-aware (`FlowBrainWiring.workspaceId`), `KvStoragePortAdapter` (optional workspace-scope) is canonical, the host injects `workspaceId`. Bespoke KV forks such as ops's are deleted after adoption |

**Unimplemented ≠ avoidance.** The fact that part of some capability is unimplemented/unwired is *not a reason to defer incorporation but a target for implementation* — an absent consumer is usually a symptom of incompleteness, not a ground for avoidance. But implement only after pinning the *place* first via the gate above.

## Common Contract for Extending Builtin Tools (assembly · source-agnostic · Studio↔AppPlayer parity)

The capability tool pack is one case of a broader contract — **the common contract for a bundle to call builtin tools and keep working even when that bundle moves hosts (Studio→AppPlayer)**. Three axes:

### ① Assembly — all-builtin ↔ partial assembly

The host **adopts capabilities (= builtin-tool bundles) opt-in**. Adoption = putting that recipe/registration into its own core wiring. Both all-builtin (every ecosystem package) and partial assembly (only what is needed) use the same mechanism — only the size of the set of `registerExposed` calls differs. The tools of a non-adopted capability are not in that host's catalog.

- vibe_studio = superset host (oriented toward incorporating the whole — [`09-instances.md`] Studio=AppPlayer superset).
- AppPlayer = a needed subset (partial assembly) or all (full embed) — assembled **by adding and removing capabilities per instance (model·custom)**. The deployment form decides.

### ② Source-agnostic — same contract whether package or bespoke

The **source of a builtin tool does not appear in the contract**:
- an ecosystem capability package (`mcp_form`, etc.) → the recipe converts operation→`KernelToolHandler` then `registerExposed`.
- a bespoke new tool (not a package, logic the host wrote directly) → registered as `<namespace>.<verb>` via the same `registerExposed`.
- a JS tool within a bundle (`manifest.tools.tools[]`) → the host's JS executor builds the handler and `registerExposed` (path 3's original usage).

All three sources terminate at **the single name `<namespace>.<rawName>` of `HostToolRegistry.registerExposed`**. The bundle only needs to know the name and is unaware of the source.

### External MCP Servers — Two Modes (do not mix)

External-server tools are treated differently from the three sources above — **one must not pre-flatten every external tool into host general tools** (the host comes to own the connection and the catalog bloats).

1. **`mcp_client` as a host tool (the app drives) — the default path.** The host does not pre-attach to a specific server. The `mcp_client` capability is raised once as `mcp.*` tools (`mcp.connect`/`call_tool`/`read_resource`/`list_tools`/`list_resources`/`disconnect`) (shape B, runtime=`KernelClientHost`). **The app/bundle directly** attaches to the desired server with those tools and drives a tool or fetches a resource (e.g. a dashboard UI). The host is merely a conduit, not the connection subject.

2. **external-tool registration — a specific remote tool as a first-class catalog tool.** When the author wants to raise a specific remote tool into the catalog under a namespace (`<serverLabel>.<verb>`) so the agent sees it and declares it via `requires` — **intentional registration at the "add tool" moment** (the builder/authoring external-tool registration, opt-in · per-add `registerExposed`). **host-settings-based auto-registration** (registering servers pinned in settings at boot) is a follow-up host-settings feature — currently out of scope.

→ mode 1 = dynamic · app-driven (not catalog-registered). mode 2 = static · intentional registration (catalog first-class). Automatic blanket registration is a non-goal.

### The place of the mcp_client tool — kernel-provided (land)

The `mcp.*` surface of mcp_client is **kernel-provided** (not a recipe). Reason: `mcp_client` · `KernelClientHost` · `McpClientKernelHost` are already kernel-native → 0 new dependency for exposure (the sole reason for isolating into a recipe = dep drag does not apply to mcp_client). Also "using external tools" = an LLM-class base primitive, not a domain capability. But ① not put into the `bk.*` facade (a separate `system/host/` surface for IO, HostToolRegistry path 3), ② `clientHost` kept optional (if not given, no mcp.*), ③ no auto-registration (the host chooses when it raises clientHost). Domain capability packages (form/browser/ingest) remain recipes due to dep drag — a distinction.

Implementation: `brain_kernel/.../system/host/client_tools.dart` — `clientTools(KernelClientHost)` + `registerClientTools(HostToolRegistry, KernelClientHost)`, the `mcp.*` 6 tools (connect·call_tool·read_resource·list_tools·list_resources·disconnect). barrel export. tests 7 PASS.

**shape = identical to `standardTools` (`InProcessToolHandler`, raw JSON).** One source feeds both host paths:
- in-process dispatcher direct (AppPlayer `ToolDispatcher` consumes it exactly like `bk.*`) → register via `clientTools(host)`.
- external endpoint (vibe_studio `ServerBootstrap`, etc.) → `registerClientTools` (`wrapInProcess` automatic).

### Host Application Structure (AppPlayer · vibe_studio isomorphic)

`mcp.*` application = the host (1) holds a `clientHost` and (2) registers `clientTools` on the dispatcher in the **same loop** as `bk.*` registration.

- **AppPlayer** (`AppPlayerCoreService._init`) — **applied**: `KernelApp.boot(clientHost: McpClientKernelHost())` + merging `standardTools` and `clientTools(kernel.clientHost!)` and registering them on the `ToolDispatcher` via the same adapt loop. `mcp.*` enters the catalog by the same path as `bk.*`. test TC-CORE-021 (verifies `mcp.connect`/`mcp.call_tool` registration).
- **vibe_studio**: same pattern + `registerClientTools`(endpoint) for external exposure.

→ a host that did not raise `clientHost` (headless · no-outbound) has no `mcp.*` (opt-in kept).

**`clientHost` choice — AppPlayer uses `McpClientKernelHost` (① a separate client).** AppPlayer's existing `ConnectionManager` is *the server list the user configured in the UI* (ServerConfig: name·isFavorite·metadata), while `mcp.*` is *connections the app opens programmatically*, so the two are **different concerns**. Tying them into one store (② an adapter) causes the conflation of the agent mutating the UI server list → separation (①) is consistent. The "two parallel paths" are not duplication but two legitimate concerns.

### ③ Studio↔AppPlayer parity — stable namespace + `requires`

Two grounds for a bundle keeping working even when it moves hosts:

1. **stable namespace.** The exposed name of a capability/tool (`form.render` · `browser.submit_form` · bespoke `<ns>.<verb>`) is identical across hosts. A bundle authored in Studio calling `form.render` resolves under the same name in AppPlayer.
2. **`requires` declaration + host verification.** The bundle declares the builtin tools · atoms it needs in the manifest `requires` (`mcp_bundle` `RequiresSection` — `builtinTools: List<String>` · `builtinAtoms: List<String>`). On activation the host **collates the bundle's `requires.builtinTools` against its own set of embedded tools** — works if satisfied, rejects if lacking (or shows the unmet tools).

→ **parity guarantee formula**: a bundle works in AppPlayer ⟺ `host.embeddedTools ⊇ bundle.requires.builtinTools`. To put a bundle made in Studio (superset) onto AppPlayer, AppPlayer embeds capabilities so as to satisfy that `requires` (exactly that set via partial assembly, or all). Source being package or bespoke is irrelevant — only the name set must be satisfied.

### Author · host responsibilities

| Subject | Responsibility |
|---|---|
| bundle author | declare `requires.builtinTools` honestly (every tool name used). |
| host | expose the set of embedded capabilities + verify `requires` on activation + explicitly reject if unmet. |
| capability/recipe | a stable `<namespace>.<verb>` name + confine tool-boundary exceptions (`{ok:false}` envelope). |

reference implementation = `os/core/brain_kernel/recipes/capability_tools/` — the universal `registerCapabilityTools()` + form(shape A) · ingest(shape B) examples, host-usage README, tests 8 PASS, brain_kernel core unmodified.

## Stateful Tool Lifecycle for Re-mount · Multi-Instance Domains

The three paths above are a model where a tool is **registered once**. But when the host has a **re-mount · multi-instance domain** (e.g. an App Builder that mounts and disposes per tab, an arbitrary host's re-mountable panel) and that domain exposes **stateful host tools** (`project.info`·`chat.send`, etc., depending on the currently open project · session) — the tool's backing state is bound to a mount instance, creating a lifecycle gap.

### Anti-Pattern (G-3) — raw re-registration per mount + mount-local capture

- ❌ each mount registers the same tool name via **raw `server.addTool` (or `registerExposed` per mount)** and **captures that mount's state (closure/bridge)**.
- the host's `addTool` = **Map overwrite by name key** → **the last mount owns the tool**.
- when that owning mount is disposed → the tool **remains** on the host while the backing points to dead (null) state → **misjudged as "absent" even though an active instance is alive** (the screen shows the project open, the tool says `{open:false}`). intermittent (does not occur if a single mount is kept alive, only in dispose/re-mount sequences).

### Contract — register-once + dynamic resolution of the active instance

1. **register a stateful host tool once at boot.** The handler, instead of captured mount-local state, **dynamically resolves the host's active instance at dispatch time** and reads its state. (No re-registration per mount.)
2. **active-instance tracking = host-owned.** The host applies **the per-instance map + activePath/activeMount pointer** pattern already used on the mount side (no singleton slot) **identically to the tool backing** — the handler resolves `activeMount?.state`. If there is no active mount, an explicit `{ok:false, reason:'no active mount'}`.
3. **the framework does not take on the routing.** `HostToolRegistry` provides only `registerExposed`/`unregisterExposed` (lifecycle primitives). "which instance is active" is a host/domain concept, so the host resolves it inside the handler. Keep the registry minimal.
4. **a genuinely ephemeral single-instance tool** (meaningful only to that mount) → `registerExposed` on mount, **`unregisterExposed` on unmount** (the primitive exists). No lingering.

→ that is, **the active-instance resolution of the tool backing = the tool counterpart of the mount-side active-pointer pattern**. Bypassing registerExposed (raw addTool + capture per mount) is the cause of this gap — closed by register-once + active resolve.

## Tool → Agent Catalog Wiring

The framework's true wiring:

```
KernelEndpoint catalog (the sum of registration path 3)
  ↓ KernelApp.toolsForAgent(agentId, {role})         ← helper
List<LlmTool> (per-role · per-bundle subset)
  ↓
AgentChatController(tools: ...) / AgentFacade.ask(...tools: ...)
  ↓
LlmRequest.parameters['tools'] (mcp_llm standard)
  ↓ natural branching by provider
  ├── mode B (anthropic / openai / gemini) — catalog forward → tool_use response → host loop
  └── mode A (claude_code) — catalog ignored + --mcp-config wiring (absorbs the host MCP server)
```

### `KernelApp.toolsForAgent` contract

```dart
List<LlmTool> toolsForAgent(
  String agentId, {
  required AgentRole role,
  List<String>? explicitAllowlist,  // manifest agents[i].tools
  String? bundleId,                  // the scope of worker / specialist
});
```

**per-role default subset** (flowbrain_core `AgentRole` 3 kinds — framework default provided):

| role | default subset | session |
|---|---|---|
| `manager` | framework `bk.*` delegation (`bk.agent.*`) + read (`bk.*.query/get/list` · `bk.workflow/pipeline/runbook` status). **mutation excluded · host-specific namespaces (`studio.*` · `app.*` · arbitrary) also excluded** (the kernel does not hard-code the host's prefix · manifest allowlist or `explicitAllowlist` widens explicitly) | master |
| `worker` | its own `<bundleId>.*` domain + `bk.<bundleId>.*` alias | non-master |
| `reviewer` | a review-friendly subset of master (read tools + agent.history) | master |

when `explicitAllowlist` is specified = overrides the default subset (glob patterns supported: `bk.fact.*` · `<bundle>.editor.*`). A narrow capability such as a specialist specifies only the allowlist, with no separate enum.

detailed canon + anti-patterns = [`10-agent-scoping.md`](10-agent-scoping.md).

## Per-Host Implementation Matrix

| Host | dispatcher implementation | endpoint implementation |
|---|---|---|
| **vibe_studio** | runtime toolExecutor (or its own map) | `ServerBootstrap` (server-backed) |
| **AppPlayer** | `ToolDispatcher` (`runtime/tool_dispatcher.dart`) | `InProcessKernelServerHost` (client default) or `ServerBootstrap` (Pro's server option) |
| **headless CLI** | its own in-memory map | `InProcessKernelServerHost` or `ServerBootstrap` |
| **arbitrary host** | free (callback path) | free (a KernelServerHost abstract implementation) |

## Non-Goals

- the logic definition of a tool handler = the bundle / host domain
- the canon of an agent's catalog-scope thesis = [`10-agent-scoping.md`](10-agent-scoping.md)
- this spec = **the interface of the three registration paths + the two naming models + code-level guarding + the agent-catalog wiring helper** only
