# 11 — IO Device Drivers

The device-driver model of the io capability. **This document is the definition (canon), and consumers (driver packages · shared wiring · hosts Studio·AppPlayer) follow it.** No per-host ad-hoc implementation or re-analysis.

---

## 0. Identity

io capability = the capability that exposes **communication/control with external devices/endpoints + local OS execution**, through a single `io.*` control contract (policy-gated device behavior).

- substrate = `mcp_io`'s `IoRuntime` = `DeviceRegistry` + `PolicyEngine` (deny-by-default · plan→commit · rate · interlock) + `AuditTrail` + `JobManager`.
- "device" = the target to read/execute/subscribe via io.*. Whether external (serial instrument · PLC · broker · endpoint) or local (OS process), the same contract.

## 1. `io.*` Surface (Fixed Contract — Driver-Independent)

| Group | Tools |
|---|---|
| Observation | `io.read` · `io.subscribe` · `io.list_devices` · `io.describe_device` |
| Control | `io.execute` · `io.plan_execute` · `io.commit_execute` · `io.emergency_stop` |
| job | `io.cancel_job` · `io.list_jobs` · `io.get_job` |
| Connection lifecycle | `io.connect_device` · `io.disconnect_device` (for on-connect drivers) |

The surface is **fixed** even as drivers grow. Bundle apps know only this surface. Routing = `target = io://<deviceId>/...` → resolve the adapter by deviceId. **Zero new verbs when a driver is added.**

## 2. Driver = adapter (sibling sub-package)

Each device type = a **sibling sub-package** of `mcp_io`, implementing `AdapterBase`, depending only on the published `mcp_io`. **No modification to the `mcp_io` core.**

Current drivers: `process` (mcp_io_process) · `serial` · `modbus` · `mqtt` · `http` · `opcua` · `can` · `scpi` · `websocket`.

## 3. Registration lifecycle (2 modes — do not confuse)

- **boot-register** — host-owned · **target-less** device. The device is the host itself, so there is no connection target (e.g. `process` = OS). At host boot, `registry.registerAdapter` once. Static.
- **on-connect** — **per-connection-target instance**. Drivers (all 8) that require a transport + a connection target (port · host:port · broker · endpoint · baseUri). An instance per connection target, so a static boot list is impossible. Registered when a target is provided at runtime.

## 4. Provisioning (on-connect drivers)

- **builder**: `type → (config params → transport + adapter)`. The builder creates the instance → `registry.registerAdapter` → `discover`.
- **2 triggers**:
  1. `io.connect_device {type, id, params}` — a bundle/app connects a device **at runtime** (app driving). Release via `io.disconnect_device {id}`.
  2. host config list — the host reads settings (user/tenant) and provisions at boot/configuration time. (config format · UI = host surface.)
- Both register into the same `IoRuntime` through the same builder → the same io.* surface.

## 5. Platform Support (per-driver declaration — required)

Each driver **declares its supported platforms**. The host registers only the **subset** that the current platform supports (no blanket "desktop only" guard).

| Driver | transport | mobile | desktop | web |
|---|---|---|---|---|
| modbus(TCP) · mqtt · http · websocket · opcua(TCP) · scpi(TCP) | `dart:io` socket | ✅ | ✅ | ❌ |
| serial | libserialport **FFI** | ❌ | ✅ | ❌ |
| can | **native** | ❌ | ✅ | ❌ |
| process | OS process | ❌ | ✅ | ❌ |

- Network drivers (socket) work on mobile too → AppPlayer mobile also exposes them.
- FFI/native (serial · can) + process = desktop only.
- web = `dart:io` absent → this whole family excluded. (A web-only transport is undefined until separately designed.) Compile-time blocking of platform-incompatible code = conditional import.

## 6. Studio ↔ AppPlayer parity (same bundle)

**The same bundle (.mbd) must work on both Studio and AppPlayer.** ⟹ the io.* surface + driver set/behavior + platform matrix must be **identical on both hosts**.

- Therefore **driver builders + provisioner + connect tools + platform declarations + policy defaults = a single shared wiring** (capability_tools recipe pattern · `publish_to:none` · no modification to the `mcp_io` core). Both hosts **consume identically**.
- Rewriting builders per host = drift = broken parity → **forbidden**.
- The bundle declares its io need via `requires.builtinTools`. The host exposes its own platform subset, and the bundle reads "is this driver available on this platform" by a consistent criterion.

## 7. Security (per-driver)

- **Common**: `PolicyEngine` deny-by-default (action/role matching) · `plan→commit` for risky actions · rate limit · `AuditTrail`.
- **2 per-driver layers**:
  - `process` = execution sandbox: executable allowlist · cwd jail · env allowlist · output cap · timeout · shell off (default).
  - connection drivers = connection ACL: allowed endpoint/target/type · credentials.
- Policy · sandbox · ACL conventions are **self-contained in the driver package or shared wiring**. The core is not polluted.

## 8. Placement (core-no-modification principle)

| What | Placement |
|---|---|
| Driver (adapter + transport) | sibling `mcp_io_*` package |
| Shared wiring (builder · provisioner · connect tools · platform declarations · policy defaults) | shared reference (recipe, `publish_to:none`) |
| `mcp_io` core | **unmodified** (do not put provisioner/wiring in the core — 8+ consumer version cascade) |
| Host (AppPlayer · Studio) | consume shared wiring + register its own platform subset + `process` (boot) + config/UI |

Consumers: AppPlayer (cherry) · Studio (sbuilder) consume the **same shared wiring**.

## 9. Non-Goals

- Merging driver wiring into the `mcp_io` core (version cascade).
- Rewriting builders per host (drift).
- web-native io (until a separate transport design).
- Pre-connecting an unused driver in advance (drivers are available, instances at connect time).

---

Related: `06-tool-registry.md` (registration paths · capability tool packs · parity conventions) · `02-bundle-interface.md` (`requires.builtinTools`) · `05-composition-patterns.md` (ecosystem app embedding).
