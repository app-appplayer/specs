# 21 — Payment Action

The contract for how an app **requests a payment**. The requester is whatever serves the DSL (a bundle app, a server app), the opener is the host, and what actually takes the money is the **payment surface**. This document fixes the boundary between the three.

A different layer from [`14-asset-credentials.md`](14-asset-credentials.md) (the credential vault) and [`19-scan-entry-identity.md`](19-scan-entry-identity.md) (entry by medium). 14 is *how a host keeps its own secrets*, 19 is *how a person enters an app*, and this is *how an app requests a payment and receives the result*.

Implementation counterpart: MCP UI DSL §4.24 (the `payment` action) · §7.3.5 (external handoff and return) · §18.11 (the Payment Profile).

---

## 0. Identity

**A payment action is a request in which the app declares only "to whom · for what" and sends everything else outside the host.**

The app declares three things, two of them conditional.

| | |
|---|---|
| `itemId` | An item identifier the payment surface knows. **A preselection, not a definition** (required) |
| `seller` | An opaque identifier of the receiving side. Not a merchant number, not an account. **Written only by an app that knows it** — a document served by a device does not write it, and the host resolves it from the device identity |
| `amount` | **Only for items where the customer sets the amount.** A tip, a donation, a counter total. Major units, greater than zero |

What an app **cannot** declare: currency · a provider name · a merchant identifier · keys · the payment address · a change of the receiving party.

If an app could name the provider, that provider's key would have to be within reach of a rendered page. Not having the field is the design.

**Items and prices belong to the payment surface.** If an `itemId` does not exist there, the payment page fails visibly — misrouting is not covered up. The same for amounts: **an amount carried on a fixed-price item is ignored** (which is correct — the seller does not let the caller name the price), and a value outside the range of a customer-entry item is **refused, not clamped**. Quietly correcting it and charging means the customer pays an amount they did not agree to.

### 0.1 The host's founding constraint — it holds neither credentials nor cards

This entire axis comes out of one sentence.

> **The host holds no payment credentials, and no card passes through the host.**

The "does not hold" column of the role table (§1) is a consequence of that sentence, not a separate rule. Which is why the following, however convenient they look, **are not part of this axis**:

| What must not be done | What it would come to hold |
|---|---|
| **Wrapping** the payment surface's sheet in host UI | A layer of host code on the path a card travels |
| **Building a backend** to open sessions | The credential to open a session |
| Putting provider keys in **host settings** | The provider credential |
| Putting the payment page in a **`webView` frame** | A card passing across a host screen |
| Letting **the host decide** payee, amount or provider for convenience | The coordinates of money |

**The test**: when a new feature arrives, ask *"would doing this give the host a credential or a card?"* If so, that feature is not part of this axis.

---

## 1. Three roles

| Role | Holds | Does not hold |
|---|---|---|
| **App** (the DSL-serving side) | Declaring `seller` · `itemId`, binding `onSuccess` / `onError` | Amount · address · keys |
| **Host** (the five AppPlayer tiers) | Assembling the payment address · opening the surface · issuing and checking the return address · the result envelope | Merchant and provider credentials |
| **Payment surface** | Merchant · provider · keys · session · settlement | The app's state, the app's credentials |

**A remote device is just one case of a server app.** There is no separate "device payment" specification. The channel (NFC · BLE · WiFi · QR) is an axis unrelated to payment and is already resolved by 16 · 17 · 18 · 19.

---

## 2. Surface

### 2.1 What the app writes (DSL)

```json
{
  "type": "payment",
  "seller": "{{app.seller}}",
  "itemId": "wash-premium",
  "onSuccess": { "type": "navigation", "action": "push", "route": "/receipt" },
  "onError":   { "type": "state", "action": "set", "binding": "msg", "value": "{{event.message}}" }
}
```

### 2.2 What the host fills in

