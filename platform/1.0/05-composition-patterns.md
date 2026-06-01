# 05 — Composition Patterns

The unbounded evolution of the framework is a change in **composition patterns**. Same layers (UI · bundle · kernel) · same interface (manifest spec) · different composition = different product.

## Composition Dimensions

| Dimension | Options |
|---|---|
| **Bundle source** | (a) local `.mbd` file (b) remote MCP server (c) both |
| **UI source** | (a) host's own chrome (b) bundle's manifest.ui (c) UI resource of a remote MCP server |
| **Tool source** | (a) host's own (b) bundle manifest.tools (c) external MCP server (d) kernel standardTools |
| **transport** | (a) in-process only (b) operate a server (c) operate a client (d) both |
| **Bundle count** | (a) single (b) multiple |
| **Agent composition** | (a) single agent (b) manager + N specialist · fan-out · mediated by fact_graph |

Each dimension × dimension = a composition pattern.

## 5 Standard Patterns

### Pattern 1 — server-serve (thin client)

```
[server (host)]
  ├── KernelApp + bundle activation
  ├── serve UI resources (manifest.ui)
  ├── serve tools (manifest.tools)
  └── external transport (operate a server)
              ↑ MCP
[client (host)]
  └── thin player — download UI resources + render + forward tool calls
```

**Example** — AppPlayer connects to an external MCP server → uses the server's UI / tools. AppPlayer = thin client.

### Pattern 2 — client-only (locally self-contained)

```
[host (single process)]
  ├── KernelApp + bundle activation (local .mbd)
  ├── own UI (host chrome + manifest.ui mount)
  ├── own tools (host registry)
  ├── external client (optional) — call another server
  └── no external server (no transport)
```

**Example** — the default mode of AppPlayer Standard (`serverHostFactory: null`). Plays a bundle within its own process.

### Pattern 3 — hybrid (local + remote composition)

```
[host (single process)]
  ├── KernelApp + local bundle activation
  ├── own UI (manifest.ui mount)
  ├── own tools (HostToolRegistry)
  ├── external client — connect to remote MCP server (tool / knowledge access)
  └── (optional) own server endpoint — accessible by other clients
```

**Example** — vibe_studio = local bundle + serve own endpoint + call external LLM / MCP server.

### Pattern 4 — multi-bundle (cross-domain · single host)

```
[host]
  ├── KernelApp
  ├── bundle_A activation (domain A)
  ├── bundle_B activation (domain B)
  ├── ...
  └── cross-domain communication = no direct bundle mediation — async via fact_graph
```

**Example** — vibe_studio's multi-tab (each tab = a separate bundle activation). Or an industrial HMI's multi-domain (multiple devices + multiple operational areas).

### Pattern 5 — multi-server (federation)

```
[host (orchestrator)]
  ├── KernelApp + local bundle
  ├── external client_A → server_1 (knowledge domain)
  ├── external client_B → server_2 (operations domain)
  ├── external client_C → server_3 (LLM provider)
  └── result composition (host holds the composition logic)
```

**Example** — enterprise environment — a single host composes per-department MCP servers (HR · finance · operations · knowledge) → a unified workflow.

## Per-Dimension Pattern Matrix (illustrative)

| Pattern | Bundle source | UI source | Tool source | transport | Bundle count |
|---|---|---|---|---|---|
| AppPlayer Standard | local or server | host chrome + manifest | host + manifest + standard | client only | single |
| AppPlayer Pro | multiple (library) | host chrome + manifest | host + manifest + standard | client (+ optional server) | multi |
| vibe_studio | local (multiple) | host chrome + manifest (workspace) | host + manifest + standard | server + client | multi |
| FlowBrain | domain knowledge (single/multiple) | host explorer | standard-oriented | server or in-process | single or multi |
| Form Studio | template (single) | form view (own) | mcp_form + standard | in-process | single |
| Industrial HMI | devices + operations (multiple) | HMI panel (own) | mcp_io + manifest + standard | client (device server) + optional server | multi |
| Headless CLI | local or server | text · stream | manifest + standard | client or in-process | single or multi |
| **User-defined** | **free** | **free** | **free** | **free** | **free** |

## Behavioral Differences (per-pattern specifics)

### Flow of Pattern 1 (server-serve)

```
1. server host = KernelApp.boot · bundle activation · endpoint.start(streamableHttp)
2. client host = KernelApp.boot · clientHost.connect(server URL)
3. client = query the server's tools/list · resources/list
4. client UI = render the server's manifest.ui (fetch resources via UiResourcePort)
5. client tool call = clientHost.connection.callTool(name, args) → server endpoint dispatch
```

### Flow of Pattern 2 (client-only)

```
1. host = KernelApp.boot (serverHostFactory null = InProcessKernelServerHost default)
2. host = local bundle activation (BundleActivation + HostToolRegistry)
3. UI tool action = dispatcher.callInProcess(<exposedName>, args)
4. agent ask = AgentChatController.sendUser → KnowledgeSystem.agents.ask
```

### Flow of Pattern 3 (hybrid)

```
1. host = KernelApp.boot (both server + client)
2. host = local bundle activation
3. host endpoint = operate a server (expose own tools)
4. host client = connect to external MCP server (tool / knowledge access)
5. UI tool action = try host dispatcher first · otherwise try external client
6. agent = use own LLM (or external LLM provider)
```

### Flow of Pattern 4 (multi-bundle)

```
1. host = KernelApp.boot
2. each bundle activation = BundleSessionBridge.openSession(activation, master: false)
3. bridge = per-session alias publication (bk.<bundleId>.<rest>)
4. active session per UI tab = scopeId applied automatically (own bundle context)
5. cross-bundle communication = mediated by fact_graph (another bundle queries `bk.fact.query` results)
```

### Flow of Pattern 5 (multi-server)

```
1. host = KernelApp.boot + clientHost wiring
2. host.clientHost.connect(server_1) · connect(server_2) · ...
3. composition of each server's tools/list = host's tool catalog
4. host orchestrator = LLM agent decides cross-server calls
5. result = host composes into its own fact_graph or a temporary store
```

## Boundaries Between Patterns

- A single host **MAY use multiple patterns simultaneously** — free composition of Pattern 3 (hybrid) + Pattern 4 (multi-bundle), etc.
- Changing a pattern = changing only the host's boot wiring + bundle activation (zero change to kernel · bundle spec)
- A new pattern = the host's new composition logic. No change to framework spec.

## Non-Goals

- Product implementation detail of a specific pattern = each host's docs
- transport wire format = mcp_server / mcp_client spec
- This spec = **composition dimensions + standard patterns + behavioral flows** only
