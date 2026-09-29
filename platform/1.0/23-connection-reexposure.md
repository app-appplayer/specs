# 23 — Connection Re-exposure (the rules for offering what a host holds)

> **Status: draft (2026-09-16).** 22 and 15 refer to these rules and do not redefine them.

The contract by which a host (AppPlayer · Studio) offers a **connection it already holds** (a board · a local server) or an **installed bundle** to another party. There are two doors, depending on whom it is offered to — the **workspace door** (devices of the same workspace) and the **market door** (others). The two doors open to different parties, but **the rules for offering must be one set.** Otherwise what is safe on one side leaks out on the other.

What this document does not define: transport ([`08-extension.md`](08-extension.md) §4) · the hub's frame protocol ([`gateway/1.0`](../../gateway/1.0)) · discovery ([`17-device-discovery.md`](17-device-discovery.md)) · introduction and linking of account peers ([`22-appplayer-peer-link.md`](22-appplayer-peer-link.md)).

---

## 0. Identity

| It is | It is not |
|---|---|
| Rules for offering a held connection **per app** | Opening up a device |
| Offering **only the declared surface** | Whatever comes along because it is connected |
| An action with permission, revocation and records | A setting that ends with one switch |
| One set shared by the account door and the market door | Different rules per door |

## 1. The unit is the app

Offering · permission · session · lifetime · revocation · records all attach to **one app**. Opening per device opens everything on the device, and at that moment "only this app, to others" becomes impossible. The boundary must be at the app for doors to be fitted separately.

One app = one logical session. Even when several share a channel (22 §5), sessions do not mix.

## 2. Two kinds — bridge and serving

| | **Bridge** | **Serving** |
|---|---|---|
| Subject | A board · a local server — a connection the host holds | A bundle installed on this host |
| Actual execution | **The device over there** does it. The host carries | Runs **on this host** (tools · JS · knowledge) |
| Surface | What the held connection offers, as declared | What the serving contract ([`serving/1.0`](../../serving/1.0)) declares |
| Risk | Who drives my device | What runs on my machine |

The two are not called by the same name. Their risks differ, so their permitted surfaces are defined differently.

### 2.1 Who uses it — **an AppPlayer coming from outside is the reason for this document**

| User | How it reaches |
|---|---|
| **This device itself** | Opens it from the launcher. Unrelated to offering — no rules needed |
| **Another AppPlayer of the account (from outside)** | A phone on a business trip uses the board the Mac at home holds. **This is the path this document defines** (22) |
| **A market counterpart** | Listings · access lists · billing (15) |

**It must not stop at connections from inside.** This device opening its own device always worked and carries no risk. What needs rules is **the path by which an AppPlayer outside uses that device** — that is where a door appears, and §3–§6 of this document are all for that path.

The host's own functions (sync · account state · market transactions) are not offered here. Those go through the market.

## 3. The declared surface (MUST)

**Offering is declaring "what".** What is not declared is answered as if it did not exist.

- Bridge = among the held connection's tools · resources · resource templates · prompts, **those declared**.
- Serving = that bundle's UI (`ui://…`) · document (`bundle://manifest.json`) · **declared tools**.

What the outside party sees is **only what is declared**. Even if the held connection offers more, what is not declared does not exist.

**Never included.**

| What | Why |
|---|---|
| Host capability tools — `secret.*` · credential transfer · `io.*` · files · payment | Registered for the device's **owner**, not part of an app's surface |
| The kernel knowledge surface — `bk.*` · `kb://` | Accumulated on that device, not something this app offers |
| Other apps' resources | Crosses the app boundary |
| Host debug surface | An operating tool, not a product surface |

If there is a path by which a bundle's tool calls enter the host's in-process dispatcher, the re-exposed surface is cut **before it**. "The bundle's tools were offered" must never become "every tool the host registered was offered".

## 4. Doors — who comes in