- **The payment address** — the host assembles it toward the payment surface configured for it. Neither address nor origin nor provider comes from the app.
- **The receiving party** — if the app wrote a `seller`, that is used; if not, it is resolved from **the verified identity of the serving origin** (the device identity gate, §6.1). If there is no means of resolution, refuse — **never fall back to a default payee.** That is not a fallback; it is a transfer to the wrong person.
- **Surface selection** — which of the three below to stand on is the host's knowledge (§2.3).
- **The provider selection screen** — when a seller accepts several, the host draws it. The app neither knows nor picks the candidates. A customer who abandons the choice produces a `cancel`.
- **The return address** — issued by the host (§3).

### 2.3 Three surfaces — where it stands is the host's knowledge

| Surface | What | Card entry |
|---|---|---|
| Hosted page | Opens the payment surface's web address in an external browser or custom tab | The provider's domain |
| **Payment home bundle** | When the payment surface **also stands as a bundle**, a host that knows how to run bundles renders it directly. No runner round trip | The provider's domain |
| Native sheet | The provider's native payment sheet | The provider's SDK |

**AppPlayer is a host that knows how to run bundles.** So where a payment surface offers a bundle, that is the first-class path, and the payment port attaches to that bundle and calls its payment tools — only the shell differs while tools and credentials stay in the same place, so screen and money path do not diverge.

**This is not putting a provider page in a `webView`.** On any surface, card entry ends on the provider's domain, and the host does not frame a provider page inside the app to imitate "payment inside the app".

The host **does not add an adapter layer.** It takes the payment surface's SDK as-is, and the tool that opens things (`url_launcher` and the like) is the host's. An SDK decides what to open and how, and does not open it — an SDK that pulls the opening tool as a dependency would let that package dictate the host build and would drag a plugin into headless and web hosts.

### 2.4 A host declares which surfaces it permits

§2.3 says what a host **can render**. Separately, a host may declare what it **permits**. The two differ — a host that knows how to open a browser may hold "we do not send customers outside" as product policy.

- A host declares a **permitted set** among the surfaces it can render. Declaring nothing means all of them.
- If **not one** permitted surface can be stood up, refuse with a reason (§4). Never quietly substitute a surface that was not permitted.
- This declaration is **the host's**, not the app's and not the payment surface's. An app demanding a surface would be the app deciding where payment stands, which is what §0 blocks.

**Why it is needed**: without the declaration, "we only take payment inside the app" is a habit rather than code. A habit quietly falls back to the browser when one surface fails to stand, and the host ships without knowing what it put on screen.

---

## 3. Return — a link is not evidence

The payment surface returns the result **as a link**. A link is a message the device sends — another app, a page in a browser, a scanned code can all send the same link.

**A return therefore means only "we came back". It is not a confirmed payment.**

- The host **does not mark a return as a verified payment.**
- An app uses `onSuccess` as **a display transition** (a receipt screen, polling, re-rendering server state).
- **Anything that opens value — starting a machine, unlocking content, releasing goods — happens only after the server side of that action re-confirms the payment against the payment surface.** `onSuccess: {"type":"tool"}` is the normal shape, and the check lives in that tool's implementation. A tool that acts because it was called has no way to tell a caller who paid from one who did not.

Without this rule written down, the most natural document — `onSuccess` turning on the machine — becomes **a document that runs the machine without payment**.

### 3.1 Return address contract

| | |
|---|---|
| Scheme | **A custom scheme is required.** `http(s)` is intercepted by another handler on the same domain |
| Token | The host issues one **fresh and unguessable per request**, carries it on the return address, and **discards** a returning token that does not match a pending request |
| A discarded return | Is not delivered to the app. An unauthorized forgery is cut before the app learns of it |
| Lifetime of the pending list | **As long as the host lives.** If the app dies mid-payment there is nothing to match against and the return is discarded — that document is gone too, so there is no recipient for the result. **The money may have moved**: the customer's recourse here is the order record, not the return link (§3.2) |

