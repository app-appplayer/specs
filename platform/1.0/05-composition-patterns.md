# 05 — Composition Patterns

The framework's unbounded evolution = changes in **composition patterns**. Same layer (UI · bundle · kernel) · same interface (manifest spec) · different composition = different product.

## Composition Dimensions

| Dimension | Options |
|---|---|
| **bundle source** | (a) local `.mbd` file (b) remote MCP server (c) both |
| **UI source** | (a) host's own chrome (b) the bundle's manifest.ui (c) a remote MCP server's UI resources |
| **tool source** | (a) host's own (b) bundle manifest.tools (c) external MCP server (d) kernel standardTools |
| **transport** | (a) in-process only (b) running a server (c) running a client (d) both |
| **bundle count** | (a) single (b) multiple |
| **agent composition** | (a) single agent (b) manager + N specialists · fan-out · mediated by fact_graph |

Each dimension × dimension = a composition pattern.

## 5 Standard Patterns

### Pattern 1 — server-serve (thin client)

```
[server (host)]
  ├── KernelApp + bundle activated
  ├── serve UI resources (manifest.ui)
  ├── serve tools (manifest.tools)
  └── external transport (running a server)
              ↑ MCP
[client (host)]
  └── thin player — download UI resources + render + forward tool calls
```

**Example** — AppPlayer connects to an external MCP server → uses the server's UI / tools. AppPlayer = thin client.

### Pattern 2 — client-only (locally self-contained)

```
[host (single process)]
  ├── KernelApp + bundle activated (local .mbd)
  ├── UI itself (host chrome + manifest.ui mount)
  ├── tools itself (host registry)
  ├── external client (optional) — calls another server
  └── no external server (no transport)
```

**Example** — AppPlayer Standard's default mode (`serverHostFactory: null`). Plays bundles within its own process.

### Pattern 3 — hybrid (local + remote composition)

```
[host (single process)]
  ├── KernelApp + local bundle activated
  ├── UI itself (manifest.ui mount)
  ├── tools itself (HostToolRegistry)
  ├── external client — connects to a remote MCP server (tool / knowledge access)
  └── (optional) own server endpoint — another client can access
```

**Example** — vibe_studio = local bundle + serving its own endpoint + calling external LLM / MCP servers.

### Pattern 4 — multi-bundle (cross-domain · one host)

```
[host]
  ├── KernelApp
  ├── bundle_A activated (domain A)
  ├── bundle_B activated (domain B)
  ├── ...
  └── cross-domain communication = not direct via bundle mediation — asynchronous via fact_graph
```

**Example** — vibe_studio's multi-tab (each tab = a separate bundle activated). Or industrial HMI's multi-domain (multiple devices + multiple operational areas).

### Pattern 5 — multi-server (federation)

```
[host (orchestrator)]
  ├── KernelApp + local bundle
  ├── external client_A → server_1 (knowledge domain)
  ├── external client_B → server_2 (operational domain)
  ├── external client_C → server_3 (LLM provider)
  └── result synthesis (host = composition logic)
```

**Example** — enterprise environment — a host composes per-department MCP servers (HR · finance · operations · knowledge) → integrated workflow.

## Per-Dimension Pattern Matrix (examples)

| Pattern | bundle source | UI source | tool source | transport | bundle count |
|---|---|---|---|---|---|
| AppPlayer Standard | local or server | host chrome + manifest | host + manifest + standard | client only | single |
| AppPlayer Pro | multiple (library) | host chrome + manifest | host + manifest + standard | client (+ optional server) | multi |
| vibe_studio | local (multiple) | host chrome + manifest (workspace) | host + manifest + standard | server + client | multi |
| FlowBrain | domain knowledge (single/multiple) | host explorer | standard-centric | server or in-process | single or multi |
| Form Studio | template (single) | form view (own) | mcp_form + standard | in-process | single |
| industrial HMI | devices + operations (multiple) | HMI panel (own) | mcp_io + manifest + standard | client (device server) + optional server | multi |
| headless CLI | local or server | text · stream | manifest + standard | client or in-process | single or multi |
| **user-arbitrary** | **free** | **free** | **free** | **free** | **free** |

## Behavioral Differences (per-pattern specifics)

### Flow of Pattern 1 (server-serve)

```
1. server host = KernelApp.boot · bundle activated · endpoint.start(streamableHttp)
2. client host = KernelApp.boot · clientHost.connect(server URL)
3. client = queries the server's tools/list · resources/list
4. client UI = renders the server's manifest.ui (fetches resources via UiResourcePort)
5. client tool call = clientHost.connection.callTool(name, args) → server endpoint dispatch
```

### Flow of Pattern 2 (client-only)

