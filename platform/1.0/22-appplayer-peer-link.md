# 22 — AppPlayer Peer Link (devices of one account lending each other a connection)

> **Status: draft (decided 2026-09-17 — MCP over a direct connection between AppPlayers · the market only shares addresses and coordinates connections · decided 2026-09-21 — the direct connection is one "connection channel" contract under which two implementations, QUIC and WebRTC, coexist (§5.0) · large data and video/voice also go over this channel (§5.6)).** The rules for what is exposed are canon in [`23-connection-reexposure.md`](23-connection-reexposure.md), and transport in [`08-extension.md`](08-extension.md) §4. This document defines only **introduction and linking within an account**.

> **Premise: a device connected to any AppPlayer of my account is usable from my other AppPlayers, anywhere.**

A board attached to the Mac at home is still that person's device on a phone on a business trip. **Which device it was first attached to is not the user's concern.** Using a board at home in Korea from a phone in the US, or a small-business owner on the move using devices at home and at the office, is what this premise looks like.

This document is how that premise is kept. The AppPlayer holding a device lends **the connection it holds** (a LAN board · a local server · a local bundle) to other AppPlayers of the same account, and the using side **draws and operates that app on its own screen.** When close by, a shorter route is taken (23 §5.2), but that is an optimization, not the premise — **the promise of reach comes first.**

---

## 0. Identity — what it is and what it is not

| It is | It is not |
|---|---|
| **Lending a connection** — the screen is drawn **by the borrowing side** | Remote control — sending a screen |
| **Only between devices of the same workspace** (§0.1) | Publishing to others — that is the market door (15 · 23 §4) |
| **Two-way** — every peer lends and borrows | A structure with one side fixed as the server |
| **Across the internet by default** | LAN only |
| **The market only shares addresses and manages connections** — data is exchanged **directly between AppPlayers as MCP** | A structure where the market carries data (relay or detour, in any form) |
| Frames are **plain MCP** | Peer-only frames · a translation layer |

### 0.1 The boundary is the workspace (decided 2026-09-23)

Every place that says "the same account" reads as **the same workspace** ([`20-account-storage.md`](20-account-storage.md) §0.1). A personal workspace has me as its only member, so the result is the same as before; family and company workspaces have several members.

- A device **resides in several workspaces at once.** The screen shows one at a time, but connections, directories and cameras live separately per workspace — the family camera must not drop while the company screen is showing.
  - **Residing means "reachable"** (2026-09-25). A workspace not on screen with nothing in use (an offered camera · serving · viewing) puts its socket down (3 minutes idle) and stands up again by **wake-up (push)** — a socket costs per device × workspace.
  - **A device that cannot be woken keeps every workspace it joined open.** Today only Android receives push tokens — if iOS · macOS · desktop folded a workspace, that workspace's devices would have no way to reach it (measured: a Mac in two workspaces disappeared from the one not on screen). Workspaces that cannot be used (no plan · unregistered · no seat · locked) are closed either way.
