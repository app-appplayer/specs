# 16 — BLE extension transport (GATT binding standard)

> status: **draft** (added 2026-07-14 · to be confirmed after validation against the ESP32 reference)
> parent: [`08-extension.md`](08-extension.md) §4 (extension transport 2-tier · seam/impl principle)
> companion: `mcp_serving/1.0` (serving addresses · entry flow) · the embedded wire contract (newline-delimited JSON-RPC)

The canonical wire binding that lets a small chip with nothing but BLE (ESP32 · STM32WB · nRF …) join the ecosystem as an **MCP server** (UI serving + tool exposure), and lets a client (AppPlayer mobile/desktop) **consume it with the standard mcp_client, unchanged**. The goal is to prevent N firmware dialects: this one document is enough for any firmware and any client to interoperate with no configuration.

---

## 1. Principle — GATT is a byte pipe, and the MCP layer is transport-blind

BLE has no general-purpose byte-stream service the way TCP does. For a bidirectional session there are really two candidates:

| path | verdict | rationale |
|---|---|---|
| **GATT** (attribute) | **adopted (baseline)** | Supported by every stack and OS (iOS/Android/ESP32/nRF/STM32WB). The de-facto pattern for serial-over-BLE (NUS shape: write char + notify char). |
| L2CAP CoC | appendix (v2 candidate) | A real byte stream, but — iOS requires the PSM to be **published as a GATT characteristic** to be discoverable (so GATT is unavoidable anyway), small-chip SDK support varies, and Android needs API 29+. |
| advertising-only | not viable | ~31 B payload, no session. |

So **GATT cannot be avoided.** This standard keeps its use minimal and fixed — one service and two characteristics used purely as a byte pipe, with the existing wire contract (newline-delimited JSON-RPC) riding on top unchanged. The MCP layer (server dispatch, client) does not know it is on BLE.

## 2. Fixed roles

- **GATT peripheral = MCP server** (the board: advertise + serve)
- **GATT central = MCP client** (AppPlayer / probe)

The general case that a bus role is not an MCP role is covered in 08 §4, but this binding **fixes** v1 to the combination above — the only practical one for embedded serving.

## 3. GATT service definition (fixed UUIDs)

| item | UUID | GATT property | direction · meaning |
|---|---|---|---|
| **MCP Serving service** | `4D435042-4C45-0001-8000-6D6370626C65` | primary service | — |
| **RX characteristic** | `4D435042-4C45-0002-8000-6D6370626C65` | Write **and** Write Without Response | central → peripheral bytes |
| **TX characteristic** | `4D435042-4C45-0003-8000-6D6370626C65` | Notify (CCCD required) | peripheral → central bytes |

- The UUIDs are fixed constants (`4D435042-4C45` = ASCII `MCPB-LE`, node `6D6370626C65` = `mcpble`). Both firmware and client may hardcode them — they are not a configuration item.
- The server MUST accept **both write modes** on RX. The client may use either (Write Without Response gives better throughput).
- The server MUST NOT send any bytes before the TX notify subscription (CCCD write).

## 4. Byte-stream reassembly (chunking)

- **Stream definition**: the TX notify payloads concatenated in arrival order are the server→client byte stream. The RX write payloads concatenated in arrival order are the client→server stream.
- **Chunk boundaries carry no meaning** (MUST NOT carry semantics) — one JSON-RPC line spanning several chunks, or several lines arriving in one chunk, are equally valid.
- Chunk size ≤ ATT_MTU − 3.
- **MTU**: the central SHOULD attempt to negotiate MTU ≥ 247 immediately after connecting. The server MUST work at the **default MTU of 23** — chunking absorbs it.

## 5. Framing (the layer above the stream — the existing contract, unchanged)

Above the reassembled byte stream the embedded wire contract applies as-is:

- **newline-delimited JSON-RPC 2.0** — UTF-8, `\n` terminates a message, blank lines skipped, leading and trailing whitespace trimmed.
- The canonical client-side implementation is `mcp_bridge`'s `ByteStreamFramer` (`encodeFrame = jsonEncode + '\n'`) — the same framer used for tcp and serial is used for BLE unchanged.
- **Minimum accepted line length**: the server MUST accept ≥ 512 B and SHOULD accept ≥ 4 KiB (sized for a `ui://app` DSL serving response — the response is produced by the server, so this is in practice a receive-buffer constraint, and a TINY profile can still accept `tools/call` with a 512 B receive buffer). On overflow, discard that line and answer with JSON-RPC error `-32600` when the id could be parsed.

The method set, the serving addresses (`ui://app` · `bundle://manifest.json`) and the tool contract are out of scope here — they follow mcp_serving/1.0 and the embedded conformance contract.

## 6. Advertising · discovery

- The server MUST include the **service UUID in the advertising PDU or the scan response** — client discovery is a single scan filter on that UUID.
- The local name is free (SHOULD be product-identifying, e.g. `mcp-<model>`).
- **Security in v1 is open**: pairing and bonding MUST NOT be required by default. A deployment MAY require bonding or encryption — that is deployment policy, outside this contract.

## 7. Conformance

Checklist a firmware has to pass:

1. §3 service and characteristic UUIDs and properties match exactly; CCCD works.
2. Chunking works at the default MTU of 23, and MTU negotiation is accepted.
3. Arbitrary chunk splitting is accepted on receive (one line split across several writes / several lines in one write).
4. `initialize → tools/list → tools/call → resources/list → resources/read ui://app → bundle://manifest.json` completes.
5. Notifications (no id) get no response; unsupported methods return `-32601`.

**The probe is the standard mcp_client completing that run** — no separate verifier, the same principle as serial: connect as central, complete the four steps above, and the firmware conforms. Alignment with existing assets:

- `mcp_bridge` ble client (bluez · Linux) — put the §3 UUIDs in its config and it *is* the probe (the write-char + notify-char + newline-framing structure is confirmed to match).
- The mobile/macOS client is the ② transport layer of 08 §4, a **Flutter BLE `ClientTransport`**, **placed by recipe** (vendored, wrapping flutter_blue_plus — being a Flutter dependency it goes into neither `mcp_bridge` nor `flutter_mcp`; Flutter hosts such as AppPlayer and Studio vendor it in common). It follows this document as written, and the ① seam (`connectExtension`) and ③ tool (`mcp.connect_extension`) are unchanged.

## 8. Reference targets

- **First reference is ESP32 (NimBLE)** — reuse the `embedded/mcp_server` C base and swap only the transport seam for this binding. This draft is confirmed once that passes.
- STM32WB / nRF and others = the specification plus LLM codegen (the no-per-target-maintained-code principle, D2).

## Appendix A — L2CAP CoC fast path (v2 candidate)

For profiles where throughput matters. Publish the PSM as an additional characteristic of this service (the iOS discovery path) → open an L2CAP CoC → from there the stream and framing are identical to §4 and §5. The GATT pipe stays as the fallback. Out of scope for v1.