| Rule | Content |
|---|---|
| Identity only as the channel authenticated it | A claim inside a frame is not identity (the same rule as gateway 1.0 §6). **An address is not identity either** |
| Workspace door | Devices of the same **workspace** (2026-09-23 · 22 §0.1). **A personal workspace allows by default** — using my devices from my hosts is the premise (22 preamble). **A workspace with several members is off by default** — sharing my device with others is something I turn on. Either way, the list that **only excludes apps to be withheld** is kept per workspace. It appears in no public listing |
| Market door | Explicit listing · access list · billing (15). Turning offering on does not open it |
| One door does not open another | An app lent to the workspace does not appear on the market, and an app sold on the market does not open my device list. Workspaces do not open each other either |
| Consent for elevated permissions | Consent given for use on this device **is not permission for use from another device.** It is asked again |
| Refusal | **Absent and not permitted get the same answer.** Answering differently lets probing alone reveal what exists |
| Secrets | Server tokens · device credentials never leave the host ([`14-asset-credentials.md`](14-asset-credentials.md)) |

## 5. A device app's identity and candidates

**An app card is bound to the device, not to an address.** An address means something only on that device, so it loses its meaning the moment the card crosses to the account. Filling the gap with "only through the device that first registered it" makes that device a permanent middleman, and even a device right next to the board cannot use it when that device is off.

### 5.1 Three properties of a candidate

| Candidate | Where it is valid | Examples |
|---|---|---|
| **Physical binding** | Only on that host | Serial · USB · BLE range |
| **Network reach** | Anyone on the same network | LAN address · found by mDNS · fixed address |
| **Borrowing** | Anywhere — while the holding host is on | Account peer (22) · hub exposure (15) |

### 5.2 Opening order

An already-open connection → what I physically reach → the same network → borrowing. **Cheapest first**, and since probing a device is not free (17), once an earlier one works the later ones are not tried.

- A device in the same space is opened **directly**. The holding host is not put on the path.
- From outside, it is borrowed.
- **A device that accepts only a single peer** may refuse a direct connection. Then it goes through the holding host — the failure itself is the correct route choice.

### 5.3 What goes to the account and what does not

| Account | Only on that device |
|---|---|
| Device identity · name · icon · **what kinds of route existed** | **Host-local handles** such as port names · IPs · radio device ids |

The place for per-device discovery facts already exists — `device/<deviceId>/discovered` (20 §2.4, written only by that device and read by others). The opening device builds candidates by combining **its own discovery + the account's card + what the lender offers**.

### 5.4 The device claims its identity

Today identity differs by transport — the host name for mDNS, the port name for serial, the device id for radio. So **two hosts seeing the same board by two routes hold different cards.**

- When a device carries a **signed identity** (17 §6 `trust.signerCert` — public key · serial · Ed25519), that key is **the deduplication key**. Whatever route it is seen by, it is one card.
- A device that does not carry one falls back to per-transport identity. That limit is not hidden — the same device may appear as two cards.

### 5.5 When it cannot be reached, say why

