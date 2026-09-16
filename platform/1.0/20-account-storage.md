# 20 — Account Storage

The storage contract by which AppPlayer, Studio and the web share **the state of one account**. **This document is canon, and its consumers — the AppPlayer Apps server · the three AppPlayer clients · Studio · bundles — follow it.** Host-local improvised arrangements are forbidden.

This is a **different layer** from [`13-datastores.md`](13-datastores.md) (persistent data sources) and [`14-asset-credentials.md`](14-asset-credentials.md) (assets and secrets). Those are about *what is manipulated and how*; this is about *how one account's state travels between devices*.

---

## 0. Identity

**Account storage = a keyed store of opaque documents bound to one account.**

- **Opaque** — the server does not interpret the body. What the server knows is only **whose it is · how much it uses · when it changed**; the meaning belongs to the client.
- **Keyed** — addressed by `(scope, key)`. List · get · conditional write · delete.
- **Account** — the market account is the sole identity axis (§1.1).

### Why opaque

If the server knew the schema, **every settings item a client adds would require a server deployment to follow**, and when the three clients are on different versions the server would be the one deciding which schema is right. The server has no basis for that judgement — the client that produced the state is what knows its meaning.

Knowledge and per-user state already went the same way ([`07-knowledge-access.md`](07-knowledge-access.md)). Same reason.

---

## 1. Surface (fixed contract)

```
list   (scope, prefix?)        → [{key, size, version, updatedAt}]
get    (scope, key)            → {body, contentType, version, updatedAt} | null
put    (scope, key, body, contentType, ifMatch?) → {version, updatedAt}
delete (scope, key, ifMatch?)  → ok
usage  ()                      → {used, limit, byScope}
```

- `body` is bytes. The server does not parse it.
- `contentType` is **a label for the client**. The server stores it and hands it back; it does not branch on it.
- `version` is an opaque string the server issues. A client **only compares** it and never interprets it.
- A body above the inline ceiling travels by **transfer ticket** (§1.2). The surface is still five calls.

### 1.1 The account

The keying axis is **the market account, one**. Ownership, orders and the wallet already stand on that uid; putting state alone under a different identity would require mapping the two axes on every call, and on the day that mapping slips there is no way to say which side is that person.

### 1.2 Large bodies — inline has a ceiling