Without a token, any inbound link confirms some payment. Even with one, **nothing is confirmed** — it only stops blind forgery, and confirmation comes only from the server side.

### 3.2 The source of truth for a payment is the order record

A return is the signal that turns the screen; what was sold is held by the **order document** on the payment surface's side. Capture is asynchronous, so appearing still pending right after a return is normal, and refunds and disputes are settled from that record too.

So the last row of §3.1 (the app dying and the return being lost) is **a change of route, not a loss** — the order record holds the result the host did not receive. Where the host has no standing to read it (as with a guest payment), the seller reads it and the seller answers the customer.

---

## 4. Failure

| Situation | Result |
|---|---|
| The person completed the payment flow | `onSuccess`, `data.status = "success"` |
| The person closed it | `onError`, `PAYMENT_CANCELLED` |
| No return arrived, or it could not be read | `onError`, `PAYMENT_UNKNOWN` |
| No payment port injected · the surface could not be opened | `onError`, `PAYMENT_UNAVAILABLE` |

- **Never route a cancel or an unknown to `onSuccess`.** A cancel is a normal outcome rather than a failure to hide, and unknown is the place where guessing is most expensive.
- **No silent no-op.** A press that produces nothing is indistinguishable from a broken document.
- **When a native sheet was demanded and cannot be opened, do not quietly switch to a browser.** Refuse with a reason — substituting means the host ships without knowing what it put on screen.

---

## 5. Tier requirements

A host that declares payment support provides the following.

1. A payment port — opening the surface + receiving the return. **Either one alone is not support** (opening only means the app never learns the result).
2. **Custom scheme registration** — iOS `CFBundleURLSchemes` · Android intent-filter. Required per tier, and must not collide with existing schemes (OAuth and the like).
3. **Discrimination between a payment return and an entry link** in the deep-link router (§6).
4. No payment is fired from a document at the `untrusted` trust grade. Since the app names the seller, an untrusted document naming a seller is a request to collect money on behalf of a stranger.
5. When the payment surface **also stands as a bundle, attach to that bundle** (§2.3). A host that cannot attach stands on the hosted page, but **never quietly substitutes another surface for one that cannot stand** (§4).
6. **Payee resolution** — a host that can receive documents without a `seller` **resolves the payee from the address that served it** (§6.0). For a document served by a device, the device identity gate (§6.1) takes that place. Without a means of resolution, such a document is refused.
7. **A surface permission declaration** — the host may declare which of the surfaces it can render are permitted, and if not one permitted surface can be stood up it **refuses with a reason** (§2.4). It never quietly substitutes a surface that was not permitted.

---

## 6. Payee · identity terminal · relationship to 19 (entry by medium)

### 6.0 A document without `seller` — the serving address decides the payee

Documents that omit `seller` are not only those served by a device. The same holds when **a payment surface serves documents at per-payee addresses** — one screen is served on behalf of several payees, so a document that names the payee makes **money follow the document rather than the address** the moment that screen is served at another payee's address.

- The host resolves the payee from **the address it connected to**. Without a means of resolution, it refuses (§4).
- **Never use a value the document carried as the basis for the payee.** Not even a value the platform injects at serve time — the moment the host reads it, the discipline shifts from "trust the address" to "trust the document", and a document the platform did not serve can carry anything.
- A document may declare **who it was written for**. That is not a payment target but **a cross-check target**. If the declaration and the address disagree, the host **pays neither and refuses**. With no declaration, behaviour is unchanged.
- Where **the host assembles the address**, it takes the payee identifier and the screen identifier **from one source in one fetch**. Fetching them separately and combining them lets a mix of two different payees hold quietly.

The device identity gate (§6.1) is one case of this rule — there, **a verified device identity** takes the place of "the address it connected to".

### 6.1 An identity terminal — the device proves the receiving party

