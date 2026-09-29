# 20 — Account Storage

The storage contract by which AppPlayer, Studio and the web share **the state of one account**. **This document is canon, and its consumers — the AppPlayer Apps server · the three AppPlayer clients · Studio · bundles — follow it.** Host-local improvised arrangements are forbidden.

This is a **different layer** from [`13-datastores.md`](13-datastores.md) (persistent data sources) and [`14-asset-credentials.md`](14-asset-credentials.md) (assets and secrets). Those are about *what is manipulated and how*; this is about *how one account's state travels between devices*.

---

## 0. Identity

**Account storage = a keyed store of opaque documents bound to one workspace.**

- **Opaque** — the server does not interpret the body. What the server knows is only **whose it is · how much it uses · when it changed**; the meaning belongs to the client.
- **Keyed** — addressed by `(workspace, scope, key)`. List · get · conditional write · delete.
- **Account** — the market account is the sole identity axis (§1.1). The account is **the person**; the **boundary** along which storage, sync and consent divide is **the workspace** (§0.1).

### 0.1 Workspace — the boundary (decided 2026-09-23)

One person uses the same app for personal life, family and work. Those three are **sandboxes over the same resources**, not different products. So the boundary sits on the **workspace**, not the account.

- **A workspace = one copy of this store.** Sync is the workspace (2026-09-23) — **a device that does not use sync has no workspace.** Such a device is a standalone install and its data belongs to that device. The moment sync is bought through a subscription a workspace comes into being (the default workspace), and adding another workspace is **adding another store**, priced separately. Screens follow this definition — while sync is off, the word "workspace" is not shown.
- **Personal = a workspace with one member.** No path exists only for personal use — there is one set of rules.
- A workspace may belong to one account alone or be joined by several accounts (members).
- **`wid` is separate from the account id** — always `w_[A-Za-z0-9]{20}` (2026-09-23). One person may keep **several personal workspaces** and several business workspaces, and the accounts in them differ. Using the account id as the workspace id would pin personal workspaces to one per account, and would close the way to hand over ownership or keep a workspace after a person leaves — **identity (the person) and container (the workspace) are different axes.**
- Zero migration is kept not by overlapping ids but by **inheriting the key name** (§2.7).
- **The workspace is the billing unit.** Plan, registered-device limit and capacity attach to each workspace (§5.1). Whether a home box, a TV and a camper each get a workspace or share one is the user's choice, and the price is paid per that unit. **Subscription, payment and cancellation are also per workspace**, and the workspace's owner pays. A member is judged by **the workspace's plan**, not their own (2026-09-23).
- **What an account holds today is that person's default personal workspace** (no migration, §2.7).
- Members, roles (owner · admin · member) and the workspace registry are held by **the market**. This document defines storage only.
- **Every workspace has a registry document** (market). A personal workspace is created **once, when sync is taken (the sign-up door)** — merely arriving does not create one (so a person who only installed the app gets no storage, 2026-09-24). Before that the account has no workspace, and calls answer without the header under the account id (the old key) (§1). Membership checks are cached (10 minutes, as the market sets) so the personal path does not become a lookup on every call.

### Why opaque

If the server knew the schema, **every settings item a client adds would require a server deployment to follow**, and when the three clients are on different versions the server would be the one deciding which schema is right. The server has no basis for that judgement — the client that produced the state is what knows its meaning.

Knowledge and per-user state already went the same way ([`07-knowledge-access.md`](07-knowledge-access.md)). Same reason.

---

## 1. Surface (fixed contract)

The workspace **is stated in a header** — the shape of the calls does not change.

```
X-AppPlayer-Workspace: <wid>   required for an account that has a workspace (missing = 400 workspace_required) · omitted only by an account with none
X-AppPlayer-Device:    <id>    the device that made the call (§5.1)

list   (scope, prefix?)        → [{key, size, version, updatedAt}]
get    (scope, key)            → {body, contentType, version, updatedAt} | null
put    (scope, key, body, contentType, ifMatch?) → {version, updatedAt}
delete (scope, key, ifMatch?)  → ok
usage  ()                      → {used, limit, byScope}
```

