# 13 — datastore Data Interface

The data interface model of the datastore capability. **This document is the definition (canon), and consumers (adapter packages · recipe wiring · hosts Studio·AppPlayer) follow it.** Per-host ad-hoc implementation · re-analysis forbidden.

datastore and io (`11-io-devices.md`) are **different layers**. They use the same "adapter + policy" skeleton but are not mixed.

---

## 0. Identity

datastore capability = the capability that handles **persistent · addressable data stores** (filesystem · database) via read/query/CRUD.

- substrate = `mcp_datastore`'s `DatasourceRegistry` + internally owned `DatastorePolicy` (deny-by-default · destructive plan→commit · audit).
- "datasource" = a persistent store handled via `fs.*`/`db.*`. File (path-addressed) · DB (query-addressed). The adapter implements `DatasourceAdapter` (→ `FsAdapter`/`DbAdapter`).

### Boundary with io (do not confuse)

| | io (`11`) | datastore (`13`) |
|---|---|---|
| Target | device/protocol endpoint | persistent · addressable data store |
| Meaning | command · stream · register · signal | **CRUD · query** over an address (path/query) |
| Example | process exec · modbus · mqtt | file · table |

Placing a DB as an io device (next to modbus) is a category error. **File operations also have two coexisting lenses**: the execution lens (`io.execute` running an OS file utility, desktop) vs the data lens (`fs.*` structured read/write/edit, all platforms). When handled as a data interface, it is datastore.

## 1. Surface (fixed contract — adapter-independent)

verbs are separated per namespace — bundling byte-path and SQL row under a single verb leaks.

| namespace | tools |
|---|---|
| `fs.*` (file) | `fs.read` · `fs.write` · `fs.list` · `fs.glob` · `fs.stat` · `fs.edit` · `fs.mkdir` · `fs.move` · `fs.remove` |
| `db.*` (database) | `db.query` · `db.exec` · `db.tx` |

The surface is **fixed** even as adapters grow. Routing = the call argument `source` (defaults to the family default when omitted) → adapter resolved from `DatasourceRegistry`. **Adding a backend introduces 0 new verbs.** `db.*` is exposed in the catalog only when a DB adapter is registered.

## 2. Adapter = sibling sub-package (isomorphic with the io drivers)

- **`mcp_datastore` core** = the `DatasourceAdapter`/`FsAdapter`/`DbAdapter` contract + `DatasourceRegistry` + `DatastorePolicy` + the built-in `FilesystemSource` (`fs.*`, `dart:io`) + `DatastoreTools` (catalog + `call`).
- **`mcp_datastore_<backend>`** = a DB-backend sibling package, implements `DbAdapter`, depends only on published `mcp_datastore`. **Core unmodified.** Planned: `mcp_datastore_sqlite` · `mcp_datastore_postgres` · `mcp_datastore_mongo`.
- The core has **0 DB-client dependencies** — heavy drivers are opt-in from the sibling. (Same as `mcp_io` not putting modbus/mqtt dependencies in its core.)

## 3. Policy — internally owned (not borrowed from io)

`DatastorePolicy` is **owned by `mcp_datastore` itself** (no dependency on `mcp_io` PolicyEngine). Its starting shape is identical (deny-by-default · destructive plan→commit · audit), but it **evolves independently** with data-specific policy: `fs` = `allowedRoots` path jail; `db` = table allowlist · read/write separation · row-level · transaction policy.

- deny-by-default: a scoped caller passes only when in an allowed role.
- destructive operations (`fs.remove`, etc.) become `PolicyNeedsCommit` → the host enforces plan→commit.
- every deny/gate is surfaced as a **codified result** (denied · needs_commit · datastore_error), not a host throw.

## 4. Platform Portability

Because the adapter is `dart:io`/a pure-Dart DB client, **it also works on mobile · web hosts** — file/DB access is available even where io's process driver (desktop-only) cannot run. A bundle app can read/write its own sandbox files without process/exec permission (file access ⊂ a narrower permission than exec).

## 5. Wiring (host adoption)

Identical to io, it follows the single reference pattern in `recipes/capability_tools/`. `datastore_example` exposes `DatastoreTools` split across the **two capability ids** `fs`·`db` (`registerCapabilityTools(registry, capabilityId: 'fs'|'db', tools: …)`). The host (Studio·AppPlayer) injects the configured `DatastoreTools` (source registry + policy). Detailed contract = the capability tool pack of `06-tool-registry.md`.

Both consumption surfaces go through the same policy core: the app (runtime `tool` action) · the agent (`HostToolRegistry` → MCP endpoint, internal/external LLM).
