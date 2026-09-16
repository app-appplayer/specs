# 06. Runtime Contract

This section defines the contract between a DSL runtime and its MCP server: protocol facilities, initialization flow, resource subscription modes, host integration points, lifecycle hook placement, error handling, and performance expectations.

Normative requirements live in [`18_Conformance.md`](18_Conformance.md) §18.2.6 (MCP protocol), §18.2.11 (performance).

## 6.1 MCP Protocol Integration

A conformant runtime communicates with a server over the Model Context Protocol (JSON-RPC 2.0 — https://spec.modelcontextprotocol.io). The DSL relies on three protocol facilities:

| Facility | Purpose |
|----------|---------|
| `resources/read` | Fetch UI definitions and bundle assets (`ui://app`, `ui://page/*`, `ui://app/info`, `bundle://...`) |
| `tools/call` | Execute server-side tools invoked by `{"type": "tool", ...}` actions |
| `notifications/*` | Receive server-pushed events, including `notifications/resources/updated` for subscribed resources |

All DSL definitions MUST be valid JSON. The runtime performs no speculative parsing: a malformed document is rejected whole (see §6.9).

## 6.2 Well-Known Resources

| URI | Returns | Since |
|-----|---------|-------|
| `ui://app` | ApplicationDefinition | v1.0 |
| `ui://page/{name}` | PageDefinition for a lazy-loaded route | v1.0 |
| `ui://page/info/{name}` | Optional lightweight page metadata | v1.0 |
| `ui://app/info` | Lightweight app metadata — see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.6 | v1.2 |
| `bundle://...` | Bundle-internal asset — see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.5 | v1.2 |

Runtimes claiming the Bundle Profile MUST resolve `bundle://` URIs inside the currently loaded bundle only.

## 6.3 Initialization Flow

1. Runtime connects to the MCP server.
2. Runtime issues `resources/read` for `ui://app`.
3. Runtime parses `ApplicationDefinition` and resolves its `version` (defaults to `"1.0"` when absent — see [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.7).
4. Runtime evaluates `theme`, `state`, and `navigation` on the ApplicationDefinition.
5. Runtime navigates to `initialRoute` (or the first declared route when absent).
6. Runtime fetches the initial page definition via `resources/read` on the mapped `ui://page/*` URI.
7. Runtime renders page content.
8. Runtime fires definition-level lifecycle hooks in order: `onInit` → `onMount` → `onReady` (see §6.8).

Subsequent route changes repeat steps 6–8 for the new page.

## 6.4 Resource Subscription

A runtime MAY subscribe to any resource URI it has read. Servers notify changes via `notifications/resources/updated`.

Two subscription modes:

| Mode | Notification payload | Runtime behavior |
|------|----------------------|------------------|
| **Standard** | URI only | Re-issue `resources/read` to fetch updated content |
| **Extended** | URI plus the new content | Apply the content directly without re-fetching |

Core Profile requires Standard mode; Extended mode is SHOULD (see [`18_Conformance.md`](18_Conformance.md) §18.2.6). A runtime MAY negotiate Extended mode at connect time; when unavailable it MUST fall back to Standard.

Subscriptions are released when the subscribing scope unmounts (e.g., a page subscription is released on `onDestroy`).

## 6.5 Tool Calls

A `{"type": "tool", ...}` action maps to `tools/call` on the server:

```json
{
  "type": "tool",
  "tool": "loadDashboardData",
  "params": { "userId": "{{app.user.id}}" },
  "onSuccess": { "type": "state", "action": "set", "binding": "page.data", "value": "{{event.result}}" },
  "onError":   { "type": "notification", "message": "{{event.error.message}}", "severity": "error" }
}
```

Tool responses populate the `event.*` binding scope for `onSuccess` / `onError` callbacks (see [`04_Actions.md`](04_Actions.md) §4.7–§4.8). A long-running tool call carries an implicit cancellation handle; the `cancel` action can target it by id (see [`04_Actions.md`](04_Actions.md) §4.9).

## 6.6 Host Integration

The runtime exposes a small surface to its embedding host:

### 6.6.1 `onExit` Registration

The host registers an exit callback through the runtime's single entry point — e.g., `MCPUIRuntime.buildUI(onExit: cb)` in the Dart implementation. `buildUI(onExit:)` is the one canonical registration path; there is no separate `registerOnExit()` or post-build registration API.

When `onExit` is registered:

- `{"type": "navigation", "action": "exitApp"}` invokes the callback (see [`04_Actions.md`](04_Actions.md) §4.2).
- The runtime automatically appends a **host close button** to `headerBar.actions` on the **root route only**, positioned at the trailing (rightmost) edge after any app-defined actions. Tapping it invokes `exitApp`. The button uses the `close` icon by default; see [`02_Widgets.md`](02_Widgets.md) §2.8.1 for the `headerBar.exitButton` customization hook.
- On inner routes, the AppBar `leading` is the automatic back button and the host close button is not rendered.

When `onExit` is not registered, `exitApp` is a no-op and the host close button is not rendered.

### 6.6.2 Tool Executor Hooks

The runtime MAY expose optional pre/post hooks around `tools/call` for logging, authentication injection, or parameter rewriting. Hook failure MUST NOT corrupt the DSL-visible tool result.

## 6.7 Page Transitions

Transition animations between routes are declared on the `navigation` action (`pageTransition` field) or at page level. See [`16_Animations.md`](16_Animations.md).

## 6.8 Lifecycle Hook Placement

The DSL uses a dual placement rule for lifecycle hooks:

| Placement | Applies to | Shape |
|-----------|------------|-------|
| **Definition-level** | `ApplicationDefinition`, `PageDefinition` | Hooks are **top-level properties** of the definition (e.g., `"onInit": [...]`), **or** grouped in a `"lifecycle": {...}` object. Both forms are valid and the two sets merge — see §1.5.3 |
| **Instance-level** | Any widget (including template instances) | Hooks are wrapped in a `"lifecycle": {...}` object property on the widget |

Definition-level placement reflects that the definition itself is the lifecycle-aware entity. Instance-level placement keeps lifecycle concerns explicitly separated from the widget's own properties and prevents collision with them — a widget carries arbitrary properties, so a bare `onMount` beside `onTap` would be ambiguous, while a definition's key set is declared.

The grouped form is therefore the one that reads the same everywhere, and is **preferred** for new documents: it is the only placement valid for both a definition and a widget. Top-level hook fields on a definition remain valid.

A runtime MUST read both placements. Reading only one is invisible to the author — a hook that is never parsed is a hook that never runs, with nothing in any log to say so.

### 6.8.1 Definition-Level Example

```json
{
  "type": "page",
  "title": "Dashboard",
  "onInit":    [ { "type": "tool", "tool": "loadDashboardData" } ],
  "onReady":   [ { "type": "resource", "action": "subscribe", "uri": "ui://metrics/live", "binding": "page.metrics" } ],
  "onPause":   [ { "type": "state", "action": "set", "binding": "page.isPaused", "value": true } ],
  "onResume":  [ { "type": "state", "action": "set", "binding": "page.isPaused", "value": false } ],
  "onDestroy": [ { "type": "resource", "action": "unsubscribe", "uri": "ui://metrics/live" } ],
  "content":   { "...": "..." }
}
```

### 6.8.2 Instance-Level Example

```json
{
  "type": "box",
  "lifecycle": {
    "onMount":   { "type": "state", "action": "set", "binding": "page.ready", "value": true },
    "onUnmount": { "type": "state", "action": "set", "binding": "page.ready", "value": false }
  },
  "child": { "...": "..." }
}
```

### 6.8.3 Hook Firing Order

For a page mount: `onInit` → `onMount` → `onReady`.
For a page unmount: `onUnmount` → `onDestroy`.

Navigation splits on one question — **does the outgoing instance survive?**

| Navigation | Outgoing page | Incoming page |
|---|---|---|
| Replaces the outgoing page | `onUnmount` → `onDestroy` | `onInit` → `onMount` → `onReady` |
| Stacks over it (outgoing kept alive) | `onPause` | `onInit` → `onMount` → `onReady` |
| Returns to a kept-alive page | `onUnmount` → `onDestroy` | `onResume` |
| Returns to a page that was replaced | `onUnmount` → `onDestroy` | `onInit` → `onMount` → `onReady` |

A destroyed instance MUST NOT fire `onPause`. §1.5.1 defines that hook as
losing active focus *without* being destroyed, and §1.5.2 draws it as half of
the `(onPause ↔ onResume)*` pair; an instance that fires it and then dies has
satisfied neither. The distinction is what the hook is *for*: an author saves
a draft, stops a timer, or parks a subscription there on the understanding
that this instance comes back. Firing it on the way out makes teardown work
placed in `onPause` appear to run, while the state it saved is discarded with
the instance and rebuilt from scratch on the next visit.

A shell that switches between pages — a tab bar, a rail, a bottom bar — keeps
them. Selecting another item is the second row of that table, not the first:
the page being left is still mounted and fires `onPause`, and coming back to
it fires `onResume` on the same instance. A page is built on its first visit,
so an application with six tabs does not run five `onInit`s before anyone has
looked at them.

**A paused page pauses what is inside it.** Instance-level `lifecycle` blocks
(§6.8.2) and embedded `view` definitions stay mounted with the page, so they
receive `onPause` and `onResume` with it. Without that, a widget that started
a timer or a subscription on mount keeps running behind a page nobody is
looking at, and never hears the resume its own document declares — the hooks
would be reachable only by destroying the page, which is the thing that is no
longer happening.

Which navigations keep an instance alive is otherwise a host decision
(§1.5.2), so a document MUST NOT assume that leaving a page will pause it
rather than destroy it. Work that must happen exactly once per document belongs on the
application, whose instance outlives every page.

Hooks within the same stage execute in definition order. A failing hook logs its error; subsequent hooks MUST still run. `onInit` hooks complete before `onReady` begins. `onDestroy` completes before the runtime releases page-scoped resources (subscriptions, channels, local state).

See [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.5 for the canonical hook list.

## 6.9 Error Handling

| Condition | Runtime behavior |
|-----------|------------------|
| Invalid JSON for a definition | Reject the whole definition; MAY render a runtime error widget in its place |
| Unknown widget type | Render an error widget at that position; log; continue rendering siblings |
| Binding resolution failure | Render empty (or declared fallback); log |
| Tool or resource error | Invoke the action's `onError` callback when present; otherwise log |
| Subscription failure | Log; attempt reconnection per the runtime's reconnect policy |

A runtime MUST NOT crash the host application on any of the above conditions. It MAY expose aggregate error state via `runtime.*` bindings or a diagnostic panel.

## 6.10 Performance Contract

Targets (SHOULD, see [`18_Conformance.md`](18_Conformance.md) §18.2.11):

- Widget tree render < 100 ms for 1000 widgets
- Binding resolution < 10 ms for 1000 expressions
- State update propagation < 16 ms (single 60 fps frame)

A runtime MAY publish measurements via a diagnostic channel; the contract applies to steady-state rendering after initialization.

## 6.11 Multi-Origin Resolution *(since v1.4, Composition Profile)*

§6.1 describes the runtime's relationship with **one** server. A document that composes several origins (see [`01_Core_Concepts.md`](01_Core_Concepts.md) §1.9) needs one more contract: how a `DefinitionSource` naming a `from` origin is resolved, and what the resolved subtree is attached to.

### 6.11.1 Connections are a host capability, not a DSL feature

The DSL does **not** define how a connection is opened, authenticated, or torn down. It defines only how a definition is addressed once a connection exists. Establishing outbound connections is the host's job, exposed to the document as ordinary tools.

The canonical host surface is the platform's outbound MCP client tool set (`mcp.connect` / `call_tool` / `read_resource` / `list_tools` / `list_resources` / `disconnect`) — an application calls `mcp.connect` in `lifecycle.onInit`, keeps the returned id in state, and refers to it as `{ "connection": "{{conn.temp}}" }`. Hosts that expose a different naming for the same capability substitute their own; the `Origin` shape is what this specification fixes.

Consequently a composing document needs no new declaration to say "I open connections" — it already declares its use of the host's tool surface through the host's existing capability-gate mechanism.

### 6.11.2 Resolution

For a `DefinitionSource` of the qualified form `{ "$ref": <uri>, "from": <origin> }`:

1. Resolve `from` to a live connection. Any binding inside it resolves first, in the scope containing the source.
2. If no such connection exists, or the origin key is unrecognised, resolution **fails** — see §6.11.4. The runtime MUST NOT fall back to the current origin.
3. Read `<uri>` from that connection via `resources/read`.
4. Parse the result as a definition. A malformed document fails resolution (§6.9 — rejected whole).
5. Attach the parsed definition to a subtree whose ambient origin is that connection.

For the binding form, steps 1–3 are skipped: the definition is already in state, and its ambient origin is the origin of the scope that holds it.

### 6.11.2a What the host must provide

Resolution is one of **three** capabilities a host wires, and a runtime claims the profile only when it has all three. They are listed together because a host that wires the first alone produces a composed screen that renders correctly and does nothing — every control inside the embedded subtree takes the app's own path, and the failure surfaces as an unrelated "no client" rather than as a missing capability.

| Capability | What it does | Missing ⇒ |
|---|---|---|
| **Resolve** | Read a definition from a named origin | `view` fails closed to `fallback` (the runtime does not implement the profile) |
| **Call** | Invoke a tool on a named origin | subtree renders; every control silently reaches the wrong server |
| **Watch** | Track a resource on a named origin | live readings render their label and never a value |
| **Read** | One-shot read on a named origin | a read returns the *embedder's* resource under the embedded document's uri — a wrong answer, not a missing one |

Read is separate from Watch on purpose: a read that leaves a subscription behind keeps the device pushing to a view that asked once.

A runtime that does not have all four MUST NOT report the profile as implemented (§18.7).

**Storage and permissions scope too.** An embedded subtree has its own storage identity: two devices on one screen that both store `config` MUST NOT share a key space, and neither may write into the embedder's. Its permission set is the **intersection** with the embedder's, never the union — an embedded document may not request, or be granted, more than the app embedding it holds. Enforce the ceiling *before* prompting: asking the user and then refusing is worse than never asking.

**Opening is the host's, and is deferred.** A document names an origin; it never opens one. A host MAY open a named origin on first use rather than holding one open per known device — and SHOULD, because devices that serve a single peer at a time are reset by a second connection, so permanent connections make the last one opened evict the others.

**A tool call MUST NOT be redirected.** If a subtree is scoped to an origin and the host wired no Call capability, the action fails. Falling back to the embedder's own server would run one device's tool name against another.

### 6.11.2b An embedded definition runs its own lifecycle

A definition is the lifecycle-aware entity (§6.8), and that does not change when it is embedded. A runtime MUST seed the embedded definition's `state.initial` into its scope and fire `onInit` → `onMount` → `onReady` once per mount, and `onDestroy` on unmount.

**Which definition owns a hook matters more once composition exists.** An application hook runs for the application as a whole; a page hook runs while that page is shown (§1.5). A standalone application and its initial page share one scope, so a subscription placed on either appeared to work — but an embedded page has its own scope, and a value written by the *application's* hook lands where the page cannot read it. The reading then renders its label and never a value, with every layer beneath reporting success.

So: **the definition that BINDS a value owns the hook that starts it.** A page that subscribes in its own `onReady` and releases in its own `onDestroy` is self-contained — it behaves identically opened on its own and embedded in another host's screen. This is the shape §6.8.1 already shows.

### 6.11.3 One runtime scope per origin

The resolved subtree runs in its **own scope**: its own state tree, its own subscription registry, its own permission and storage identity. It is not a sub-tree of the embedder's state.

`notifications/resources/updated` arriving on a connection dispatch to the scope(s) bound to that connection, never to the embedder's scope. Two `view`s bound to the same connection share that connection but hold separate scopes.

The scope's lifetime is the **mount**, not the render. A scope rebuilt per frame discards everything the embedded definition put in it — a value arrives, the write triggers a rebuild, and the rebuild throws the value away, so the reading never appears while every layer beneath reports success. Runtimes MUST create the scope once per mounted source and MUST replace it when the source changes (a different origin must never inherit the previous one's state).

Bindings in the embedded subtree MUST re-evaluate when that scope's state changes. A subtree evaluated once at mount can render a live reading's label and never its value.

**A source is compared by value, not identity.** An embedded application whose route is a `ui://` uri resolves one level further, and the nested source is rebuilt each frame — structurally equal, a different object. Comparing by identity reads that as a changed origin, so the subtree resolves, renders, rebuilds and resolves again: on real hardware the view flickered between its content and its loading indicator and then stayed on the indicator. A route value that is a bare uri also MUST inherit the embedding view's origin — the route belongs to the application that declared it.

### 6.11.4 Failure and lifecycle

- **Resolution failure is local.** A route whose source fails to resolve reports a navigation error; a `view` whose source fails renders its `fallback` and fires `onError` while its siblings and the embedding page render normally.
- **Disconnect.** When a connection drops, scopes bound to it enter the failed state and render `fallback`. Their subscriptions are dropped.
- **Reconnect.** On reconnect the runtime MUST re-resolve the source and remount the scope. Local state inside the embedded scope does not survive; a definition that must survive reconnection persists through its own origin. (Remount rather than resume is chosen because the origin may have changed what it serves.)
- **Depth and cycles.** Runtimes MUST enforce a maximum nesting depth for embedded definitions and MUST detect cycles in the origin/URI pair chain. On either, the offending source fails resolution and renders `fallback` — it MUST NOT recurse.

## 6.12 Asset Resolution *(since v1.4)*

§6.11 says how a *definition* reaches a runtime. This section says how an **asset** does. Every widget slot typed `AssetRef` — `image.src`, `icon.icon`, `avatar.src`, `lottieAnimation.src`, `BackgroundImage.image`, `mediaPlayer.{source,poster}`, `rive.src`, `lightbox.images[]`, and the `app` / `theme` asset slots — resolves through the one contract below. A runtime MUST NOT implement asset loading per widget: two widgets given the same `AssetRef` MUST resolve it identically.

### 6.12.1 Resolution is by scheme, and the scheme set is open

A runtime dispatches on the reference's scheme prefix (or, for the object form, reads `uri` through `resources/read` on the resolved origin — §6.12.3). The schemes named in `AssetRef` are the ones this document defines; a host MAY resolve others.

**An unknown scheme is not an invalid document.** A runtime that meets a scheme it does not resolve MUST treat it as an *unresolvable asset* (§6.12.4), not as a schema violation. Rejecting the document would make every runtime's gaps into authoring errors, and an author cannot know in advance which runtime will render their page.

### 6.12.2 A binding is resolved first

`AssetRef` admits a binding in every position. A runtime MUST resolve the binding **before** dispatching on scheme — an asset whose source arrives in state is the normal case, not an edge one, and a slot that dispatches on the literal `"{{item.picture}}"` will find no scheme and fail on a document that is correct.

### 6.12.2a An empty string is not a reference

`""` is not a valid `AssetRef`: there is no asset it could name. A slot whose source may legitimately be absent declares a **binding**, and the runtime treats a binding that resolves to empty or `null` as an unresolved asset (§6.12.4) — the same path an unsupported scheme takes.

Stated because the alternative is worse in a way that only shows up later: admitting `""` into the type would make every asset slot silently accept a typo that produced an empty string, and the author would see the fallback and conclude the asset was missing rather than misspelt.

### 6.12.3 Ambient origin

An `AssetRef` in object form without `origin`, and a `bundle://` reference, both resolve against the **ambient origin**: the origin of the definition that declares them, never the embedding document's (§6.11.3). An embedded subtree's `bundle://logo.png` is its *own* bundle's logo. A runtime that resolves it against the embedder's bundle serves one server's asset under another's identity — the same failure [`07_Security.md`](07_Security.md) §7.10 names for definitions.

### 6.12.4 Unresolvable is declared, never silent

Assets differ from definitions in one way that matters: a runtime is not expected to resolve all of them. Constrained hosts exist, and a device that can inline bytes may have no filesystem, no network stack, and no room to hold a bundle.

So the contract is **honesty, not completeness**:

- A runtime MUST publish the set of `AssetRef` forms it resolves ([`18_Conformance.md`](18_Conformance.md) §18.2.12). "Publish" means discoverable by the host that embeds it, not merely documented.
- An asset that cannot be resolved — unsupported scheme, missing capability, read failure, or a payload the runtime cannot decode — MUST take the slot's **declared fallback path** where the widget defines one (`image` has `fallback`, `fallbackUrl`, and `fallbackBehavior`), and MUST otherwise render as an absent asset per §6.9.
- A runtime MUST NOT render an implementation detail in place of the asset. A box reading `Base64 not supported` states the runtime's limitation in the user's screen; the author asked for a picture, and the failure belongs in the diagnostic channel (§6.9), not the layout.
- A runtime MUST NOT silently substitute a different origin's asset, or the embedder's, for one it could not resolve.

### 6.12.5 Reading is asynchronous

`bundle://`, `client://`, and origin-served references are read asynchronously. A runtime whose asset path is synchronous can only ever support the forms that need no I/O, and will appear to support the contract while resolving a strict subset of it. Asset resolution MUST therefore be modelled as an asynchronous read with a pending state, and a slot awaiting bytes MUST render its declared loading state rather than its fallback — a fallback shown while a read is in flight reports a failure that has not happened.

### 6.12.6 Size is a host policy, not a document one

A host MAY decline to inline or cache an asset above a size it chooses. That decision is a resolution failure like any other and takes the path in §6.12.4; it MUST NOT be reported as a malformed reference, and the threshold MUST NOT appear in the document. An author writes what the asset *is*, not how large the host will tolerate it being.

### 6.12.7 Who resolves `bundle://`, and when

§6.12.3 says *which* bundle a `bundle://` reference names. This says who reads
it, because the answer has been left to each host and the two ways of doing it
are not interchangeable.

A `bundle://` reference is resolved **against the bundle the client already
holds** — the ambient origin's bundle (§6.12.3). It is never a request to the
server that sent the document: a server naming `bundle://logo.png` is naming
the running app's own asset, not one of its own files. A server's own files
reach the client by the serving conventions in
[`mcp_serving`](../../../mcp_serving/spec/1.0/README.md) §4, which need no
scheme in this DSL.

Two placements are conformant:

1. **Resolved before the runtime sees the document.** The host walks the
   definition (and every page it later loads) and replaces each `bundle://`
   string with something the runtime already resolves — usually a `data:` URI
   or a local path. The runtime never meets the scheme.
2. **Resolved inside the runtime.** The host gives the runtime read access to
   the active bundle, and the runtime resolves the scheme like any other
   (§6.12.5, asynchronously).

Placement is a host choice. What is **not** a choice:

- **A host MUST apply the same placement to every document it loads,
  regardless of how the document arrived.** A host that resolves `bundle://`
  for a locally installed bundle and not for a document read from a connected
  server makes the same reference render in one path and vanish in the other,
  and the author has no way to tell which path their document will take.
- Placement 1 satisfies §6.12.3 by construction, because the resolver belongs
  to the document's own bundle. Placement 2 does not: a runtime resolving the
  scheme itself MUST know which bundle is ambient for the subtree being built,
  or an embedded subtree's `bundle://logo.png` will find the embedder's logo —
  the substitution §6.12.4 forbids.
- A host with no bundle loaded has nothing to resolve against, and
  `bundle://` there is an unresolvable asset (§6.12.4) — not an error in the
  document, which may be perfectly valid in a host that does hold the bundle.


### 6.12.8 An asset travels as a reference, not as state

A tool result, and therefore state, is a value the runtime copies, merges and
re-parses on every delivery. An asset is not that: it is bytes with an
identity, and every layer that knows the identity can skip the bytes.

Authors SHOULD carry assets as references (`bundle://`, `resource://`, a URL)
and let the host resolve them. Embedding the bytes — a `data:` URI, base64 in
a tool result — is legal and sometimes the only option, but it MUST be
understood as giving up the identity:

- The reference is what a cache is keyed on. Bytes carried in state are
  re-sent in full every time the tool is called again, re-parsed with the
  rest of the payload, and re-merged into state; a reference costs the same
  few dozen characters however many times it arrives.
- Only decoding can be recovered after the fact. A runtime MAY cache the
  decode of a `data:` URI keyed on the URI string, and one that does removes
  the repeated decode — it cannot remove the transfer or the parse, because
  those already happened before the runtime saw the value.
- Size, per §6.12.6, is a host policy. An inlined asset is not subject to it:
  the host never sees a reference it could decline, so a document that
  embeds bytes bypasses the one place that limit is meant to live.

**Send an asset at the size it is drawn at.** Everything above is about the
second delivery; the first one is paid whatever happens, and its cost is set
by the transport, not by any cache. The same picture at its source resolution
and at the size the screen actually uses differed by 1,268 ms against 57 ms on
a local pipe — and a pipe is the fastest transport there is. On a serial link
the same difference is tens of seconds, which is the difference between a
screen appearing and a device that cannot present one.

No layer below the author can do this. A runtime that receives bytes cannot
know what they were meant to be, and a host applying §6.12.6 sees a reference
it may decline, not a picture it may resize.

This is guidance, not a constraint on the wire format. A runtime MUST NOT
reject a document for carrying an inline asset, and MUST NOT impose a size
limit on state (§6.12.6 governs *assets*, and the host cannot tell which
string in a payload was meant to be one).

## 6.13 Declared Behaviour *(since v1.4)*

§6.12.4 fixes honesty for assets. This section states the same rule for the
**behaviour a widget declares** — playing media, loading a page, drawing a map,
speaking, signing — because the failure mode there is worse and was found in the
field: a runtime that cannot perform a behaviour can still draw something that
looks like it is performing it.

### 6.13.1 A declared behaviour is performed or reported (MUST)

A widget whose contract declares an effect — sound comes out, a page loads, a
document renders, an animation plays — MUST either perform that effect or report
that it cannot. There is no third state.

A runtime MUST NOT render a facsimile of the behaviour succeeding. Concretely,
and each of these has been shipped by an implementation of this spec:

- a media transport whose position advances on a timer while nothing is decoded,
  firing `onPlay` and `onEnded` as though playback occurred;
- a web view that reports a successful load and renders the URL as text;
- a map that draws a coloured rectangle where tiles would be.

Each satisfies "parse and render" and each tells the user the opposite of the
truth. **A silent failure is recoverable; a simulated success is not** — nobody
looks for a bug in something that appears to work, and the author ships a
document believing the effect reached the user.

### 6.13.2 Inability is a capability fact, not a rendering (MUST)

A runtime that lacks a behaviour MUST publish that fact the way §6.12.4 requires
for asset forms — discoverable by the embedding host, not merely documented — and
MUST route the individual failure to the widget's declared error path
(`onError` where the widget defines one) and to the diagnostic channel (§6.9).

The screen is not the diagnostic channel. A box reading "video not supported"
is the substitution §6.12.4 already forbids for assets, and it is forbidden here
for the same reason: the author asked for an effect, and the layout is not where
the runtime's limits are reported.

### 6.13.3 A host-provided capability is the normal case (informative)

Most of these behaviours are platform powers, not rendering: an audio decoder, a
web engine, a tile source. A runtime is expected to accept them from its embedder
rather than carry them, exactly as it accepts asset resolution. The contract
above is written so that a runtime with none of them is still conformant — it
declares what it has and reports what it does not — while one that fakes them is
not.

