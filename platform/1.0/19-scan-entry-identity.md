# 19 — Scan entry, identity & custody (entering an app from a physical medium)

> status: **draft** (2026-07-29 new · rev.4 after conformance audit · awaiting approval · no implementation yet)
> peer: [`04-ui-host.md`](04-ui-host.md) §origin scoping (an entry never crosses origins) · [`05-composition-patterns.md`](05-composition-patterns.md) (which composition an entry resolves to) · [`08-extension.md`](08-extension.md) §4 (connection layers · the `Authorization: Bearer` mapping a grant rides · the embed a `listing` target uses) · [`17-device-discovery.md`](17-device-discovery.md) (how a `localServer` target is found and how it proves itself) · [`14-asset-credentials.md`](14-asset-credentials.md) (where a durable credential lives — a scan grant is NOT one)
> binding: MCP UI DSL — a new client-extension section for the `identity.*` / `entry.*` surfaces (§8 here fixes the contract the DSL section must expose)

A person taps a physical thing — a code on a windshield, a tag on a machine, a card, a printed document, a link in a message — and lands inside an app, on the **right page**, with the **right amount of identity**. Later the thing changes hands: the salesperson leaves, the vehicle is sold, the equipment moves to another team. The printed code does not change.

This standard fixes three things so that both halves work: how an entry link resolves to a target, how identity participates **as a parameter rather than as a fork**, and how a medium's **custody** is rebound without reissuing it.

The rules this document exists to protect:

> **A single entry point serves an anonymous passer-by and a signed-in owner. Which one arrives is decided at scan time, not at design time.**
>
> **A medium outlives the parties it speaks for. Changing the person never means printing a new code.**

Everything below follows from refusing to build two systems, and from refusing to treat a physical medium as disposable.

---

## 1. Principle — identity and custody are parameters of an entry, not classes of entry

The tempting design is two products: a "public page" and an "app". It fails immediately in ordinary business, because the same medium is scanned by both populations, often minutes apart. A stranger needs to reach the party responsible for a vehicle without an account. That party scans the identical code and expects their own management surface. A technician scans it and expects service history. One medium, one link, three surfaces.

The second tempting design is a medium bound to a person. It fails on the first personnel change. A company issues a card for a salesperson; the salesperson leaves; the customers holding that card are the company's asset and must not be lost. A driver sells the car; the code on the windshield belongs to the vehicle, and the contact behind it must move to whoever drives it now.

So both are modelled as **parameters**, open at every level at once (guest · end user · tenant operator · machine), never pre-forked into modes or editions.

Three consequences are normative:

- An entry declares an **identity policy** (§4.2), not an audience.
- The runtime exposes an **identity state** (§8.1) that an app branches on, and identity may change **during** a session without restarting it (§5.3).
- A medium has an **owner** and a **holder**, and both are rebindable at any time by an authorized act (§6). Neither is baked into the code on the medium.

## 2. Terms

| term | meaning |
|---|---|
| **medium** | the physical or digital carrier: printed code, NFC tag, card, link, message |
| **entry link** | the resolvable URL the medium carries (§3) |
| **entry code** | the opaque identifier inside the entry link |
| **resolver** | the role that turns an entry code into an `EntryTarget` (§4). Any backend may implement it |
| **entry target** | the resolved answer: what to open, where inside it, and with how much identity |
| **grant** | short-lived, narrowly scoped authority issued for **one scan** (§5.2) |
| **principal** | whoever the current session acts as — guest or identified (§5.1) |
| **promotion** | a guest session becoming identified without losing its place (§5.3) |
| **owner** | the party that controls a medium's binding — a tenant or a person (§6.1) |
| **holder** | the party a medium currently speaks for (§6.1) |
| **rebinding** | changing where a medium points, or whom it speaks for, without reissuing it (§6.2) |

### 2.1 Roles — who does what

The standard names roles, not products. One deployment may satisfy several roles with one system; a different deployment may split them across four.

