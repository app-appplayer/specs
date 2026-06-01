# 10. Distribution: Pack, Sign, Install

This document defines how a bundle moves from authoring to a host's
install root: pack `.mbd/` → optional sign → install (with policy
gates).

References, in the `mcp_bundle` package:
- `lib/src/install/mcp_bundle_packer.dart`
- `lib/src/install/bundle_signer.dart`
- `lib/src/install/mcp_bundle_installer.dart`
- `lib/src/install/install_policy.dart`
- `lib/src/install/installed_bundle.dart`
- `lib/src/install/runtime_descriptor.dart`
- `lib/src/install/trust_store.dart`
- `lib/src/models/integrity.dart`

## 10.1 The Pack Step

`McpBundlePacker.packDirectory(mbdPath, {computeIntegrity, signer})`
turns an `.mbd/` working tree into deterministic `.mcpb` bytes.

### 10.1.1 Determinism Rules

Already covered in [`07_Resources.md`](07_Resources.md) §7.4.1 —
recap:

- `manifest.json` is the first archive entry.
- All other entries are sorted lexicographically.
- `lastModTime`, `mode`, `ownerId`, `groupId` are fixed.
- Paths use forward slash.
- `manifest.json` is canonical-JSON-encoded.

These rules together guarantee `pack(mbd) == pack(mbd)` byte-for-byte
across hosts and across runs — the precondition for content-hash
integrity.

### 10.1.2 Integrity Auto-Compute

When `computeIntegrity` is true (default) or a `signer` is supplied,
the packer:

1. Loads `manifest.json` and parses the `McpBundle`.
2. Determines the integrity scope from the existing
   `bundle.integrity.contentHash.scope` (default
   `ContentScope.canonicalJson`).
3. Determines the algorithm from
   `bundle.integrity.contentHash.algorithm` (default
   `HashAlgorithm.sha256`).
4. Computes the hash over the payload-for-scope:
   - `canonicalJson` — canonical JSON bytes of the bundle map.
   - other scopes — host extension.
5. Updates `bundle.integrity.contentHash` with the computed value.
6. Optionally appends a signature (next section).
7. Writes the updated `manifest.json` into the archive.

A bundle authored without an `integrity` block gets one synthesized
during pack. A bundle with a pre-existing `integrity.contentHash` has
its `value` recomputed (the original `value` is overwritten — the
declared `scope` and `algorithm` are kept).

## 10.2 Signing

### 10.2.1 `BundleSigner`

```dart
abstract class BundleSigner {
  String get keyId;
  SignatureAlgorithm get algorithm;
  PayloadRefType get payloadRefType;
  Future<Signature> sign(Uint8List payload);
}
```

Reference: `bundle_signer.dart`.

The signer is **caller-supplied** — `mcp_bundle` does not manage keys.
A host that wants to sign supplies a `BundleSigner` implementation
backed by its key store (OS keychain, HSM, file-based keys, ...).

### 10.2.2 Signature Block

The signer's `sign(payload)` returns a `Signature`:

```json
{
  "keyId": "release-2026",
  "algorithm": "ed25519",
  "value": "<base64>",
  "payloadRef": { "type": "contentHash" },
  "signedAt": "2026-05-15T08:00:00Z",
  "expiry": null
}
```

Fields:

| Field | Description |
|-------|-------------|
| `keyId` | Identifier matched against `TrustStore.lookup(keyId)` at install. |
| `algorithm` | `ed25519` / `rsa-pss-2048-sha256` / ... — from `SignatureAlgorithm` enum. |
| `value` | Base64-encoded signature bytes. |
| `payloadRef` | What the signature covers — `{type: "contentHash"}` (canonical default), `{type: "fullArchive"}`, or `{type: "fileHash", path: "..."}`. |
| `signedAt` | ISO-8601 timestamp. |
| `expiry` | Optional expiry timestamp. |

A bundle MAY carry zero or many signatures in
`integrity.signatures[]`. The installer's `requireSignature` policy
flag controls whether at least one verified signature is mandatory.

## 10.3 `InstallPolicy`

The installer's behavior is controlled by `InstallPolicy`:

```dart
class InstallPolicy {
  const InstallPolicy({
    this.onConflict       = InstallConflictPolicy.replace,
    this.requireIntegrity = true,
    this.requireSignature = false,
    this.limits           = const InstallLimits(),
  });
}
```

### 10.3.1 `InstallConflictPolicy`

| Value | Behavior |
|-------|----------|
| `failIfExists` | Throw `BundleAlreadyInstalledException` when `<installRoot>/<id>/` exists. |
| `replace` (default) | Atomic swap: stage → rename existing aside → rename new into place → delete the displaced tree. |
| `skipIfExists` | Leave the existing install alone and return its metadata. |

### 10.3.2 `requireIntegrity`

When `true` (default), an install without a parseable
`IntegrityConfig` is rejected. When `false`, integrity is verified
only when declared.

### 10.3.3 `requireSignature`

When `true`, at least one signature in `integrity.signatures[]` MUST
verify against the trust store. When `false`, signatures are not
required (but if present, they are still verified — failing
signatures cause install rejection regardless).

### 10.3.4 `InstallLimits`

ZIP / extraction caps to defend against malformed or hostile inputs:

