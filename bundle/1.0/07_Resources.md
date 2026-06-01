# 07. Resources & On-Disk Layout

A bundle exists in two physical forms:

- **`.mbd/` working tree** — a directory with `manifest.json` at the
  root, plus reserved sub-folders for content.
- **`.mcpb` archive** — a deterministic ZIP of an `.mbd/` for
  distribution.

Both forms carry the same logical bundle. Authoring tools read /
write `.mbd/`; marketplaces ship `.mcpb`.

References, in the `mcp_bundle` package:
- `lib/src/io/bundle_resources.dart`
- `lib/src/io/mcp_bundle_loader.dart`
- `lib/src/install/mcp_bundle_packer.dart`

## 7.1 Reserved Folders

The bundle format owns 12 folder names under the root:

```
<bundle>.mbd/
├── manifest.json    ← single source of declarations
├── ui/              ← UI definitions (mcp_ui_dsl JSON)
├── assets/          ← binary / text assets (icons, fonts, files)
├── skills/          ← skill module definitions
├── knowledge/       ← knowledge sources / retriever configs
├── facts/           ← atomic SPO fact records (companion to inline facts[])
├── workflows/       ← workflow definitions (companion to inline workflows[])
├── pipelines/       ← pipeline definitions (companion to inline pipelines[])
├── runbooks/        ← runbook definitions (companion to inline runbooks[])
├── tools/           ← tool definitions / bundled JS sources (companion to inline tools[])
├── profiles/        ← profile definitions
├── philosophy/      ← philosophy / ethos definitions
└── agents/          ← agent definitions (4-axis bindings + runtime cfg)
```

The reference enum (declaration order):

```dart
class BundleFolder {
  static const ui         = BundleFolder._('ui');
  static const assets     = BundleFolder._('assets');
  static const skills     = BundleFolder._('skills');
  static const knowledge  = BundleFolder._('knowledge');
  static const facts      = BundleFolder._('facts');
  static const workflows  = BundleFolder._('workflows');
  static const pipelines  = BundleFolder._('pipelines');
  static const runbooks   = BundleFolder._('runbooks');
  static const tools      = BundleFolder._('tools');
  static const profiles   = BundleFolder._('profiles');
  static const philosophy = BundleFolder._('philosophy');
  static const agents     = BundleFolder._('agents');
}
```

Reference: `bundle_resources.dart:17`.

A folder MUST be present only when the bundle has content for it.
Empty folders are allowed but discouraged.

### 7.1.1 Folder Conventions

| Folder | Convention |
|--------|-----------|
| `ui/` | Each `<rel>.json` is a valid mcp_ui_dsl file (`ApplicationDefinition` or `PageDefinition`). The path `ui/<rel>.json` maps to MCP resource URI `ui://<rel>`. |
| `assets/` | Free-form. Paths used elsewhere (e.g. `manifest.icon`) are resolved against `assets/`. |
| `skills/` | Each file may declare one `SkillModule` or a list. Inline `manifest.skills` is equivalent. |
| `knowledge/` | Each file may declare a `KnowledgeSource`. Inline `manifest.knowledge` is equivalent. |
| `facts/` | Each file may declare one `Fact` or a list of facts (typically newline-delimited or per-fact JSON files). Companion to inline `manifest.facts[]` for large-volume facts. |
| `workflows/` | Each file may declare a `WorkflowEntry`. Companion to inline `manifest.workflows[]` for large or generated workflow definitions. |
| `pipelines/` | Each file may declare a `PipelineEntry`. Companion to inline `manifest.pipelines[]`. |
| `runbooks/` | Each file may declare a `RunbookEntry`. Companion to inline `manifest.runbooks[]`. |
| `tools/` | Each file may declare a `ToolEntry`, plus the bundled JS source files referenced by `kind: js` tools (e.g. `tools/export.js`). Companion to inline `manifest.tools[]` for large schemas or scripts. |
| `profiles/` | Each file may declare a `ProfileDefinition`. |
| `philosophy/` | Each file may declare a `Philosophy`. |
| `agents/` | Each file may declare an `AgentDefinition`. |

When the same id appears in both the inline section and the
matching folder, **folder content takes precedence**. The host SHOULD
warn.

