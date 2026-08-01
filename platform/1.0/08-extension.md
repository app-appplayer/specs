# 08 — Extension Standard

The standard procedure to follow when building a new instance (product · host · domain · etc.) on top of the framework. The real entry point of the user's framing "**same framework, infinite instances**".

## Extension Dimensions

| Dimension | What is added |
|---|---|
| New host product | UI surface + composition-pattern decision + KernelApp boot |
| New domain bundle | `.mbd` manifest authoring + category wiring |
| New LLM provider | implement mcp_llm's LlmProvider or extend CustomLlmProvider |
| New transport | extension transport = implement `ClientTransport`/`ServerTransport` + inject (impl outside the core) · or implement `KernelClientHost`/`ServerHost` wholesale |
| New tool kind | bundle's ToolKind enum + host's handler-composition logic |
| New facade (knowledge) | add a facade in flowbrain_core (kernel domain) |

Each one need only follow the framework's **interface spec** (the specs in this directory).

## 1. Building a New Host Product

### Step 1 — boot

```dart
import 'package:brain_kernel/brain_kernel.dart';

Future<void> main() async {
  final app = await KernelApp.boot(
    workspaceId: '<host-defined>',
    kvStorage: <storage decision>,
    llmProviders: <LLM pool decision>,
    config / uiResource / observability / bundleSource: <port impls>,
    serverHostFactory: <ServerBootstrap.factory when server-only>,
    clientHost: <McpClientKernelHost() when client-only>,
    chatLogDir: <path>,
    bundleRegistryStorageDir: <path>,
  );
  ...
}
```

### Step 2 — endpoint + tool registration (composition-pattern decision)

| Use case | endpoint composition |
|---|---|
| self-contained (client only) | `serverHostFactory: null` → InProcessKernelServerHost · no external transport |
| external exposure (server only · server+client) | `serverHostFactory: ServerBootstrap.factory` + `ep.start(KernelTransportKind.streamableHttp, port)` |
| external call (client only · server+client) | `clientHost: McpClientKernelHost()` + `app.clientHost.connect(...)` |

tool registration:
- host-home standard 45 tools = `ep.addStandardTools(app)`
- host's own tools = `HostToolRegistry.registerExposed(bundleId: 'host', rawName: ...)`

### Step 3 — bundle activation wiring

```dart
Future<void> activateBundle(McpBundle bundle) async {
  // 6 facade assets
  await BundleActivation(system: app.system, bundleId: bundle.id).activate(bundle);

  // domain tools (manifest.tools.tools[])
  for (final tool in bundle.manifest.tools.tools) {
    final handler = _buildHandlerForToolEntry(tool);  // host composes the per-kind handler
    hostToolRegistry.registerExposed(
      bundleId: bundle.id,
      rawName: tool.name,
      description: tool.description ?? '',
      inputSchema: tool.inputSchema,
      handler: handler,
    );
  }

  // domain access to knowledge tools (optional · when the domain needs access to its own knowledge)
  final session = bridge.openSession(activation);
  for (final entry in standardTools(app).entries) {
    bridge.registerTool(name: entry.key, handler: wrapInProcess(entry.value), inputSchema: ...);
  }

  // UI (manifest.ui)
  await hostUiRuntime.mount(bundle.manifest.ui);

  // wiring (manifest.wiring)
  hostChrome.attachWiring(bundle.id, bundle.manifest.wiring);

  // settings + chat
  hostSettings.attachSections(bundle.manifest.settings.sections);
  hostChatPanel.attachSlashCommands(bundle.manifest.chat.slashCommands);
}
```

### Step 4 — UI surface