A document served by an unattended device does not write `seller`. Who receives follows from **which device this is**, and the host establishes that by **identity verification**, not by reading a field.

| # | Step | Party |
|---|---|---|
| 1 | Device connection (NFC · BLE · QR) → the device serves its identity and what it sells | Device |
| 2 | Request a challenge | Host → identity gate |
| 3 | The device signs the challenge (ECDSA P-256 / SHA-256) | Device |
| 4 | Submit the identity proof → status + a payee reference | Host → identity gate |
| 5 | Open the payment session | Host → payment surface |
| 6 | Approval | Customer |
| 7 | **Return the approval token to the device** → the device acts | Host → device |

- **The device holds no payment credential.** One identity key, and money does not pass through the device.
- **A challenge is single-use and expires quickly.** It is deleted on consumption, so a captured proof cannot be replayed.
- The identity gate answers three ways — **allow** (payee resolved) · **lookup only** (identity is valid but there is no payment context: unverified, unmapped) · **deny** (revoked · suspended · signature mismatch · invalid challenge). **Lookup-only is not a fault but an incomplete setup** — and must read that way.
- If step 7 fails, the customer paid and received nothing. **The host reports a failure at this step distinctly from a payment failure** — the party who must undo it is different.

#### A device does not declare a price

What the identity gate returns is **the receiving party**, and step 5 is **opening that payee's payment surface**. Item and price live there, in the item document the payee registered, and the device only points with an `itemId`.

**There is no path that trusts an amount declared by a device.** A tampered device could name any amount, and "the server sets the amount" is a payment surface's first discipline. A device where the customer enters an amount (a counter, a top-up) expresses that item as a **customer-entry item** — §0's `amount` is that place.

#### There are two doors because the consumers differ

| Consumer | Path | Credential |
|---|---|---|
| **A consumer with a backend** | Resolve identity → its own backend opens the payment session directly | Needs the session-opening credential. **The caller declares amount and currency** |
| **A consumer without a backend (AppPlayer)** | Resolve identity → **open the payee's payment surface** | Guest surface only. Amount and currency are **derived by the surface from the item** |

AppPlayer is a client — it cannot hold a session-opening credential, and building a backend to hold one is not the business of this axis. So **AppPlayer uses only the lower door.** The upper door belongs to consumers with their own backend, and the two coexist.

### 6.2 Relationship to 19

- **Using a custom scheme here is a case 19 §3.1 permits** — using one alone on public media is forbidden because it fails silently when the app is absent, but **a link issued inside the app for an installed counterpart** is the exception. A payment return is exactly that. 21 cites this exception and creates no new rule.
- **A payment return is not an entry code.** 19 §9.2 fixes that the three acquisition paths (scanner · intercepted link · deferred entry) all produce the same entry code and follow the same path thereafter. A payment return must not enter that path, so **the host's deep-link router discriminates the two before interpreting.** On failure to discriminate, the payment return is discarded.

---

## 7. Out of scope

- **Payment confirmation · settlement · refunds** — these belong to the payment surface and its server. This document defines only the request and the delivery of the result.
- **An in-app native sheet (v2)** — to be defined once a surface exists that issues a payee's publishable credential to a client. Until then the host refuses a native demand (§4).
- **Payment node hardware** — the device payment node draft is the axis where *the device is the payment target*, a different layer from this document. This document is the axis where *an app requests a payment*.
- **Lifecycle and trust grades in general** — safepage `FR-DEVICE-001~004` owns those.
- **Provider-registered price items** — items that follow a product price registered on the payment company's side are a connector resolution path, not a place the payment home bundle opens. Out of scope for v1 of this document.
- **Subscriptions and entitlement (tiers)** — activation, suspension, revocation and restoration of entitlement is an axis maintained by the payment surface's entitlement declaration, and **it is not something a document lives in** (that is the host's and the portal's own axis). Out of scope for v1.