| role | responsibility | does **not** |
|---|---|---|
| **issuer** | mints media, holds the registry, owns custody and the management surface (§6) | render the target · handle the scan |
| **resolver** | answers an entry code with an `EntryTarget` (§4), issues grants (§5.2), **operates the entry host domain and its no-app landing** (§3.2, §3.4) | decide how the target renders |
| **entry host** | claims the link, acquires the code, resolves, reaches the target, binds the credential, renders (§9) | own the binding · decide who may rebind |
| **distribution** | supplies installable targets and their trust chain when a target is a `listing` | participate in `open` / `optional` entries (§4.3) |

The issuer and the resolver are usually the same backend, since the registry is what an entry code dereferences against. The entry host is deliberately separate: a medium's issuer changes bindings all day without ever shipping client software, and the client resolves late (§3.3) precisely so that it needs no advance knowledge of what any code points at.

Conformance is defined for two of these roles — the entry host and the resolver (§11). An issuer conforms by driving a conformant resolver.

## 3. The entry link

### 3.1 Form

An entry link intended for public media MUST be an **HTTPS link** claimed by the host application as a platform app link / universal link:

```
https://<entryHost>/<entryPath>/<entryCode>
```

A custom scheme (`<scheme>://…`) MUST NOT be the sole form on public media. A custom scheme fails silently when the app is absent, which is the majority case for a stranger scanning a physical thing. Custom schemes remain valid for links minted **inside** an app for an installed peer.

### 3.2 The no-app path is part of the contract

Because the link is HTTPS, the platform's normal behaviour is the required behaviour:

- **app installed** → the entry host intercepts and resolves in-app
- **app absent** → the browser opens that same URL, and the resolver's landing surface takes over

The landing surface MUST identify the issuer and MUST offer the entry host from the operating system's app store. Only the target's own UI needs the host; the trust gate does not, so a stranger always sees *who* is asking before deciding whether to install anything.

Two distinct stores exist and MUST NOT be conflated:

| store | who may pass | role |
|---|---|---|
| **operating system app store** | anyone, no platform account | how a first-time scanner obtains the entry host |
| **platform marketplace** | account required | how an identified user obtains a `listing` target (§4.3) |

Installing the entry host from an OS app store is therefore open to a guest. Acquiring a bundle from the platform marketplace is not — which is why guest entries never resolve to `listing` (§4.3), and never need to.

A resolver MAY additionally render the target's substance on the web when the target has a web equivalent. It is not required to: a served UI DSL app renders in the entry host, and the honest no-app surface is the gate plus the install path plus §3.5.

### 3.3 The code is opaque

The entry link MUST NOT encode its destination. It carries an entry code that the resolver dereferences **after** the host has intercepted it. Three consequences, all load-bearing:

- **Rebinding.** The destination — and the party behind it — changes without reissuing the physical medium (§6). This is the entire economic point of a fixed medium with a dynamic target.
- **Tamper.** A destination in the link is a destination an attacker edits.
- **Late binding of kind.** Whether the entry opens a served app, a local node, an installed bundle, a store listing, or an external action is resolved at scan time. The link does not know, and MUST NOT need to know.

### 3.4 Claiming the entry host

App-link interception requires the entry host domain to publish the platform association files naming the host application. The entry host is therefore an **operational commitment**, not merely a URL. Two shapes are conformant:

- **shared entry host** — one domain operated for many owners, one association, each owner allocated code space. The default: no per-owner setup.
- **owner's own domain** — the owner publishes association files for the host application on their domain. Their brand appears in the link.

Three failure modes are common enough to be normative prohibitions, because each degrades **every** scan silently rather than visibly:

- **No redirects into a claimed host.** Interception matches the URL that was opened, so a vanity domain that redirects to the claimed domain is never intercepted. A branded domain must itself be claimed.
- **No in-page navigation to the entry link.** On some platforms a link opened from within a page of the same site, or navigated to by script, does not trigger interception. A landing surface therefore MUST NOT rely on scripted navigation to hand off to the app; it offers the store path (§3.2) and, when the app is present, whatever explicit hand-off the platform honours.
- **One entry host application per domain.** When several applications claim the same domain, platforms differ: some prompt, some pick by install order, some stop verifying altogether. A domain MUST be claimed by exactly one entry host application. Product variants that must coexist on one device SHOULD be separated by **path space on one domain with one claiming application**, not by several applications claiming the same paths.