Outside the framework spec — host's freedom. Only the following conventions:
- UI does NOT call facades directly
- tool action = via the dispatcher or the endpoint
- resource = bridge.readResource (kb:// URI)
- chrome wiring = wire the manifest.wiring slots

## 2. Building a New Domain Bundle

### Step 1 — follow the manifest spec

The schema of `specs/mcp_bundle/`. The category table of this spec's `02-bundle-interface.md`.

```json
{
  "id": "my_domain",
  "name": "My Domain",
  "version": "0.1.0",
  "requires": { ... },
  "tools": { "tools": [...] },
  "agents": [...],
  "facts": [...], "skills": [...], "profiles": [...], "philosophies": [...],
  "flows": [...], "pipelines": [...], "runbooks": [...],
  "ui": {...},
  "wiring": {...},
  "settings": {...},
  "chat": {...}
}
```

### Step 2 — author the per-category assets

- own domain tools (manifest.tools.tools[]) — JS / external · name `own.namespace` (not `bk.`)
- own agent definitions
- own 4-axis assets (facts · skills · profiles · philosophies)
- own operational logic (flows · pipelines · runbooks)
- own UI (mcp_ui_dsl) + wiring (chrome slot wiring)
- settings + chat slash commands

### Step 3 — compatibility gate

```json
"requires": {
  "kernelVersion": "^0.1.0",
  "uiSpec": "mcp_ui_dsl@1.3",
  "builtinAtoms": ["text@1.3", ...]
}
```

= the host verifies before activation. On incompatibility, reject + inform the user.

### Step 4 — packing

`mcpb`'s packager (mcpb_packager) → `.mbd` directory or `.mcpb` compressed format. The host's BundleSourcePort fetches it.

## 3. Building a New LLM Provider

### Step 1 — options

| Choice | path |
|---|---|
| add an mcp_llm native provider (beyond Anthropic / OpenAI / Gemini / Cohere / Mistral / Groq / Bedrock / VertexAI) | implement mcp_llm's LlmProvider + register a factory |
| CLI subprocess wiring (Claude Code · gemini CLI · ollama · etc.) | extend `CustomLlmProvider` + `mcp_llm.registerProvider('name', factory)` |
| private wrapper | `CustomLlmProvider` or own LlmProvider |

### Step 2 — recipe location

The `brain_kernel/recipes/<vendor>/` pattern. First instance = `brain_kernel/recipes/claude_code/` (Claude Code subscription provider). Self-contained:
- `lib/<vendor>_provider.dart` — provider + factory
- `bin/main.dart` — verifier
- `pubspec.yaml` — brain_kernel path + mcp_llm
- `README.md` — thesis · input/output normalization · verification

### Step 3 — host wiring

```dart
import 'package:my_recipe/my_provider.dart';

final factory = MyProviderFactory();
final provider = factory.createProvider(LlmConfiguration(...));
final adapter = LlmPortAdapter.fromInterface(modelId: 'my-model', provider: provider);

await KernelApp.boot(
  llmProviders: {'my-model': adapter, ...},
);

// when an agent's ModelSpec.id == 'my-model', it routes automatically
```

## 4. Adding a New Transport

Transports split into two tiers:

| tier | definition | location |
|---|---|---|
| **standard transport** | defined by the MCP spec (stdio · SSE · Streamable HTTP) | mcp_client / mcp_server core |
| **extension transport** | outside the spec, a wire the ecosystem adds (ws · tcp · serial · usb · ble · user protocol) | **outside the core** — a separate package, injected |

### Principle — seam in the kernel, impl in the host

- **kernel = injection seam (pure · 0 FFI · additive).** The `KernelClientHost` / `KernelServerHost` contract + an injection point that accepts an arbitrary `ClientTransport` / `ServerTransport`. Being a single point shared by the three axes (AppPlayer · Studio · FlowBrain), one seam means all three inherit it.
- **concrete transport impl = outside the kernel, host opt-in injection.** FFI (serial/usb/ble) · Flutter-channel transports implement `ClientTransport` / `ServerTransport`, live in a separate package, and the host injects them.

**Absolute rule:** do NOT put FFI · Flutter · platform-specific dependencies into the **brain_kernel · mcp_client · mcp_server core.** Doing so pollutes *every* consumer of that core — the FlowBrain pure-Dart headless server, the mobile host, the CLI. **Keeping the kernel pure = guaranteeing pure-Dart instances the "freedom to go pure".**

### Standard 3 Layers (client injection · shared by the three hosts)

The standard for **consuming** — connecting the kernel to an external MCP server (embedded board · hub-exposed local · etc.) over an extension transport. Fixed into three layers — a host follows these three rather than wiring ad hoc.

| Layer | Location | What |
|---|---|---|
| **① seam (contract)** | **core** `brain_kernel` (`package:brain_kernel/mcp_host.dart`) | `ExtensionTransportConnect` capability interface — `connectWith({id, transport})`. The reference impl `McpClientKernelHost` implements it. It is a **separate interface** from the abstract `KernelClientHost` (the type of `KernelApp.clientHost`), so `KernelClientHost` is unchanged (cascade 0). Canonical use = the **`connectExtension(clientHost, {id, transport})` helper** (probe + inject, throws on failure; it seals in one place the cast footgun that `ExtensionTransportConnect` is unrelated to `KernelClientHost?` so an `is` test does not promote the variable). |
| **② transport impl** | **mcp_bridge** (desktop FFI) · **flutter_mcp** (mobile channel) | serial/usb/ble/tcp/ws `ClientTransport`. FFI isolated (the absolute rule above). |
| **③ host surface (tool)** | **recipe `recipes/extension_transport/`** (`publish_to: none`, mcp_bridge dep) | `registerExtensionConnectTool(registry, clientHost)` → the **`mcp.connect_extension`** tool (companion to the kernel's `mcp.connect` — `mcp.connect` drives only the kernel-built stdio/http/sse, while extension transports are built by this tool via mcp_bridge and injected through the ① seam). The connection lands in the kernel `mcp.*` registry by `id` → drive it afterward with `mcp.list_tools`/`call_tool`/`read_resource`/`disconnect`. Only a host that exposes board-connect vendors it (otherwise no mcp_bridge FFI is pulled in). |

- **Hub is a separate recipe.** The transport for consuming a hub-exposed local (`HubConsumerTransport`) is built by `recipes/gateway_node/` (mcp_gateway dep) and is likewise injected through the ① seam (`connectWith`) (`15-hub-channel.md` §8). So **only layer ② splits by dependency (mcp_bridge vs mcp_gateway); the ① seam and the `mcp.*` drive surface are single.**
- Per-host adoption = the ③ tool is opt-in (Studio adopts board-connect · AppPlayer, if hub-only, skips it and uses just the seam helper). The consume drive surface is always the kernel `mcp.*`.

### Standard Consumer Embed (marketplace) — platform-owned, host supplies one InstallSink

**Embedding and using the marketplace must be identical across every host** (AppPlayer Pro · Studio Pro · standalone). That is the reason the platform/recipe exists — if each host re-implements install / connect / isInstalled ("ad-hoc"), it is a hollow interface, not a standard.

**Principle: the platform owns the mechanism; the host injects only the one irreducible atom.** (The ① seam above is this standard's connection foundation — connection already has a standard, so the host does not re-write it.)

The recipe `recipes/marketplace_embed` entry point is **`buildMarketEmbed({installSink, clientHost?, connectServer?, injectExtensionTransport?, fetchBundle?, ledger?, authToken?})`**.

The platform **always owns** (no host re-implements): catalog UI · login wall (owned by the marketplace package; no anonymous browsing; identical mandatory login for every embedder) · sha256 verify gate · **owned/installed ledger logic (`MarketInstallLedger`)** · `isInstalled` (the ledger answers) · the **hub relay assembly** (`HubRelayConnection` → `connectHubViaGateway`).

| Wire | Standard default (thin consumer host: Studio · standalone) | Runtime-owning host override (AppPlayer) |
|---|---|---|
| **install** | — (always the host atom) | **`InstallSink`** — `install(verifiedMbdPath, listingId) → InstallRecord` (materialize the verified archive into my runtime **+ register in chrome**: launcher card / tab) + `open(record)` (**launch / render** — only on the Open action). **install ≠ run.** |
| **connectServer** | `clientHost.connect(streamableHttp)` (kernel seam) | `connectServer` override — my own connection/app model (AppPlayer = `openAppFromServer` + launcher card). The server analog of `InstallSink`. |
| **hub inject** | `connectExtension` (① seam, `clientHost` is `ExtensionTransportConnect`) | `injectExtensionTransport` primitive (the host's `connectExtensionTransport`). **The relay assembly is the platform's**; only the inject is pluggable. |
| **fetch** | `http.Client` GET | `fetchBundle` override (hardened downloader — retries · pinned CAs) |
| **ledger** | `InMemory` / `Stored` | backed by the real install registry (AppPlayer = launcher list) |

- **A thin consumer host (Studio) is fully turnkey**: `buildMarketEmbed(installSink:, clientHost:)` only → connect · hub · fetch all default. **No `isInstalled` stub** (the ledger answers).
- **A runtime-owning host (AppPlayer)** overrides only its runtime materialization (server = app model · fetch = Dio · ledger = launcher list). The verify gate, ledger logic, hub assembly, catalog, and login are still the platform's — nothing is re-implemented. An override is not "ad-hoc re-implementation" but a runtime binding of the same shape as `InstallSink`.
- **Connection credential (contract).** An entitlement's `accessToken` rides the **MCP-standard `Authorization: Bearer`** header — mapped by appplayer_core's `TransportFactory` streamableHttp branch (symmetric with the sse branch's `bearerToken`). Server-specific schemes pass through `transportConfig['headers']` (an explicit header wins over the derived one). No server-specific header name (e.g. X-API-Key) is hardcoded in the core — servers accept the standard Bearer.
- **install ≠ run (contract).** `HostCapabilities.runBundle` = install only (download → sha256 verify → materialize + register in chrome + record the ledger; it does NOT launch). Running is the separate `openInstalled(listingId)` (resolve the install record from the ledger → `InstallSink.open`). Only the "Open" action hits `open`. This mirrors AppPlayer core's `installBundleFromFile` ↔ `openAppFromBundle` split.
- **`open` presentation is host-defined (non-contract).** How `InstallSink.open` renders / navigates is the host chrome's concern — not part of the contract. Launcher-style hosts push a renderer route (AppPlayer Pro = `AppRendererScreen`, Studio = activate the tab), but a **single-app (dedicated-device) variant** returns to the home route instead of pushing (AppPlayer X = `popUntil(isFirst)` → the shell-less boot router renders full-screen; no pushed renderer / no close button). Because that tier is **install == run** (installing IS opening, holding a live connection), it also skips the server-install metadata prefetch (the open reads the same metadata itself). The contract surface (install ≠ run · ledger · isInstalled) is unchanged.
- **Bundle format.** The marketplace download artifact is a **`.mcpb` packed archive** (produced by `McpBundlePacker`; `installBytes` ZipDecodes it). The verify gate writes the temp file as **`.mcpb`**. `.mcpb` = the internal install archive; `.mbd/` = an unpacked directory for in-place loading (the marketplace only ever hands the former). **Installers are uniformly extension-strict**: file installs accept `.mcpb` only (enforced by `McpBundleInstaller.installFile`; directories go through `installDirectory`), and Studio's install surface matches. Lenient (content-sniffing) installers are forbidden — they swallow a producer handing the wrong artifact form, masking the defect until a stricter host surfaces it (the temp-file case was exactly this).
- The low-level `buildMarketHostCapabilities` (closures injected directly, now including installBundle · openInstalled) remains for special overrides, but **hosts use `buildMarketEmbed`**.

### Platform Branching (build integrity)

Every native transport branches by platform via **conditional import + stub**:

```
serial_transport_io.dart    // dart:io + FFI
serial_transport_stub.dart  // web / unsupported → UnsupportedError
serial_transport.dart       // conditional import (dart.library.io ? _io : _stub)
```

→ on unsupported platforms the stub compiles so **the build does not break**, and availability is **determined at compile time** (not a runtime crash). Existing pattern = mcp_client's `http_client_io / _web / _stub`, `event_source_io / _web / _stub`.

### impl Location

| kind | location |
|---|---|
| desktop FFI (serial · usb · ble) | **mcp_bridge** — the FFI opt-in home (isolating the libserialport/libusb/bluez dependency) |
| mobile platform-channel (Android USB-serial · BLE) | **flutter_mcp** (Flutter dependency) |
| pure-Dart network (ws · tcp) | either of the above two (dart:io · no FFI) |

mcp_bridge doubles as the transport↔transport bridge + the opt-in home for those transport impls. The host chooses between (a) reuse via the bridge (serial↔standard, core unchanged) or (b) injecting the transport directly (mcp_bridge export + kernel seam).

### When a Wholesale New Host Is Needed

The above is the case of *adding a new wire under the existing mcp_client/server model*. When the connection model itself differs (non-MCP client, IPC, in-memory bus, etc.), implement the `KernelClientHost` / `KernelServerHost` abstraction wholesale (0 dependency on mcp_server / mcp_client):

```dart
class MyTransportKernelClientHost implements KernelClientHost { /* connect · ... */ }
```

the host wires `KernelApp.boot(serverHostFactory: ..., clientHost: MyTransportKernelClientHost())`. = USB · IPC · in-memory bus · a user-arbitrary protocol all share the same path.

## 5. Adding a New Tool Kind

Add to `mcp_bundle`'s `ToolKind` enum (e.g. `wasm` · `embedded` · `cloud-2` · etc.) — realized example: **`ts`**, the compiled real-code tool of a `type: server` bundle; built and executed by the serving runtime, hosts have no executor and treat it as a declaration. Add a per-kind handler-composition clause inside the host's BundleActivation logic:

```dart
Future<RegistrationResult> registerTool(ToolEntry tool) async {
  switch (tool.kind) {
    case ToolKind.js: return _registerJsTool(tool);
    case ToolKind.mcp: return _registerMcpTool(tool);
    case ToolKind.wasm: return _registerWasmTool(tool);  // new kind
    ...
  }
}
```

Changing mcp_bundle's ToolKind = package domain (`packages/mcp_bundle`).

## 6. Adding a New Facade (knowledge category)

Add a facade in flowbrain_core (e.g. `documents` · `events` · `entities` · etc.). Impact = brain_kernel standardTools + the 8 facades expand to 9+ · BundleActivation's registration logic + the kb:// URI scheme + a per-facade wrapper.

= a large task (kernel domain — a `flowbrain_core` + `brain_kernel` cascade). Proceed only after explicit user instruction.

## Compatibility + Forward-Evolution Principles

- new instance = does NOT violate the existing framework spec
- new category / kind / facade = additive (preserves existing compatibility)
- manifest forward compat = unknown enum / partial load policy (the kernel's graceful handling)
- new composition pattern = host design's freedom (no change to the framework spec)

## Non-Goals

- per-instance product detail = each host's docs
- manifest schema detail = mcp_bundle
- this spec = **the extension interface + the standard procedure** only