- Rooms on the account channel (§4.1) are **per workspace**. The ticket carries `wid`, and the pipe connects only devices of the same workspace. "One socket per device" reads as **"one socket per device per workspace"**.
- The device-to-device ticket is not "a short-lived proof of the same account" but **"a short-lived proof of membership in the same workspace"**.
- Consent to expose (cameras · apps) is taken **per workspace**. The default is off, and what is turned on in one does not show in another. An admin can turn it off by policy (the market's part).
- In a workspace with several members, **a standing "viewing" indicator is mandatory** (§5.7) — whoever is watching my camera may not be my own device.

## 1. Terms

| Term | Meaning |
|---|---|
| **Peer** | One AppPlayer signed in with a market account (Pro · cloud web). X is a single-app device of a dedicated market and does not take part; Custom has no market sign-in and does not take part for now |
| **Account channel** | Where the workspace's devices attach to the market (rooms are per workspace · §0.1). The directory · addresses · connection-management signals · wake-ups · notices (`sync.changed`) flow here. **Data does not flow here** |
| **Link** | One **direct connection** between two AppPlayers. Devices connect to each other using the addresses the market shared |
| **App channel** | The strand one app uses on top of a link. One app = one channel = one MCP session |
| **Lender / borrower** | The device holding the connection / the device using it. One device is both at once |

## 2. Use cases

| Case | Lender | Borrower |
|---|---|---|
| A home device from outside | The Mac at home | A phone on a business trip |
| Small-business owner | Office PC + the Mac at home | Phone on the move — devices of both places in one launcher |
| The other direction | Phone (a sensor attached by radio) | Office Mac |
| From the web | The Mac at home | Cloud web |

**The route is chosen each time.** If the phone is on the same network or within range of the device it opens it directly; otherwise it goes through the holding device (23 §5.2). To the user it is **the same icon, the same app**, and which route opened it is not asked.

## 3. Layers

```
  App            drawn by the borrower's runtime                              (as today)
  ─────────────────────────────────────────────────────────────────────────────────────
  MCP            client ── app channel ── server (bridge: re-exposes the held connection)   (one channel per app)
                 frames are plain MCP (JSON-RPC). No translation · each AppPlayer is server and client (§5 · 23)
  ─────────────────────────────────────────────────────────────────────────────────────
  Connection     one contract · two implementations — QUIC (quinn, native pairs) | WebRTC (pairs involving the web · video/voice) (§5.0)
  channel
  ─────────────────────────────────────────────────────────────────────────────────────
  Introduction   the market (account service) — directory · address sharing · connection management · wake-up   (§4) — outside the data path
```

## 4. Introduction — the account service

**Introduction happens where sync storage lives.** The service that keeps account state already knows that account's devices, and it is also where the subscription attaches. The machinery for selling to others (hub nodes · listings · tenants · wallets) is not dragged in between my own devices.

| What flows | What does not |
|---|---|
| Who is there · what they offer · device names and **public keys** | Market machinery — listings · tenants · wallets |
| **Addresses** (where a device receives) · connection-management signals · wake-ups · account notices | **App data (MCP frames) — not a single byte** · public listings · routes to other accounts |

- **The directory**: each device writes its own and reads others' — `device/<deviceId>` (20 §2.4) already has that shape. When the apps it offers change, that device rewrites its own entry. The directory can be stale, so **whether something actually opens is answered at the moment of opening** (if it is not in the directory the opening side re-reads once — an app another device newly offered after attaching is not in the old directory), and if it fails, the reason is given (23 §5.5).
- **The directory is read once and held.** Who is present now (online) is answered by the account channel, not the directory, so devices coming and going (presence) does not re-read the directory. Re-reading covers **that one device**, in two cases only — ① when that device newly joins the channel (it may have edited its entry while away), ② when that device sends `signal {"op":"peer.changed"}` (a device that rewrote its entry **so that the content changed** sends it to every device it reaches). The device list (`/market/me/devices`) is re-asked only when 10 minutes have passed or the channel announces a device it does not know.
- **The workspace's devices are decided by the registered-device registry (`registered`)** (2026-09-25). Profiles (`device/<id>`) remain after release, so on a plan with a limit, profiles not in the registry are not presented as devices (measured: 13 devices showed in a workspace with 5 seats). The answer carries **which workspace answered (`wid`)**, and if it differs from the one asked, the answer is not used (20 §1). Measured 2026-09-22: re-reading every record on every presence change meant that one phone dropping its channel every few minutes on LTE made the devices read records 2,952 times an hour.
- **It shares addresses and manages connections** (§4.1). A device uploads the addresses it can receive on, receives the addresses of devices of the same account and connects **directly**. App data does not pass through the market.
- **A notice that the account changed** is a signal on the same channel — a device whose upload stored something sends `signal {"op":"sync.changed"}` to every device it reaches, and the receiving device pulls from storage (20 §3.3). The notice carries no content: what changed is answered by storage, and the channel carries only "look now". Directory attach/detach moves only when devices come and go, so it cannot replace this notice.
- It issues the ticket for **the account channel** (§4.1) and the **device-to-device ticket** (a short-lived proof by which the other side confirms membership in the same workspace). The account channel ticket is **bound to that device** (2026-09-22) — the device sends `deviceId` when taking the ticket, and the ticket carries that id (`did`). The pipe refuses when the query's `deviceId` differs from the ticket's `did`. Taking the ticket is also **the registered-device check** — a new device beyond the count the plan sets gets no ticket (20 §5.1).
- **A device that is off** is woken. A device that cannot be woken receives only while it is attached.
- **The subscription opens this feature.** It opens **from the minimum grade** (20 §5) — the place that opens sync and device linking while giving almost no storage, and that is where what this feature actually sells lies. No subscription, no sync — and no introduction.
- **Devices already paired** know each other's keys. On the same network they can find and attach to each other even when the account service is unreachable — what the subscription buys is the introduction office, not the connection itself.

### 4.1 Address sharing and connection management — the market's place

A device cannot know by itself what address it appears as outside its own router, nor where the other side is. So **one place both sides already reach** collects and distributes addresses and aligns connections. That place is **already paid for** as the account service. This is the whole of the market's role; once a connection stands, data goes device to device.

- **Address candidates** — found and uploaded through **the very UDP socket** the device uses for QUIC:
  - Same-network address (LAN IP:port) — on the same router this reaches directly
  - Public address — the outside address obtained by sending a **STUN binding request** from that socket (the port the router attached to that socket). STUN is a request that only reports an address; no data passes. Cloud Run cannot receive UDP, so the STUN endpoint is a UDP endpoint outside the account service (public STUN for now)
  - IPv6 global address
  - A port opened on the router (UPnP) — a later improvement
- **Connection coordination** — at the moment one side wants to open, the account channel gives both sides **the other's candidates · certificate fingerprint · a time to connect simultaneously**. A router blocks packets arriving from outside first, so at the agreed time both sides **dial QUIC to each other from the same socket** — the outgoing packets open a path in each router, and the following packets come in. The first connection to stand is used and the rest are closed. Measured (2026-09-17 · Mac = home router · phone = LTE carrier NAT): **3/3 direct** · phone→Mac 90–120 ms · Mac→phone had its first packet dropped before the other's hole opened and was established after a 1-second retransmit.
- **The account channel is one WebSocket** — it carries only the directory · addresses · connection signals · notices. The Cloud Run request timeout is the connection's lifetime, so it **switches over before the timeout** (50 minutes · new socket with a new ticket first · the old socket's `replaced` is read as a hand-over · signals sent during the switch go to the new socket in order — reflecting a defect measured 2026-09-17). If this channel drops, **device-to-device connections already standing are not dropped** — data does not pass through here.
- **Sync storage is not used as a channel.** Storage holds the directory and address records.
- **The market's cost is introduction and signals only** — a toll-level cost. No cost proportional to data volume arises at the market.