- `body` is bytes. The server does not parse it.
- `contentType` is **a label for the client**. The server stores it and hands it back; it does not branch on it.
- `version` is an opaque string the server issues. A client **only compares** it and never interprets it.
- **Grammar.** Scope ids — `app/<appId>` · `knowledge/<scopeId>` are `[A-Za-z0-9._:-]{1,64}`; `shell/<product>` · `device/<deviceId>` · `bundle/<ref>` are `[A-Za-z0-9._-]{1,64}`. A key is `[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}`. Anything else the server **refuses** (400). A refusal is not unreachability — the client does not queue it for resending but reports it on the spot. How a name chosen by a person is laid out as a key is decided by whoever produces that key (§2.1.2).
- A body above the inline ceiling travels by **transfer ticket** (§1.2). The surface is still five calls.
- **The header is part of the address.** The same `(scope, key)` in a different workspace is a different document. The server checks that **the account is a member** of the header's workspace and refuses otherwise (403). `usage()` is that workspace's too.
- **No header is not "the default workspace"** (2026-09-25). A call that forgot its workspace and a call that meant the default workspace are the same request to the server; answering with the default workspace would let the omission **succeed against another workspace's records.** So:
  - For **an account holding at least one workspace**, workspace calls (storage · quota · subscription state · device registry · connection tickets) without the header are **refused** (400 `workspace_required`). A client that does not know its workspace **waits rather than asks.**
  - **An account with no workspace** comes without the header — a workspace is created when sync is taken, so a person who only installed the app has none to name, and with none to choose there is none to confuse. The server answers under the account id (the old key).
  - At the **sign-up and payment doors**, no header means "my personal workspace" — a personal workspace is created only at that door.
  - **Account calls** (workspace list · create · accept invitation · push token) do not read the header. Workspace versus account is **divided by the path** — there is no separate "whole account" marker.
  - A workspace response carries **which workspace answered (`wid`)**. If it differs from the one asked, the client does not use the answer.

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

The table below exists **once per workspace** (§0.1). The same `(scope, key)` in a personal workspace and in a company workspace are different documents — using the same app on both sides without the data mixing is the core of the sandbox.

| scope | What | Who writes | Quota |
|---|---|---|---|
| `shell/common` | Account preferences that cross products (§2.1) | Any product | Counted (small) |
| `shell/<product>` | That **product's** settings · layout (§2.1) | That product | Counted |
| `app/<appId>` | Per-app data — **independent of product** (§2.1) | That app | Counted |
| `knowledge/<scopeId>` | Knowledge **the user accumulated** (§2.3) | That bundle | Counted |
| `device/<deviceId>` | **Facts about that device** (§2.4) — differs per workspace (what it offers differs) | That device **only** | Counted (small) |
| `bundle/<ref>` | **A bundle's own bytes** (§2.6) | AppPlayer | Counted |
| `shared` | **The shared folder** — user documents across apps | **The user** (§2.5) | Counted |

**Entitlement does not live in this store.** It belongs to Apps, sits outside quota, and survives regardless of subscription.

### 2.1 Different products, different shells

AppPlayer and Studio are **different products**. Their settings and screens do not overlap, and putting them in one bucket means each side carries keys the other does not know every time one adds an item — and a desktop layout lands in Studio.

So shell state is **per product**. The mirroring unit is the product too — devices converge *within the same product*.

```
shell/common              the account's preferences   any product (§2.1.1)
shell/appplayer           AppPlayer's                 mobile · desktop (Pro) · web (AppPlayer Cloud) share one
shell/studio              Studio's
```