The shape where `put` / `get` carry the body inline is **for small bodies**. `bundle/<ref>` (a bundle's own bytes, §2.6) is measured in MB and cannot pass inline.

> **A body above the inline ceiling travels by transfer ticket. The surface stays five calls.**

```
put(scope, key, size, contentType, ifMatch)   no body, just the size
   → {transfer: {url, method, expiresAt}}     upload here
   → the version is issued when the upload completes   ← until then no record is visible

get(scope, key)
   → small  {body, ...}
   → large  {transfer: {url, method, expiresAt}, size, contentType, version, updatedAt}
```

**This round trip runs beneath the port.** The sketch above is between server and adapter; the caller makes one `put(scope, key, body, …)` regardless of size and gets a receipt. **Callers must not choose their call by size** — the same conditional would be copied into every caller, and each would get it slightly differently wrong. Taking the ticket, uploading, and waiting for the version are the adapter's job.

**The receipt means "a record now exists", not "the bytes arrived".** For a body that went by ticket, `put` returns **when the version is issued**, not when the upload finished. If no version is issued the write **failed**. Treating upload completion as success hands the caller **a version that does not exist yet**, and the next conditional write built on it breaks — the failure surfaces one step away from its cause.

**Exceeding the ceiling and exceeding the limit are different.** A body above the inline ceiling (`inlineCeiling`) is *not refused* — it simply takes the other road. A size accepted by no road at all is a separate value (`maxBodyBytes`), and exceeding that is a refusal (§4).

**An incomplete upload must not become a record.** That is why the version is issued at completion — a bundle cut off midway that appears in a listing and is downloaded by another device makes the failure surface far from its cause, at install.

**Silent truncation is forbidden.** An inline `put` above the ceiling is **refused, by name**. Storing a trimmed copy reports a loss as a success.

The ceiling's value is decided by the server and **announced by `usage()`** — a client holding it as a constant is the first thing to be wrong on the day the server changes it.

#### 1.2.1 A 409 for a large body carries no body

§3 puts the current body in the conflict response. **For a large body that response would itself be that large, so it carries only the `version`.**

Nothing is lost — **the scopes where large bodies live are not merge targets.** The `ref` in `bundle/<ref>` addresses content, so two devices writing the same ref write the same bytes. There is no reason to ship merge material to a place with no difference to merge.

> Hence `ref` **must identify content** — a bundle id alone will not do. With an id alone, different versions contend for the same slot, the sentence above ("same ref = same bytes") becomes false, and the body-less 409 turns into a response that really does lose something.

> Rule: **`current` is included when the body is of inline size.** Otherwise only `version` is included, and a client that needs the body fetches it with `get`'s ticket.

---

## 2. scope — what lives where

A scope is **a boundary**, not a folder. Permissions, quota and the unit of synchronization all divide here.

| scope | What | Who writes | Quota |
|---|---|---|---|
| `shell/common` | Account preferences that cross products (§2.1) | Any product | Counted (small) |
| `shell/<product>` | That **product's** settings · layout (§2.1) | That product | Counted |
| `app/<appId>` | Per-app data — **independent of product** (§2.1) | That app | Counted |
| `knowledge/<scopeId>` | Knowledge **the user accumulated** (§2.3) | That bundle | Counted |
| `device/<deviceId>` | **Facts about that device** (§2.4) | That device **only** | Counted (small) |
| `bundle/<ref>` | **A bundle's own bytes** (§2.6) | AppPlayer | Counted |
| `shared` | **The shared folder** — user documents across apps | **The user** (§2.5) | Counted |

**Entitlement does not live in this store.** It belongs to Apps, sits outside quota, and survives regardless of subscription.

### 2.1 Different products, different shells

AppPlayer and Studio are **different products**. Their settings and screens do not overlap, and putting them in one bucket means each side carries keys the other does not know every time one adds an item — and a desktop layout lands in Studio.

So shell state is **per product**. The mirroring unit is the product too — devices converge *within the same product*.

```
shell/common              the account's preferences   any product (§2.1.1)
shell/appplayer           AppPlayer's                 mobile · desktop · web share one
shell/appplayer_pro       Pro's
shell/studio              Studio's
```

`<product>` is a product identifier. Different platforms of the same product (mobile / desktop / web) **share one bucket** — the differences between them (window size, connected devices) are device facts rather than product state and go to `device/<deviceId>` (§2.4).

#### 2.1.1 There are exactly two intersections

Sharing everything happens within one product; what crosses products is **explicitly two things**.

| | Why it crosses |
|---|---|
| `shell/common` | Language, theme — **that person's preferences**. A different product is still the same person |
| `app/<appId>` | **An app's data is the app's.** An app used in AppPlayer and opened in Studio must show the same data — which product opened it is not that app's concern |

`shared` (§2.5), `knowledge` (§2.2) and entitlement likewise cross products. The only thing bound to a product is **the shell**.

What goes into `shell/common` is decided **conservatively**. When in doubt it goes to the product side — promoting to common later is easy, while splitting out of common touches every product already using it.

### 2.2 Isolation

A scope reads and writes only its own. `app/<appId>` cannot see `app/<otherId>`. Whatever a bundle asks for is interpreted within that bundle's scope — the server enforces this and does not rely on client good faith.

### 2.3 Knowledge — what was shipped and what accumulated

Knowledge is of two kinds, and **because their origins differ, so do their homes.** Same criterion as bundles (§2.6 — a cache if the device has somewhere else to keep it, data if it does not).

| | Where | Quota | Why |
|---|---|---|---|
| **Knowledge shipped in a bundle** | Inside the bundle | Not counted | It is part of the bundle. Re-fetching the bundle brings it |
| **Knowledge the user accumulated** | `knowledge/<scopeId>` | Counted | That person made it. **We are the only copy** |
| **Derived indexes** | Local | Not counted | Rebuildable from the source. Each device builds its own |

`scopeId` is the one from [`07-knowledge-access.md`](07-knowledge-access.md) — the per-bundle namespace. It is used as the key as-is rather than invented anew. When a bundle calls `bk.fact.write` the bridge applies the scopeId, and the result lands in this scope.

**Not uploading derived indexes matters.** An index is larger than its source and unreadable when it travels between devices on different engine versions. Only sources travel; each device builds its own index.

### 2.4 `device/<deviceId>` — a device's facts are written by that device alone

Every device has facts of its own: which bundles are actually materialized on it, which LAN devices it discovered, its window size, what USB is attached.

These are **facts about that device, not about the account**, so they are not mirroring targets. They must still be kept — another device has to be able to see "this is installed on my desktop", and recovering a device requires knowing what was there.

> **Single writer. `device/<deviceId>` is written only by that device.**

This rule **removes §3's conflicts for this scope**. Two devices never touch the same key, so there is nothing to merge and `ifMatch` is a formality.

Other devices **can read and cannot write.** That is what makes "my device list" work while keeping one device from manipulating another's state.

```
device/<deviceId>/profile      { name, platform, syncEnabled }
device/<deviceId>/installed    bundles materialized on this device
device/<deviceId>/discovered   LAN devices this device found (§5, 17-device-discovery)
```

**Sharing is a separate problem.** Making a device found by device A *usable* from device B is a question of reachability and credentials and is not the subject of this document. What is defined here goes as far as **keeping and viewing**.

### 2.5 `shared` — the one place isolation breaks, and therefore the user mediates

Documents used across apps are needed: something made in one app opened in another, put in and taken out by the user directly.

But **letting apps read this scope freely makes §2.1 meaningless** — isolation would stand while a room anyone may read sits next to it, and one app could take everything another app left there.

So the discipline is one rule.

> **The user mediates. An app holds no standing access.**

| Who | Regarding `shared` |
|---|---|
| **The user** | Full read and write. AppPlayer presents it as a file management screen |
| **An app** | **Only what it is handed.** Limited to what the user picked, and only for that item |

For an app to open something in `shared`, it **has the user pick** (export / import). Only the chosen result is passed to that app, and the app cannot see the listing. The same shape as a mobile OS document picker, for the same reason — **being given and taking are different.**

The mediator is AppPlayer, the client, not the server. The server keeps `shared` as opaquely as any other scope and looks only at **whether it belongs to that account**.

### 2.6 `bundle/<ref>` — a bundle's bytes live in **that device's storage**

Where a bundle's bytes live depends on whether the device has storage of its own.

**If the device has storage, they live there.** Desktop and mobile install to their own disk, and what came from Apps does not pass through account storage — the original is at Apps, so it is a rebuildable cache, and uploading it here would keep a purchase twice. What remains in this scope is **whatever there is nowhere to re-fetch from**: what was authored directly and what was sideloaded.

**If the device has no storage, this scope is that place.** A browser has no disk to install to. On that host, "install on this device" means **install into the storage allocated to the account**, and then what came from Apps lives here too. That is not keeping it twice — this is the first copy.

> Rule: **the bytes go wherever that host can install them.** Where device storage is that place, account storage stands aside; where there is no device storage, account storage is that place.

This split is **a fact about the host, not a user choice.** It is not exposed as a setting — offering "store on device" in a browser offers a choice that cannot be made, and turning on "store in account" on a desktop sends a bill that counts the same bytes twice.

Quota **counts** what lives here (§4). That installing on the web consumes quota is not a side effect; it counts exactly the room an install actually takes on that host. When quota runs short an install is **blocked like any other write, and names what was blocked** — it does not quietly half-install.

---

## 3. Conflict — the server does not adjudicate, it only detects

This is the hard part of synchronization, and the discipline is one rule.

> **The client merges. The server's duty ends at returning versions honestly.**

The server does not know the body and therefore cannot merge. If the server decided by "last writer wins", **a device coming back from being offline would silently erase someone else's edit.**

```
put(scope, key, body, ifMatch: version)
   version matches    → write, return the new version
   mismatch           → 409 + {current: {body, version}}   ← what is on the server right now, included
                          (large bodies: version only — §1.2.1)
   no ifMatch         → unconditional overwrite (first write, or a deliberate force, only)
```

**Shipping the current body with the 409 is the core of the contract.** Without it a client has to `get` again in order to merge, and if it changes again in between the client circles the same spot.

### 3.1 What a client must do

- Hold the `version` it read and present it as `ifMatch` when writing.
- On a 409, **merge and write again.** Never overwrite silently.
- When the kind cannot be merged, **ask the user.** Never pick arbitrarily.

---

## 4. Quota

Per account. `usage()` returns `{used, limit, byScope}`.

### 4.1 `byScope` is by **family**

```
shell · appState · knowledge · devices · bundles · shared
```

Not per individual scope. `app/<appId>` is dynamic — one account may hold hundreds — and aggregating per scope would put **every app's writes into one aggregate document**, which is where write contention appears.

The detail comes from `size` in `list`. What §4.2 requires — that it be visible what uses how much — holds in two steps: **narrow down where by family, then see what by listing.**

`usage()` announces **both** size boundaries together (§1.2).

| | |
|---|---|
| `inlineCeiling` | The maximum that passes inline. Above it, **a transfer ticket, not a refusal** |
| `maxBodyBytes` | The maximum accepted by any road. Above it, **a refusal** |

Both are decided and announced by the server. Held as client constants, the client is the first thing to be wrong on the day the server changes them. And learning a limit **only from a refusal** means the caller learns it after spending the whole upload — being told "not accepted" after uploading 600MB is different from knowing before choosing.

A value of 0 means the server did not say. **Do not read it as "unlimited"** — leave it unknown and let the refusal speak.

### 4.2 When it is full

- **Only writes are blocked.** `get`, `list` and execution are unaffected.
- A refusal **names what was blocked** — which scope and which key, not "save failed".
- `byScope` shows what uses how much. Without it, someone unable to delete has no option but to buy more.

### 4.3 When a subscription ends

**Reading remains.** Writing stops and synchronization stops; the contents of account storage, entitlement and execution are unaffected. An expiring subscription must not reclaim the past.

The grace period and what follows it are Apps policy and not the subject of this document.

---

## 5. Participation in sync — the device declares it

A device registers itself in `device/<deviceId>/profile` (§2.3).

Unless `syncEnabled`, that device **neither uploads nor downloads.** It never turns itself on — a work PC must not pull down a personal desktop, and a device used briefly must not upload without saying so.

**The web is the exception.** A browser has no persistent local storage, so turning it off leaves nothing behind. The switch is meaningless on the web and is a choice only on native (mobile · desktop · Studio).

A device with sync off still **writes its own `device/<deviceId>`.** That is not mirroring but the device recording its own facts; what the switch governs is *exchanging account state*.

---

## 6. What this store is **not**

- **Not a file system.** No directories, no moves, no partial writes. Large bodies travel by transfer ticket (§1.2), but that exchanges **a whole record** — it is not streaming, and there is no reading part of a record or appending to one.
- **Not bulk asset storage.** Assets that are not account state (a tenant's media, datasets) belong to the data sources of [`13-datastores.md`](13-datastores.md). The large bodies that live here are only **those the device has nowhere else to keep** — if device storage can hold it, it goes there; if there is none, here is that place (§2.6).
- **Not a secret store.** Credentials live in the vault of [`14-asset-credentials.md`](14-asset-credentials.md). Putting a secret here makes it travel between devices exactly as far as account storage does.
- **Not the source of truth for entitlement.** Entitlement is at Apps (§2).
- **Not a collaboration space.** `shared` too is **within one account** (§2.5). Sharing between people is a distribution question for Apps and is not defined here.

---

## 7. Consumers

| Who | With what |
|---|---|
| **AppPlayer Apps** (server) | Implements this surface. Opaque keeping · version issuance · quota · enforcing isolation |
| **AppPlayer** (mobile · desktop · web) | Reader and writer of `shell/appplayer` · `app/*` · `device` · `bundle/*`, and **the merging party**. The **mediator** for `shared` (§2.5) |
| **Studio** | The same account. Holds **its own `shell/studio`** and shares `shell/common` · `app/*` · `shared` · knowledge with AppPlayer |
| **Bundles** | Only their own `app/<appId>` · `knowledge/<scopeId>`. The kernel enforces the scope (§2.3) |

Kernel-side wiring is an adapter placed at the `kvStorage: KvStoragePort` slot of [`03-kernel-runtime.md`](03-kernel-runtime.md) — no new port is created.