## 5. Links and app channels

### 5.0 Connection channel — one contract, two implementations (decided 2026-09-21)

Direct connections between devices are handled by **one contract, the "connection channel"**. QUIC and WebRTC are **two implementations** of that contract, and the layers above (MCP bridge · borrowing · lending · apps) do not know which implementation it is. With only one, the upper layers would have to be rewritten for the pairs that need the other (pairs involving the web · video/voice) — so both are plugged into the same place from the start.

| Contract | Meaning |
|---|---|
| `open(appId)` → app channel | One app = one channel = one MCP session (§5.2). An open channel is a pipe carrying JSON-RPC one line at a time |
| Incoming app channel | A channel the other side opened (app id and pipe) |
| Limits | Per-message ceiling · flow-control method — the implementation states its own values and the upper layer splits to fit (§5.3 · §5.5) |
| Peer · liveness | The other device's id · visible address · closure |
| Kind | `quic` · `webrtc` — for records and diagnostics. Not used as a basis for branching in upper layers |

**What is used for which pair**

| Pair | Channel | Basis |
|---|---|---|
| Native ↔ native (Pro to Pro) | **QUIC** | Large bodies · flow control · loss detection measured superior (§5.1) |
| Pairs involving the web (cloud ↔ Pro · cloud ↔ cloud) | **WebRTC data channel** | The only standard path by which a browser punches its own hole (§5.5) |
| Video · voice (any pair) | **WebRTC media** | Codecs · jitter buffer · bandwidth adaptation are in the standard — not rebuilt on a data channel or QUIC (§5.6) |

- A pair **may hold both channels together** — a native pair with data over QUIC and a call over WebRTC media.
- The choice is made by the opening side from the other's kind (device info in the directory). **The signal op name states the transport** (`link.*` = QUIC · `rtc.*` = WebRTC) — there is no separate capability-negotiation step.
- In both implementations **the market carries only addresses and signals** (§4.1). Neither channel detours through the account channel.
- The contract lives in the shared recipe (`peer_link`), and the implementations live where they are used — QUIC in native hosts that need a native library, WebRTC on both native and web.

### 5.1 Link — a direct QUIC connection between AppPlayers