```
1. host = KernelApp.boot (serverHostFactory null = InProcessKernelServerHost default)
2. host = local bundle activated (BundleActivation + HostToolRegistry)
3. UI tool action = dispatcher.callInProcess(<exposedName>, args)
4. agent ask = AgentChatController.sendUser → KnowledgeSystem.agents.ask
```

### Flow of Pattern 3 (hybrid)

```
1. host = KernelApp.boot (both server + client)
2. host = local bundle activated
3. host endpoint = running a server (exposes its own tools)
4. host client = connects to an external MCP server (tool / knowledge access)
5. UI tool action = tries the host dispatcher first · if absent, tries the external client
6. agent = uses its own LLM (or an external LLM provider)
```

### Flow of Pattern 4 (multi-bundle)

```
1. host = KernelApp.boot
2. each bundle activation = BundleSessionBridge.openSession(activation, master: false)
3. bridge = per-session alias publication (bk.<bundleId>.<rest>)
4. per-UI-tab active session = scopeId applied automatically (its own bundle context)
5. cross-bundle communication = mediated by fact_graph (another bundle queries the result of `bk.fact.query`)
```

### Flow of Pattern 5 (multi-server)

```
1. host = KernelApp.boot + clientHost wiring
2. host.clientHost.connect(server_1) · connect(server_2) · ...
3. the combined tools/list of each server = the host's tool catalog
4. host orchestrator = the LLM agent decides cross-server calls
5. result = the host synthesizes into its own fact_graph or a temporary store
```

## Boundaries Between Patterns

- One host can **use multiple patterns simultaneously** — free combinations such as Pattern 3 (hybrid) + Pattern 4 (multi-bundle)
- Changing a pattern = changing only the host's boot wiring + bundle activation (zero change to kernel · bundle spec)
- A new pattern = new composition logic on the host. No change to the framework spec.

## Ecosystem App Embedding (host-agnostic capability seam)

If the 5 patterns above are *kernel · bundle · transport* composition, this is a different axis — **a host embeds a published ecosystem Flutter app as a widget** and fills the per-host differences via an **injection interface**. The first · canonical instance = AppPlayer **Marketplace** (`_meta/APPPLAYER_BASIS.md` §1 "the supply branches entering the stage").

### Principle

- An ecosystem app (e.g. `marketplace_app`) is an **embedded library** that does not `runApp`. Its public surface = the embed widget + backend injection (config) + **host capability injection (`HostCapabilities`)** + capability gates (`can*`).
- The core / FEAT are **host-unaware.** Per-host differences (running a bundle · connecting to a server · payment · token) are all injected via the `HostCapabilities` implementation.
- An unsupported capability = not a throw but a `can*=false` **gate** (UI disabled/hidden). → A single app codebase covers the capability spectrum of native host · web, etc. with one code path.

### Rules (conformance — common to all consumer hosts)

1. **Every consumer host (AppPlayer Pro · Studio · web …) embeds via the same seam.** Widget embed + injection of config · `HostCapabilities`. The per-host branch = a single `HostCapabilities` implementation, no branching in the app code. → "Studio uses it the same way as Pro" is not a coincidence but a contract.
2. **The host implementation lives in each host's tree.** Do not put the host implementation in the ecosystem app package (preventing reverse dependency · work-tree pollution). The app package = interface + minimal default (web) only.
3. **A host lacking a capability = `can*=false` gate.** (e.g.: IAP on a desktop authoring host.) When there is no session the token provider = null (public route).
4. When a host composition change creates a new capability, only that host's `HostCapabilities` implementation is updated — no change to the app · other hosts.
5. The backend is **consumed** (not reinvented) — the source of truth for settlement · entitlement = saas. The backend origin is injected via config.

### Authoritative Contract

This spec defines only the *pattern · rules*. The **single source of truth** for the interface · implementation table · IAP `accountToken` · payment flow = `saas_app/marketplace/app/docs/03_DDD/host-capabilities.md` (code `marketplace_app/lib/src/host.dart`). The topology of the supply branches = `_meta/APPPLAYER_BASIS.md` §1.

### Instances (current — summary, detail in the authoritative contract)

| consumer host | runBundle | connectServer | payInApp | authToken | implementation location |
|---|---|---|---|---|---|
| AppPlayer Pro | ✓ `appplayer_core` | ✓ `appplayer_core` | ✓ IAP | host session | AppPlayer tree |
| Studio | ✓ kernel bundle | ✓ kernel `clientHost` | gateable | host session / null | Studio tree |
| web | gate | gate | gate (card = saas SDK) | web login | app package default |

## Non-Goals

- Product implementation detail of a specific pattern = each host's docs
- transport wire format = the mcp_server / mcp_client spec
- This spec = **composition dimensions + standard patterns + behavioral flows** only
