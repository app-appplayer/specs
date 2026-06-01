# 00 — Overview

## 1. Definition

**MakeMind Platform** = a framework based on MCP (Model Context Protocol). It produces an arbitrary product by composing the three layers **UI + bundle + kernel (operations)**. Every composition (server · client · bundle · tool · knowledge) is expressed over the MCP protocol.

## 2. Source of truth for ideas

- **Single framework, infinite instances** — varying the composition pattern of UI · bundle · kernel → diverse products
- **MCP-native** — the common protocol for all connections
- **Bundle is self-contained** — a single `.mbd` behaves through the same composition interface on any host
- **Kernel is decoupled** — whatever surface the UI is, whatever domain the bundle is, the kernel operational logic is shared

## 3. baseline behavior → extension

```
[baseline]
  MCP server ↔ MCP client = serving + consumption
  = the common protocol for all connections

[extension 1] server-side self-containment
  server = serves UI + bundle manifest + tools + knowledge all together
  → client = thin player (renderer)

[extension 2] local + remote hybrid
  composition of a local bundle + the UI / tools / knowledge of a server (or another bundle)
  → partial self-containment + partial remote serve

[extension N] ... infinite
  combinations of multiple servers + multiple bundles + multiple UI surfaces
  same framework · new composition pattern
```

## 4. 3 layers (brief)

```
[UI layer]         user surface — chrome · runtime · widget · DSL · result exposure
       ↕
[bundle layer]     conformance interface — manifest.tools/agents/6 facade/ui/wiring/settings/chat/requires
       ↕
[kernel layer]     operational logic — facade(8) · agent · standardTools(41) · bridge ·
                   HostToolRegistry · KernelApp.toolsForAgent (per-agent scoping) ·
                   hostMcpServerSpec({endpointLabel}) · KernelServerHost.addPrompt
                   (MCP prompts wrapper) · 4 ports · LLM pool (live view)
```

Details: [01-layers.md](01-layers.md)

## 5. Instances (examples — all the same framework)

| product | UI | bundle | kernel |
|---|---|---|---|
| **vibe_studio** | builder chrome · DSL workspace · multi-tab | studio's own tools + builder bundle | brain_kernel |
| **AppPlayer** (Standard · Pro · X · Custom) | player chrome · launcher · library | playback-target bundle · external MCP server | brain_kernel |
| **FlowBrain** (knowledge operations) | knowledge explorer · graph · search | domain knowledge bundle | brain_kernel |
| **Form Studio** | form view · pdf/html renderer | template + agent + 8 facade data | brain_kernel + mcp_form |
| **industrial HMI** | HMI panel · real-time graph | equipment tools + operational knowledge | brain_kernel + mcp_io |
| **arbitrary user host** | own | own .mbd | brain_kernel |

Details: [09-instances.md](09-instances.md)

## 6. spec layout

| spec | area covered |
|---|---|
| 00-overview (this document) | overall definition |
| 01-layers | responsibilities of the 3 layers + composition points |
| 02-bundle-interface | bundle manifest |
| 03-kernel-runtime | kernel logic |
| 04-ui-host | responsibilities of the UI host |
| 05-composition-patterns | composition patterns |
| 06-tool-registry | tool registration paths · two-name model |
| 07-knowledge-access | knowledge access · scopeId |
| 08-extension | standard for a new instance |
| 09-instances | current + potential instances |
| 10-agent-scoping | per-agent capability scoping — the advantage of introducing a true kernel (canonical specification) |

## 7. Non-goals

- Values / principles / motivations of ideas
- Package inventory
- Workspace directory structure
- This spec = **the structural contract of the framework only**