- **The transport is QUIC (quinn).** An AppPlayer keeps one QUIC endpoint on one UDP socket and **acts as both server and client** (receiving and dialing must use the same port for the path opened in the router to be usable).
- **Why QUIC** (measured 2026-09-17 · same device pair): a large body flows in one stream (8 MiB arrived — a data channel above 256 KiB did not arrive) · flow control is built in, so a fast producer **waits** (zero loss — the data channel died and lost data at 16.8 MB) · loss of the peer is known **within a configured time** (3 s — ICE declared failure after 2 min 10 s) · the stack is light.
- **Only the same account attaches** — certificates are self-made by each device, and the other side is accepted **only when it matches the fingerprint** received over the account channel (no certificate authority · no device certificate issuance).
- **Each AppPlayer is both MCP server and client.** The lender connects a received stream by app id to the **bridge** (23 · `McpForwarder` + the declared surface), and the borrower is an ordinary MCP client on the stream.
- **When a direct connection cannot be made, nothing relays.** On a network where every candidate is blocked the link does not stand, and the screen says **why it cannot** (23 §5.5). There is no path by which the market carries data, in any form.
- **Implementation**: a Rust (quinn) native library called from Dart (macOS · Windows · Linux · Android · iOS).

### 5.2 A stream per app

On top of the direct connection, **one QUIC bidirectional stream per app = one MCP session**. The side opening the stream puts the app id on the first line, and the lender accepts **only if it offers that app and the other side is allowed** (23 §4 — refusal and absence give the same answer). MCP messages (JSON-RPC, one per line) follow. Closing the app closes only that stream.

### 5.3 Size and speed

The size and frequency of frames are the app's to decide. A QUIC stream has no size ceiling and built-in flow control, so there is no splitting and no separate ceiling. Real-time streams (extended subscriptions — mcp_ui_dsl 06 §6.4 Extended) must flow without breaks. Measured (LTE phone ↔ Mac): 8 MiB 3.0–3.7 s · a 10-per-second stream for 10 s, 99/99.

### 5.4 Disconnection

| What | Result |
|---|---|
| Account channel switch-over · drop | Device-to-device connections **stay** — data does not pass through there |
| Device-to-device connection drops (keep-alive · idle timeout) | Known within the set time. Re-coordinate over the account channel, reconnect and re-subscribe |
| The phone switches networks (Wi-Fi ↔ LTE) | Re-coordinate with the new address and connect (connection migration is a later improvement) |
| The lending device | Session ends. The borrower sees the reason (§7) |
| The lending device is offline (the account list says so) | The borrower **does not dial on a timer**; it waits for news that the device is back online (presence on the account channel → device list) and attaches right then. The same holds while the app is open on screen — a timer cannot succeed sooner than that news. Measured 2026-09-21: three devices dialing every second rang a connection-state notice on each failure and rewrote **their own device records about 160 times a minute** |
| Writing the device record | If the content (excluding `at`) equals what was last written, it is not written. `at` is the "last seen" other devices see, so it is refreshed only every 10 minutes |

### 5.5 Pairs involving the web (browser) — WebRTC data channel (the second implementation of §5.0)

A browser cannot send UDP first, so it cannot do §5.1's QUIC hole punching. The only standard path by which a browser punches its own hole is WebRTC, so **pairs involving the web connect directly over a WebRTC data channel** (2026-09-17). The premise is the same — the account channel carries signals only, no relay.

| Item | Rule |
|---|---|
| Transport selection | The op name states the transport — `link.offer` is QUIC (§5.1), `rtc.offer` is WebRTC. No capability negotiation |
| Who opens | **Only the borrower offers**; the lender only answers (no simultaneous-offer collision). Channels in the other direction open on the same connection |
| Signals | `rtc.offer {id, sdp}` · `rtc.answer {id, sdp}` · `rtc.candidate {id, candidate, sdpMid, sdpMLineIndex}` · end of candidates = `candidate: null` · `id` = attempt id (candidates of an old attempt arriving late are discarded) |
| Pre-negotiation channel | Before the offer, a pre-agreed channel (`_link`, negotiated · id 0) is created — an offer with no m-line gathers no candidates (measured) |
| Route | ICE (STUN) direct only. No TURN · no pipe detour |
| Timeout | The direct-connection timeout (a few seconds) is when "it cannot" is said — ICE's own failure declaration takes 2 min 10 s (measured) |
| Failure text | States the ICE state together with the candidate kinds both sides gathered (host/srflx). Symmetric NAT is named only when confirmed by measurement |
| Channel | One app = one data channel (label = app id) = one MCP session · incoming messages are preserved with a single subscription |
| Size · flow | The per-message ceiling is read from the negotiated value (`maxMessageSize` · SDP `a=max-message-size`) and messages are split to it — without the attribute the standard default of 64 KiB (RFC 8841) is assumed · the negotiated value is confirmed by measurement on a Pro ↔ web pair · flow controlled with `bufferedAmountLowThreshold` (pushing into the buffer kills the channel — measured at 16.8 MB) |
| Identity | The DTLS fingerprint in the SDP is the authentication, and the SDP came over the account channel, so it passed account authentication — no separate fingerprint field |
| Pro | Keeps a WebRTC answerer for web peers (Pro to Pro is QUIC) |