### 3.5 Deferred entry — installing SHOULD NOT lose the scan

An entry that sends someone to an app store aims to resume **that same entry** on first launch. Landing on the host's home screen instead is the failure the person cannot recover from: the medium is often no longer in front of them.

Platforms differ in what they make possible, so the obligation is graded:

- The host **MUST NOT** silently land on its home surface after an install that originated from an entry.
- Where the platform provides a mechanism to carry the entry code across the install, the host **MUST** use it and resume the entry.
- Where it does not, the host **MUST** present an explicit recovery on first launch — rescan, or re-open the link — naming what was being opened if it knows.
- Resolution happens **after** the install, never before it — custody may have changed in between (§6.4), and a stale target is exactly what §4.3 forbids replaying.
- A deferred entry carries no authority across the gap. Any grant is minted at resolution, after the app is present (§5.2).

### 3.6 In-app browsers

Some scanners open URLs inside their own embedded browser rather than the system browser. Interception generally does not occur there, so the entry silently degrades to the landing surface even on a device where the app is installed.

A landing surface MUST therefore be usable in that context and SHOULD offer opening in the system browser. A host MUST NOT assume interception is the only way its own users arrive.

## 4. Resolution

### 4.1 `EntryTarget`

The resolver's answer. Field names and shapes are normative so any host resolves any owner's medium identically.

| field | type | meaning |
|---|---|---|
| `status` | `ok` · `revoked` · `expired` · `denied` | anything but `ok` renders the trust gate (§9) |
| `issuer` | object | `{ name, verified }` — who stands behind this medium, shown before anything else renders |
| `target.kind` | `server` · `localServer` · `bundle` · `listing` · `external` | what to open |
| `target.ref` | string | server endpoint · local node identity (§4.1.1) · bundle id · listing id · external URI (`tel:` / `mailto:` / https) |
| `target.route` | string? | the page **inside** the target |
| `target.params` | map? | parameters bound at that page (§8.1) |
| `identityPolicy` | `open` · `optional` · `required` | §4.2 |
| `grant` | object? | `{ token, expiresAt, scope[] }` — the scan's own authority (§5.2) |
| `steward` | object? | `{ kind, ref, route }` — the management entry for this medium (§6.5). Present **only** when the resolved principal is authorized to manage it |
| `notice` | object? | `{ kind, message }` — a resolver-supplied disclosure (§4.1.2) |
| `reason` | string? | present when `status ≠ ok`, for the gate's message |

#### 4.1.1 `localServer` — the thing you scanned is the thing you connect to

A medium is frequently attached to the machine that serves its own UI: equipment on a plant floor, an instrument, a controller, a board. The useful entry there is not a cloud address but **the node in front of you**.

For `localServer`, `target.ref` is a discovery candidate in the sense of 17 — `{transport, endpoint, displayName?}` — optionally with a transport hint (17 mDNS · 16 BLE · 11 serial/USB). The entry host resolves it by **discovering and probing the node** (17 Stage 2), not by dereferencing DNS. The node proves itself with the signed manifest trust of 17 §6; the entry code does not vouch for it.

Two properties follow, and both are the point:

- **The medium identifies the machine; the machine serves the UI.** Nothing about the screen has to exist in a cloud, so a site with no internet still gets a real app from a sticker.
- **Reachability is a precondition, not an assumption.** When the node is not discoverable or fails probe, the entry fails with a reason — it MUST NOT silently fall back to a cloud twin, which would render a different machine's state under the same code (§4.3).

**Carrying a grant to a local target.** The Bearer mapping of 08 §4 exists on HTTP transports; BLE and serial have no header slot. A grant therefore MAY have nowhere to ride. Where it cannot be presented, the resolver MUST NOT issue scope that depends on it — this standard does not require a constrained device to validate a token it never receives.

