# 04. Tools Section

The `tools` section is the bundle's declaration of **host-callable
units of behavior**. Every UI button, every chat command, every
wiring slot ultimately resolves to a tool name. The host MCP server
registers each declared tool under `<bundleNamespace>.<name>` and
makes it callable from external MCP clients, the same host's other
bundles, the chat LLM, and the bundle's own UI / wiring.

Reference, in the `mcp_bundle` package: `lib/src/models/tools_section.dart`.

## 4.1 `ToolsSection`

```json
"tools": {
  "schemaVersion": "1.0.0",
  "tools": [ ToolEntry, ToolEntry, ... ]
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `schemaVersion` | string | no (default `"1.0.0"`) | Section schema version. |
| `tools` | `ToolEntry[]` | no (default `[]`) | Tool declarations. |

The section may carry tool entries inline under `tools.tools[]` and /
or split them across files inside the `tools/` reserved folder (see
[`07_Resources.md`](07_Resources.md) §7.1). Folder content takes
precedence on id conflict. The `tools/` folder also carries the
bundled JS source files referenced by `kind: js` tools — see §4.7.

## 4.2 `ToolEntry`

```json
{
  "id": "newProject",
  "name": "studio_builder.newProject",
  "description": "Create a new package project.",
  "kind": "js",
  "target": { "entry": "tools/newProject.js", "fn": "newProject" },
  "inputSchema": {
    "type": "object",
    "properties": {
      "name":   { "type": "string" },
      "parent": { "type": "string" }
    },
    "required": ["name"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "ok":   { "type": "boolean" },
      "path": { "type": "string" }
    },
    "required": ["ok"]
  },
  "agentScope": ["studio.manager", "studio.ui_implementer"],
  "metadata": {}
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string? | no | Optional cross-reference id. Distinct from `name` — `name` is the dispatch handle, `id` is a stable handle for cross-section reference. |
| `name` | string | **MUST** | Non-empty. The handle the host uses to invoke this tool. Conventionally namespaced (`<bundle>.<verb>`). |
| `description` | string? | no | Human-readable description. |
| `kind` | enum [`ToolKind`](#43-toolkind) | no (default `host`) | Dispatch discriminator. |
| `target` | object | no (default `{}`) | Kind-discriminated dispatch payload — see §4.4 — §4.7. |
| `inputSchema` | object? | no | JSON Schema (or MCP `inputSchema`) describing the tool's input. Free-form map — the validator does not enforce dialect. |
| `outputSchema` | object? | no | JSON Schema describing the tool's response shape. Free-form map (same dialect-agnostic policy as `inputSchema`). When present, hosts MAY use it to validate responses or to drive `mergeState` auto-merge integrity checks (see [`mcp_ui_dsl 1.3` §3.10](../../ui_dsl/1.3/03_Data_Binding.md)). |
| `agentScope` | string[]? | no | Agent ids / role names that may invoke this tool. `null` = unscoped (host-default access). |
| `metadata` | object | no (default `{}`) | Host-specific extension. |

The validator's checks for `tools[]` are intentionally **lenient**:

1. `name` MUST be non-empty.
2. `name` MUST be unique within the section.
3. `kind` MUST NOT round-trip as `unknown` (forward-compat discrimination
   only, not a valid input).

It does **not** enforce kind-specific `target` shape — hosts get the map
as-is and decide how to dispatch.

## 4.3 `ToolKind`

```
host | mcp | cloud | js | unknown
```

| Kind | Dispatched by | `target` shape | Typical use |
|------|---------------|----------------|-------------|
| `host` | Host runtime in-process | `{}` (empty) | Host builtin tools (`studio.*`). The host already knows the dispatch path. |
| `mcp` | External MCP server | `{ "transport": "http" \| "stdio", "url"?, "command"?, "args"? }` | Outbound MCP connections — remote HTTPS servers, local stdio binaries. |
| `cloud` | External HTTPS endpoint | `{ "url": "https://..." }` | SaaS / cloud APIs. |
| `js` | Bundled JavaScript executed in `flutter_js` | `{ "entry": "tools/foo.js", "fn": "foo" }` | The bundle's own scripts — most domain tools. |
| `unknown` | — | — | Forward-compat round-trip slot for unrecognized `kind` strings. Validator rejects. |

`kind` is parsed case-insensitively (`HOST` → `host`).
`wireValue` round-trips the canonical lowercase string.

Most domain bundles populate `tools[]` with `js` entries plus a few
`mcp` references. `host` is reserved for host-builtin declarations
(domain bundles do not declare those — the host registers them
itself). `cloud` is a thin wrapper over an HTTPS call.

## 4.4 `kind: host` — Host Builtin

```json
{ "name": "studio.workspace.save", "kind": "host", "target": {} }
```

The host already knows how to dispatch built-in names. The bundle MAY
re-declare a host builtin to put it in its `tools[]` for documentation
or to feed `wiring`/`chat` references — most bundles do not, and the
host SHOULD accept references to undeclared host tools as long as they
appear in the host's tool registry.

## 4.5 `kind: mcp` — External MCP Server

```json
{
  "name": "github.create_issue",
  "kind": "mcp",
  "target": {
    "transport": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"]
  }
}
```

```json
{
  "name": "remote.search",
  "kind": "mcp",
  "target": {
    "transport": "http",
    "url": "https://mcp.example.com"
  }
}
```

| `target` field | Type | Description |
|----------------|------|-------------|
| `transport` | string | `"http"` (Streamable HTTP) or `"stdio"`. |
| `url` | string? | Required for `transport: http`. |
| `command` | string? | Required for `transport: stdio` — executable path. |
| `args` | string[]? | Optional argv. |

The MCP wire protocol (initialize / tools/list / tools/call) is
unchanged by this spec — the bundle only declares the connection.

## 4.6 `kind: cloud` — HTTPS Endpoint

```json
{
  "name": "weather.lookup",
  "kind": "cloud",
  "target": { "url": "https://api.example.com/weather" }
}
```

| `target` field | Type | Description |
|----------------|------|-------------|
| `url` | string | HTTPS endpoint. The host POSTs the tool's input as JSON and consumes the response. |

Authentication, retry, and request shaping are host-defined. A future
spec patch may standardize an `auth` sub-object.

## 4.7 `kind: js` — Bundled Script

```json
{
  "name": "notes.export",
  "kind": "js",
  "target": { "entry": "tools/export.js", "fn": "exportAll" }
}
```

| `target` field | Type | Description |
|----------------|------|-------------|
| `entry` | string | Bundle-root-relative path. Conventionally inside the `tools/` reserved folder (e.g. `tools/<name>.js`). Authors MAY use any in-bundle path; the host resolves the string against the bundle root. |
| `fn` | string | The exported function name within `entry`. The host calls `fn(args)`. |

The host loads `entry` from the bundle filesystem and invokes `fn`
with the validated `args` object.

### 4.7.1 JS Function Contract

```js
// tools/export.js
async function exportAll(args) {
  args = args || {};
  const format = args.format || 'json';

  // Read from bundle filesystem
  const data = await host.fs.read('data/notes.json');

  // Call another MCP tool
  const path = await host.mcp.callTool('studio.fs.write', {
    path: 'export.' + format,
    content: data,
  });

  // Notify the user
  await host.ui.notify('Exported.', 'success');

  return { ok: true, path: path.body };
}
```

Inputs:

- `args` is the JSON object the caller passed, validated against
  `inputSchema` when present. Missing required fields cause the host
  to refuse the call before `fn` runs.

Outputs:

- Any JSON-serializable value. Conventionally `{ok: true|false,
  ...payload}`.
- The host's MCP layer wraps the return value as the standard MCP
  `CallToolResult` (`{ body, isError }`).
- When the caller is `mcp_ui_runtime`, the [§3.10 auto-merge in
  `mcp_ui_dsl 1.3`](../../ui_dsl/1.3/03_Data_Binding.md)
  may merge top-level keys of the return value into runtime state —
  authors design return shapes accordingly.

Errors:

- `throw` propagates to the host as a tool error. The MCP `isError`
  flag is set on the wrapped result.

### 4.7.2 Execution Isolation

JS runs inside `flutter_js` (in-process JS interpreter). Each bundle
gets an **isolated context** — globals from one bundle do not leak
into another. The only host-provided global is `host` (next section).

## 4.8 JS Host Bridge — `host.<atom>.<verb>`

The `host` global is the JS tool's only path back to host capabilities.
It is structured by **atom** (coarse capability category) and
**verb** (the operation).

```
host.<atom>.<verb>(args) -> Promise<value>
```

### 4.8.1 Atom Roster

The reference Studio host advertises:

| atom | Primary verbs | Use |
|------|--------------|-----|
| `mcp` | `callTool(name, args)` | **Most important.** Call any other MCP tool — host builtin, this bundle's own tools, other active bundles' tools. The single funnel. |
| `fs` | `read`, `write`, `list`, `mkdir`, `exists`, `delete` | Bundle-scope file I/O. Paths are relative to the bundle root. |
| `workspace` | `save`, `undo`, `redo`, `revert`, `history` | Workspace-level controls. |
| `ui` | `notify(text, severity)`, `dialog(spec)`, `prompt(question)` | Host UI interactions. |
| `agent` | `dispatch(target, message)`, `set_model(...)` | Multi-agent routing. |
| `bundle` | `info()`, `activate(id)` | Bundle metadata + activation. |
| `kb` | `query`, `put`, `get`, `list`, `delete` | Knowledge / domain KV. |
| `bus` | `emit(topic, data)`, `subscribe(topic, fn)` | Domain event bus. |

The atom catalog is **host-defined**, not spec-defined. This spec
fixes the bridge shape (`host.<atom>.<verb>(args) -> Promise<value>`)
and the gating mechanism (`requires.builtinAtoms`); the host
advertises which atoms it implements.

### 4.8.2 Permission Gate

```json
"requires": { "builtinAtoms": ["mcp", "fs", "ui", "workspace"] }
```

Atoms that the bundle does NOT declare are **not exposed** to its JS.
Calling `host.kb.query(...)` from a bundle that did not declare `kb`
MUST produce a `host.kb is undefined` error at the JS level. This is
both a security gate and an explicit-intent declaration the user can
inspect before activation.

## 4.9 `host.mcp.callTool` — The Single Funnel

```js
const r1 = await host.mcp.callTool('studio_builder.gotoPage',
                                   { pageId: 'tools' });
const r2 = await host.mcp.callTool('studio.project.info', {});
const r3 = await host.mcp.callTool('app_builder.exportProject', {});
```

`callTool(name, args)` returns a `CallToolResult`-shaped object:

```js
{ body: <tool's return value>, isError: false }
```

Same-bundle, host-builtin, and cross-bundle calls all use this single
verb. The chat LLM uses the same MCP surface — JS tools and LLMs
share one tool table.

## 4.10 Wiring Reference

Tool **declarations** live in `tools[]`. Tool **invocation bindings**
(which UI slot calls which tool) live in `manifest.wiring.*` and
`manifest.chat.slashCommands[]`. See [`06_Wiring.md`](06_Wiring.md).

A wiring slot's `tool` field MAY name:

- a tool declared in this bundle's `tools[]`, or
- a host builtin tool name, or
- a tool declared in another active bundle.

The host resolves the name through its MCP tool registry at call
time. A wiring slot referencing an unresolved tool MUST fail at
activation (host error) or, less strictly, render the UI element in
a disabled state — host policy.

## 4.11 Conformance

A bundle MUST:

1. Declare `name` non-empty for every entry in `tools[]`.
2. Use unique `name` values within a single `tools[]`.
3. Use a known `kind` value (`host` / `mcp` / `cloud` / `js`).
4. For `kind: js`, ensure `target.entry` resolves to a file inside the
   bundle and `target.fn` exports a function.

A host MUST:

1. Refuse to activate a bundle with duplicate `name` or `kind:
   unknown` entries.
2. Reject JS host-bridge calls to atoms not declared in
   `manifest.requires.builtinAtoms[]`.
3. Surface tool errors (`throw` in JS / non-2xx in cloud / non-zero
   stdio in mcp) as MCP `isError: true` results.

A host MAY:

- Cache `kind: js` script compilation across calls within an
  activation.
- Coalesce identical `kind: cloud` requests.
- Apply a per-tool timeout from a host policy.