### 7.1.2 Sections With No Folder (v1.0)

The following slots are **inline-only** in v1.0 — there is no
reserved folder for them. Their data appears only inside
`manifest.json`:

- `requires` — see [`02_Manifest.md`](02_Manifest.md) §2.7.
- `bindings`, `tests`, `policies`, `factGraphSchema`,
  `factGraphSection`, `compatibility`, `integrity`, `extensions`.

All 12 reserved folders listed in §7.1 are part of the v1.0 contract;
the additive expansion that introduced `facts/`, `workflows/`,
`pipelines/`, `runbooks/`, and `tools/` keeps `BundleResources` API
back-compatible — earlier consumers that only used the original
folders continue to work without change.

## 7.2 `BundleResources` API

Every reserved folder is exposed through a typed read / write surface:

```dart
final ui = bundle.uiResources;
await ui.write('app.json', jsonEncode(definition));
final raw = await ui.read('app.json');
final list = await ui.list(extension: '.json');
```

Reference: `bundle_resources.dart`.

`McpBundle` exposes one named getter per reserved folder, all
returning `BundleResources` bound to that folder:

| Folder | Getter |
|--------|--------|
| `ui/` | `bundle.uiResources` |
| `assets/` | `bundle.assetResources` |
| `skills/` | `bundle.skillResources` |
| `knowledge/` | `bundle.knowledgeResources` |
| `facts/` | `bundle.factsResources` |
| `workflows/` | `bundle.workflowsResources` |
| `pipelines/` | `bundle.pipelinesResources` |
| `runbooks/` | `bundle.runbooksResources` |
| `tools/` | `bundle.toolsResources` |
| `profiles/` | `bundle.profileResources` |
| `philosophy/` | `bundle.philosophyResources` |
| `agents/` | `bundle.agentResources` |

The generic accessor `bundle.resources(BundleFolder folder)` returns
the same `BundleResources` object for any folder enum value. All
getters throw `StateError` when the bundle was loaded from inline
JSON / a remote URL with no on-disk directory (see §7.5).

The read / write methods:

| Method | Returns |
|--------|---------|
| `read(rel)` | UTF-8 text. Throws `BundleResourceNotFoundException` on miss. |
| `readBytes(rel)` | Raw bytes. |
| `readJson(rel)` | Parsed JSON (text + `jsonDecode`). Throws `BundleResourceParseException` on invalid JSON. |
| `write(rel, content)` | UTF-8 text — creates parent dirs. |
| `writeBytes(rel, bytes)` | Raw bytes — creates parent dirs. |
| `writeJson(rel, value, indent: 2)` | JSON-encode + write. |
| `exists(rel)` | bool. |
| `delete(rel)` | No-op when missing. |
| `list({extension})` | Recursive listing of relative paths, forward-slash separated, sorted lexicographically. |

### 7.2.1 Path Safety

All `rel` arguments MUST be relative, forward-slash-separated paths.
The implementation rejects:

- empty string;
- absolute paths (`/foo`, `C:\foo`);
- any segment equal to `..`.

Hosts implementing this spec MUST apply equivalent path-traversal
guards. A bundle that successfully embeds an absolute or `..`-bearing
path is malformed.

## 7.3 `manifest.json` Layout

`manifest.json` MUST live at the root of `.mbd/`. The file is a
JSON object whose top level is the `McpBundle` shape (see
[`01_Bundle_Schema.md`](01_Bundle_Schema.md)).

```jsonc
// <bundle>.mbd/manifest.json
{
  "schemaVersion": "1.0.0",
  "manifest": { ... },
  "ui":        { ... },   // optional inline (deprecated for ui)
  "tools":     { ... },
  "wiring":    { ... }    // (lives under "manifest" actually)
}
```