What authorizes the node's own operations is **not this standard's concern**. An entry gets a person to the right node and the right page; whether that node then permits an operation is decided by the node and by whatever access control the deployment runs, on its own terms. Reaching a node is never authorization, and this document defines none for it.

An entry code still dereferences against a resolver, so a fully disconnected site needs a resolver reachable **on its own network**. Nothing in §4 requires the resolver to be on the internet; the roles (§2.1) are about responsibility, not topology.

#### 4.1.2 Vocabularies and localization

- `notice.kind` is a closed set: `custodyChanged` · `targetMoved` · `degraded` · `advisory`. A resolver that needs another kind uses `advisory` and carries the specificity in the message; hosts MUST render an unrecognized kind as `advisory` rather than dropping it.
- `grant.scope` entries are opaque to the entry host — it forwards them and exposes them to the app (§8.1). Their vocabulary is the origin's, since the origin is what enforces them (§7.1).
- Human-readable fields (`issuer.name`, `notice.message`, `reason`) MUST be resolved for the **host's requested locale**, and the host MUST send one. A resolver that has no translation returns its default and says which locale it returned; a host MUST NOT machine-translate an issuer's identity.

### 4.2 `identityPolicy`

| value | meaning | never |
|---|---|---|
| `open` | renders as guest. The app MUST NOT prompt for sign-in | — |
| `optional` | renders as guest; the app MAY offer promotion, and MUST render usefully if it is declined | blocking content behind sign-in |
| `required` | identification precedes rendering | rendering a partial surface first |

`optional` is the load-bearing value. `open` and `required` are its degenerate cases, and a host that implements `optional` correctly gets both for free. A host that implements only `open` and `required` has built the two-product design this standard exists to prevent.

### 4.3 Resolution rules (normative)

- **`open` and `optional` MUST resolve to an account-free surface.** A guest cannot pass the platform marketplace's account wall (§3.2), so a guest entry MUST NOT resolve to `listing` and MUST NOT require acquiring a bundle. In practice: `server`, `localServer`, or `external`. Installing the **entry host itself** from an OS app store is not an account wall and remains available to a guest (§3.2, §3.5).
- **`required` MAY resolve to any target kind**, including install paths.
- **Context survives every interstitial.** `route` and `params` MUST arrive at the rendered page after any install, sign-in, or permission step. An entry that lands on the app's own home screen has failed.
- **A route that no longer exists fails visibly.** Bindings outlive app versions, so a resolved `route` may be absent from the target. The host MUST NOT silently render the target's default as though the request had been honoured: it renders the default **and** surfaces that the requested page was unavailable, or fails with a reason. Silent substitution here is how a rebinding error looks exactly like success.
- **No silent substitution, generally.** An unresolvable, revoked, or denied entry renders the trust gate. It MUST NOT fall back to another target, the app's home, or a generic page — the same rule origin scoping already applies to unknown origins (04).
- **One entry, one origin.** A resolved entry renders under exactly one origin's identity. An entry MUST NOT compose targets from several origins at load time; embedding remains an explicit act of the loaded app (05).
- **Following `steward` is a new entry.** The management surface may live at another origin, so a host follows it by performing a **fresh resolution** under the identified principal — not by composing it into the current render. This is what keeps §one-entry-one-origin true while §6.5 stays useful.
- **Resolution is not cacheable past its validity.** A host MUST re-resolve rather than replay a stored `EntryTarget`, because custody may have changed since (§6.4).

## 5. Identity model

### 5.1 States

| state | principal is | typical source |
|---|---|---|
| `guest` | the **bearer of this scan** | the grant issued for this entry (§5.2) |
| `identified` | a person, tenant, or service | a sign-in the host performed |

There is no third state. An entry that can produce neither a grant nor a sign-in is `denied` and never renders (§4.3).

The subject is exposed **party-agnostically**, so one app serves every level without branching on which kind of party arrived:

