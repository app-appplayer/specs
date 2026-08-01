# MCP UI DSL — Specification

MCP UI DSL is a JSON-based domain-specific language for declarative, server-driven user interfaces in the [Model Context Protocol](https://modelcontextprotocol.io) ecosystem. One canonical vocabulary; many implementations.

## Versions

- **[1.4/](1.4/)** — current canonical specification

## Quick Start

For 1.3:

- New to MCP UI DSL? → [`1.4/00_Overview.md`](1.4/00_Overview.md)
- Implementing a renderer? → [`1.4/01_Core_Concepts.md`](1.4/01_Core_Concepts.md) then [`1.4/18_Conformance.md`](1.4/18_Conformance.md)
- Authoring a UI? → [`1.4/02_Widgets.md`](1.4/02_Widgets.md) and [`1.4/04_Actions.md`](1.4/04_Actions.md)
- Looking for a specific name or alias? → [`1.4/17_Naming.md`](1.4/17_Naming.md)
- Tracking what changed across versions? → [`CHANGELOG.md`](CHANGELOG.md)

## Document Map (1.3)

### Foundation
- [`1.4/00_Overview.md`](1.4/00_Overview.md) — Motivation, scope, terminology, document map
- [`1.4/01_Core_Concepts.md`](1.4/01_Core_Concepts.md) — Application, Page, Widget tree, Definition lifecycle

### Building Blocks
- [`1.4/02_Widgets.md`](1.4/02_Widgets.md) — Canonical widget catalog
- [`1.4/03_Data_Binding.md`](1.4/03_Data_Binding.md) — Binding syntax, expressions, prefix resolution
- [`1.4/04_Actions.md`](1.4/04_Actions.md) — Action system
- [`1.4/05_Theme.md`](1.4/05_Theme.md) — Theme system (M3 + DTCG + shadcn semantic + HCT seed)
- [`1.4/05a_Tokens_Reference.md`](1.4/05a_Tokens_Reference.md) — Token 3-tier (reference / alias / component)
- [`1.4/05b_DTCG_Interchange.md`](1.4/05b_DTCG_Interchange.md) — DTCG W3C JSON interchange (Tokens Studio · Style Dictionary · Claude Design)

### Runtime and Security
- [`1.4/06_Runtime_Contract.md`](1.4/06_Runtime_Contract.md) — MCP protocol integration
- [`1.4/07_Security.md`](1.4/07_Security.md) — Sandboxing, input validation, resource access, state isolation

### Extended Capabilities
- [`1.4/08_Client_Extensions.md`](1.4/08_Client_Extensions.md) — Client actions, permissions, channels
- [`1.4/09_Templates.md`](1.4/09_Templates.md) — Template system (static and stateful)
- [`1.4/10_Advanced_Widgets.md`](1.4/10_Advanced_Widgets.md) — Charts, canvas, code editor, terminal, etc.
- [`1.4/11_Bundle_Metadata.md`](1.4/11_Bundle_Metadata.md) — ApplicationDefinition metadata, `bundle://`, `ui://app/info`
- [`1.4/12_Internationalization.md`](1.4/12_Internationalization.md) — i18n bindings, pluralization, formatting

### Cross-Cutting
- [`1.4/13_Accessibility.md`](1.4/13_Accessibility.md) — `accessibility` object, roles, keyboard, live regions
- [`1.4/14_Responsive_Events.md`](1.4/14_Responsive_Events.md) — Breakpoints, MediaQuery, event phases
- [`1.4/15_Offline_Sync.md`](1.4/15_Offline_Sync.md) — Offline queue, conflict resolution
- [`1.4/16_Animations.md`](1.4/16_Animations.md) — Page transitions, shared elements, implicit/explicit

### Normative References
- [`1.4/17_Naming.md`](1.4/17_Naming.md) — Canonical naming rules + Legacy alias registry
- [`1.4/18_Conformance.md`](1.4/18_Conformance.md) — Profiles, MUST/SHOULD/MAY, conformance tests

### Schemas and References
- [`1.4/schema/`](1.4/schema/) — JSON Schemas (app, page, widgets, theme)
- [`1.4/widgets/`](1.4/widgets/) — Per-category widget detail (layout, display, input, list, navigation, scroll, animation, interaction, dialog, advanced, utility)
- [`1.4/generated/`](1.4/generated/) — Generated reference (widgets index, LLM prompt card)

## Principles

1. **Single source of truth per concept.** Each fact defined once.
2. **Additive evolution.** Canonical names never change; new names introduced via aliases.
3. **Profile-based conformance.** Implementations declare which profiles they support.
4. **Normative vs informative clearly separated.** MUST / SHOULD / MAY tagged; examples marked as informative.
5. **Examples use canonical names only.** Legacy aliases live in one registry.
6. **No ghost references.** Every cross-reference resolves.
7. **Discoverable in 60 seconds.**

## Conformance

Conformance is declared by **Profile** (Core / Client / Bundle / Advanced / Template / Composition), not by version. See [`1.4/18_Conformance.md`](1.4/18_Conformance.md).

## Language

Specification prose is in English. Code examples are JSON. Comments within examples use `//` inline (informative only; not part of JSON).

## License

MIT — see [LICENSE](LICENSE).
