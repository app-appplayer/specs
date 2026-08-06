# 17 — Nearby device discovery

> status: **draft** (added 2026-07-14 · rev.2 2026-07-15 — §6 trust/signature carriage · rev.3 2026-07-28 — §7 coupling with the onboarding axis · rev.4 2026-08-06 — §7.6c mirrored, §7.6d reconnect persistence, §7.6e observation as trigger)
> companions: [`16-ble-transport.md`](16-ble-transport.md) (BLE binding · advertising) · [`08-extension.md`](08-extension.md) §4 (the three connection layers) · the embedded node docs (board product documentation) · AppPlayer Pro MOD-DISC (the consuming pipeline)

The standard for finding and connecting to a **nearby** MCP server (an embedded board, a local device) without going through the cloud. The discovery pipeline itself — policy, candidate stream, the "discovered" surface — belongs to the host (AppPlayer Pro's `DiscoveryManager` is the reference); this document fixes only the **per-transport announce/scan rules and the confirmation procedure**. Follow them and any board is found by any host's discovery surface.

## 1. Model — two stages: announce-scan → probe-confirm

**Stage 1 (announce/scan, cheap)**: obtain *candidates* using the per-transport rules. A candidate is `{transport, endpoint, displayName?, meta?}`. **No manifest is required at this stage** — a BLE or TCP board cannot hand over a document before a connection exists.

**Stage 2 (probe-confirm, common to every transport)**: build the candidate's `ClientTransport` → `initialize` → read `bundle://manifest.json` → settle the candidate against its real identity (manifest id, name, whether it has a UI, its tools) and reflect that on the discovery surface. **Failing to complete the probe drops the candidate** — being visible to a scan does not make something an MCP node. The probe doubles as conformance verification.

Registration and connection are separate from discovery: once the user picks a confirmed candidate, the connection is made through the three layers of 08 §4 (the `connectExtension` seam). Automatic registration (autoRegister) is for channels that carry a basis for trust only — host policy, e.g. a USB Secure partner chain. How that basis is carried and verified is §6.

## 2. Stage 1 rules (per transport)

| transport | device side (announce) | client side (scan) |
|---|---|---|
| **BLE** | [`16-ble-transport.md`](16-ble-transport.md) §6 — the service UUID MUST be in the advertising | scan filtered on the service UUID. Candidate endpoint = device id, displayName = local name |
| **WiFi/LAN** | SHOULD publish mDNS/DNS-SD **`_mcp._tcp.local`** (§3) | browse `_mcp._tcp` (LAN-scoped, which is what "nearby" means here). Candidate endpoint = host:port + TXT |
| **serial/USB-CDC** | none — the physical connection is the announcement | enumerate ports (desktop = libserialport; Android = USB host). **Every port is a candidate** — Stage 2 sorts out what is what. VID/PID hints may order the list but MUST NOT filter it |
| **USB raw** | none | enumerate devices (libusb). Same principle as above |

## 3. mDNS contract (`_mcp._tcp`)

- service type = **`_mcp._tcp.local`**, instance name = the display name.
- **TXT records**:

| key | required | value |
|---|---|---|
| `proto` | MUST | `ndjson` = raw TCP + newline-delimited JSON-RPC (the embedded contract) · `http` = MCP streamableHttp |
| `id` | SHOULD | manifest id (`<vendor>.<model>`) |
| `v` | MAY | version |
| `path` | MUST when `http` | endpoint path |

- The SRV record is authoritative for the port. The recommended default for a TCP listener is **6270** (the phone keypad spelling of M-C-P is 6-2-7; it is a convention for manual entry where there is no mDNS, not a rule).
- A different `proto` means Stage 2 attaches over a different transport — which is what lets embedded boards and general MCP servers coexist under one service type.

## 4. Host integration (informative — not normative)

- AppPlayer Pro: the scanners above are wired as implementations of `appplayer_launcher`'s `DiscoveryTransport` interface (the `usb` · `mdns` · `bt` slots, where only InMemory used to sit). A candidate is settled on the "Discovered" tab → user confirmation (notify by default) → connection through the three layers.
- Other Flutter hosts such as Studio consume the recipe's scanner API directly — Stage 1+2 are identical without a `DiscoveryManager`.
- Scanner implementations are placed by **recipe** (the BLE scanner lives alongside the `ble_transport` recipe; the mDNS scanner is pure Dart). The core packages (`mcp_client`, `brain_kernel`) are untouched.

## 5. Conformance

- Board: obey the announce rule for its own transport — BLE advertising (16 §6) or mDNS publication (§3) — and complete the Stage 2 probe.
- Client scanner: MUST NOT require a manifest in Stage 1 · MUST NOT surface a candidate that failed its probe · MUST NOT filter serial enumeration by VID/PID.
- Client host: re-attach subscriptions, notification fan-out and a read-once every time a link is re-established (§7.6c) · absorb truncated provisioning-status notifications with a read (§7.4a) · do not give up reconnecting while the app is open, and keep detection and retry pacing as separate values (§7.6d) · if observation is used as a trigger, scope it to servers an open app is waiting on and keep the time-based retry as the floor (§7.6e).
- Board (optional): a manifest `trust` signature (§6). Required to appear on a host whose policy enforces signatures.

## 6. Trust and signature — how a manifest carries a signature

The convention for carrying a discovered device's **identity signature** in its manifest. It is the evidence a host's signature-enforcement policy (e.g. AppPlayer Pro FR-DISC-005 `enforceSignature`, ON by default) and USB Secure partner auto-registration (FR-DISC-007) verify against. The key-distribution model is a chain: MakeMind root → manufacturer certificate → the certificate embedded in the device.

### 6.1 Carriage — the optional manifest field `trust`

The manifest object served at `bundle://manifest.json`, which Stage 2 reads, may carry an optional `trust` field:

```json
{
  "manifest": {
    "id": "acme.vault01",
    "name": "Acme Vault",
    "version": "1.0.0",
    "trust": {
      "role": "partner",
      "signerCert": {
        "publicKeyBase64": "<32-byte Ed25519 pubkey, base64>",
        "serial": "<issuer-assigned serial>",
        "notBefore": "2026-01-01T00:00:00Z",
        "notAfter": "2031-01-01T00:00:00Z",
        "algorithm": "ed25519"
      },
      "signatureBase64": "<Ed25519 signature, base64>"
    }
  }
}
```

- The `signerCert` schema is the same shape as a root CA entry (the fingerprint is derived automatically as SHA-256 over the raw public key, hex).
- `role` selects the root pool to verify against: **`partner`** (the hardware manufacturer chain) or **`marketplace`**. Any other value is treated as untrusted, and MUST be dropped where signatures are enforced.

### 6.2 What is signed (canonical bytes)

The canonical JSON of the manifest object **with the `trust` field removed** — object keys sorted ascending recursively, no superfluous whitespace, UTF-8 encoded. Array order is preserved. The signature algorithm MUST be **Ed25519**.

### 6.3 Verification (client)

1. Is `trust.role` in the permitted set (§6.1)?
2. Does the `signerCert` chain verify against that role's root CA, including expiry and CRL?
3. Does `signatureBase64` verify over the §6.2 canonical bytes with `signerCert.publicKey`?

All three passing means the signature is valid. The host hands the result to the discovery pipeline as trust evidence for the candidate (e.g. `TrustEvidence.signatureValid` · `partnerChainValid`). With enforcement ON, a candidate with an invalid or absent signature is not surfaced and the outcome is audited (`app_signature_failed`); a valid `partner` chain may serve as the basis for auto-registration (§1).

### 6.4 Cost on the device

The signature is **computed at manufacture or provisioning time and baked into the static manifest** — the device performs no runtime cryptography (an MCU C base only serves strings). Session-level mutual authentication (challenge-response, key agreement) is a separate layer; this section covers device *identity* only.

## 7. Coupling with the onboarding axis (rev.4 · 2026-07-28)

Discovery presumes **the device is already on the network**. A factory-fresh board has no credentials, therefore no address, and therefore never appears in Stage 1 at all. What comes before is the onboarding axis (the embedded node's FR-PROV), and the two are **different axes**:

| | advertises | identity | next action |
|---|---|---|---|
| onboarding | `mcp-prov` (a dedicated GATT service) | **not** an MCP server | inject credentials → join → persist → **reboot** |
| discovery | `_mcp._tcp` · the MCP service UUID | an MCP server | probe-confirm → register |

### 7.1 Do not mix them in the discovery pipeline (MUST NOT)

Onboarding candidates MUST NOT enter the host's discovery pipeline (the §1 candidate stream, §6 trust verification, probe-confirm). That pipeline's contract is "discover an MCP server", and feeding it something that is not one applies signature enforcement, trust policy and probing to something they were not written for. An onboarding candidate cannot pass a probe, so under §1 dropping it is the correct outcome.

### 7.2 Do merge them on the surface (SHOULD)

On the **user-facing surface**, however, they belong in one list. To the user the two are a single job — "there is a new device, make it usable" — and splitting the screen means **the user has to know the device's wireless state before knowing where to look**. That is the implementation's stages leaking into the UI.

The recommended shape is one list with **per-row actions**:

```
Discovered
  📡 mcp-prov-A4F2   Needs Wi-Fi setup   [Set up ▸]   ← onboarding axis
  🌡  esp32.temp      192.168.0.21        [Add ▸]      ← discovery axis (probe-confirmed)
```

When onboarding succeeds the device reboots and **reappears in the same list as a serving node**. That continuity is the value of merging them.

### 7.3 Radio mutual exclusion (MUST)

BLE onboarding and BLE discovery scanning **contend for the same adapter**. A GATT connect attempted while a background scan is running starves (observed on hardware: connect-scan starvation, `reason=211`). The host MUST stop discovery scanning for the duration of a commissioning run and resume afterwards. Once the two axes share one screen this coordination is not optional.

### 7.4 Deliver the network list in pages (MUST)

The first screen of onboarding is **the user choosing their own network**. If that list is silently truncated the user concludes their router is absent and abandons onboarding — and neither the device nor the host leaves a trace, because the device logs "scan succeeded" and the host renders a valid short list normally.

A GATT attribute value is capped at **512 bytes** by ATT and that cannot be raised. So the list is delivered in **pages**. One page looks like:

```json
{"gen": 3, "from": 0, "total": 23, "more": true, "aps": [{"ssid": "...", "rssi": -48, "secure": true}]}
```

- The host reads a page and, while `more` is true, **records `from + aps.length` as the cursor** and reads again, repeating until `more` is false.
  - BLE = **write** `{"from":N}` to the same `wifiList` characteristic (READ + WRITE)
  - HTTP = `GET /wifi-scan?from=N`
- The device **retains the full scan result**. It MUST NOT drop APs for payload-size reasons.
- A response without `more` or `gen` is read as false and ignored respectively — a device predating the page contract works in a single round trip (backward compatibility), and because its characteristic is read-only the host **MUST NOT write before the first read**.
- If `more:true` but `aps` is empty the host **ends** the traversal: the cursor does not advance, so continuing is an infinite loop.
- The host's traversal MUST have an upper bound. Onboarding is where a user first meets the device, and a traversal that never returns is a screen with no way out.

A page exceeding one MTU is normal — the stack delivers it via ATT Read Blob. **Do not shrink the list to fit a single read.**

### 7.5 Onboarding a phone uses must pass the phone's connectivity check (MUST)

The user of the HTTP portal (SoftAP) approach is a phone. Right after associating, a phone decides for itself whether the network is usable, and if it decides no it **moves traffic to cellular** — after which typing the address by hand still does not reach the device. The endpoint can be alive and the portal still unreachable.

So the portal needs three things: a **DNS responder** (with no resolver in the lease, name lookup fails and the network is discarded), **DHCP option 6** (advertise this device as the DNS server), and **a 302 redirect of unregistered URLs to the portal** (the signal an OS captive-portal detector looks for). Details are in the embedded node's FR-PROV.

A debugging principle belongs with it: at this layer, **"is the server healthy"** and **"is the path healthy"** look like the same silence. Log accepted TCP connections and inbound DNS queries separately, and give the device a self-check that calls its own server once — that separates the two.

### 7.6 Report failures with a diagnosable reason (MUST)

A join retries, and each attempt can fail for a different reason. **Reporting only the last reason destroys the diagnosis**: a wrong password shows up as a 4-way-handshake failure on the first attempt and as a generic connection failure on every retry after that.

- The device MUST **classify each attempt** and accumulate them so a specific reason is not overwritten by a generic one. Among specific reasons, the **earliest** one describes what the user actually did.
- Standard reasons: `auth` (credential mismatch) · `no-ap` (no such network) · `disconnect` (anything else).
- The host MUST **translate these codes into the user's language**. Showing the raw code tells a user who mistyped their password "disconnect" — true, but it does not tell them what to do next. An unrecognized code is passed through rather than hidden.

### 7.6a Registering is not connecting (MUST)

`Add` registers a device as **reachable**; it does not hold a connection. When a composed screen needs it (a `view` naming an origin), the host opens it **at that moment**.

The reason not to hold connections open is measured, not theoretical: many of these boards **serve only one peer at a time**. Holding a connection per registered device means the most recent connect resets the earlier ones (`Connection reset by peer`) and the tiles on screen die in rotation.

For the same reason **the discovery probe MUST NOT re-dial an already-confirmed node every cycle**. Re-probing opens a second connection, and that resets the one the host is holding for composition. Continuing to announce is itself the liveness signal, so **probe once at first discovery and trust the announcement afterwards** (a sweep expires it when the announcements stop). USB pays this as a DTR reset and a TCP board as peer eviction — only the shape of the cost differs; the rule is the same.

### 7.6b One connection per device, and everything above it is a shared resource (MUST)

One device is used by several screens at once — its own standalone app, and any composed tile naming it as an origin. The host MUST open **one connection per device id** and let everyone share it. Opening one per consumer means the second dial is refused on the single-peer boards of §7.6a, leaving nothing on screen but an error that looks like a device fault. Even on a server that accepts concurrent peers, two links split subscriptions and state tracking in two.

Sharing the connection makes **everything layered on top a shared resource too.** Each must be managed by consumer count rather than by owner:

| resource | rule |
|---|---|
| connection | one per id. The only moment it is released is an **explicit termination by the user** — leaving a screen is not termination, since apps keep running in the background |
| subscription | **reference counted**. `resources/subscribe` goes out on the wire for the first consumer only, `unsubscribe` for the last only. The server only knows "subscribed or not", so whoever releases first would otherwise cut the other's stream |
| notification | **fan out**. Client implementations commonly hold a single handler per method. Used as-is, a later registration silently replaces an earlier one and that consumer never receives another notification — the registration was not lost, its owner changed, so closing the screen does not bring it back |

All three defects **are reported by no layer.** The subscription is alive and so is the socket. The symptom is visible only on screen: a tile stops, and pressing `Subscribe` on that page moves the value **exactly one step**. That step is not a notification — it is the read-once included in the subscribe operation. Had notifications been arriving the value would have kept flowing, which makes this "changes exactly once" the decisive signal that the notification path is severed.

### 7.6c The link is re-established, and what sat on it does not come along (MUST)

§7.6b fixes that a connection is released only on explicit termination by the
user. But **the platform terminates it for you** — mobile suspends the process
in the background and kills the socket. On return the host re-establishes the
connection, and at that moment the client is a **new** one: the server has no
idea what the old link had subscribed to.

So every time a link is re-established the host MUST re-attach what sat on it:

| re-attach | why |
|---|---|
| `resources/subscribe` | a subscription belongs to the connection. A new link has subscribed to nothing |
| notification fan-out registration | handlers are held on the old client |
| a read-once right after re-subscribing | values kept flowing on the device. A returning screen must show the **current** value, not the number that was frozen there |

Like the three in §7.6b, **no layer reports this defect.** Tool calls resolve the
current client per call and keep working, and the screen renders. The only thing
stopped is one stream. The decisive signal is that **pressing `Subscribe` does
nothing** — the runtime binding never went away, so from the runtime's point of
view it is already subscribed and there is nothing to do.

This is not limited to the full-screen app: a **composed tile's summary runtime**
watching the same device must be re-attached too, or it is the one surface still
frozen after a resume.

### 7.6d A dropped link is not given up on while the app is open (MUST)

§7.6c fixes what to re-attach once a link is up again. The question before it —
**does it come up at all** — is this one.

A host MUST NOT end reconnection by attempt count. With a cap, an app sitting on
screen reaches a point where nothing happens any more, and the only recovery
left to the user is leaving the app and coming back. Measured case: a phone
hotspot switched off and back on minutes later — the board was back, the host
had already given up.

| rule | |
|---|---|
| no giving up (MUST) | while the app is open (while the host's reconnect monitor runs), retries do not stop |
| detection ≠ retry pacing (MUST) | the observation interval (keepalive, liveness) and the retry interval are **different values**. Fused into one knob, tuning detection to be fast also accelerates retries and burns an attempt cap in seconds |
| an open app does not back off (SHOULD) | an app on screen is the explicit statement that this connection is supposed to exist. Widening the gap buys nothing there. Backoff is for connections **nobody is looking at** |
| retries do not overlap (MUST) | the interval applies **between dials**. A 1s interval on a dial that takes five seconds is one dial every six, not five overlapping ones — and on the single-peer boards of §7.6b overlapping dials refuse each other |
| a dial is bounded (MUST) | an unanswered dial that hangs forever stops the retry loop with it. The host connector carries the deadline |

### 7.6e What came back can be observed (SHOULD)

A timer can only guess *when to look again*. A signal knows. A device that comes
back is **visible before it is connectable** — BLE advertises, an mDNS node
re-announces on rejoin (RFC 6762 §8.3), a USB port reappears in the enumeration.
A host may use these observations as the retry trigger, and then the interval no
longer decides recovery latency.

Three things are normative:

1. **Observation is not a dial.** Receiving an advertisement, running a browse
   window, reading a port list opens no session on the device, so none of it is
   subject to the single-peer constraint of §7.6b. Do not confirm with a probe —
   that is a connection.
2. **Observe only what an open app is waiting on (MUST).** Observing everything
   ever registered *is* an always-on scan, and undoes from behind the reason
   discovery is scoped to a screen's lifetime (§7.3 radio exclusion, power). No
   app waiting, no observation.
3. **Observation accelerates, it does not replace (MUST).** No signal source
   covers every transport — a remote / cloud server is never named by one, and
   **network regain is its only external signal**; an advertisement can be
   missed. The time-based retry must remain as the floor.

A network-regain signal cannot name a server, so it applies to **every failed
connection**.

### 7.7 Where the capability lives

The onboarding capability is **owned by the host**, not by a bundle. Provisioning is the precondition for everything else, so putting it in a bundle creates a loop: you need the network to fetch the bundle, and the bundle to create the network. A bundle-side surface remains valid as a demo or alternate path, but not as the default one.

The reference implementation is AppPlayer Pro (a merged "Discovered" tab plus an onboarding sheet); the capability is the `ble_provisioning` recipe's `BleProvisioningCapability` (`candidates → wifiScan → commission`).

Opening a discovered device as a composition origin is the `composition_host` recipe's `openOrigin` hook. The host opens it at that moment using the transport information in the registration manifest — and **MUST NOT discriminate by transport kind**: a USB board once registered and opened fine standalone while being unreachable from composed screens alone, and there was no reason a user could see.