> Note: `wiring` and `chat` live **inside** `manifest`, not at the
> bundle root — they are manifest sub-objects (see [§2.8](02_Manifest.md#28-wiring-studio-host-surface)).

Other top-level keys are listed in [`01_Bundle_Schema.md`](01_Bundle_Schema.md).

## 7.4 `.mcpb` Archive Format

`.mcpb` is a **deterministic ZIP** of an `.mbd/` tree.

Reference: `mcp_bundle_packer.dart`.

### 7.4.1 Determinism Rules

The packer enforces:

| Rule | Value |
|------|-------|
| **Entry order** | `manifest.json` first; remaining entries sorted lexicographically. |
| **lastModTime** | Fixed `1980-01-01T00:00:00Z` (DOS epoch — `315532800`). |
| **mode** | `420` (octal `0644`). |
| **owner / group** | `0` / `0`. |
| **Path separator** | Forward slash, even on Windows. |
| **`manifest.json` payload** | Canonical JSON (sorted keys, no insignificant whitespace). |

These ensure `pack(<dir>)` produces byte-identical output across
platforms and runs — a precondition for content-hash integrity.

### 7.4.2 Integrity Computation

When packed with `computeIntegrity: true` (default), the packer:

1. Walks the on-disk tree in path order.
2. Computes a content hash whose **scope** is determined by
   `manifest.integrity.contentHash.scope`:
   - `canonicalJson` — hash the canonical-JSON-encoded
     `manifest.json` only;
   - other scopes (e.g. file-tree-merkle) — host-defined extension.
3. Updates `manifest.integrity.contentHash` with the computed hash.
4. Optionally signs via the supplied `BundleSigner` and appends to
   `integrity.signatures[]`.

See [`10_Distribution.md`](10_Distribution.md) for the full pack /
sign / install pipeline.

### 7.4.3 Compression

Each entry is compressed via the `archive` package's deflate. The
packer instructs `compress = true` so empty files / already-tiny
metadata still produce predictable output.

## 7.5 Loading

The reference loader supports four sources:

| Source | Entry point | Notes |
|--------|-------------|-------|
| `.mbd/` directory | `McpBundleLoader.loadDirectory(path)` | Reads `manifest.json` + reserved folders eagerly. Sets `bundle.directory`. |
| Installed bundle | `McpBundleLoader.loadInstalled(installRoot, id)` | Reads from a pre-installed location. |
| Inline JSON map | `McpBundleLoader.loadJson(map)` | For tests / in-memory composition. `bundle.directory` = `null`. |
| Remote `.mcpb` URL | `McpBundleLoader.loadUrl(uri)` | Fetches + unpacks in memory. |

Reference: `mcp_bundle_loader.dart`.

A bundle loaded inline (`directory == null`) can NOT call
`bundle.uiResources` etc. — those getters throw `StateError`.
Authoring tools that need both inline JSON and folder I/O must
install the bundle to disk first.

## 7.6 Storage Adapters

The `BundleStoragePort` abstraction lets hosts plug their own
backing store:

```dart
abstract class BundleStoragePort {
  Future<Uint8List> readBytes(String key);
  Future<void> writeBytes(String key, Uint8List bytes);
  Future<bool> exists(String key);
  Future<void> delete(String key);
  Future<List<String>> list({String? prefix});
}
```

Reference: `bundle_storage_port.dart`.

Built-in adapters:

| Adapter | Backing |
|---------|---------|
| `FileStorageAdapter` | Local filesystem. |
| `HttpStorageAdapter` | HTTP(S) GET / PUT (read-only by default). |
| `MemoryStorageAdapter` | In-memory map (tests / fixtures). |

Hosts MAY add their own adapter (S3, GCS, IPFS, ...) without
spec-level changes.

## 7.7 Conformance

A bundle MUST:

1. Place `manifest.json` at the root of `.mbd/` and as the first
   entry in `.mcpb`.
2. Use only the 12 reserved folder names listed in §7.1 for content
   directories.
3. Use forward-slash paths inside the manifest and in `.mcpb`
   entries.

A host loading a bundle MUST:

1. Reject path components equal to `..` or starting with `/` /
   drive-letter prefixes inside reserved-folder reads.
2. Treat inline-section / folder content as equivalent (folder wins
   on conflict; warn).
3. Default `schemaVersion` to `"1.0.0"` when `manifest.json` omits it.

A host MAY:

- Lazy-load reserved-folder content (read on demand) instead of
  eagerly at activation.
- Cache parsed folder content across activations of the same
  bundle version.
