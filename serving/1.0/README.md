# MCP Serving — 1.0

How a bundle is served over MCP, so a client (AppPlayer-class) connects to a server and runs the result **identically to a local bundle**. The server may back the content with an `.mbd` on disk or generate it in code; the client cannot tell. This spec introduces **no new concept** — it only states the MCP address that carries each part of a bundle, and the guarantees around it.

## What a bundle is (the thing being served)

A bundle is one **document** plus the larger payloads it points to.

- The document is `manifest.json` (model: `Bundle` / `BundleManifest`). Its **top-level keys are sibling sections**: `manifest` (pure metadata — `id` · `name` · `version` · `entryPoint` · `dependencies` · `platform` · `icon` · `publisher` …), `requires`, `ui` (config), `tools` (declarations), `settings`, `wiring`, `chat`, `behavior`, and the knowledge sections (`facts` · `skills` · `profiles` · `philosophy` · `workflows` · `pipelines` · `runbooks` · `agents` · `flow` · `factGraph`).
- The `manifest` key is **not a container** for the others — it is one section among siblings. The document is the root; every section is a child of the document, not of `manifest`.
- Larger payloads live outside the document as their own resources: UI page definitions, knowledge entry bodies, and binary assets.

## Serving surface — complete map

The **bundle document is the entry**: a client reads it first; it declares which sections are present and (via `entryPoint`) where the UI begins. Every other row is a payload the document points to. Existing addresses are normative in their home specs; only the document resource is defined here.

| bundle part | MCP address | direction | home |
|---|---|---|---|
| **bundle document** — `manifest` metadata + `requires` + `settings` + `wiring` + `chat` + tool declarations + `ui` config (all top-level keys) | resource `bundle://manifest.json` | `resources/read` | **here** |
| ui application definition | resource `ui://app` | `resources/read` | mcp_ui_dsl §6 |
| ui page `{name}` | resource `ui://page/{name}` | `resources/read` | mcp_ui_dsl §6 |
| app metadata (lightweight) | resource `ui://app/info` | `resources/read` | mcp_ui_dsl §11.6 |
| knowledge entry | resource `kb://<facade>/<id>` | `resources/read` | platform 07 |
| assets (icon · font · image …) | reference `bundle://<path>` | resolved per asset | mcp_ui_dsl §11.5 |
| tools (executable) | `tools/list` · `tools/call` | call | platform 06 |
| behavior (executable) | `bk.behavior.run` · `resume` · `list` | call | platform 03 |

`settings`, `wiring`, `chat`, and `requires` have **no separate address** — they are top-level keys of the bundle document, delivered by reading it. `tools`, `ui`, and the knowledge sections appear **twice**: declared in the document (so the client knows they exist) and served at their own address (the payload / executable surface). The document is the only address this spec adds; `bundle://` is the existing scheme (mcp_ui_dsl §11.5), here resolving the document file rather than an asset.

## Entry flow

The client **discovers** what a server offers — it never assumes the bundle document is present. Every address except the bundle document predates this spec, so a server built before it keeps working.

1. Client connects and discovers the served surface via `resources/list` and `tools/list` (standard MCP).
2. **Enriched entry** — if `bundle://manifest.json` is served, read it. The document yields `manifest` metadata, `requires`, `settings`, `wiring`, `chat`, tool declarations, `ui` config, and the list of declared knowledge sections.
3. **Baseline entry** — if the bundle document is *not* served (an existing server), the mcp_ui_dsl init flow applies unchanged: read `ui://app` as the entry. The result is a UI-only bundle (plus whatever `tools`/`kb://` the server happens to expose) — identical to today.
4. Fetch each declared payload at its address: `ui://app` (then `ui://page/*` per `entryPoint` / routes), `kb://<facade>/<id>` per knowledge entry, `tools/list`; register `bk.behavior.*` if a behavior section is present.
5. Reconstruct the in-memory `Bundle` and run it through the same activation path a local bundle uses.

## Rules

1. **Source-agnostic.** The server MAY serve from an `.mbd` or generate the content. The client depends only on the addresses above, never on the source.
2. **Equivalence.** Reconstruct-then-run MUST behave identically to running the same bundle locally (the local path: install → kernel boot → knowledge + standard tools registered → settings store → UI built). Serving changes the source, never the execution.
3. **Partial — existing servers keep working.** Any subset of addresses MAY be served (a partial bundle). The client consumes what it understands and MUST ignore the rest without error. The bundle document is additive: a client MUST NOT require `bundle://manifest.json`. A server that predates this spec — a pure `ui://app` mcp_ui_dsl server, a tools-only server — is a valid partial bundle and MUST run unchanged.

Version `1.0`. Status: draft. MUST / SHOULD / MAY per RFC 2119.
