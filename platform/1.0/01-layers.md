# 01 — 3 Layer Structure

MakeMind framework = the three abstract layers **UI · bundle · kernel**. Each has clearly separated responsibilities + standardized composition points.

## Layer 1 — UI

**Responsibility** — everything in the user surface:
- chrome (titlebar · statusbar · sidebar · launcher and other host-specific frame)
- runtime (mcp_ui_dsl · own widget · agent chat panel · form view, etc.)
- widget · DSL · rendering
- input capture · result exposure
- external communication (HTTP · WebSocket · stdio and other transports)

**Contract**:
- UI wires the **bundle's manifest.ui** + **manifest.wiring** (apart from the host's own chrome)
- UI MUST NOT call the **kernel facade** directly — it goes through the dispatcher or an endpoint
- UI surface is free (Flutter · web · embedded · voice · AR, etc.)

**Implementation examples**:
- vibe_studio's chrome + DSL workspace + multi-tab
- AppPlayer's player chrome + launcher + library
- external MCP client (Claude Desktop · user CLI)

## Layer 2 — Bundle

**Responsibility** — the conformance interface. Every category inside the `.mbd` manifest:

```
bundle.manifest = {
  id, name, version, requires,        ← identity + compatibility
  tools.tools[],                       ← domain's own logic tools
  agents,                              ← agent definition
  skills, profiles, philosophies,      ← 4 axis
  facts,                               ← knowledge assets
  flows, pipelines, runbooks,          ← operational logic
  ui,                                  ← UI definition (mcp_ui_dsl)
  wiring.{lifecycle,domainActions,    ← chrome slot wiring
          titlebar,statusbar,
          settings,chat},
  settings.sections[],                 ← settings dialog spec
  chat.slashCommands[],                ← slash commands
}
```

**Contract**:
- Bundle = **self-contained** — behaves through the same interface on any host
- The bundle author only needs to follow the **manifest spec** (implementation is up to the host)
- The bundle tool name uses **its own namespace** (MUST NOT start with `bk.*` — that is the knowledge tool space)

**Implementation spec**: the `mcp_bundle` package schema. See [`02-bundle-interface.md`](02-bundle-interface.md).

## Layer 3 — Kernel (Operations)

**Responsibility** — operational logic + 8 facades + standard tools + tool registration paths:

```
kernel layer:
  ├── KnowledgeSystem (flowbrain_core)
  │   └── 8 facade — facts · skills · profiles · philosophies · workflows · pipelines · runbooks · agents
  │   └── 4 axis — Profile · Skill · Fact · Philosophy (4-axis prompt composition)
  │   └── agent layer (self-contained · each agent has its own LLM context)
  │
  ├── standardTools (41) — bk.<facade>.<verb> standard tool wrappers
  │
  ├── tool registration path (3 kinds — see 06-tool-registry.md)
  │   ├── addStandardTools(app) — host home knowledge tools
  │   ├── BundleSessionBridge.registerTool — knowledge tool domain wrapping
  │   └── HostToolRegistry.registerExposed — general tools (host + domain logic)
  │
  ├── per-agent capability scoping (10-agent-scoping.md)
  │   ├── KernelApp.toolsForAgent(agentId, {role, explicitAllowlist?, bundleId?})
  │   │   └── role-specific default subset (manager bk.* delegation + read · worker per-bundle · reviewer read-only)
  │   └── manifest.agents[i].tools allowlist override (host-specific namespace explicit)
  │
  ├── MCP surface wrapper (3 kinds — KernelServerHost abstraction)
  │   ├── addTool / removeTool / toolDefinitions
  │   ├── addResource / removeResource
  │   └── addPrompt / removePrompt / promptDefinitions (KernelPrompt* wrapper)
  │
  ├── BundleSessionBridge — 4 standard channel API · session · scopeId · alias publication
  ├── BundleActivation — registers 6 facade assets (skills/profiles/philosophies/facts/flows/agents)
  ├── 4 ports — ConfigPort · UiResourcePort · ObservabilityPort · BundleSourcePort
  ├── LLM pool — AgentLlmSessions (LlmPortAdapter per modelId · live UnmodifiableMapView)
  │   └── providerName getter (UI banner · post-boot swap consistency)
  ├── KernelApp.hostMcpServerSpec({endpointLabel}) — external dial-back spec (multi-endpoint safe)
  └── Host adapters — KernelServerHost · KernelClientHost (mcp-dependency-0 abstraction)
```

**Contract**:
- Kernel = **headless** — zero UI dependency
- Kernel = **abstracts the MCP dependency** — mcp_server / mcp_client are reference impl only (injected by the host)
- Kernel = **single source** — shares operational logic across all instances

**Implementation spec**: the `brain_kernel` package. See [`03-kernel-runtime.md`](03-kernel-runtime.md).

## Composition points (inter-layer interfaces)

### UI ↔ bundle

- UI renders the DSL / own-widget mapping of `manifest.ui`
- UI wires the chrome slots (titlebar · statusbar · domainActions · settings, etc.) of `manifest.wiring.*`
- UI activates the slash commands of `manifest.chat.slashCommands`

### UI ↔ kernel

- UI **MUST NOT call the facade directly** — via dispatcher / endpoint
- UI tool action (mcp_ui_dsl `tool` action) = dispatcher's `callInProcess` or endpoint's `callTool`
- UI resource (mcp_ui_dsl `resource` action) = bridge's readResource via the `kb://<facade>/<id>` URI

### bundle ↔ kernel

- On bundle activation the host wires each manifest category into the kernel layer:
  - 6 facade assets → `BundleActivation.activate(bundle)` (skills/profiles/philosophies/facts/flows/agents)
  - tools (manifest.tools.tools[]) → `HostToolRegistry.registerExposed` (general tools)
  - knowledge tool exposure (kernel 41) → `BundleSessionBridge.registerTool` (domain wrapping)
  - **agents[i].tools allowlist (optional)** → `AgentHost.toolsFor` forwards to `KernelApp.toolsForAgent(role, explicitAllowlist: ...)` (per-agent scoping · 10-agent-scoping.md)
  - 4 ports → injected by the host at boot

### bundle ↔ bundle (cross-bundle)

- No direct composition — asynchronous via fact_graph (per raw 2026-05-23 §6)
- On external call = `bk.<bundleId>.<rest>` alias (isolation)

### host ↔ external (MCP)

- Host = free combination of MCP server (external LLM / client connects) + MCP client (calls another server)
- 4 modes — server only · client only · both · neither (in-process)
- KernelServerHost / KernelClientHost abstraction = the host is free to choose the wire library

## Responsibility cross-ref

| layer | canonical specification | reference implementation |
|---|---|---|
| UI | [`../ui_dsl/`](../ui_dsl/) | the `mcp_ui` runtime · host chrome |
| bundle | [`../bundle/`](../bundle/) | the `mcp_bundle` package · `.mbd` manifest (user space) |
| kernel | [`03-kernel-runtime.md`](03-kernel-runtime.md) | `brain_kernel` · `flowbrain_core` |
