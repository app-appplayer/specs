# 06. Wiring

The **wiring surface** binds the tools a bundle declares (in
[`tools[]`](04_Tools.md)) to the host chrome that the user actually
sees and clicks. It is split across three top-level sections plus a
`chat` section:

```
wiring.domainActions[]   ← row-2 domain icons
wiring.lifecycle[]       ← system slots (project/edit/history/...)
wiring.settings[]        ← settings dialog actions (alongside top-level `settings.sections[]`)
chat.slashCommands[]     ← `/` composer chips
chat.agent               ← optional default chat agent id
settings.sections[]      ← user-facing settings form schema (top-level)
```

### Location: top-level (canonical) + manifest-nested (legacy alias)

`wiring`, `chat`, and `settings` are **top-level sections** of the
bundle JSON (siblings of `manifest`, `tools`, `knowledge`, …),
mirroring the dual-location pattern documented for
[`requires`](02_Manifest.md#27-portability-contract--requires).
Authoring tools and runtime hosts MUST emit them at the top level.

For backward compatibility with bundles that previously embedded
these sections inline as `manifest.wiring` / `manifest.chat` /
`manifest.settings`, loaders SHOULD accept both positions —
top-level wins on collision. New bundles MUST NOT use the nested
position. The model layer surfaces the canonical top-level slots
(`McpBundle.wiring` / `McpBundle.chat` / `McpBundle.settingsSection`).

This surface is **defined by the Studio host model** and standardized
here so any host that wants to host bundle UI can implement the same
slots.

A host that does not implement the relevant chrome MAY ignore the
entire wiring surface — bundles remain conformant. Bundles MAY
carry both `wiring` for chrome hosts and arbitrary `extensions` for
host-specific chrome.

## 6.1 The Single Principle

**Every clickable thing in the chrome = one MCP tool, 1:1.**

The host does not own a parallel "programmatic API". When a slot
needs both a dialog mode and a programmatic mode, the difference is
expressed via the tool's `inputSchema` — pass an arg to skip the
dialog, omit it to surface one.

```jsonc
// Tool: studio.project.open
// inputSchema: { properties: { path?: string } }
// path absent  → dialog mode
// path present → programmatic mode
```

This is why every wiring entry below has a `tool` field and nothing
that would compete with it.

## 6.2 `wiring.domainActions[]` — Row-2 Domain Icons

Permanent domain affordances rendered in the host's "row 2"
(below the system row). The bundle decides which icons appear and
which tool each fires.

Two `kind` discriminators:

### 6.2.1 `kind: "button"` — Single Trigger

```json
{
  "kind": "button",
  "tool": "studio_builder.export",
  "icon": "upload",
  "tooltip": "Export .mcpb",
  "args": { "format": "mcpb" }
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `kind` | string | **MUST** | Discriminator — `"button"`. |
| `tool` | string | **MUST** | Tool name to invoke on click. |
| `icon` | string | **MUST** | Icon identifier (host-defined registry). |
| `tooltip` | string | no | Hover tooltip. |
| `args` | object | no | Fixed arguments passed to the tool on click. (Legacy alias `arguments` accepted; emitters MUST use `args`.) |

### 6.2.2 `kind: "selectGroup"` — Radio / Page Selector

Multiple icons grouped together; only the item whose `value` matches
the group's `stateKey` is emphasised.

```json
{
  "kind": "selectGroup",
  "stateKey": "currentRoute",
  "items": [
    {
      "icon": "preview",
      "tooltip": "UI",
      "tool": "studio_builder.gotoPage",
      "args": { "pageId": "ui" },
      "value": "/ui"
    },
    {
      "icon": "build",
      "tooltip": "Tools",
      "tool": "studio_builder.gotoPage",
      "args": { "pageId": "tools" },
      "value": "/tools"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `kind` | string | **MUST** | Discriminator — `"selectGroup"`. |
| `stateKey` | string | **MUST** | Runtime state path (dot path supported, e.g. `currentRoute` or `view.mode`). The host looks up this key in the runtime state. |
| `items` | object[] | **MUST** | At least one item. |

Each `item`:

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `icon` | string | **MUST** | Icon id. |
| `tooltip` | string | no | Hover tooltip. |
| `tool` | string | **MUST** | Tool to invoke. |
| `args` | object | no | Fixed arguments. |
| `value` | scalar | **MUST** | Group's `stateKey` lookup is compared against this; equality emphasises the item. |

#### Host Behavior

The host MUST:

1. Walk `domainActions[]` and discriminate on `kind`.
2. For `kind: "button"`, render a single chrome action; no emphasis
   tracking.
3. For `kind: "selectGroup"`, render N adjacent chrome actions with
   a group flag; resolve `stateKey` via dot-path against runtime
   state; emphasise items whose `value` matches.
4. Render adjacent items of the same group without dividers.

#### Binding Conventions

- **Chrome-driven routing groups** SHOULD use `stateKey:
  "currentRoute"` (a chrome ↔ bundle convention). The bundle's UI
  DSL binds its router widget to `value: "{{currentRoute}}"` and
  declares the initial value in `state.initial.currentRoute`. The
  chrome's renderer / navigate verb writes to this key; the router
  swaps.
- **Domain-arbitrary groups** — the bundle freely chooses its own
  `stateKey` and `value` set (e.g. `mode`, `locale`). The chrome
  knows nothing about the names; the manifest is the source of
  truth.

#### Legacy `emphasisedWhen` (Deprecated)

Pre-v1.0 bundles MAY carry the flat shape:

```json
{
  "tool": "...",
  "icon": "...",
  "tooltip": "...",
  "args": {...},
  "emphasisedWhen": { "key": "currentRoute", "equals": "/ui" }
}
```

When `kind` is absent and `emphasisedWhen` is present, hosts MUST
interpret the entry as a single-item `selectGroup` (no neighbour
grouping). Emitters MUST emit `kind` explicitly.

## 6.3 `wiring.lifecycle[]` — System Slots

The host chrome owns a fixed set of system buttons (new project,
open, save, undo / redo, history, revert, close, export, ...). The
bundle decides which tool each slot fires; the bundle does NOT draw
the buttons.

```json
"lifecycle": [
  { "slot": "project.new",   "tool": "studio_builder.newProject" },
  { "slot": "project.open",  "tool": "studio_builder.openProject" },
  { "slot": "project.save",  "tool": "studio_builder.saveProject" },
  { "slot": "project.close", "tool": "studio_builder.closeProject" },
  { "slot": "edit.undo",     "tool": "studio_builder.undo" },
  { "slot": "edit.redo",     "tool": "studio_builder.redo" },
  { "slot": "history.show",  "tool": "studio_builder.showHistory" },
  { "slot": "edit.revert",   "tool": "studio_builder.revert" }
]
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `slot` | string | **MUST** | Host-defined slot name (`project.new`, `edit.undo`, ...). |
| `tool` | string | **MUST** | Tool name to invoke when the chrome fires the slot. |

### 6.3.1 Standard Slot Catalog

Hosts implementing the Studio chrome model SHOULD recognize at
least:

| Category | Slots |
|----------|-------|
| **Project** | `project.new`, `project.open`, `project.save`, `project.close`, `project.export`, `project.info` |
| **Edit** | `edit.undo`, `edit.redo`, `edit.revert` |
| **History** | `history.show` |

Hosts MAY define additional slots in their own namespace. Bundles
that bind to host-specific slots are still conformant — they just
work on those hosts.

A bundle that omits a slot binding MUST NOT cause the host to fail —
the host either greys out the chrome button or hides it (host
policy).

## 6.4 `wiring.settings[]` — Settings Dialog Actions

Rows in the settings dialog's domain panel.

```json
"settings": [
  {
    "label":   "Reset preferences",
    "icon":    "refresh",
    "tool":    "studio_builder.resetPrefs",
    "args":    { "scope": "all" },
    "tooltip": "Reset to defaults"
  }
]
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `label` | string | **MUST** | Row label. |
| `tool` | string | **MUST** | Tool name to invoke on click. |
| `icon` | string | no | Optional icon id. |
| `args` | object | no | Fixed arguments. |
| `tooltip` | string | no | Tooltip / secondary text. |

The shape mirrors `domainActions` `kind: "button"`. Settings rows
do not group / emphasise — they are flat triggers.

## 6.4a `wiring.titlebar` / `wiring.statusbar` — Chrome User-Zone Payloads

Two optional single-string payloads rendered in the host chrome's
**user-zone**: the segment of the titlebar (after fixed
server / status / version pills) and the segment of the statusbar
(between the fixed lint group and the right-edge spec / locale).

```json
"wiring": {
  "titlebar":  "scenarios: {{count}} · ready",
  "statusbar": "warnings: {{warnings}} · build: {{buildState}}"
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `titlebar` | string | no | Single payload rendered in the titlebar user-zone. |
| `statusbar` | string | no | Single payload rendered in the statusbar user-zone. |

### Host Behavior

The host MUST:

1. Render the payload verbatim in the named zone when present.
2. Resolve any `{{key}}` tokens against the active tab's runtime
   state before rendering. Unknown keys render as the empty string.
3. NOT split the user-zone into sub-cells — the bundle's tool
   composes the final string before assigning. Single-payload
   semantics is what lets a bundle freely choose its own layout
   without coupling to host chrome internals.

A host that does not surface a titlebar / statusbar user-zone MAY
ignore the corresponding field. Bundles MAY omit either.

Updates to the rendered payload flow through the bundle's tool
calls (typically a `state.set` action that writes the new string
into a state key the chrome reads, or — when the bundle wants the
host to re-evaluate templated tokens — a `patchManifest` on
`wiring.titlebar` directly). Both pathways round-trip through
`mcp_bundle` cleanly; the typed model preserves the field.

## 6.5 `chat.slashCommands[]` — Composer Chips

Chips surfaced when the user types `/` in the chat composer.

```json
"chat": {
  "slashCommands": [
    {
      "command": "/find",
      "template": "/find ",
      "description": "search docs"
    },
    {
      "command": "/build",
      "tool": "studio_builder.exportPackage"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `command` | string | **MUST** | The slash trigger token (must start with `/`). |
| `template` | string | one of `template` or `tool` | When set, selecting the chip pre-fills the composer with this string and hands control back to the user. |
| `tool` | string | one of `template` or `tool` | When set, selecting the chip dispatches this tool **directly** (LLM-bypassed). |
| `description` | string | no | Hint text rendered with the chip. |
| `args` | object | no | Fixed arguments for `tool`-mode entries. |

A chip MUST set exactly one of `template` or `tool`. Setting both is
an error.

## 6.6 End-to-End Example

A minimal bundle that wires three buttons + one slash command:

```json
{
  "schemaVersion": "1.0.0",
  "manifest": {
    "id": "com.example.notes",
    "name": "Notes",
    "version": "0.4.0",
    "wiring": {
      "domainActions": [
        {
          "kind": "selectGroup",
          "stateKey": "currentRoute",
          "items": [
            { "icon": "preview", "tooltip": "Notes",  "tool": "notes.gotoPage", "args": { "pageId": "list" }, "value": "/list" },
            { "icon": "build",   "tooltip": "Edit",   "tool": "notes.gotoPage", "args": { "pageId": "edit" }, "value": "/edit" }
          ]
        },
        {
          "kind": "button",
          "tool": "notes.export",
          "icon": "upload",
          "tooltip": "Export"
        }
      ],
      "lifecycle": [
        { "slot": "project.save", "tool": "notes.save" }
      ],
      "settings": [
        { "label": "Reset", "tool": "notes.reset", "icon": "refresh" }
      ]
    },
    "chat": {
      "slashCommands": [
        { "command": "/find", "template": "/find " },
        { "command": "/save", "tool": "notes.save" }
      ]
    }
  },
  "tools": {
    "tools": [
      { "name": "notes.gotoPage", "kind": "js", "target": { "entry": "tools/gotoPage.js", "fn": "gotoPage" } },
      { "name": "notes.export",   "kind": "js", "target": { "entry": "tools/export.js",   "fn": "exportAll" } },
      { "name": "notes.save",     "kind": "js", "target": { "entry": "tools/save.js",     "fn": "save" } },
      { "name": "notes.reset",    "kind": "js", "target": { "entry": "tools/reset.js",    "fn": "reset" } }
    ]
  },
  "requires": { "builtinAtoms": ["mcp", "fs", "ui"] }
}
```

The chrome reads `wiring.domainActions[]`, renders the icons,
matches `currentRoute` against each item's `value`. When the user
clicks the **Edit** icon:

```
chrome → wiring matches selectGroup item value:'/edit'
       → tool=notes.gotoPage, args={pageId:'edit'}
       → host.mcp.callTool('notes.gotoPage', {pageId:'edit'})
       → tools/gotoPage.js#gotoPage runs
       → host.mcp.callTool('studio.renderer.activate', {target:'/edit'})
       → chrome navigates to '/edit'
       → state.currentRoute='/edit'
       → next chrome paint emphasises Edit
```

## 6.7 Reference Resolution

Every `tool` field in the wiring surface is a **MCP tool name**
resolved by the host's tool registry at activation:

1. The host indexes all declared `tools[]` from active bundles.
2. Host builtin tools are added (always present).
3. The wiring surface's `tool` references are resolved against this
   merged registry.

When a wiring slot references a name **not** in the registry:

- **MAY refuse** to activate the bundle (strict host).
- **MAY** render the slot disabled (lenient host).
- **MUST** emit a warning during activation.

Tools that come from another bundle's `tools[]` resolve naturally —
this is how cross-bundle integration works without coupling.

## 6.8 Mutator Tools (Studio Host Convention)

The Studio host typically exposes a family of `studio.builder.*`
mutator tools that the chat LLM uses to **modify the active bundle's
own manifest / files**:

| Mutator | Patches |
|---------|---------|
| `studio.builder.patchManifest` | Top-level manifest fields. |
| `studio.builder.addTool` | `tools.tools[]`. |
| `studio.builder.addKnowledgeSource` | `knowledge.sources[]`. |
| `studio.builder.addAgent` | `agents.agents[]`. |
| `studio.builder.writeUI` | `ui/<rel>.json` files. |

These are host-side conveniences, not part of the spec contract —
listed here only as the standard model so other Studio-style hosts
align.

## 6.9 Conformance

A bundle's wiring surface MUST:

1. Set `kind` on every `domainActions[]` entry, OR carry the legacy
   `emphasisedWhen` shape (interpreted as single-item selectGroup).
2. Reference tool names that are either declared in `tools[]` or
   known to the host as builtins.
3. Set exactly one of `template` or `tool` on each `slashCommands[]`
   entry.

A host implementing the wiring surface MUST:

1. Resolve `domainActions[*].tool` and `lifecycle[*].tool` and
   `settings[*].tool` and `slashCommands[*].tool` against the merged
   tool registry at activation.
2. Pass the entry's `args` (when present) as the tool's input.
3. For `selectGroup`, look up `stateKey` via dot-path and compute
   emphasis per item.

A host MAY:

- Render the system row icons (`lifecycle[*].slot`) in any visual
  style; the bundle does not control rendering.
- Provide additional non-spec slots in its own namespace.