### 5.6 Large data · video · voice — what the channel gives apps (direction · 2026-09-21)

The connection channel is not a part only for borrowing. Once **a path by which AppPlayers reach each other directly** stands, it is used for large data and real-time media too — sending those through the market or another relay creates cost proportional to data volume (contrary to the §4.1 "toll") and adds latency.

| Use | Channel | Rule |
|---|---|---|
| Large data (files · bundles · record dumps) | That pair's data channel (QUIC stream · WebRTC data channel) | Follows flow control — built into QUIC, `bufferedAmountLowThreshold` on WebRTC (§5.5). Never pushed in all at once |
| Video transmission · video call · voice call | WebRTC media track | Opened between devices of the same account. Signals use the same path and identity as `rtc.*` (§5.5) |
| Where a relay path exists elsewhere in the platform (e.g. a server app through the market hub) | — | Large data and media are **not sent through that relay.** The channel standing directly between AppPlayers is used |

- Among video, the **device camera** is fixed as a contract in §5.7. The rest (files · calls) is still **direction**. The surface by which an app calls on this capability (UI DSL action · host capability) is defined separately — what is decided now is "the channel contract is shaped so it does not block these uses": upper layers do not know the channel kind, and the place that opens media is provided by the WebRTC implementation.
- On a network where a direct connection cannot stand, these uses cannot stand either — they are not switched to a relay; the reason is given (§5.1 · 23 §5.5).

### 5.7 Device camera — real-time video (contract · decided 2026-09-22)

This fixes §5.6's video as a contract for the first time. The subject is **a camera attached to a device** (built-in · USB webcam), watched in real time by other devices of the same account. Only the platform (host) implements it — the UI DSL does not change.

**It is a device capability, not an app.** MCP has no real-time media, and the camera belongs to the host (23 §2.1: "host capabilities are outside the app surface"). So it is offered **separately** from app lending (§6), with separate consent.