`<product>` is a product identifier. Different platforms of the same product (mobile / desktop / web) **share one bucket** — AppPlayer Pro on desktop and mobile and AppPlayer Cloud on the web are the same AppPlayer, so they keep one app list in `shell/appplayer`, and an app installed or removed on one side appears the same on the other.

An app-list entry **leaves a trace** when removed: `removedAt` (when it was removed) · `removedBy` (the `deviceId` of the removing device) · `body` (the content to rebuild it, kept after removal). `removedBy` is **a record** — restoring uses the remaining `body`; this value answers "which device removed this app from every device". An entry a device **could not build** (no content, or building failed) is not something that device removed, even though it is absent locally, so it is not uploaded as a removal — one device's failure to understand would remove the app from every device (measured 2026-09-17: an H723 card was removed 84 seconds after being added, and nobody could say who removed it).

**An entry's body carries no device facts** — if the time the device last opened the app (`lastUsedAt`) were in the body, the value would differ per device, every upload would rewrite the entry, and together with the §3.3 notice two open devices would wake each other endlessly (measured 2026-09-17: 60 times a minute between a Mac and an emulator). Both comparison and upload leave this value out. **The description a device read from the source** (`metadataJson` — the last-received shape of name · description · publisher) is sent in the body but **not counted as a change** — each device re-reads it on its own, so two devices can read one card differently (measured the same day: the Mac's ESP32 card had a different description, and multi_device had no publisher), and counting that as a change starts the same fight. It is sent so that a newly building device has something to show before it connects. Other fields (name · icon, etc.) are compared — they may be values a person edited. In turn, **when the account-side entry changed since the last agreement and differs from this device's card, pulling replaces that card with the account's** (device facts and observed values stay this device's). If it differs while the account did not change, it was edited on this device, so it is uploaded. Without taking in existing cards, two devices' cards never converge and each pull re-uploads its own.

Differences between platforms (window size, connected devices) are device facts rather than product state and go to `device/<deviceId>` (§2.4). X is a working device that installs one app from a dedicated market and performs that app's function in place, so it **does not take part in account sync** (settings, app list and device profile alike) — it connects to the market, but has nothing to carry and nothing to receive.

#### 2.1.1 There are exactly two intersections

Sharing everything happens within one product; what crosses products is **explicitly two things**.

| | Why it crosses |
|---|---|
| `shell/common` | Language, theme — **that person's preferences**. A different product is still the same person |
| `app/<appId>` | **An app's data is the app's.** An app used in AppPlayer and opened in Studio must show the same data — which product opened it is not that app's concern |

`shared` (§2.5), `knowledge` (§2.2) and entitlement likewise cross products. The only thing bound to a product is **the shell**.

What goes into `shell/common` is decided **conservatively**. When in doubt it goes to the product side — promoting to common later is easy, while splitting out of common touches every product already using it.

#### 2.1.2 `appId` is the app a person has

An app's data follows **the app the person obtained**, not a bundle file. So `appId` is decided by how the app was obtained.

| How obtained | `appId` |
|---|---|
| Installed from a market listing | `listing:<listingId>` — the same data when the same listing is reinstalled, even if the listing carries a new bundle id |
| Otherwise (direct install · local registration) | `bundle:<manifest.id>` |

`appId` is **decided by the host; the bundle does not choose it.** If a bundle could name its own `appId`, it could open another's scope by name (§2.2).

The device copy (the kernel key-value store) uses the same `appId` — so what was written on the device need not be renamed when it goes up to the account.

`appId` stays within the §1 grammar (64 characters including `:`). A bundle's `kb` is **one record per key** in this scope. The key is the bundle's key after `kb/`, with UTF-8 bytes outside `A-Z a-z 0-9 . _ / -` written as `:` plus two uppercase hex digits (`:` itself too — `a:b` → `kb/a:3Ab`, `메모 1` → `kb/:EB:A9:94:EB:AA:A8:201`). Every host must use the same layout for one app's data to be readable by another host. The layout is reversible, and a prefix stays a prefix, so `list(prefix)` remains a prefix query. A key whose stored form exceeds 256 characters is refused by the host as `KB_INVALID_KEY`, and a write the account refused is not queued.

### 2.2 Isolation

A scope reads and writes only its own. `app/<appId>` cannot see `app/<otherId>`. Whatever a bundle asks for is interpreted within that bundle's scope — the server enforces this and does not rely on client good faith.

**Workspaces are isolated from each other too.** Outside the workspace the header points to cannot be opened even by name. The server first checks that the account is a member of that workspace and refuses otherwise (403).

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

### 2.7 Migration — what the account holds today is the default personal workspace (2026-09-23)

Nothing is moved. What account storage holds today is, as it stands, **that person's default personal workspace**; it only gets a name. Devices are already running on top of it, so the moment of moving would itself be the point of failure.

- ~~A call arriving **without** a workspace header is interpreted as that account's default workspace (`defaultWid`) — old builds keep running.~~ → Withdrawn 2026-09-25: workspace calls without the header from an account that has a workspace are refused (§1). With no official real users there was no reason to support old builds, and this interpretation turned an omission into a normal response.
- **The key name is inherited.** Existing documents and the registered-device registry are keyed by the account id. The person's first workspace registry records **the inherited key name** (the old account id), and the server reads and writes with it. Workspaces created later use their own `wid` as the key name. **The ids separate without moving documents.**
- Registered devices (§5.1) become the default workspace's registered devices.
- **A new workspace starts empty.** The personal one is not copied into it — copying would break the sandbox.

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
- **Sync is bound to one workspace** (2026-09-25). The store is pinned to the workspace at the moment of joining (it does not read "the current workspace" per call), and when the workspace changes, the old sync **stops, including any pull or push in progress.** Missing either lets the old workspace's agreement be compared against the new workspace's list, **writing the old workspace's apps into the new one as "removed"** (measured: 0.4 seconds after switching), or bringing old-workspace apps into the new list.

### 3.3 Changes on other devices are received without waiting

The store has **no means to wait for changes** (its surface is only `list · get · put · delete · usage`). So if a device pulls only at start, **an open device never learns of another device's changes** — measured 2026-09-17: after the Mac removed an H723 card, two open cloud sessions tried every second to borrow the removed app again, and the emulator kept re-dialing it until a person pressed "Sync now". Sync must be **two-way** for "travels between devices" to hold.

| Trigger | Rule |
|---|---|
| **Notice from the changing device** (main path) | **Only when an upload actually stored something**, `signal {"op":"sync.changed"}` goes over the account channel (22 §4.1) to every device reachable now. A receiving device pulls. Changes pulled and applied are not uploaded, so notices do not bounce between devices |
| **Returning to the screen** | When the app or tab becomes visible again, it pulls — covering notices missed while the pipe was down |
| **Periodic floor** | 5 minutes — for when both of the above missed |

- Pulls run **one at a time.** If triggers overlap and two run, the same merge overwrites the other's agreement.
- **A trigger arriving during a pull causes one more pull after it finishes** (once, however many arrive). A pull that has already read the account cannot contain later changes — answering with it misses the change until the next floor. That is why triggers are not gathered by time (debounce).
- The reason the store gets no change notification is the same as §4.1: the channel is the pipe; the store is registry and bodies.

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

**Where to take part is created by the subscription.** The storage limit of account storage is given by **the subscription grade** (Apps grants the limit, and writes from an account with a limit of 0 are refused — §4). There are three grades.

| Grade | Storage limit | What opens |
|---|---|---|
| None (sign-in only) | 0 | Nothing — settings, apps and bundle `kb` stay inside each device, and the device profile is not written |
| **Minimum** | Small | Sync · **device link** (22) |
| AppPlayer Cloud | Large | All of the above + cloud execution |

**What the minimum grade sells is not storage space.** Its limit is close to nominal; what actually costs money is introducing devices to each other and **sharing addresses and coordinating the connection** (22 §4 · §4.1). Data goes device to device. The limit divides the grades, but what is sold is not confused with it.

The client checks the grade at sign-in and when a device turns sync on, and **does not cache** the judgment (a cancellation takes effect immediately). If it cannot check, it treats the account as **having no grade**.

> The judgment surface (`GET /market/me/cloud`) already answers **`planId` and `bytes` (the capacity that subscription gives)** besides `active`. There is an order of reading — **`active` is the door, and `bytes` is the room inside it.** A cancelled subscription still arrives with its limit as `active:false`, so judging the grade from the limit alone opens the door for someone who cancelled. The judgment is **`active && bytes > 0`**.
>
> **What it answers, though, goes only as far as "minimum grade or above".** `planId` is an opaque string and no plan carries **a grade marker**, so a client cannot tell "Minimum" from "Cloud" (confirmed 2026-09-16 — the market's **client contract** has no `tier`-like field and no plan-list surface at all. Whether the server knows the grade is a separate question; if it does not, that is where it stops). A side that only opens the device link has enough, but **a side that must gate cloud execution does not** — plans must mark their grade. Dividing by the limit number would bake policy into the client, so it is not done.
>
> And the consumers are a separate problem. The readers today fold everything into `active` and drop `bytes` and `planId`.

A device registers itself in `device/<deviceId>/profile` (§2.3).

Unless `syncEnabled`, that device **neither uploads nor downloads.** It never turns itself on — a work PC must not pull down a personal desktop, and a device used briefly must not upload without saying so.

**The web is the exception.** A browser has no persistent local storage, so turning it off leaves nothing behind. The switch is meaningless on the web and is a choice only on native (mobile · desktop · Studio).

A device with sync off still **writes its own `device/<deviceId>`.** That is not mirroring but the device recording its own facts; what the switch governs is *exchanging account state*.

### 5.1 Registered devices — the plan sets the count, the server counts (2026-09-22)

Plans are divided by **how many devices may be registered to one workspace** (2026-09-23: from the account axis to the workspace axis). **The workspace is the billing unit** — plan, limit and capacity all attach per workspace. Whether one person keeps personal · family (home box · TV) · camper · company separate or together is **that person's choice, and the price attaches per that unit.** **The count is a plan setting, not a number baked into code** (`cloud.plans[].limits.devices` · empty = unlimited). A workspace with several members may derive the count from **the number of member accounts** (the market's part). It is easier to understand than a limit per feature, contains other limits (concurrent camera viewing and the like), and the cost (sync · registry reads) scales with the number of devices.

- **What is counted is registered devices** (not concurrent connections). Counting concurrent connections lets several people share one account by splitting the hours.
- **The server holds the registry.** A device sends its id with every call to the account — account storage calls in the `X-AppPlayer-Device` header, pipe ticket requests in the body `deviceId` (22 §4).
- **Registration is a person's choice** (2026-09-24). Using is not registering — a call from an id never seen before is **`device_unregistered` (403)**. A device enters through "Add this device" (`POST /market/me/devices`), and above the limit it gets **`device_limit` (403)** (carrying the limit and the current count — the person needs to know what to release). If a device quietly took a seat while being used, the person would not know how the registry filled up.
- **The limit is a plan setting** — `cloud.plans[].limits.devices`. Empty = unlimited. Code does not know grades and **bakes in no number** (§5).
- **Capacity also belongs to that workspace's plan.** Workspaces do not split an account total — each separately paid unit gets its own (2026-09-23).
- **On a plan with a limit, a call without an id is refused** (`device_required`, 400). Without a limit there is nothing to count, so a call without an id passes — this is not a fallback path but "nothing to guard".
- **Lowering the plan does not cut devices already registered.** Until the count drops under the limit, only **new registrations** are blocked — if devices in use stopped the moment the plan was lowered, that would be an incident, not a plan.
- **Release is done by a person** — from the devices page, through a confirmation (one tap does not remove someone's device). Releasing frees the seat and also removes that device's `device/<deviceId>` (it leaves the registry and its quota returns). A released device **does not sync in that workspace until it is added again** (writes get `device_unregistered`).
- **The devices page is provided by the market package** (`marketDevices`, 2026-09-26) — like the plans page, the host only opens it and it follows the host theme. One page handles the registry (name · last used · this device) · usage/limit · adding this device · release · buying more devices.
- **Buying more devices goes through payment** (2026-09-25). There is no path to a larger allowance without payment.

  | Billing | Base | How to extend | Who |
  |---|---|---|---|
  | Personal (`sync` · `cloud-5gb`) | 5 devices | +5 once (max 10) | The person pays on the plans screen |
  | Business · admin-paid (`business`) | 5 per member | +5 per member (max 10) | The member requests → an admin approves and pays |
  | Business · per-member (`business-member`) | 5 per person | +5 for oneself | The person pays (no admin approval) |

  The allowance is read from the subscription — once the payment is confirmed (a zero-amount payment also passes the payment door; measured at tens of seconds) the registry's `limit` changes. Before confirmation it is "checking".
- **An app's markers ("not yet added" · "all in use") follow the registry** (2026-09-26). A marker learned from a refusal must not clear only on the screen where the device was added — a workspace under a marker stops writing, so there would be no call to clear it. When the device was added on another device or tab, the limit rose, or a seat freed, **it reads the registry and reconciles**: in the registry → clear both · absent with a free seat → "not yet added" · full → unchanged · unreadable → unchanged.
- **An upgrade is the same device. Uninstalling and reinstalling is a new device** (2026-09-26). The device id lives in app storage and survives updates (Android `install -r` · iOS install over with the same signature · store updates). Uninstalling and reinstalling comes in with a new id and the old seat remains — **release the leftover and add anew.** The id is not moved into OS keychain storage to prevent this.
- **The last-used time** is recorded — at most once an hour (every write is a cost).
- **Devices unused for a long time are released automatically** — setting `cloud.deviceRelease.idleDays` (empty = off). Once a day, registered devices whose last use is older than that many days are released (the same as a release a person pressed — the record is removed too). It keeps devices whose id changed when browser storage was cleared, and abandoned devices, from holding seats indefinitely.
- **The web keeps only one device id in the browser** (2026-09-22). Account data is not left behind, but the identifier must be, or the same browser becomes a new device on every reload. A window that cannot use storage gets a new id on every load (the automatic release above collects them).
  → **To be replaced (direction set 2026-09-26): cloud web is one device per account, and wherever one signs in is that device.** This removes what happened when every browser · private window · data wipe became a new device, took a seat and left a ghost (the old 42 `web-` entries). The id is one bound to the account (issued by the server) · signing in elsewhere signs the old session out (the server checks the session on every workspace call; the old one gets `session_replaced`) · seat rules unchanged (the same id is naturally one device). This refusal is a fact about the session, not the workspace — it does not touch workspace markers. Implemented by: market · cloud.
- **The server guards the single writer** (§2.4): a call that sent an id and writes to **another's** `device/<id>` is refused.
- **The id is self-declared by the device** — it is not a signed identity. What this prevents is "several people sharing one account", not a modified app copying an id (if copied, two devices contend for one seat and overwrite each other's records).

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

Kernel-side wiring is twofold. The kernel's own state attaches at the `kvStorage: KvStoragePort` slot of [`03-kernel-runtime.md`](03-kernel-runtime.md). A bundle's `kb` must be written per key with a version and `ifMatch` for conflicts to be visible (§3 · bundle spec 04_Tools §4.8.1), and a kv port without versions cannot carry that. So the host implements `KbAccountRecords` — this surface (§1) narrowed to one app's `app/<appId>` — over account storage, and the kernel's `AccountKbRecordStore` places the device copy and the offline queue on top of it. No other new port is created.