```
identity.subject.kind : guest | user | tenant | service
identity.subject.ref  : opaque string, stable for that principal
```

An app that needs "may this principal manage this medium" does not inspect the subject kind — it reads whether the resolver granted a steward entry (§6.5). Custody is a relation between a principal and a medium; it is not a property of an identity state.

### 5.2 The grant — guest is not powerless

A guest is not anonymous in the useless sense. Someone presented this medium's code now. The resolver converts that into a **grant**: a short-lived, single-medium, scope-limited authority.

Normative properties:

- **short-lived** — minutes, not sessions
- **scoped** — enumerated capabilities for this medium only; never a general credential
- **not durable** — the app MUST NOT persist a grant; it is not an asset credential (14) and never enters a vault
- **opaque to the app** — the app reads `entry.grant.scope` to decide what to *offer*; it never reads or forwards the token

Short life is also what makes custody change safe (§6.4): a grant issued to the previous holder's visitor expires on its own rather than needing revocation.

Abuse control (rate, time window, geography, revocation) belongs to the resolver, at issuance. It does not belong to the app, which cannot enforce it.

### 5.3 Promotion

Under `optional`, a guest may become identified mid-session. Promotion MUST:

- **preserve the entry context** — same route, same params, same origin
- **not restart the app** — bindings re-evaluate; the session survives
- **be reversible** — releasing identity returns to `guest` under `optional`, and ends the session under `required`
- **re-authorize the connection** — the host rebinds the origin's credential (§7.1). The app takes no part

If custody changed while the promotion was in flight, the re-authorized principal may see a different surface than the guest did. That is correct, and the resolver SHOULD accompany it with a `custodyChanged` notice rather than letting the screen change unexplained.

## 6. Custody — the medium is an asset

### 6.1 Owner and holder are different parties

| role | is | changes when |
|---|---|---|
| **owner** | the party that controls the binding — `tenant` or `user` | the asset itself changes hands |
| **holder** | the party the medium currently speaks for | the responsible person changes |

Both are **party-agnostic**. A company-issued card is owned by a tenant and held by an employee. A personally issued card is owned and held by the same person — that is not a special mode, it is the case where the two references coincide. A vehicle code is owned by whoever owns the vehicle and held by whoever currently drives it. No separate "enterprise" and "personal" mechanisms exist.

### 6.2 Operations

All are authorized against the **owner**, performed at the resolver, and MUST work without reissuing or physically touching the medium.

| operation | effect | example |
|---|---|---|
| **rebind** | change `target`, `route`, `params` | contact number changes; the service page moves |
| **reassign** | change the **holder** | the salesperson leaves; a different driver takes the vehicle |
| **transfer** | change the **owner** | the asset is sold; a company card converts to personal |
| **suspend / revoke** | stop resolving (`status`) | the card is lost; the medium is retired |

**Transfer MUST be explicit and recorded, and SHOULD require acceptance by the incoming owner.** Without a two-sided act there is no way to distinguish a sale from a seizure.

### 6.3 The medium is the durable thing

Normative: a design in which changing the person requires a new code has failed this standard. Everything already distributed — printed cards in customers' wallets, a sticker on a windshield, a plate on a machine — MUST keep resolving across `rebind`, `reassign`, and `transfer`.

This is what makes the medium an asset rather than a consumable: the accumulated reachability survives the people.

### 6.4 Effect is immediate — and what it cannot reach

After `reassign`, `transfer`, or `revoke`:

- subsequent resolutions MUST reflect the new state
- grants issued before the change MUST NOT be renewed; they lapse on their own short expiry (§5.2)
- an identified session of the previous holder MUST be re-evaluated **at the origin**, not merely hidden in the client (§7.1)
- a host MUST NOT serve a cached `EntryTarget` past its validity (§4.3)

What custody cannot reach is a **copy already on a device**. An installed bundle keeps running, and one that works offline keeps working. Custody governs the entry and what the origin will do — it is not a remote kill switch for distributed code. A deployment whose control depends on withdrawal MUST NOT make an installed `listing` its only enforcement point; the enforcing capability belongs at an origin the entry can re-evaluate.

