# 07. Security

**Status:** Normative.
**Conformance:** [`18_Conformance.md`](18_Conformance.md) §18.2.8.
This section defines the security model of the MCP UI DSL: what runtimes MUST guard against, what they MUST reject, and what boundaries they MUST enforce. Permission declarations and the permission prompt system are described in [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.4; this document covers the runtime-enforcement layer.

---

## 7.1 Expression Sandbox

Binding expressions (`{{...}}`) MUST be evaluated in a sandbox with the following guarantees:

- **No host-runtime access.** Expressions MUST NOT reach the file system, network, DOM, platform channels, process APIs, or any implementation-language globals (e.g., `window`, `document`, `dart:io`, `System`).
- **No dynamic code loading.** `eval`, `Function(...)`, dynamic imports, and equivalent constructs MUST be disabled.
- **Bounded execution time.** Each expression evaluation has a time budget. The default SHOULD be 50ms; runtimes MAY configure this.
- **Bounded memory and recursion.** Runtimes MUST enforce a maximum call-stack depth and a maximum iteration count for loop-like operations (see [`03_Data_Binding.md`](03_Data_Binding.md)). The defaults SHOULD be depth ≤ 10 and iterations ≤ 1000.
- **Read-only state access.** Expressions MUST NOT mutate state. State mutation occurs only through `state` actions.

A sandbox violation MUST surface as an evaluation error returning `null` and logging a diagnostic, not as a runtime crash.

### 7.1.1 Allowed Globals

A sandboxed expression MAY reference a documented allowlist of pure helpers (e.g., math functions, date parsing). Runtimes MUST NOT expose implementation globals by default.

```json
{
  "security": {
    "expressions": {
      "allowedGlobals": ["Math", "Date", "JSON"],
      "timeout": 50,
      "maxDepth": 10,
      "maxIterations": 1000
    }
  }
}
```

---

## 7.2 Input Validation

- **Strict JSON parsing.** Incoming definitions MUST be parsed with a strict parser. Trailing commas, comments, and unquoted keys MUST be rejected.
- **Type-name validation.** Widget `type` strings and action `type` strings MUST be validated against the canonical list plus legacy aliases registered in [`17_Naming.md`](17_Naming.md). Unknown `type` values MUST render an error placeholder; they MUST NOT execute code or fall through to an implementation-language reflection path.
- **Schema validation.** Required fields (§ Core Profile) absent from an incoming object MUST produce a parse error scoped to that node, not a silent fallback.
- **Value validation.** Declared field types (string, number, enum) MUST be validated before use. Invalid enum values MUST be replaced with the documented default and logged, not cause a crash.

### 7.2.1 User-Input Validation

Interactive widgets (`textInput`, `numberField`, `form`, etc.) MAY declare a `validation` block. Runtimes MUST apply the declared constraints before binding the value into state.

`validation` accepts two shapes:

**Shape A — Constraint object** (sanitization-oriented, single constraint set):

```json
{
  "type": "textInput",
  "validation": {
    "kind": "email",
    "sanitize": true,
    "maxLength": 255,
    "pattern": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
    "message": "Enter a valid email"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `kind` | enum | `text`, `number`, `email`, `url`, `phone`, `date`. |
| `sanitize` | boolean | Whether to strip or normalize input per `kind`. |
| `maxLength` | number | Maximum character length. |
| `pattern` | string | Regex the input MUST match. |
| `message` | string | User-facing error message shown when invalid. Optional; runtime provides a default if omitted. |

**Shape B — Rule array** (UX-oriented, multiple constraints, each with its own message):

```json
{
  "type": "textInput",
  "validation": [
    { "rule": "required", "message": "Name is required" },
    { "rule": "minLength", "value": 2, "message": "Too short" },
    { "rule": "email",    "message": "Invalid email format" }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `rule` | enum | `required`, `minLength`, `maxLength`, `min`, `max`, `pattern`, `email`, `url`, `phone`, `number`, `integer`, `date`. |
| `value` | any | Rule parameter (e.g., `minLength: { value: 2 }`). Omitted for rules that take no parameter (`required`, `email`). |
| `message` | string | User-facing error message when the rule fails. |

Runtimes MUST support **both shapes** and evaluate rules in declaration order. Validation runs before the value is written to the bound state path. A field fails validation when any rule (Shape B) or any declared constraint (Shape A) rejects the input. The failure message surfaces through the enclosing `form` widget (§2.6.23) if present; otherwise the field displays it inline.

---

## 7.3 Resource Access Verification

- **Server-side authorization.** `resources/read` requests on `ui://…` resources MUST be authorized by the server. The DSL runtime does not bypass server authorization.
- **Client resource permissions.** Access to `client://` resources (see [`08_Client_Extensions.md`](08_Client_Extensions.md)) MUST go through the permission system. A client action without its required permission MUST fail with `PERMISSION_DENIED` rather than silently succeed.
- **`bundle://` scope.** `bundle://` URIs MUST resolve only within the currently loaded bundle. Path traversal (`..`) and absolute-path escape MUST be rejected.
- **URL scheme allowlist.** URL fields (`image.src`, `link.href`, `webView.src`, etc.) MUST validate the URI scheme. Runtimes MUST reject schemes not in a documented allowlist (`https`, `http` when explicitly permitted, `data` with type restrictions, `client://`, `bundle://`).

### 7.3.1 Path Validation

For any client action that accepts a path (`client.readFile`, `client.writeFile`, `client.listFiles`, `client.saveFile`):

1. Normalize the path (remove `.`, collapse `//`).
2. Reject `..` segments in non-absolute contexts.
3. Resolve symlinks and validate the resolved path against the permission scope.
4. Match against the permission's `paths` glob allowlist and `excludePaths` denylist.

Failures MUST return `INVALID_PATH` or `PERMISSION_DENIED` per [`08_Client_Extensions.md`](08_Client_Extensions.md) error registry.

### 7.3.2 Command Sanitization

For `client.exec`:

- The command MUST appear in the permission's `commands` allowlist.
- Argument values MUST be passed as discrete array elements; the runtime MUST NOT interpolate user-supplied strings into a shell-interpreted command line.
- Shell metacharacters in argument values MUST be rejected when the permission's `args.allowPatterns` defines a regex.

### 7.3.3 Network Restrictions

For `client.httpRequest`:

- The target domain MUST match the permission's `domains` allowlist (wildcards per [`08_Client_Extensions.md`](08_Client_Extensions.md)).
- Non-localhost traffic MUST require HTTPS unless the permission explicitly allows HTTP.
- Request and response sizes MUST be bounded by the permission's `maxRequestSize` / `maxResponseSize`.

---

## 7.4 State Isolation

- **Application boundary.** `app.*` state is shared within one application instance. It MUST NOT leak across applications mounted in the same runtime process.
- **Page boundary.** `page.*` state is isolated per page instance. Navigating away and back yields a fresh page scope unless explicitly preserved.
- **Template instance boundary.** `local.*` state *(since v1.3)* is isolated per template instance. Two instances of the same stateful template MUST NOT share `local.*` slots.
- **Binding boundary.** A binding expression MUST NOT read state outside the scopes documented in [`03_Data_Binding.md`](03_Data_Binding.md). There is no ambient global state.

### 7.4.1 Protected State Paths

An application MAY declare state paths that actions cannot mutate:

```json
{
  "security": {
    "state": {
      "readOnlyPaths": ["user.id", "session.token"],
      "protectedPaths": ["user.role", "permissions"]
    }
  }
}
```

Runtimes MUST reject `state` actions targeting `readOnlyPaths` and MUST require elevated trust (see [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.4 trust levels) to mutate `protectedPaths`.

---

## 7.5 XSS and Injection Prevention

- **Text escaping.** Binding values inserted into text widgets (`text`, `richText`, etc.) MUST be treated as literal strings. Runtimes MUST NOT interpret the bound value as markup.
- **No raw HTML by default.** Raw HTML insertion is NOT supported by any Core Profile widget. Widgets that render untrusted content (`webView`, `markdown`) MUST be in the Advanced Profile and MUST document their own sandbox posture.
- **Attribute injection.** URL-valued fields (`image.src`, `link.href`) MUST be scheme-validated (§7.3) before use; runtimes MUST NOT build dynamic URIs by string concatenation of bindings without re-validating the result.
- **Style injection.** Bindings that resolve to style values (colors, dimensions) MUST be validated against the expected value format (e.g., hex color, positive number). A malformed value MUST fall back to the documented default.

---

## 7.6 Bundle Integrity *(since v1.2)*

- A bundle's manifest MUST include a checksum covering the bundle's asset tree.
- Runtimes that support the Bundle Profile SHOULD verify the checksum before mounting the bundle. A checksum mismatch MUST abort mounting with a diagnostic; the runtime MUST NOT render partial content from an inconsistent bundle.
- Signed bundles (SHOULD, where supported) carry a publisher signature alongside the checksum; runtimes MAY refuse to mount unsigned bundles depending on host policy.

---

## 7.7 Template Safety *(since v1.1, Template Profile)*

- Template parameters MUST be validated by `TemplateParamDefinition.validate()` before substitution into the template body. Type mismatches MUST abort the template invocation with a diagnostic.
- Template bodies are subject to the same expression sandbox (§7.1) and input-validation rules (§7.2) as regular page definitions.
- Remote template libraries *(since v1.3)* MUST be fetched over authenticated channels. Library responses MUST be validated before registration; a malformed library MUST NOT register partial entries.
- Name collisions between local and remote templates resolve to the local definition (see [`09_Templates.md`](09_Templates.md)); a remote library MUST NOT override a locally registered template.

---

## 7.8 Offline Data Encryption *(since v1.1)*

When an offline queue, cached resources, or persisted state are written to disk:

- Runtimes SHOULD encrypt at-rest using the host platform's keystore (e.g., OS keychain, Keystore, DPAPI) or an equivalent key-management facility.
- Encryption keys MUST NOT be embedded in the bundle or in the DSL.
- Session-scoped permission grants (see [`08_Client_Extensions.md`](08_Client_Extensions.md)) MUST be cleared on application close. Permanent grants MAY be persisted, but MUST be stored encrypted at-rest and MUST be keyed by MCP server identifier.

---

## 7.9 Security Diagnostics

Every security-enforcement failure (sandbox violation, rejected path, denied permission, blocked domain, template validation failure, bundle checksum mismatch) MUST:

1. Return a structured error conforming to the unified error envelope in [`08_Client_Extensions.md`](08_Client_Extensions.md).
2. Be logged via the runtime's diagnostic channel.
3. NOT crash the runtime or leak stack-trace details into the rendered UI.
