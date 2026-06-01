# 09 — Instance Matrix

Instances of the same framework. Currently in-progress instances + potential instances + arbitrary user instances. Each instance = a composition decision on top of the framework spec (00-08).

## Current Instances

### vibe_studio

| Dimension | Composition |
|---|---|
| UI | builder chrome (Flutter) + mcp_ui_dsl workspace + multi-tab |
| bundle | studio itself + domain-bundle authoring (multi-bundle) |
| kernel | brain_kernel |
| transport | server (ServerBootstrap) + client (McpClientKernelHost) |
| composition pattern | Pattern 3 hybrid + Pattern 4 multi-bundle |
| LLM | multi-provider (anthropic · openai · gemini + claude_code subscription) |

Location = `tools/appplayer_vibe_studio/debug/vibe_studio/` (active)

### AppPlayer

| Variant | Composition |
|---|---|
| **Standard** | UI player chrome + external MCP server connect or local bundle · transport client only · Pattern 1 or Pattern 2 |
| **Pro** | UI player chrome + launcher + library + (optional) server endpoint · multi-bundle · free combination of Pattern 1+3+4 |
| **X** | standalone shell (1 bundle embed) · client only · Pattern 2 |
| **Custom** | whitelabel · user environment · free composition |

Implementation = `appplayer_core` (Core) · `appplayer` (Standard) · `appplayer_pro` (Pro).

### FlowBrain

| Dimension | Composition |
|---|---|
| UI | knowledge explorer + graph + search (host design) |
| bundle | domain knowledge bundle (multi-bundle possible) |
| kernel | brain_kernel |
| transport | in-process · or server (running a knowledge service) |
| composition pattern | Pattern 2 or Pattern 1 |
| use | personal · team · organizational knowledge |

Implementation = `flowbrain_core`.

### FlowOS

| Dimension | Composition |
|---|---|
| UI | unified OS chrome (host decision) |
| bundle | package integration + modularization |
| kernel | brain_kernel (or direct) |
| transport | server / standalone / embedded |
| composition pattern | free combination of all patterns |
| use | OS-level operation |

Implementation = the FlowOS host.

## Potential Instances (thesis-anchored)

### Form Studio (memory thesis)

| Dimension | Composition |
|---|---|
| UI | form view + pdf/html renderer (own widget · or mcp_ui_dsl) |
| bundle | template + agent + 8-facade data |
| kernel | brain_kernel + mcp_form (template/binding/5 renderer) |
| composition pattern | Pattern 2 or Pattern 3 |
| use | AI document/report automation |

Rationale = memory `project_form_studio_app_thesis`.

### Industrial HMI

| Dimension | Composition |
|---|---|
| UI | HMI panel + real-time graph (own widget) |
| bundle | equipment tools + operational knowledge |
| kernel | brain_kernel + mcp_io (transport adapter) |
| composition pattern | Pattern 1 (equipment server) + Pattern 4 (multi-device) |
| use | industrial operation / manufacturing / medical / etc. |

Rationale = memory `project_vibe_studio_domain_catalog` · `project_appplayer_usb_secure_device`.

### Vertical SaaS instances (marketplace)

| Dimension | Composition |
|---|---|
| UI | per-domain UI (POS · LMS · student management · ELN · CRM · ERP, etc.) |
| bundle | vertical domain bundle |
| kernel | brain_kernel |
| composition pattern | free per domain (usually Pattern 1 + 4) |
| use | per-domain SaaS |

Rationale = memory `project_vibe_studio_vertical_saas_thesis`.

### Arbitrary User Host

| Dimension | Composition |
|---|---|
| UI | user's own |
| bundle | user's own `.mbd` |
| kernel | brain_kernel |
| composition pattern | user's choice |
| use | arbitrary product |

= the framework's true value — a user can build their own product on the same framework.

## Composition Matrix (summary)

| Instance | UI | bundle source | transport | bundles | LLM |
|---|---|---|---|---|---|
| vibe_studio | builder chrome | local (authored) | server + client | multi | multi-provider + claude_code |
| AppPlayer Standard | player chrome | local or server | client only | single | host choice |
| AppPlayer Pro | player chrome + launcher | multiple (library) | client (+ optional server) | multi | host choice |
| AppPlayer X | standalone | embed (1) | client | single | host choice |
| AppPlayer Custom | whitelabel | site-specific | client + server | free | site decision |
| FlowBrain | knowledge explorer | domain (1+) | in-process / server | single or multi | host choice |
| FlowOS | unified OS | package | all | all | all |
| Form Studio | form view | template + 8 facade | in-process | single | arbitrary |
| Industrial HMI | HMI panel | equipment + operation (multi) | client (equipment) + optional server | multi | arbitrary |
| vertical SaaS | domain UI | vertical bundle | free per domain | free | arbitrary |
| arbitrary host | free | free | free | free | free |

## Cross-Instance Commonality

Shared by all instances:
- **brain_kernel** (8 facades + 41 standardTools + bridge + HostToolRegistry + 4 ports + LLM pool)
- **bundle manifest spec** (the `mcp_bundle` schema)
- **MCP protocol** (transport · message format)
- **kb:// URI scheme** (knowledge-asset reference)
- **three tool-registration paths** (06-tool-registry.md)
- **two naming models** (knowledge-tool isolation)

= per-instance variation = **UI surface + bundle source + composition-pattern decision** only. No framework spec change.

## Adding a New Instance — Guide

Follow the standard procedure in [`08-extension.md`](08-extension.md). Each instance = one row added to this matrix. No framework spec change.

## Non-Goals

- Per-instance product detail · design · business model = out of scope (each host's domain)
- Cross-instance commercial relationships = out of scope
- This spec = **the framework-composition-decision matrix of instances** only