### 6.5 Management is reachable from the medium itself

When the resolved principal is authorized to manage the medium, the resolver SHOULD include a `steward` entry (§4.1) pointing at the management surface, and the app SHOULD offer it. Scanning your own thing is the shortest path to rebinding it — that is what makes "change the contact" a ten-second act for a driver rather than a support ticket.

The steward surface is an ordinary `required` entry (§10.3), reached by a fresh resolution (§4.3). It is not a separate product, a separate app, or a separate link.

### 6.6 Data boundary — reachability transfers, personal data does not

What a medium carries across a custody change is **the ability to be reached** and its binding. What it does not carry is the previous holder's personal data.

Normative:

- Interaction records accrued through a medium belong to its **owner** when the owner is a tenant. That is what preserves the company's customer touchpoints when an employee leaves.
- Data scoped to a **person** — their private records, their own contacts, anything held under their identity rather than the medium's — MUST NOT be inherited by the incoming holder.
- A resolver MAY surface a `custodyChanged` notice so that a returning scanner is not silently redirected to a stranger. By default a disclosure MUST NOT reveal the previous holder's identifiers; naming them is an owner-controlled choice, not a platform default.

## 7. Authority, trust, and what this standard does not defend

### 7.1 Authority lives at the origin, never in the client branch

The identity state is a **presentation input**. It decides what a screen offers. It decides nothing about what the system permits.

Normative: a host MUST NOT treat client-side identity state as an authorization decision, and a served origin MUST evaluate every request against the credential on the connection. Two screens differing by `identity.state` is a UX affordance; the server refusing a guest's privileged call is the security boundary. An app whose protection is a conditional in its layout is non-conformant.

The same applies to custody: `entry.canSteward` tells a screen whether to show a management affordance. Whether the rebinding is permitted is decided at the resolver, on every operation.

This is why guest entries prefer served UI: the origin already holds the owner's authority and acts on the guest's behalf, so the guest needs no account anywhere. The relay pattern (§10.1) is that arrangement.

### 7.2 What an entry proves

| claim | strength |
|---|---|
| "this code exists and is currently bound" | proven by resolution |
| "the issuer stands behind it" | as strong as issuer verification at the resolver |
| "the scanner possesses the medium" | **not proven** — only that the code was presented. A printed code is copyable, and a link forwards |
| "the scanner is a particular person" | only under `identified` (§5.1) |
| "the local node is the machine it claims to be" | as strong as its manifest trust (17 §6) — the entry code does not vouch for it |

`entry.params` are resolver-supplied context, not credentials. An app MUST treat them as input, never as authority — the origin re-derives whatever it needs to enforce.

### 7.3 Out of scope

This document is an **entry standard**. It gets a person from a medium to the right surface with the right amount of identity, and it says honestly what that proves (§7.2). It is not an access-control standard, and it deliberately defines none:

- **Authenticating the medium itself** — challenge-response media and the key management behind them are a different layer's subject. Where such a layer exists, its verdict is an input the resolver may weigh; this standard neither models it nor depends on it.
- **Authorizing operations** — at a served origin, at a local node, or on a device. Every privileged act is decided where it executes (§7.1).

What this standard does about the residual risk is keep the blast radius small: grants are short and narrow, guest surfaces do not disclose parties (§10.1), and nothing renders under an identity it did not earn.

## 8. Runtime surface

### 8.1 State (read-only bindings)

| path | meaning |
|---|---|
| `identity.state` | `guest` · `identified` |
| `identity.subject.kind` | `guest` · `user` · `tenant` · `service` |
| `identity.subject.ref` | opaque principal reference |
| `identity.canPromote` | whether promotion is available here (policy + host capability) |
| `entry.route` | the route this entry resolved to |
| `entry.params.*` | the resolved parameters — **untrusted input** (§7.2) |
| `entry.issuer.name` · `entry.issuer.verified` | for the app's own trust affordances |
| `entry.grant.scope` | what this scan is allowed to attempt |
| `entry.canSteward` | whether a management entry was granted for this medium (§6.5) |
| `entry.notice` | a resolver-supplied disclosure to render, when present (§4.1.2) |

