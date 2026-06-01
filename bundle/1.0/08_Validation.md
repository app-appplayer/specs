# 08. Validation

The reference validator (`McpBundleValidator`) checks a parsed
`McpBundle` against three independent passes — schema, reference,
integrity — and returns a `ValidationResult` carrying `errors[]` and
`warnings[]`.

References, in the `mcp_bundle` package:
- `lib/src/validator/mcp_bundle_validator.dart`
- `lib/src/validator/validation_result.dart`

## 8.1 Validation Result

```dart
class ValidationResult {
  final bool isValid;
  final List<ValidationError> errors;
  final List<ValidationWarning> warnings;
  final Map<String, dynamic> metadata;
}
```

`isValid == errors.isEmpty`. Warnings never affect validity. Hosts
SHOULD surface warnings to the bundle author / user but MUST NOT
refuse activation on warnings alone.

Each issue carries:

| Field | Description |
|-------|-------------|
| `code` | Stable error code — see [§8.2](#82-error-codes). |
| `message` | Human-readable description. |
| `location` | Dot-path within the bundle (e.g. `manifest.id`, `tools.tools[3].name`). |
| `severity` | `error` / `warning` / `info`. |

## 8.2 Error Codes

| Code | Meaning | Pass |
|------|---------|------|
| `MISSING_REQUIRED` | A required field is absent or empty. | schema |
| `INVALID_TYPE` | Field's runtime type does not match its declared type. | schema |
| `INVALID_PATTERN` | String fails its regex (e.g. `id` not matching reverse-DNS). | schema |
| `INVALID_VALUE` | Value out of bounds or outside an enum. | schema |
| `UNKNOWN_REFERENCE` | An id-reference does not resolve (e.g. agent's `skillIds[i]` not in `skills.modules[*].id`). | reference |
| `CIRCULAR_REFERENCE` | A reference graph contains a cycle. | reference |
| `DUPLICATE_ID` | Two entries in a section share the same `id` / `name`. | schema or reference |
| `HASH_MISMATCH` | A computed content hash differs from the declared `integrity.contentHash.value`. | integrity |
| `SIGNATURE_INVALID` | A signature in `integrity.signatures[]` fails verification. | integrity |
| `SIGNATURE_EXPIRED` | A signature's expiry timestamp is in the past. | integrity |

Reference: `McpValidationCodes` in `mcp_bundle_validator.dart:31`.

The host MAY add host-specific codes; spec-defined codes MUST keep
these meanings.

## 8.3 Schema Pass

`_validateSchema` walks the bundle and emits `MISSING_REQUIRED`,
`INVALID_TYPE`, `INVALID_PATTERN`, `INVALID_VALUE`, `DUPLICATE_ID`
for every section that the bundle carries.

Per-section minimums:

| Section | Required checks |
|---------|----------------|
| Manifest | `id` non-empty + reverse-DNS pattern; `name` non-empty; `version` semver. |
| `ui` (typed inline) | `pages[*].id` non-empty + unique. |
| `skills` | Module `id` non-empty + unique within `modules[]`. |
| `assets` | `assets[*].path` non-empty + unique. |
| `policies` | Policy `id` unique. |
| `factGraphSchema` | Type names unique within their category. |
| `flow` | Flow `id` unique; step `id` unique within a flow. |
| `knowledge` | `sources[*].id` unique. |
| `bindings` | Binding `id` unique. |
| `tests` | Suite `id` unique. |
| `requires` | Items must be strings. |
| `tools` | Entry `name` non-empty + unique within `tools[]`; `kind` not `unknown` (lenient — see [`04_Tools.md`](04_Tools.md) §4.2). |
| `facts` | Required SPO fields non-empty (`subject` / `predicate`); `id` unique when present. |
| `workflows` / `pipelines` / `runbooks` | Entry `id` non-empty + unique. |

Sections not listed above pass schema validation when they parse
without throwing — the validator does not enforce shape on
free-form entries (steps, stages, procedures).

## 8.4 Reference Pass

`_validateReferences` walks cross-section id references:

| Validation target | Checked against |
|-------------------|-----------------|
| `agents.agents[*].profileIds[i]` | `profiles.profiles[*].id` |
| `agents.agents[*].skillIds[i]` | `skills.modules[*].id` |
| `agents.agents[*].factSourceIds[i]` | `knowledge.sources[*].id` and `facts.facts[*].id` |
| `agents.agents[*].philosophyIds[i]` | `philosophy.philosophies[*].id` |
| `skills.modules[*].knowledgeSources[i].sourceId` | `knowledge.sources[*].id` |
| `manifest.wiring.*.tool` and `manifest.chat.slashCommands[*].tool` | `tools.tools[*].name` (host MAY also resolve against builtin registry) |

Unresolved references emit **warnings** (`UNKNOWN_REFERENCE`),
NOT errors. Bundles ship with cross-bundle references that resolve
only at host activation time, so the validator can not be strict.

`CIRCULAR_REFERENCE` errors are emitted when a self-referencing
graph is detected (e.g. an agent listing itself in its own scope, a
profile extending itself transitively).

## 8.5 Integrity Pass

`_validateIntegrity` runs only when `bundle.integrity != null`.
Its checks:

1. If `contentHash` is declared, recompute the hash according to
   `contentHash.scope` and compare against `contentHash.value`.
   Mismatch → `HASH_MISMATCH`.
2. If `files[]` is declared, recompute each file's hash and compare.
3. If `signatures[]` is declared, verify each against the trust
   store / public key the validator was given. Failure →
   `SIGNATURE_INVALID`.
4. If a signature carries `expiry` in the past → `SIGNATURE_EXPIRED`.

A bundle without `integrity` skips this pass — `validateIntegrity`
returns valid. The installer's `InstallPolicy.requireIntegrity` may
make integrity mandatory at install time; see [`10_Distribution.md`](10_Distribution.md) §10.3.

## 8.6 Composing Passes

```dart
final r = McpBundleValidator.validate(bundle);
if (!r.isValid) {
  for (final e in r.errors) {
    print('${e.code} at ${e.location}: ${e.message}');
  }
}
```

The full `validate(bundle)` runs all three passes. Hosts may run
individual passes:

- `validateSchema(bundle)` — at parse time.
- `validateReferences(bundle)` — at cross-section integration.
- `validateIntegrity(bundle)` — at install / update.

`ValidationResult.merge` combines results from multiple bundles or
multiple passes.

### 8.6.1 Partial-Entry Validation

Whole-bundle validation operates on a parsed `McpBundle`. Authoring
tools that mutate a single section (e.g. inserting one new
`knowledge.sources[i]`, one new `agents.agents[i]`) need a narrower
pass that returns issues for **a single entry map**, without
requiring the rest of the bundle to be in a parseable state.

An implementation MUST expose a partial-entry validator surface for
every knowledge-category entry type the spec defines:

| Entry path | Required-on-insert |
|------------|--------------------|
| `knowledge.sources[i]` | `id`, `name` non-empty. `type` MUST be an `KnowledgeSourceType`. |
| `knowledge.sources[i].documents[j]` (and `knowledge.documents[i]`) | at least one of `id` or `path` non-empty. |
| `agents.agents[i]` | `id`, `name`, `role` non-empty. |
| `facts.facts[i]` | `id` non-empty. |
| `skills.skills[i]` (canonical) / `.modules[i]` (legacy alias) | `id`, `name` non-empty. |
| `profiles.profiles[i]` | `id`, `name` non-empty. |
| `philosophy.philosophies[i]` | `id`, `name` non-empty. |
| `workflows.workflows[i]` | `id`, `name` non-empty. |
| `pipelines.pipelines[i]` | `id`, `name` non-empty. |
| `runbooks.runbooks[i]` | `id`, `name` non-empty. |

The validator returns `null` when the entry is valid and a
`List<String>` of issues otherwise. The implementation MUST run two
layered checks:

1. **Explicit required-field check** — the entry map MUST carry every
   field listed above, non-null and non-empty. Section `fromJson`
   factories typically coerce missing identifiers to `''`; the
   explicit pass closes that silent-fallback gap so empty ids are
   rejected at authoring time rather than surfacing at the next
   parse.
2. **Smoke parse** — run the entry's `fromJson` factory. Caught
   exceptions surface as a single `'shape: <message>'` issue covering
   hard structural errors (wrong type, missing nested object) that
   the explicit pass does not enumerate.

Cross-reference checks (e.g. agent's `skillIds[i]` resolving to an
existing skill) are out of scope for partial-entry validation — they
remain in the whole-bundle reference pass per [§8.4](#84-reference-pass)
because the entry alone does not carry the surrounding context.

Reference: `PartialValidators` in the `mcp_bundle` package
(`lib/src/validator/partial_validators.dart`).

## 8.7 Author Workflow

A bundle's reference validation lifecycle:

```
authoring tool                       host loader               installer
─────────────                         ───────────               ─────────
load + mutate + write
  partial-entry pass (per insert) →
  write manifest.json (mutate)    →
                                    parse  →  schema pass    →
                                              reference pass →
                                                                       install:
                                                                         integrity pass
                                                                         requires gate
                                                                         compatibility gate
                                                                         activation
```

Authoring tools SHOULD run [§8.6.1](#861-partial-entry-validation)
before each entry insert and surface issues to the LLM / UI as a
diagnostic so invalid entries never reach disk. They SHOULD also run
the whole-bundle three-pass validation on save and surface
errors / warnings inline. The host loader SHOULD run schema +
reference at activation. The installer MUST run integrity per
[`10_Distribution.md`](10_Distribution.md).

## 8.8 Authoring Transaction

Multiple authoring surfaces may mutate the same `.mbd/` concurrently
— a UI editor and a chat-driven LLM agent in the same host process,
or an external LLM connected via MCP from a different process. A
naive `load → mutate → write` pattern is last-write-wins: the
slower writer overwrites the faster writer's changes without any
diagnostic. Pre-insertion validation (§8.6.1) closes the shape-error
window but cannot protect against this concurrent-edit window.

Implementations exposing a `.mbd/` mutate API SHOULD wrap the
`load → mutate → write` cycle in three layered guards:

1. **In-process FIFO mutex** (REQUIRED) — keyed on the absolute
   `mbdPath`. Same-isolate concurrent calls serialize. Eliminates
   the race between two coroutines in one process. Without this
   guard, even single-process authoring under async fan-out
   (concurrent slash-commands, parallel LLM tool calls) reorders
   non-deterministically.
2. **OS file lock** (RECOMMENDED, opt-in) — advisory lock on a
   hidden sentinel under the bundle directory (e.g. `.mbd-lock`).
   Required when other processes might mutate the same bundle (host
   shell GUI + headless mcp-server + external LLM client). Off by
   default so the common single-process path avoids the syscall
   overhead.
3. **Optimistic checksum** (REQUIRED) — capture a content hash
   (sha256 over `manifest.json` bytes) at the load step, verify
   against disk immediately before the commit write. A mismatch
   means another writer slipped through despite the locks (e.g.
   filesystem without advisory lock support, or `useFileLock=false`
   when a peer mutator did acquire the lock). Surface the mismatch
   as a conflict diagnostic so the caller can retry from a fresh
   load.

The mutate API SHOULD return a discriminated reason on guard failure
so callers can distinguish race from application errors:

| Reason | Meaning | Caller response |
|--------|---------|-----------------|
| `timeout` | In-process mutex held past the caller's timeout. | Retry after a backoff or surface a busy diagnostic. |
| `lockFailed` | OS file lock unavailable. | Retry, escalate to user, or fall back to checksum-only mode. |
| `conflict` | Checksum mismatch at commit time. | Reload, re-apply the mutation, retry. |

A mutation closure that throws MUST abort the transaction before the
write — disk stays untouched. A closure that signals a no-op (e.g.
returning a null `updated` bundle) MUST skip the write entirely
while still observing the guard scope (read-only inspection under
the mutex is a valid use case).

Mutation surfaces that bypass `McpBundleWriter` (e.g. authoring tool
shells reading `manifest.json` directly with `jsonDecode` and
writing back with `jsonEncode`) MUST NOT be used in tools that
target the spec — they bypass the mutex, the checksum, and the
partial-entry pre-insertion validation, and historically have
been the source of every observed authoring-layer race.

Reference: `McpBundleMutator` /
`BundleMutationException` / `BundleMutationReason` in the `mcp_bundle`
package (`lib/src/io/bundle_mutator.dart`).

## 8.9 Conformance

A validator implementing this spec MUST:

1. Use the codes in [§8.2](#82-error-codes) with the listed
   meanings.
2. Set `isValid` based solely on `errors.isEmpty`.
3. Never raise `error` for unresolved cross-bundle references —
   those are `warning` only.
4. Re-compute and compare integrity hashes deterministically (same
   inputs → same hash).

A validator MAY:

- Add additional codes for host-specific checks.
- Skip the reference pass for performance during partial loads.
- Promote a warning to an error under a strict-mode flag.

A bundle MAY ship that produces validator warnings — warnings are
informational. A bundle that produces errors is malformed and SHOULD
be refused by hosts at activation.
