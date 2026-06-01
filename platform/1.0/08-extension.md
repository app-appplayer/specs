# 08 — Extension Standard

The standard procedure to follow when building a new instance (product, host, domain, etc.) on top of the framework. The actual entry point for the "**same framework, infinite instances**" framing.

## Extension Dimensions

| Dimension | What is added |
|---|---|
| New host product | UI surface + composition-pattern decision + KernelApp boot |
| New domain bundle | `.mbd` manifest authoring + category wiring |
| New LLM provider | Implement mcp_llm's LlmProvider or extend CustomLlmProvider |
| New transport | Implement KernelServerHost / KernelClientHost |
| New tool kind | bundle's ToolKind enum + host's handler-construction logic |
| New facade (knowledge) | Add a facade in flowbrain_core (kernel domain) |

Each only needs to follow the framework's **interface spec** (the specs in this directory).

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
    serverHostFactory: <ServerBootstrap.factory for server-only>,
    clientHost: <McpClientKernelHost() for client-only>,
    chatLogDir: <path>,
    bundleRegistryStorageDir: <path>,
  );
  ...
}
```

### Step 2 — endpoint + tool registration (composition-pattern decision)

| Use case | endpoint composition |
|---|---|
| Self-contained (client only) | `serverHostFactory: null` → InProcessKernelServerHost · no external transport |
| External exposure (server only · server+client) | `serverHostFactory: ServerBootstrap.factory` + `ep.start(KernelTransportKind.streamableHttp, port)` |
| External invocation (client only · server+client) | `clientHost: McpClientKernelHost()` + `app.clientHost.connect(...)` |

Tool registration:
- host home standard 45 tools = `ep.addStandardTools(app)`
- host's own tools = `HostToolRegistry.registerExposed(bundleId: 'host', rawName: ...)`

### Step 3 — bundle activation wiring

```dart
Future<void> activateBundle(McpBundle bundle) async {
  // 6 facade assets
  await BundleActivation(system: app.system, bundleId: bundle.id).activate(bundle);

  // domain tools (manifest.tools.tools[])
  for (final tool in bundle.manifest.tools.tools) {
    final handler = _buildHandlerForToolEntry(tool);  // host constructs handler per kind
    hostToolRegistry.registerExposed(
      bundleId: bundle.id,
      rawName: tool.name,
      description: tool.description ?? '',
      inputSchema: tool.inputSchema,
      handler: handler,
    );
  }

  // knowledge tools' domain access (optional · when a domain needs access to its own knowledge)
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

Outside the framework spec — host's discretion. Only the following contracts:
- UI MUST NOT call facades directly
- tool action = via dispatcher or endpoint
- resource = bridge.readResource (kb:// URI)
- chrome wiring = manifest.wiring slot wiring

## 2. Building a New Domain Bundle

### Step 1 — follow the manifest spec

The [`../bundle/`](../bundle/) schema. The category table in [`02-bundle-interface.md`](02-bundle-interface.md).

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

### Step 2 — author per-category assets

- Own domain tools (manifest.tools.tools[]) — JS / external · name `own.namespace` (not `bk.`)
- Own agent definitions
- Own 4-axis assets (facts · skills · profiles · philosophies)
- Own operational logic (flows · pipelines · runbooks)
- Own UI (mcp_ui_dsl) + wiring (chrome slot wiring)
- settings + chat slash commands

### Step 3 — compatibility gate

```json
"requires": {
  "kernelVersion": "^0.1.0",
  "uiSpec": "mcp_ui_dsl@1.3",
  "builtinAtoms": ["text@1.3", ...]
}
```

= host validates before activation. On incompatibility, reject + notify the user.

### Step 4 — packing

`mcpb`'s packager (mcpb_packager) → `.mbd` directory or `.mcpb` compressed format. The host's BundleSourcePort fetches it.

## 3. Building a New LLM Provider

### Step 1 — options

| Choice | path |
|---|---|
| Add a native provider to mcp_llm (beyond Anthropic / OpenAI / Gemini / Cohere / Mistral / Groq / Bedrock / VertexAI) | Implement mcp_llm's LlmProvider + register factory |
| CLI subprocess wiring (Claude Code · gemini CLI · ollama · etc.) | Extend `CustomLlmProvider` + `mcp_llm.registerProvider('name', factory)` |
| Private wrapper | `CustomLlmProvider` or own LlmProvider |

### Step 2 — recipe location

The `brain_kernel/recipes/<vendor>/` pattern. First case = `brain_kernel/recipes/claude_code/` (Claude Code subscription provider). Self-contained:
- `lib/<vendor>_provider.dart` — provider + factory
- `bin/main.dart` — verifier
- `pubspec.yaml` — brain_kernel path + mcp_llm
- `README.md` — rationale · I/O normalization · verification

### Step 3 — host wiring

```dart
import 'package:my_recipe/my_provider.dart';

final factory = MyProviderFactory();
final provider = factory.createProvider(LlmConfiguration(...));
final adapter = LlmPortAdapter.fromInterface(modelId: 'my-model', provider: provider);

await KernelApp.boot(
  llmProviders: {'my-model': adapter, ...},
);

// auto-routed when an agent's ModelSpec.id == 'my-model'
```

## 4. Building a New Transport

An abstract implementation of `KernelServerHost` or `KernelClientHost`. Zero dependency on mcp_server / mcp_client:

```dart
class MyTransportKernelServerHost implements KernelServerHost {
  // addTool · addResource · callTool · start · shutdown · etc.
}

class MyTransportKernelClientHost implements KernelClientHost {
  // connect · ...
}
```

The host wires it via `KernelApp.boot(serverHostFactory: MyTransportKernelServerHost.factory, clientHost: MyTransportKernelClientHost())`.

= USB · IPC · in-memory bus · arbitrary user protocol all follow the same path.

## 5. Adding a New Tool Kind

Add to `mcp_bundle`'s `ToolKind` enum (e.g. `wasm` · `embedded` · `cloud-2`). Add a per-kind handler-construction clause to the host's BundleActivation logic:

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

Changing mcp_bundle's ToolKind = package domain (the `mcp_bundle` package).

## 6. Adding a New Facade (Knowledge Category)

Add a facade in flowbrain_core (e.g. `documents` · `events` · `entities`). Impact = brain_kernel standardTools + 8 facades expanding to 9+ · BundleActivation's registration logic + kb:// URI scheme + per-facade wrapper.

= a large task (kernel domain — `flowbrain_core` + `brain_kernel` cascade).

## Compatibility + Forward-Evolution Principles

- New instance MUST NOT violate existing framework spec
- New category / kind / facade = additive (preserves existing compatibility)
- Manifest forward compat = unknown-enum / partial-load policy (kernel's graceful handling)
- New composition pattern = host design freedom (no framework spec change)

## Non-Goals

- Per-instance product detail = each host's docs
- Manifest schema detail = mcp_bundle
- This spec = **extension interface + standard procedure** only