`identity` and `entry` are new binding roots and MUST NOT collide with the existing reserved roots (`app`, `page`, `route`, `theme`, `i18n`, `event`, `sync`, `runtime`).

`entry.params` is a distinct root from route parameters: route parameters describe *where in the app*, entry parameters describe *what was scanned*. Collapsing them loses the scan context on any internal navigation.

Owner and holder references are **not** exposed to the rendered app. An app that needs them is acting as the management surface and reaches them through the origin, under an identified principal — not through the entry bindings a guest can also see.

### 8.2 Actions

| action | effect |
|---|---|
| `identity.promote` | begin promotion (§5.3). No-op when `canPromote` is false |
| `identity.release` | drop to `guest` (`optional`) or end the session (`required`) |

Credentials never appear in action arguments or results.

### 8.3 Reactivity

An identity change MUST re-evaluate identity- and entry-bound expressions in place. An app MAY declare a lifecycle hook for the transition; it MUST NOT be required to, since a declarative surface should express the difference as bindings rather than as an event handler.

## 9. Host obligations

1. **Claim the link** — register the entry host for app links / universal links (§3.4), one application per domain, and route an intercepted link to resolution rather than to the home surface.
2. **Acquire the code** — from a scanner (camera / NFC), an intercepted link, or a deferred entry recovered on first launch (§3.5). All three produce the same entry code and MUST take the same path afterwards.
3. **Resolve late, never cache past validity** — the kind of target and the party behind it are answers, not assumptions (§3.3, §4.3). Send the locale (§4.1.2).
4. **Reach the target** — dial a `server` endpoint, or discover and probe a `localServer` node over whatever transport reaches it (§4.1.1). A target that cannot be reached fails visibly; it is never swapped for a reachable one.
5. **Bind the credential** — grant or signed-in credential onto the origin connection where the transport has a slot for it (§4.1.1). Opaque to the app.
6. **Honour or disclose the route** — render the resolved route, or render the default **and** say the requested page was unavailable (§4.3).
7. **Render the trust gate** — issuer identity, any `notice`, and, when requested, the scope being asked for, **before** the target's own UI. `status ≠ ok` renders the gate's failure form and stops.
8. **Isolate guest state** — a guest session's storage and permission scopes are ephemeral and origin-scoped, and MUST NOT be keyed to a durable identity. On promotion, the host MUST NOT silently merge guest state into the identified principal's state.

## 10. Generic entry patterns

Business-neutral shapes. An owner composes their case from these rather than inventing a new mechanism; each is a policy plus an authority arrangement, and all four coexist on one platform.

### 10.1 Relay — a stranger acts toward the responsible party, seeing nothing about them

`identityPolicy: open`. The guest triggers a brokered action (notify, request, alert). The origin holds the holder's contact and performs it.

Normative: a guest surface MUST NOT expose the counterparty's personal identifiers. The platform brokers the action; it does not disclose the party. Applies to any "reach the responsible party" case — vehicles, lost property, care tags, unattended equipment, building faults. Because the contact lives at the origin and not on the medium, `reassign` (§6.2) is the whole of "the driver changed".

Relay is also where `bearer` weakness is contained: since a copied code can only *cause a brokered action*, the exposure of a photographed medium is nuisance traffic, which the resolver rate-limits (§5.2) — not disclosure.

### 10.2 Attest — anyone verifies a claim about the thing

`identityPolicy: open`. The guest reads verified facts: authenticity, validity, issuer, current status. Read-only, no correlation to the reader. Certificates, warranties, documents, provenance, compliance tags.

An attest surface states facts about the **current** binding. A claim about the chain of custody is a separate, explicit attestation — never a side effect of the binding log.

### 10.3 Steward — the responsible party manages the medium

`identityPolicy: required`. The identified principal rebinds the target, reassigns the holder, transfers ownership, inspects history, or revokes (§6.2). Same medium as the relay or attest surface above it; different principal, different surface, same code.