| Item | Contract |
|---|---|
| Offering | **Per camera · off by default** (the opposite of apps' default on — a camera shows people). The owner turns on **one camera at a time** in the "This device" section of that device's "Cameras" screen — the rear can be offered while the front is not. At the moment of turning on, the OS camera permission is obtained on that device (so no permission dialog appears on an unattended device when viewed remotely). Only cameras turned on are listed in the device record (§4 directory) as `media: [{id, name, kind:"video", audio}]`; turning one off removes it and **immediately ends that camera's sessions in progress** |
| Sound | **Per camera · off by default · separate consent** — the microphone is included only when "Include sound" is on (OS microphone permission at the moment it is turned on). The record's `audio: true` marks it. A session of a camera with sound sends a video track and a sound track together. The viewer can mute (not muted by default — it is what the viewer chose). **Turning sound on or off is not a decision to end viewing** — the offering side closes that camera's sessions with `media.close {sid, reason:"changed"}`, and the viewer **reopens immediately** to continue with the new setting (measured 2026-09-22: turning off only the sound showed "The video has ended" on the viewing devices) |
| Address | `client://stream/<deviceId>/<sourceId>` — used as-is as the `source` of the UI DSL `video` widget. The host's `MediaPort` resolves this form and opens a session (the DSL's `client://<type>/<path>` is the place the host resolves) |
| Channel | **WebRTC media track on every pair** — even between natives, video opens over WebRTC, not QUIC (loss recovery · codecs · bandwidth adaptation · common with browsers, §5.6). One connection per session. Not mixed with app data channels |
| Who opens | **The viewer offers** (receive only) · the offering side answers (send only). The offering party is always the viewer, so the direction is the same when the web is involved |
| Signals | Account channel `signal` (§4.1), bound by session id `sid`: `media.offer {sid, source, sdp}` → `media.answer {sid, sdp}` or `media.refused {sid, reason}` · both sides `media.candidate {sid, candidate}` · either side `media.close {sid}` |
| Refusal reasons | `not-offered` (switch off · no such source) · `camera-unavailable` (OS permission denied · in use by another app) · `busy` (old builds only — no longer sent) · `other-camera` (on a device that runs one camera at a time — phones · tablets — another device is already watching a different camera) — the viewer shows the reason as-is. On such a device, when **the same device** asks for a different camera, that device's old session is closed and the new one opened. On any device, a new camera opens only after a capture that is shutting down has fully stopped (when switching, opening a new camera while the old one still holds the device yields no video). Android is given `facingMode` along with `deviceId` — on devices with the old camera API where the plugin cannot match the id, it picks by direction (without it, everything is front). A camera just released (permission check at switch-on · a capture shutting down) may briefly fail to open — **the offering side retries within itself** a few times (0.5 · 1 · 2 s) and answers `camera-unavailable` only if it still fails. Every re-request by the viewer sends an offer and candidates over the account channel (data cost), so a viewer that received a refusal does not re-request on its own — only when a person presses "Try again". Reconnecting after a network drop also ends — after **6 attempts** at 1→30 s intervals (about a minute) it stops and shows "Try again". A stopped session reattaches once when news arrives that the device is back or its record changed (records it reads anyway). It does not dial a device that is off |
| One capture, shared | A source has one capture. With several viewing devices the same track is shared — device load does not scale with the number of viewers (23 §6.1). When the last session closes, the camera turns off. Cameras have **no separate limit** — viewers are the account's devices, and the plan's **registered-device count** (5 for personal, 20 §5.1) already bounds them (decided 2026-09-22). A viewer **always** tells the other side with `media.close` about attempts it abandoned itself (drop · signal failure · timeout) — it does not announce only those the other side ended (`close` · `refused`) |
| Visibility · cutting off | The offering device shows camera sessions in "In use now" (23 §6) alongside app sessions, and can cut them off |
| Lifetime | When the viewing screen (widget) closes, `media.close`. **A session that ended because the connection dropped reopens itself while the screen is open** (1 → 2 → 4 … up to 30 s apart, immediately when the offering device comes back online). **A session the offering side ended is not reopened** — `media.close` (owner cut off · that camera turned off) and `media.refused` are a person's decision, so the reason is shown and it stops. Only `reason:"changed"` (settings changed) reopens immediately. There is no rewind or length (`seek` does nothing and no length is reported) |
| Viewing screen | **Switch to another camera** offered by the same device (front ↔ rear — a new session) · **zoom · pan** (on the viewer's screen — no control is sent to the offering device) · full screen · mute if there is sound |
| Background | iOS uses the camera and microphone only while the app is on screen (OS constraint) — a locked or backgrounded iPhone · iPad shows as "off" in the list. Android can keep offering in the background with a camera foreground service. Mobile background is defined separately · **on Android, with the screen off the app and account channel stay alive while the OS reclaims only the camera** — the offering side then closes sessions with `media.close {reason:"away"}` and stops the capture (the viewer, instead of a frozen picture, sees "that device's screen is off" and waits, reattaching when news of that device arrives). When the screen comes back it sends `media.ready {source}` to the devices that were watching so they **reattach on their own** (that device never left the account channel, so there is no other news to wake them). Separately, a viewer **whose picture has not arrived for 25 seconds** abandons that attempt and re-requests — some paths die without a word |
| No relay | STUN only. When a direct path cannot stand, it is not switched to a relay; the reason is given (§5.1) |

- There are two entrances: ① an app screen (UI DSL `video`, the address above) ② a platform screen — the built-in "Cameras". One screen holds this device's switches and the list of cameras other devices offer; tapping one opens a viewing screen drawn by the host (home is not divided by device, so it lives in a built-in rather than a device section of the launcher). Both use the same session layer.
- Device names are what a person recognizes — the host name on desktop, the model on phones and tablets (the mobile host name is always `localhost`, so they all looked the same).
- Capture requests are given as numbers (`width: 1280, height: 720, frameRate: 30`) — the iOS plugin reads `{ideal: n}` only as a string and otherwise falls to 640×480 (measured on iPad 2026-09-22).
- iOS uses the camera only while the app is on screen — a locked or backgrounded iPhone · iPad shows as "off" in the offering list (mobile background is defined separately).
- The device record's `media` also follows §4's "read once and hold" — flipping a switch changes the record's content, so `peer.changed` goes out.

## 6. What is lent

**The rules are set by 23** — the unit is the app, only the declared surface, host capability tools and kernel knowledge are outside the surface, elevated permissions require consent again, revocation is immediate, and who is using it is visible.

This document adds two things.

- **Within an account the default is "usable".** Using my devices from my AppPlayers is the premise (preamble), so permission is not asked again per device. Only apps to be withheld are excluded. This **applies to the account door only** — the market door remains closed by default and appears in no public listing.
- **The host's own functions (sync · account state · market transactions) are not offered this way** — those go through the market (23 §2.1).

## 7. User experience

| Situation | What the borrower sees |
|---|---|
| Opening | "Opening · the Mac at home · ESP32 lights" |
| Open | The app behaves as if local |
| A network where a direct connection cannot stand | "The Mac at home cannot be reached directly from this network" — not switched to a relay |
| The lending device is off | "The Mac at home is off · last seen 18:03" |
| Offering turned off on that device | "This app is no longer offered" |
| In the same space | Opens directly without borrowing — the user does not see the difference (23 §5.2) |

Response timeouts assume international latency (hundreds of ms).

## 8. Division of roles

| Who | What |
|---|---|
| **Account service (market backend)** | Directory · **address sharing · connection management** (reporting public addresses · simultaneous-connect time) · wake-up · tickets · subscription gate. **Outside the data path** |
| **Account channel** | Signals only — directory · addresses · connection signals · notices. Carries no app data |
| **Recipe** | MCP forwarder (bridge) · declared surface |
| **Native library (Rust quinn)** | UDP socket · STUN · simultaneous connect · QUIC streams — called from Dart |
| **AppPlayer Pro** | Wiring · offering switches · "My other devices" in the launcher · network permission declarations |
| **AppPlayer Cloud (web)** | Direct connection over the WebRTC data channel (§5.5) |
| **Studio** | Decided after this document is final |

## 9. Verification

Experiment record (2026-09-17 · Rust quinn · scratch):
- Mac loopback: simultaneous connect 0.6 ms · 8 MiB 41 ms · slow-receiver stream 2290/2290 zero loss · peer loss after kill -9 detected in 3.03 s
- **LTE phone (carrier NAT) ↔ Mac (home router)**: public STUN + simultaneous connect **3/3 direct** · RTT 57–69 ms · initialize 177–200 ms · 8 MiB 3.0–3.7 s · stream 99/99 ×3
- Comparison: iroh 1.2 (coordination via the n0 relay) on the same pair was 2/2 direct but 8 MiB took 48–49 s · with the relay off the connection failed

Product verification (after implementation):
1. **Direct connection established** — same network · different network (LTE) · which candidate it stood on · account channel bytes not proportional to data
2. **Repeated operation** — H723 two-step check · ESP32 · MCP Demo
3. **Stream continuity** — ESP32 extended subscription over a long period (including account channel switch-over)
4. **Recovery from drops** — device-to-device drop · lending device restart · phone network switch
5. **Networks where a direct connection cannot stand** — whether the reason is given without relaying

Record (09-17): the 09-16–17 implementation broke this premise — data passed through the account pipe. That implementation was set aside and replaced by the direct connection above.

## 10. Relation to other documents

- [`23-connection-reexposure.md`](23-connection-reexposure.md) — **what is exposed and how.** This document only uses those rules.
- [`08-extension.md`](08-extension.md) §4 — the two transport tiers and the injection slot. The peer link transport plugs into that slot too.
- [`15-hub-channel.md`](15-hub-channel.md) — the market door. The links in this document **do not pass through the hub** — the publishing and permission models are not mixed.
- [`17-device-discovery.md`](17-device-discovery.md) — how devices are found in the same space.
- [`20-account-storage.md`](20-account-storage.md) — where the directory lives, and sync. This document makes no copies.
- [`14-asset-credentials.md`](14-asset-credentials.md) — secrets never leave the lending device.
