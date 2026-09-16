# 04 — UI Host

The UI layer's responsibility + interface. The framework's most flexible area — host free (Flutter · web · embedded · voice · AR · CLI · any surface).

## Definition of host

**host** = a process-level entry point. The owner of the UI surface · runtime · kernel boot · bundle activation. Within one host: N endpoints · N bundles · N agents.

```
host process
  ├── UI surface (chrome · runtime · widget · DSL)
  ├── KernelApp (single instance of the operational layer)
  ├── active bundles (entries of BundleActivationRegistry)
  ├── active endpoints (_endpoints of KernelApp)
  └── (optional) external transport (server / client / both)
```

## host Responsibilities

### 1. KernelApp boot

```dart
final app = await KernelApp.boot(
  workspaceId: '<host-defined>',
  kvStorage: <host-chosen storage>,
  llmProviders: <host-chosen LLM map>,
  config / uiResource / observability / bundleSource: <host port impls>,
  serverHostFactory / clientHost: <host-chosen mcp wiring>,
);
```

- the host provides all external dependencies as ports
- decides the LLM provider (Anthropic · OpenAI · Gemini · Claude Code subscription · CustomLlmProvider · etc.)
- decides the transport (server / client / both / in-process)

### 2. endpoint Wiring

The host decides N endpoints:

```dart
// single endpoint (simple host)
final ep = app.addEndpoint(label: 'main');

// multiple endpoints (per-domain isolation · vibe_studio pattern)
final epA = app.addEndpoint(label: 'domainA', appVersion: '0.1.0');
final epB = app.addEndpoint(label: 'domainB', appVersion: '0.1.0');

// in-process only (no external transport · AppPlayer pattern)
final ep = app.addEndpoint(label: 'inproc');  // serverHostFactory null = InProcessKernelServerHost
```

### 3. bundle Activation

The host wires per-category from the manifest:

```
host:
  ├── 6 facade assets → BundleActivation.activate(bundle)
  ├── domain tools (manifest.tools.tools[]) → HostToolRegistry.registerExposed
  ├── knowledge tool exposure (optional) → BundleSessionBridge.registerTool (bk.* enforced)
  ├── manifest.agents[i].tools allowlist (optional) → AgentHost
  │     forwards via KernelApp.toolsForAgent(role, explicitAllowlist: ...)
  │     (per-agent scoping · 10-agent-scoping.md · host-specific namespace explicit)
  ├── UI → host UI runtime mounts manifest.ui
  ├── wiring → host chrome wires manifest.wiring
  ├── settings → host settings dialog wires manifest.settings
  └── chat.slashCommands → host chat panel wires
```

### 4. tool dispatch

The host wires two dispatcher sites:

| dispatcher | call site |
|---|---|
| **script layer** (in-process) | UI tool action · agent tool use · own logic |
| **external endpoint** (transport) | external MCP client (CLI subprocess · another host · external LLM) |

The same tool is registered at both sites simultaneously (`HostToolRegistry.registerExposed` does this automatically). Tool registration path detail = `06-tool-registry.md`.

### 5. session Lifecycle

The host starts / ends the bridge session at the moment the UI is activated / deactivated:

```dart
// activation (tab open · launcher click · dashboard enter)
final session = bridge.openSession(activation);

// Zone-scoped scopeId within dispatch
await bridge.runScoped(session, () async {
  await bridge.callTool(session, 'bk.fact.write', args);
});

// deactivation (tab close · launcher off · dashboard exit)
await bridge.closeSession(session);
```

## Areas Where the host Is Free

### UI surface

- Flutter / web / embedded / voice / AR / CLI / any
- use mcp_ui_dsl · custom widgets · both · neither
- chrome model free (titlebar / launcher / multi-tab / sidebar / fullscreen · etc.)

### LLM provider Decision

