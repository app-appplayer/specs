# 14 — Assets & Credentials

The model for describing operational assets (db · homepage · API · repo · deploy · MCP …) under one shape, keeping their secrets (credentials) in an OS-keychain vault separate from the asset description, and migrating them between machines. **This document is the definition (canon), and consumers (bundle authoring · hosts Studio·AppPlayer · recipe wiring) follow it.** Per-host ad-hoc convention forbidden.

The asset model is a **different layer** from `secure.*` (at-rest seal/open, `06-tool-registry.md`): an asset describes *what · where · by which means* it is operated, while a credential is the secret used to reach that asset.

---

## 0. Identity

- **asset** = a ref to an operational target. A bundle/fact carries only the asset's *description* (kind · location · operating means · credential key) — **never the secret body**.
- **credential vault (`secret.*`)** = a `credentialRef → secret` keyed store (OS keychain). No plaintext get.
- **migration** = the keychain key cannot leave the machine, so credentials move as a passphrase-sealed portable blob.

### Boundary with secure (do not confuse)

| | `secure.*` (06) | `secret.*` (this doc) |
|---|---|---|
| Model | stateless at-rest seal/open | `credentialRef → secret` keyed vault |
| Key | machine-keychain symmetric key (non-portable) | (vault) machine keychain · (migration) passphrase-derived |
| Use | sealing bytes (cookies · profiles) | storing · resolving · migrating asset credentials |

## 1. asset Fact Convention (the model)

Schema unchanged — a convention over the existing knowledge fact. `category: "asset"` + `key=<assetId>` + `value=<title>` + `metadata`:

| field | values | meaning |
|---|---|---|
| `kind` | `db`·`file`·`code`·`repo`·`data`·`homepage`·`deploy`·`api`·`mcp` | asset kind |
| `location` | `local`·`embedded`·`external` | location (an attribute only — does not partition what is managed) |
| `locator` | `path`·`dsn`·`url`·`bundle://`·`git@` | address |
| `capability` | `db`·`fs`·`io`·`browser`·`mcp` | operating means (which capability handles it) |
| `credentialRef` | vault key | credential indirection (**secret body is never here**) |

Intent: internal and external assets under one model, with location as an attribute. A market-shared bundle leaks only the `credentialRef`; the secret body never leaves.

### 1.1 `fs` locator = project-portable

An `fs` asset `locator` is **legitimate both inside and outside** the project. An asset may be an external folder the project **operates on**, or a file the project owns. Portability normalization applies **only to links inside the project**:

- A locator **inside the project** → stored **relative** to the project root (the active workspace bundle) → survives a folder rename · copy · move (self-contained and portable).
- A locator **outside the project** (a target of work, a remote, …) → **kept absolute as-is**. Relativizing something external means nothing; it is not a candidate for it.

- **Storing (normalization)**: on asset registration (`knowledge_fact_save category:"asset"`), if the `fs` locator is **inside the project** the host normalizes it relative to the active workspace bundle directory (`<projectRoot>/<wsId_>.mbd`, the host tab's `currentProject`). Absolute paths outside the project, `http(s)://`, `bundle://` and the like stay **as-is** as external refs. (Export / vendoring is only for when an external target should be brought into the project — a choice, not a portability requirement.)
- **Reading (resolve)**: `asset_open` reads through the host's `studio.fs.*` capability, and `fs.*` resolves a relative locator against the **current** active project root (`registerFsTools(activeProjectRoot:)`). It works unchanged from wherever the project was moved to.
- **Anchor-identity invariant**: the store-side relativization anchor = the read-side resolve anchor = the host tab's `currentProject` (= `wsBundleDir(projectRoot, wsId)`). Break this single anchor and assets stop opening after a rename.
- This resolve lives in the host fs layer (above), so **bundle apps inherit it automatically** — an individual builtin does not reimplement it.

## 2. `secret.*` Keyed Vault (fixed surface)

| namespace | tools |
|---|---|
| `secret.*` | `secret.set` · `secret.exists` · `secret.remove` · `secret.list` |

- **No plaintext `get`** (vault discipline). A secret is used only when a capability resolves it internally — never returned to a tool · UI · agent. `list` returns keys only.
- Storage = `appplayer_secure`'s `SecureStorage` (production = OS keychain), namespace `appplayer.credentials` (isolated from the package's identity / at-rest keys).

## 3. `credentialRef` Indirection

asset fact `metadata.credentialRef` → vault key → (when a capability operates the asset) `secret` resolve. Separating the asset description from the secret = zero secret leakage when a bundle/fact is shared.

## 4. Credential Migration

Moving credentials between machines:

1. The operator's **passphrase** seals the `{ref: secret}` map → a portable opaque blob.
2. On the target machine, the same passphrase opens it → vault write.

- crypto = `appplayer_secure`'s `PassphraseSealer` (PBKDF2-HMAC-SHA256 → AEAD, salt · nonce embedded, **key derived purely from the passphrase** → machine-independent). Contrast with the machine-keychain key of `secure.*`/`AtRestSealer`.
- core = `recipes/secure_capability/`'s `CredentialMigrator` (`seal(refs, passphrase)` / `restore(blob, passphrase)`). **Deciding which refs to migrate (collecting asset facts) is the host's job**; the recipe does the vault read/seal · open/write core only.

## 5. Security Discipline

- The secret body is **never in a bundle · fact · migration-blob plaintext** (the blob is AEAD ciphertext).
- Tools never return a secret in plaintext (no get; import does not echo plaintext).
- A wrong passphrase / tampering → AEAD authentication failure (`SecError(aeadAuthenticationFailed)`), writing nothing.

## 6. Wiring (host adoption)

Like io · datastore, it follows the single reference `recipes/secure_capability/` — `secret_example` (`secretCapabilityTools(SecureStorage)`) + `credential_migration` (`CredentialMigrator`) + `secure_example`. The host (AppPlayer Pro/X/Custom · Studio) vendor-copies it (as with `io_drivers`) and registers `registerCapabilityTools(registry, 'secret', secretCapabilityTools(store))` + wraps migration tools over `CredentialMigrator`. Asset-fact collection · UI are the host's. Registration contract = `06-tool-registry.md`.