| Field | Default |
|-------|---------|
| `maxCompressedBytes` | 64 MiB |
| `maxUncompressedBytes` | 512 MiB |
| `maxCompressionRatio` | 100 (per entry) |
| `maxEntryCount` | 10000 |
| `maxEntryPathLength` | 1024 chars |
| `maxPathDepth` | 32 segments |

Hosts MAY tighten these — they MUST NOT loosen them past the listed
defaults without explicit operator opt-in.

## 10.4 `TrustStore`

```dart
abstract class TrustStore {
  TrustedPublicKey? lookup(String keyId);
  bool isRevoked(String keyId);
}
```

Reference: `trust_store.dart`.

`mcp_bundle` never persists or fetches keys. Hosts inject a populated
`TrustStore` per install call and back it with bundled trust roots,
OS keychains, TOFU pinning, or remote revocation lists.

Built-in implementations:

- `EmptyTrustStore` — rejects every signature (default).
- `InMemoryTrustStore` — useful for tests and small hosts.

When `requireSignature == true` and the trust store returns `null` for
every signer, the install MUST fail.

## 10.5 `RuntimeDescriptor` and `CompatibilityConfig`

Hosts declare their runtime via `RuntimeDescriptor`:

```dart
class RuntimeDescriptor {
  const RuntimeDescriptor({
    required this.version,
    this.features = const <String>{},
  });
  final String version;
  final Set<String> features;
}
```

Bundles declare their requirements via `CompatibilityConfig` at
`McpBundle.compatibility`:

| Field | Description |
|-------|-------------|
| `schemaVersion` | Required bundle schema version. |
| `requirements` | Generic package/version map. |
| `minRuntimeVersion` / `maxRuntimeVersion` | Host runtime range. |
| `requiredFeatures` | Host capability flags that MUST be present. |
| `incompatibleWith` | Bundle ids that MUST NOT be installed alongside. |

The installer compares `RuntimeDescriptor` against
`CompatibilityConfig`:

- `version` must satisfy `min ≤ version ≤ max` when bounds are
  declared.
- Every entry in `requiredFeatures` must be in
  `runtime.features`.

Rejection codes follow the [`08_Validation.md`](08_Validation.md)
patterns; failures are wrapped as `BundleIncompatibleException`.

## 10.6 The Install Flow

```
caller                    installer                  filesystem
──────                    ─────────                  ──────────
installBytes(bytes)  →
                  decode ZIP (limits enforced)
                  parse manifest
                  validate compatibility (descriptor)
                  validate integrity (hashes, signatures)
                  validate requires (atoms / tools)
                  stage to <installRoot>/.staging/<id>/
                                                    write extracted files
                  acquire <installRoot>/.lock
                  swap or skip per onConflict
                  release lock
                                                    rename into place
                  write .install.json sidecar
                                                    write sidecar
                  return InstalledBundle
```

The installer entry points:

| Entry | Source |
|-------|--------|
| `installBytes(bytes, ...)` | Raw `.mcpb` bytes. |
| `installFile(path, ...)` | `.mcpb` file path. |
| `installDirectory(mbdPath, ...)` | `.mbd/` directory; packs in memory first via `McpBundlePacker.packDirectory`. |
| `installUrl(uri, ...)` | HTTP(S) GET; honors `InstallLimits.maxCompressedBytes` against `Content-Length` pre-check. |

All entries return an `InstalledBundle` describing where the bundle
landed and its parsed manifest.

## 10.7 `InstalledBundle` and `.install.json`

Each install directory carries an `.install.json` sidecar:

```json
{
  "registrySchema": "1.0.0",
  "id": "com.example.notes",
  "version": "0.4.0",
  "installedAt": "2026-05-15T08:00:00Z",
  "source": "https://...",
  "verifiedSignatures": ["release-2026"],
  "policy": {
    "requireIntegrity": true,
    "requireSignature": false
  }
}
```

Hosts use the sidecar to reconstruct install metadata without
re-parsing the bundle. The exact set of fields is host-extensible —
the installer only requires `registrySchema`, `id`, `version`.

## 10.8 Uninstall

The installer also exposes:

- `list(installRoot)` — enumerate installed bundles.
- `get(installRoot, id)` — read one `InstalledBundle`.
- `uninstall(installRoot, id)` — delete `<installRoot>/<id>/`.

Uninstall is best-effort recursive deletion. Hosts that want to
preserve user data (config, caches) outside the bundle directory
SHOULD store it in a separate location.

## 10.9 Conformance

A pack implementation MUST:

1. Produce byte-deterministic output per the rules in
   [§10.1.1](#1011-determinism-rules).
2. When `computeIntegrity` is true, populate
   `manifest.integrity.contentHash` based on the declared scope.

A signer MUST:

1. Produce a valid `Signature` with `keyId`, `algorithm`,
   base64-encoded `value`, and an `ISO-8601` `signedAt`.

An installer MUST:

1. Enforce `InstallLimits` before extracting any entry.
2. Reject on path traversal (`..`) or absolute entry paths inside
   the archive.
3. Verify integrity per `requireIntegrity` and signatures per
   `requireSignature`.
4. Apply the `requires` portability gate (atoms / builtin tools)
   against the `RuntimeDescriptor.features` set.
5. Honor `InstallConflictPolicy` deterministically.

A trust store MUST:

1. Return `null` from `lookup(keyId)` when the key is unknown.
2. Return `true` from `isRevoked(keyId)` when revocation is recorded
   — the installer treats revoked keys as untrusted.