A device card is **visible on every host of the account** (the premise in 22's preamble). So "cannot open" is not an answer — when something is visible but will not open, **why it cannot be reached** decides what the person does.

| Situation | What to say |
|---|---|
| Physically bound device · the holding host is off | "Plugged into the Mac at home over USB · the Mac is off" |
| Network-reach device · on a different network | "Opens when on the same network" |
| The holding host stopped offering | "That device no longer offers this app" |

## 6. While alive — lifetime · revocation · visibility

| Rule | Content |
|---|---|
| Revocation is immediate | Turning offering off or removing the device ends that app's sessions on the spot |
| Who is using it is visible | **A host with re-exposure on must show who is using what right now and be able to cut it off.** Without that, it cannot be turned on |
| Calls are recorded | The device over there sees everything as **what this host did**. Without recording which party called what, the owner has no way to know later |
| One device, one connection | Re-exposure **does not open a second connection** to the device. It uses the connection the host holds |
| Subscriptions are reference counted | Even when the borrowing side lets go, the subscription of this host's screen remains. The device only knows "subscribed" |
| Request ids do not mix | When several parties use one connection, the host swaps ids in and back out |

### 6.1 Many share one connection — device load · liveness · reconnection (approved 2026-09-21)

"One device, one connection" is the rule that a device is not opened twice. What was not defined is what reaches the device when **this host's screen and several borrowing devices** ride that one connection, who judges whether the connection is alive and how, and what happens to sessions when the connection stands again. What came out of that gap (measured 2026-09-21, ESP32 · 7 devices):

| Observation | Cause |
|---|---|
| ~10 requests to the device each time a device borrowed (4 kinds of list · several resource reads · the start tool) | The lender forwarded received requests straight to the device · a surface snapshot was taken fresh on every borrow |
| A slow device failed to answer ping within 4 s → the host dropped the connection | Liveness judged by ping alone — the device had been sending notifications every second all along |
| Every drop ended all lent sessions → everyone borrowed again at once → ~50 requests → the next drop | Sessions shared the connection's lifetime · resumption came all at once |

**Principle: the load a device receives is independent of the number of users.** Whether one or seven use it, the device must receive the same requests. The device's throughput is fixed (TINY is single-task · slow radio), while users multiply.

**Common to both doors.** Lending through the account door (22) and hub relaying through the market door (15) both put several users on one device connection. This section is a rule for the host holding the connection and does not distinguish doors — if each door did it differently, one door would knock the device over.

#### 6.1.1 What reaches the device — a shared copy per connection

The shared copy lives **at the host's connection slot**. Placed only in lending, requests opened by this host's own screen would still go straight to the device (that was the stall when opening on the Mac). This host's screen and borrowing devices see the same copy.

| Kind | What | To the device |
|---|---|---|
| Connection info | `initialize` result (server info · capabilities) | Once per connection. Afterwards a copy to every user |
| Lists | `tools/list` · `resources/list` · `resources/templates/list` · `prompts/list` | Once per connection. Again only when a `*/list_changed` notification arrives |
| Document resources | `ui://…` · `bundle://manifest.json` — **documents the device offers** | Once per connection. Documents change only when firmware or the bundle changes, and then the connection stands anew. Again when `resources/updated` arrives for that uri |
| Subscribed state resources | `sensor://…` etc., subscribed by someone | Subscriptions are reference counted (§6). A read by a newly joining party is answered with **the last value received** (extended notifications carry the body) |
| Unsubscribed state resources | A one-time read (`Read once`) | To the device. If a read of the same uri is already in flight, **its response is shared** (N reads do not go out at the same moment) |
| Calls | `tools/call` | Always to the device. Order and concurrency per §6.1.4 |

**Only connections whose surface is fixed are kept per connection.** A serving device (a board · local serving — its surface is set by the manifest, and when it changes the connection stands anew) reads lists and documents once per connection. A general MCP server can change lists without notice (MCP permits it when `listChanged` is not declared), so **only lists that declare `listChanged`** are kept, and for the rest requests at the same moment are only merged. Whether the surface is fixed is told by the host holding the connection — the shared layer does not guess.

**Which resources are documents is declared by the device.** When the serving manifest (`bundle://manifest.json`, mcp_serving) names document resources, that is followed. Only without a declaration is **the scheme used as the default** — `ui://` · `bundle://` are offered documents (mcp_ui_dsl · mcp_serving). Anything else is treated as state: MCP's default is that it can change without notice, so an unsubscribed state value is not answered from a copy. The host does not add documents by guessing.

#### 6.1.2 Is it alive — a shared connection belongs to no one person

| Rule | Content |
|---|---|
| Incoming traffic is the evidence | If anything comes in from the device (a response · a notification), it is alive. A check request (ping) is sent **only when quiet** |
| Late is not dead | A late ping does not drop the connection if something else came in meanwhile. It is dropped when **nothing comes in for the set time** and ping gets no answer either. Set time = **30 s without receiving** — longer than lwIP's retransmission ladder (1.5 + 3 + 6 + 12 s) on lossy 2.4 GHz. A retransmitting device is alive and TCP eventually delivers. A new connection rides the same radio, so dropping gains nothing (2026-09-21: a "4 missed pings" judgment dropped such a board at a deadline shrunk by RTT to 4.2 s) |
| The deadline belongs to that device | A single ping's wait follows the response time measured on that connection, not a fixed 4 s (for recording lateness). **Dropping is judged only by time without receiving, not by the wait deadline** |
| A drop is everyone's business | Dropping a shared connection stops every user on it — the judgment looks at the whole connection, not one user's single request |

#### 6.1.3 When the connection stands again — sessions outlive the connection

- When a shared connection drops and stands again, borrowed sessions are **not ended but reattached to the new connection**: subscriptions are re-established and the copy is refilled once. Users see only a brief gap ("reconnecting").
- Sessions end only when the device **is gone** (offering revoked · card removed · unreachable beyond the set time).
- **Requests in progress at the moment of the drop are not carried over to the new connection.** They end with an error on the spot and the user calls again — a tool call moves something on the device, and carrying it over could execute it twice. List and document reads are answered once the copy is refilled.
- The current approach — ending everyone on a drop and having everyone borrow again — is replaced by this rule; that approach creates the next drop through simultaneous resumption.

#### 6.1.4 Protecting the device — request flow

| Rule | Content |
|---|---|
| Concurrent request cap per device | **Declared by the device** (serving manifest). Without a declaration, the grade default (TINY = 1 · otherwise unlimited). Requests over it wait **at the host** — not in the device's receive buffer |
| Joining uses the copy | When a new user attaches, lists and documents are answered from the copy, so no device request arises. Seven attaching at once is invisible to the device |
| Notifications are received once and fanned out | Device → host once, host → each user. A lagging user receives only the latest value (22 §5.3) |
| Releasing a connection is counted too | Each connection counts its **holders** (apps on this host's screen · each borrowed session). Closing a screen or ending a session **releases only its own share**, and the connection closes only when the last holder releases. Brief openings (metadata reads · open timeout) do not close it while someone holds it. Only clearing the device (card removal · offering revoked) closes it immediately (§6) |

#### 6.1.5 Visible — what to measure per connection

Per connection, measure **requests sent to the device · waiting · response time · number of users · drops**. Warnings **carry the cause** (a warning that records only where it failed is as good as none).

#### 6.1.6 How it is verified

1. While users grow from 1 → 7, **the number of requests sent to the device does not grow** (measured by per-connection counters).
2. While one user joins and leaves, **other users' stream gaps are ≤ 1.5 s**.
3. Even when the device answers 10 s late once, the connection and all sessions are kept.
4. Forcing the connection down and standing it up again continues the sessions without ending them.
5. Opening and closing on this host's screen does not stop the borrowing side.
6. A tool call in progress at the moment of a drop ends with an error and does not execute twice on the device.
7. Closing the app on this host or re-reading metadata does not close the connection while borrowed sessions exist.

**What this section does not fix.** A healthy device on the same network answers a request within tens of ms. If the device itself is slow (2026-09-21 ESP32: request ACK 0.5–1.5 s), that is a device-side defect and is investigated separately. This section keeps one slow device from stopping everyone on it.

Placement: shared copy · liveness · flow = the core connection layer. The connection manager **wraps the MCP client a connector (different per host) produces in the shared layer** as soon as it receives it (`appplayer_core` `SharedClient`) — so on any host, whatever the user (this host's screen · runtime · lending · hub relay), the same rules apply. **The standard MCP client library is not changed** — the rules of this section (document schemes · fixed surface · bodies in extended notifications) are this platform's conventions, not the MCP standard. Device firmware is not changed either.

## 7. What carries it

- **Account door = plain MCP.** A channel per app, so there is nothing to merge. Frames are not translated (22 §5).
- **Market door = gateway frames** (15 §8). It brokers several providers to several consumers and applies per-consumer policy, so that layer earns its keep.
- The channel itself is one of the transports in 08 §4. This document does not define the channel.

## 8. Relation to other documents

- [`08-extension.md`](08-extension.md) §4 — the two transport tiers and the injection slot.
- [`15-hub-channel.md`](15-hub-channel.md) — the market door. The hub's nodes · sessions · relay · gateway frames.
- [`17-device-discovery.md`](17-device-discovery.md) — how devices are found, and signed identity (the basis of §5.4).
- [`20-account-storage.md`](20-account-storage.md) — the boundary between what lives in the account and what lives on the device (§5.3).
- [`22-appplayer-peer-link.md`](22-appplayer-peer-link.md) — introduction and linking on the account door.
- [`serving/1.0`](../../serving/1.0) — the serving surface.
- [`14-asset-credentials.md`](14-asset-credentials.md) — secrets never leave the host.
