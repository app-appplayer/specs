# 00. Overview

## 0.1 What is MCP UI DSL?

MCP UI DSL is a JSON-based domain-specific language for declarative, reactive user interfaces in the Model Context Protocol (MCP) ecosystem. A server (or an AI agent) emits a JSON definition; any MCP-capable client renders it without application-specific code.

Three properties are definitional:

1. **Declarative.** The JSON describes *what* the UI is, not *how* to construct it. Runtimes own the rendering strategy.
2. **Server-driven.** The authoritative definition lives on the server. Clients fetch, render, and subscribe to updates; they do not author UI.
3. **Language-neutral.** The DSL names and semantics are independent of Flutter, React, Vue, Python, or any specific implementation. A valid definition renders equivalently on every conformant runtime.

## 0.2 Why MCP UI DSL?

- **Cross-platform parity without duplication.** One definition serves every client.
- **AI-authored UIs.** LLMs can generate valid JSON from a single-document reference.
- **Live updates via MCP.** Resources, subscriptions, and tools are first-class — the UI reacts to server state without polling logic in the definition.
- **Permissioned client capabilities.** File system, network, system info, channels — all available through explicit permission grants (see [`08_Client_Extensions.md`](08_Client_Extensions.md)).

## 0.3 Scope of this Specification

This specification defines:

- The shape of an ApplicationDefinition and PageDefinition
- The widget catalog (canonical names, properties, aliases)
- Data binding syntax, expression language, and prefix resolution
- The action system
- The theme system
- The MCP protocol contract used by the DSL
- The security model (sandbox, validation, access control)
- Client-side extensions (actions, permissions, channels)
- The template system
- Advanced widgets (charts, canvas, code editor, etc.)
- Bundle metadata and serving
- Internationalization
- Accessibility
- Responsive layout and event propagation
- Offline behavior and sync
- Animations
- Conformance profiles

This specification does **not** define:

- How a specific runtime implementation is structured internally.
- Which widgets are mapped to which native platform widgets (a Flutter implementation's `text` → Flutter `Text`, a React implementation's → `<span>`; these are implementation choices).
- Any single product's UI design language beyond the theme tokens.

## 0.4 How to Read This Specification

Start here. Then:

| If you are... | Read... |
|---------------|---------|
| A runtime implementer | [`01_Core_Concepts.md`](01_Core_Concepts.md), [`18_Conformance.md`](18_Conformance.md), then sections matching your target profiles |
| A UI author (human or AI) | [`02_Widgets.md`](02_Widgets.md), [`03_Data_Binding.md`](03_Data_Binding.md), [`04_Actions.md`](04_Actions.md) |
| A bundle packager | [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) |
| A client-capability integrator | [`08_Client_Extensions.md`](08_Client_Extensions.md) |
| Seeking a specific name | [`17_Naming.md`](17_Naming.md) |

## 0.5 Terminology

| Term | Definition |
|------|------------|
| **Application** | A complete MCP UI app: routes, pages, theme, global state. Defined by an `ApplicationDefinition`. |
| **Page** | A full-screen view within an application. Defined by a `PageDefinition`. Canonical `type` value is `"page"`; `"screen"` is a legacy alias. |
| **Widget** | A UI component declared by a `"type"` field in JSON. The widget catalog is in [`02_Widgets.md`](02_Widgets.md). |
| **Action** | An operation triggered by user interaction or lifecycle events. Types: `state`, `navigation`, `tool`, etc. See [`04_Actions.md`](04_Actions.md). |
| **Binding** | A reactive expression in `{{...}}` syntax. Resolves against state, theme, route params, etc. See [`03_Data_Binding.md`](03_Data_Binding.md). |
| **Template** | A reusable, parameterized widget tree invoked via `"type": "use"`. See [`09_Templates.md`](09_Templates.md). |
| **Channel** | A bidirectional real-time data stream between client and server (Client Profile). See [`08_Client_Extensions.md`](08_Client_Extensions.md). |
| **Bundle** | A packaged MCP application containing UI definitions, assets, and manifest. See [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md). |
| **Runtime** | The client-side implementation that parses a definition and renders widgets. |
| **Generator** | A library (or AI agent) that produces DSL JSON. |
| **Profile** | A named subset of DSL capabilities used for conformance declarations. See [`18_Conformance.md`](18_Conformance.md). |
| **Canonical name** | The preferred identifier for a concept. Emitters MUST use canonical names. |
| **Legacy alias** | An accepted alternative name for a canonical identifier. Accepted on input; never emitted. Registered in [`17_Naming.md`](17_Naming.md). |

## 0.6 Requirement Levels

This specification uses **MUST / SHOULD / MAY** per RFC 2119 / RFC 8174:

- **MUST** — absolute requirement. A non-conformant runtime fails.
- **SHOULD** — strong recommendation. A runtime MAY diverge with documented reason.
- **MAY** — optional. Either behavior is conformant.

Requirements are anchored in [`18_Conformance.md`](18_Conformance.md) per profile. Other sections describe behavior and reference the conformance file for normative binding.

## 0.7 Versioning and `since:` Markers

Features introduced after v1.0 carry a `since: vX.Y` marker in their heading or schema table. The marker is informational — it tells implementers which MCP UI DSL version first introduced the feature.

Implementations declare support via **Profiles** (see [`18_Conformance.md`](18_Conformance.md)), not via version numbers. A runtime may implement all of v1.0 but only the Core Profile parts of v1.1.

## 0.8 JSON Conventions

- **Key style:** camelCase.
- **Enum value style:** lowercase single-word or camelCase multi-word.
- **Comments:** JSON does not support comments; inline `//` comments in this document's examples are informative only and MUST be removed from any actual JSON sent across the wire.
- **Required fields** are marked `required: true` in schema tables. Absence MUST produce a parse error.
- **Optional fields** with defaults show the default value in the schema table.

See [`17_Naming.md`](17_Naming.md) §17.1 for the complete naming conventions.
