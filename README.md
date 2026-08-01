# Specifications

Specifications for the MakeMind / AppPlayer ecosystem — its UI DSL, bundle format, serving contract, and the platform that composes them. These run on the [Model Context Protocol](https://modelcontextprotocol.io); the `mcp_` in package names is a naming convention, not an affiliation. One vocabulary per concept; many implementations. Each spec is versioned under its own directory.

## Specs

- **[ui_dsl/](ui_dsl/)** — UI DSL: declarative, server-driven UIs.
- **[bundle/](bundle/)** — Bundle: a manifest plus sibling sections packaging an application.
- **[serving/](serving/)** — Serving: serving a bundle so a client runs it identically to a local one.
- **[platform/](platform/)** — Platform: the structural contract composing UI, bundle, and kernel into a framework.
- **[form/](form/)** — Form: a schema-based document generation engine — fixed templates, schema-constrained fill, six render formats.
- **[gateway/](gateway/)** — Gateway: the dispatch protocol carrying full MCP across a mediated relay (hub topologies, NAT-bound providers).

## Language

Specification prose is in English. Code examples are JSON; comments within examples are informative only.

## License

MIT — see [LICENSE](LICENSE).