- API-key based (Anthropic · OpenAI · Gemini · Cohere · Mistral · Groq · Bedrock · VertexAI)
- subscription based (Claude Code subscription via CustomLlmProvider · `brain_kernel/recipes/claude_code/`)
- arbitrary CLI / local LLM (Ollama · etc.) via CustomLlmProvider
- per-agent different provider possible
- `LlmPortAdapter(providerName: ...)` specified → provider visible in UI banner / chat header
- `AgentLlmSessions.providers` = live `UnmodifiableMapView` — post-boot adapter swaps (e.g. `forKernel` factory rebind after transport start) are reflected in the dispatch path immediately as new entries
- **No automatic fallback** — when the user does not specify → throw + UI banner (aligned with `feedback_no_fallback_anywhere`)

### transport Decision

| mode | host composition |
|---|---|
| server only | KernelServerHostFactory injected · clientHost null |
| client only | KernelServerHostFactory null · clientHost wired |
| both | KernelServerHostFactory + clientHost both |
| in-process | both null (kernel's InProcessKernelServerHost default) |

For a multi-endpoint host (vibe_studio's multi-domain pool · user-arbitrary multi-transport) = extract the explicit dial-back spec via `KernelApp.hostMcpServerSpec(endpointLabel: '<label>')`. `endpointLabel` unspecified = first non-null spec (safe only for a single-endpoint host).

### bundle Composition Decision

- single bundle (single-bundle host · simple player)
- multiple bundles (multi-bundle host · vibe_studio pattern)
- simultaneous bundle activation (cross-domain · multi-agent)
- bundle dependency (manifest.requires gate)

### chrome Slot Policy

- wire all slots of manifest.wiring (vibe_studio · AppPlayer Pro pattern)
- partial wiring (selected slots only — embedded pattern)
- no slot wiring (headless pattern)

## Contracts the host MUST Follow

### Tool Registration

- general tools (domain + host's own) = `HostToolRegistry.registerExposed` only (no bridge use)
- knowledge tool wrapping = `BundleSessionBridge.registerTool` only (`bk.` start enforced)
- standardTools = `addStandardTools(app)` (host home context)

### MCP surface wrapper (3 kinds)

- tool registration = `host.addTool` / `host.removeTool` (the 3 paths above wrap these)
- resource registration = `host.addResource` / `host.removeResource`
- **prompt registration = `host.addPrompt` / `host.removePrompt`** (KernelPrompt* wrapper · usable by builtin/bundle without directly importing `package:mcp_server` · see 03-kernel-runtime.md "MCP Prompts surface")

### Name Isolation

- general tool = `<bundleId>.<rawName>` (HostToolRegistry automatic)
- knowledge tool = `bk.<bundleId>.<rest>` (bridge auto-aliases)
- host's own tool = host namespace (e.g. `host.menu.refresh`)

### scopeId

- applied automatically on `bridge.callTool` / `bridge.readResource` calls within a domain context
- pass-through for a master session
- explicit cross-bundle = `kb://fact/<other_bundle>.foo`

### Security · Permissions

- secrets such as API keys = MUST NOT be recorded in the manifest artifact (.mbd) (FR-LLM-005)
- handling keys such as `ANTHROPIC_API_KEY` on external LLM calls = host policy (aligned with §5 of the recipe)
- a mode such as bypassPermissions = explicit opt-in by the host

## host Implementation Matrix

| host | UI surface | transport | bundle composition |
|---|---|---|---|
| **vibe_studio** | Flutter chrome + mcp_ui_dsl + multi-tab | server + client | multi-bundle · simultaneous activation |
| **AppPlayer Standard** | Flutter player chrome | client only | single-bundle |
| **AppPlayer Pro** | Flutter player chrome + launcher + library | client + (optional) server | multi-bundle · library |
| **AppPlayer X** | Flutter standalone shell | client | bundled (1 bundle embedded) |
| **AppPlayer Custom** | Flutter whitelabel | client + server | site-specific |
| **FlowBrain** | Flutter knowledge explorer | in-process (or server) | single or multi |
| **headless test / CLI** | text · stream | in-process · client | arbitrary |

## Non-Goals

- Widget spec of the UI DSL = [`../ui_dsl/`](../ui_dsl/)
- Visual design of the chrome = host free
- host policy (membership · billing · distribution) = out of scope (host domain)
- This spec = **only the host's framework composition responsibilities**

## Multi-Origin Composition (UI DSL v1.4 Composition Profile · 2026-07-28)

For one bundle to compose several MCP servers into a single product (a temperature sensor + a humidity sensor + a controller = one app), the host must guarantee **per-origin isolation**. A host that cannot do this simply **does not claim** the Composition Profile — `view` then behaves fail-closed and the document degrades into a visible error, never a silently wrong render.

What a host must satisfy in order to claim it ([`../ui_dsl/1.4/07_Security.md`](../ui_dsl/1.4/07_Security.md) §7.10.1 · [`../ui_dsl/1.4/18_Conformance.md`](../ui_dsl/1.4/18_Conformance.md) §18.7.1):

| Obligation | Content |
|---|---|
| Scope | Each embedded definition gets **its own state tree · subscription registry · permission and storage identity**. No implicit access to embedder state |
| Channel | Embedder→embedded data travels through `view.props` **only** — explicit and one-way |
| Permissions | The **intersection** (what the embedder was granted for that origin ∩ what the embedded definition requires). Never escalated by embedding |
| Dispatch | `tool` calls, resources and subscriptions in an embedded subtree go to **that scope's origin**. Never to the embedder's origin |
| Unknown origin | **Refuse.** No fallback to one's own origin — that draws one server's UI under another server's identity |
| Notification routing | `notifications/resources/updated` reaches only the scope bound to that connection |
| Failure isolation | A dead origin affects only that `view`'s `fallback`. Siblings and the embedding page keep rendering |
| Recursion | An embed depth limit plus origin cycle detection |
| Definition cache | **Reuse** an origin's definitions while attached. Re-reading on every mount adds a round trip per re-entry, which looks like a tile reconnecting while the connection is in fact fine. One invalidation point — reopening the origin — is enough |
| Lifecycle | An embedded definition runs its own hooks: `onInit→onMount→onReady`, and on unmount `onPause→onUnmount→onDestroy`. **Running the start and omitting the release** leaves a node streaming after its tile is gone ([`../ui_dsl/1.4/06_Runtime_Contract.md`](../ui_dsl/1.4/06_Runtime_Contract.md) §6.11.2b) |
| Shared resources | When one device is used by both a standalone screen and a composed tile, connection, subscription and notification all become shared resources. Rules = [`17-device-discovery.md`](17-device-discovery.md) §7.6b |

**All four capabilities must be wired for the claim to hold** (corrected 2026-07-29 — the first edition said three). "Dispatch" in the table above is wiring separate from the resolver, and the first implementation omitted it. The result is **a screen that looks finished and does nothing**: buttons in the embedded subtree went to the app's own session and landed where that device had no client (`session.tool.no_client`), leaving live values as labels. A host with only resolve **must not claim** the profile.

The four runtime-side seams are `registerDefinitionResolver` (fetch) · `registerOriginToolCaller` (act) · `registerOriginResourceWatcher` (track) · `registerOriginResourceReader` (one-shot read).

Missing the fourth produces **a wrong answer rather than a missing one** — the embedded document's uri reads the **embedder's** resource and shows that value as if it were the device's.

On the host side the `composition_host` recipe (`buildCompositionHooks`) assembles the set in one piece, going through the kernel's `mcp.read_resource` / `call_tool` / `subscribe_resource` / `unsubscribe_resource` ([`06-tool-registry.md`](06-tool-registry.md) mode 1 as-is). AppPlayer, being a published package, **vendors** the recipe and wires them all behind `useKernelDefinitionResolver()`.

**Two things the runtime must hold** (each broke silently in the field):

- **A scope's lifetime is the mount, not the render.** Creating a fresh scope every frame means writing a value triggers a rebuild that discards the state just written — the value can never reach the screen while every layer below reports success.
- **Bindings in an embedded subtree must re-evaluate on that scope's state changes.** Evaluating once at mount leaves live values as labels.

**Opening an origin is deferred.** The document names an origin and the host opens it at first use. Holding a permanent connection per registered device means the last connection resets the earlier ones on a single-peer board (measured: `Connection reset by peer`).