### 10.4 Entitle — a holder consumes a right

`identityPolicy: required`, or `optional` where the grant alone suffices for a low-value right. Tickets, seats, sessions, quotas, access windows. Consumption decrements at the origin; the client never adjudicates.

The recurring composition is **10.1/10.2 over 10.3**: a public face for whoever finds the thing, and the responsible party's face behind the same code. That pairing is exactly what `optional` and `required` on one entry express, and exactly what two separate products cannot.

## 11. Conformance

### 11.1 Entry host

- [ ] An HTTPS entry link opens in-app when installed, and lands on the issuer-identifying gate with an install path when not
- [ ] Exactly one entry host application claims a given domain (§3.4)
- [ ] The target's kind is resolved after interception — no destination is read from the scanned link
- [ ] An entry that routed through an app store never lands silently on home: it resumes, or offers explicit recovery (§3.5)
- [ ] `open` and `optional` entries render without a platform account and without acquiring a bundle
- [ ] A `localServer` entry discovers and probes the node, and fails visibly rather than substituting a remote twin
- [ ] `route` and `params` reach the rendered page across install, sign-in, and permission interstitials
- [ ] A resolved route that no longer exists is disclosed, not silently replaced (§4.3)
- [ ] A guest session carries its grant's scope and no durable credential
- [ ] Promotion under `optional` preserves route, params, and origin, and does not restart the app
- [ ] Release returns to guest (`optional`) or ends the session (`required`)
- [ ] Identity- and entry-bound expressions re-evaluate on identity change
- [ ] A revoked, expired, or denied entry renders the trust gate and never substitutes another target
- [ ] Following `steward` performs a fresh resolution rather than composing another origin
- [ ] Guest storage and permission scopes are ephemeral and are not merged on promotion
- [ ] The locale is sent on resolution, and issuer identity is never machine-translated

### 11.2 Resolver

- [ ] An entry code reveals nothing about its destination
- [ ] `rebind`, `reassign`, and `transfer` take effect without reissuing the medium, and previously distributed media keep resolving
- [ ] A resolution never outlives its stated validity, so a custody change is visible on the next scan
- [ ] Grants are short-lived, single-medium, and scope-limited (§5.2)
- [ ] Scope is never issued for a target where the resolver cannot have it evaluated (§4.1.1)
- [ ] A management entry is included when the principal is authorized, and withheld otherwise
- [ ] A custody change is disclosable without revealing the previous holder by default
- [ ] The no-app landing identifies the issuer, offers the install path, and works inside an embedded browser (§3.2, §3.6)
- [ ] Guest entries never resolve to `listing`

## 12. Open items

Deliberately unfixed here; each needs its own decision before implementation.

- **Grant issuance protocol** — how a resolver mints and a host presents the grant on a connection. §5.2 fixes properties, not wire format.
- **Promotion providers** — which sign-in methods a host offers, and whether the resolver may constrain the set per entry.
- **Ownership transfer protocol** — the two-sided offer/accept act of §6.2, and how it is attested afterwards.
- **Path-space allocation across product variants** — the concrete layout that keeps one claiming application per domain (§3.4) while several host variants ship.
- **Deferred entry mechanism** — the per-platform means of carrying the code across an install, and what "explicit recovery" looks like where none exists (§3.5).
- **Association-file distribution** — how an owner claims their own domain for the host application without hand-operating it (§3.4).
- **Multiple entries per medium** — one physical thing addressing several targets by context (time, location, reader role).
- **Resolver federation** — a host resolving entry codes across owners, and how issuer verification is attested across them.
- **Local resolver deployment** — how a site runs a resolver on its own network for `localServer` entries, and how its media are minted and revoked without the internet (§4.1.1).
- **Web surface ownership** — how much of the target renders on the web landing versus prompting for the app.
- **Offline scan** — a medium scanned with neither a network nor a local resolver. Out of scope here; offline authority is a credential concern (14), not an entry concern.
