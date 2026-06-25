# MakeMind Platform Spec

The composition model of every package, tool, bundle, and UI in the MakeMind framework has converged into a single structure. This spec defines that framework as a **structural contract** — the technical spec of **"this is how it MUST behave."** It defines structure only: not values, motivations, or product instances.

## Thesis in one line

> **MakeMind = an MCP-native composition framework of UI + bundle + kernel. Depending on the composition pattern, an arbitrary product (studio · player · knowledge system · document automation · industrial HMI · user-defined) is produced.**

= single framework, infinite instances.

## Spec file layout

| spec | area |
|---|---|
| [`1.0/00-overview.md`](1.0/00-overview.md) | overview · definitions · scope |
| [`1.0/01-layers.md`](1.0/01-layers.md) | the three abstract layers (UI · bundle · kernel) + composition points |
| [`1.0/02-bundle-interface.md`](1.0/02-bundle-interface.md) | bundle manifest interface (conformance spec) |
| [`1.0/03-kernel-runtime.md`](1.0/03-kernel-runtime.md) | kernel operational-logic spec |
| [`1.0/04-ui-host.md`](1.0/04-ui-host.md) | responsibilities + interface of the UI host |
| [`1.0/05-composition-patterns.md`](1.0/05-composition-patterns.md) | composition patterns (server-serve · client-only · hybrid · multi-bundle · multi-server) |
| [`1.0/06-tool-registry.md`](1.0/06-tool-registry.md) | three tool registration paths + two-name model |
| [`1.0/07-knowledge-access.md`](1.0/07-knowledge-access.md) | knowledge tool access + scopeId + bundleId prefix |
| [`1.0/08-extension.md`](1.0/08-extension.md) | the standard to follow when building a new instance |
| [`1.0/09-instances.md`](1.0/09-instances.md) | current instance matrix + potential instances |
| [`1.0/10-agent-scoping.md`](1.0/10-agent-scoping.md) | per-agent capability scoping |
| [`1.0/11-io-devices.md`](1.0/11-io-devices.md) | io device-driver model (sibling adapters · fixed `io.*` surface · policy gating) |
| [`1.0/12-flowbrain-runtime.md`](1.0/12-flowbrain-runtime.md) | FlowBrain runtime wiring (4-axis ask composition · work-time gate · review) |
| [`1.0/13-datastores.md`](1.0/13-datastores.md) | datastore data interface (`fs.*`/`db.*` · sibling DB adapters · internal policy · separate layer from io) |

## Related specifications

- [`../bundle/`](../bundle/) — bundle manifest implementation spec
- [`../ui_dsl/`](../ui_dsl/) — UI DSL implementation spec

## Change contract

This spec defines a framework-level contract. On change:

1. Verify conformance with the reference implementation.
2. Check impact on the instance matrix ([`1.0/09-instances.md`](1.0/09-instances.md)).

## License

MIT — see [../LICENSE](../LICENSE).
