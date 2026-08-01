# 03. Data Binding

Data binding is the mechanism by which values in state, theme, route, environment, and event scopes flow into widget properties and back. It is the reactive core of MCP UI DSL.

## 3.1 Syntax

A binding is any string containing one or more `{{ expression }}` segments.

```json
{ "type": "text", "text": "{{count}}" }
{ "type": "text", "text": "Total: {{count}} items" }
{ "type": "text", "text": "{{count > 0 ? 'Has items' : 'Empty'}}" }
```

- A string with no `{{ }}` is a literal.
- A string equal to a single `{{ expression }}` resolves to the expression's native value (number, boolean, object, array, null). It is not coerced to string unless concatenated with surrounding text.
- Mixed strings (`"Total: {{count}}"`) coerce the interpolated value to string per §3.3.

## 3.2 Expression Language

### 3.2.1 Grammar

```
Expression      := Literal | Variable | BinaryOp | UnaryOp | Conditional | FunctionCall | MethodCall | ArrayAccess
Literal         := String | Number | Boolean | Null | Array | Object
Variable        := Path
Path            := Identifier ( "." Identifier | "?." Identifier | "[" Expression "]" )*
BinaryOp        := Expression BinaryOperator Expression
UnaryOp         := UnaryOperator Expression
Conditional     := Expression "?" Expression ":" Expression
FunctionCall    := Identifier "(" ArgumentList? ")"
MethodCall      := Expression "." Identifier "(" ArgumentList? ")"
ArrayAccess     := Expression "[" Expression "]"
```

### 3.2.2 Operators

| Operator | Meaning |
|----------|---------|
| `+` `-` `*` `/` `%` | Arithmetic |
| `==` `!=` `<` `<=` `>` `>=` | Comparison |
| `&&` `\|\|` `!` | Logical |
| `?:` | Conditional (ternary) |
| `??` | Null coalescing |
| `?.` | Optional chaining |
| `.` `[]` | Member / index access |

### 3.2.3 Operator Precedence

From highest to lowest:

1. `()` `[]` `.` `?.`
2. `!` unary `-` unary `+`
3. `*` `/` `%`
4. `+` `-`
5. `<` `<=` `>` `>=`
6. `==` `!=`
7. `&&`
8. `||`
9. `??`
10. `?:`

## 3.3 Type Coercion

| Operation | Rule |
|-----------|------|
| `String + Number` | String concatenation |
| `Number + Number` | Numeric addition |
| Boolean in numeric context | `true` → 1, `false` → 0 |
| `null` in string context | Empty string `""` |
| Undefined property access | `null` |
| `{{x}}` inside a larger string | Converted to string via default formatter |

## 3.4 Prefix Resolution Order

An unprefixed identifier is resolved by searching scopes in order. The first scope that contains the identifier wins. Explicit prefixes bypass this resolution and target their scope directly.

1. List iteration context (`item`, `index`, `isFirst`, `isLast`, `isEven`, `isOdd`)
2. Template instance scope (`local.*`) *(since v1.3 — see [`09_Templates.md`](09_Templates.md))*
3. Current page state (`page.*`)
4. Application state (`app.*`)
5. Theme tokens (`theme.*`) — see [`05_Theme.md`](05_Theme.md)
6. Route parameters (`route.params.*`)
7. Runtime capabilities (`runtime.*`)
8. i18n keys (`i18n.*`) — see [`12_Internationalization.md`](12_Internationalization.md)

If no scope matches, the expression evaluates to `null` and the runtime MAY log a warning.

Explicit prefixes always win: `{{app.count}}` accesses application state even if `count` also exists on the page.

## 3.5 Binding Prefix Catalog

Full prefix catalog is maintained in [`17_Naming.md`](17_Naming.md) §17.2.5. Summary:

| Prefix | Since | Scope | Access |
|--------|-------|-------|--------|
| *(bare)* | v1.0 | Resolved by order in §3.4 | Read-write (resolved target) |
| `page.` | v1.0 | Current page state (explicit alias of the bare resolution target) | Read-write |
| `app.` | v1.0 | Application-wide state | Read-write |
| `local.` | v1.3 | Template instance state | Read-write |
| `route.params.` | v1.0 | Current route parameters | Read-only |
| `theme.` | v1.0 | Theme tokens | Read-only |
| `event.` | v1.0 | Event data (inside action callbacks) | Read-only |
| `i18n.` | v1.0 | Localized strings | Read-only |
| `client.` | v1.1 | Client environment | Read-only |
| `client.theme.` | v1.1 | Host environment theme | Read-only |
| `client.env.` | v1.1 | OS environment variables (allowlisted) | Read-only |
| `client.file.` | v1.1 | Client file metadata | Read-only |
| `client.system.` | v1.1 | Client system info | Read-only |
| `client.network.` | v1.1 | Network connectivity | Read-only |
| `permissions.` | v1.1 | Permission grant state | Read-only |
| `channels.` | v1.1 | Bidirectional channel data | Read-only |
| `resources.` | v1.1 | Subscribed MCP resources | Read-only |
| `sync.` | v1.1 | Offline sync status | Read-only |
| `runtime.` | v1.1 | Runtime capability flags | Read-only |

### 3.5.1 List Iteration Context

Inside a `list`, `grid`, or `use` template iteration:

| Variable | Type | Description |
|----------|------|-------------|
| `item` | any | Current element of the iterated collection |
| `index` | number | Zero-based index |
| `isFirst` | boolean | `index == 0` |
| `isLast` | boolean | `index == length - 1` |
| `isEven` | boolean | `index % 2 == 0` |
| `isOdd` | boolean | `index % 2 == 1` |

### 3.5.2 Event Data (`event.*`)

Inside a callback action, `event.*` resolves to data from the triggering event. Shape depends on the source:

| Source | Typical fields |
|--------|----------------|
| Text input `onChange` | `event.value` |
| Form `onSubmit` | `event.values` |
| Gesture / tap | `event.target`, `event.data` |
| Drag/drop | `event.data` (payload) |
| Channel `onMessage` | `event.message`, `event.channel` |

### 3.5.3 Permission Binding Forms *(since v1.1)*

- `{{permissions.<resource>.<action>}}` → boolean
- `{{permissions.<resource>.<action>.status}}` → one of `granted`, `denied`, `pending`, `revoked`, `unavailable`

### 3.5.4 Channel Binding Forms *(since v1.1)*

- `{{channels.<name>.active}}` → boolean
- `{{channels.<name>.state}}` → one of `connecting`, `connected`, `disconnected`, `reconnecting`, `failed`, `stopped`
- `{{channels.<name>.<dataPath>}}` → latest value received on the channel

## 3.6 Built-in Functions

The following 16 functions are required in the Core Profile. Implementations SHOULD also provide the extended set listed in §3.6.3.

### 3.6.1 Core Built-ins (v1.0)

| Function | Purpose |
|----------|---------|
| `toUpperCase(s)` | Uppercase a string |
| `toLowerCase(s)` | Lowercase a string |
| `trim(s)` | Strip leading and trailing whitespace |
| `round(n, digits?)` | Round to nearest, optional digit precision |
| `floor(n)` | Round down to integer |
| `ceil(n)` | Round up to integer |
| `max(a, b, ...)` | Largest numeric argument |
| `min(a, b, ...)` | Smallest numeric argument |
| `length(v)` | Length of string or array |
| `contains(s, needle)` | `true` if `s` contains `needle` |
| `replace(s, old, new)` | Replace all occurrences |
| `split(s, sep)` | Split string into array |
| `join(arr, sep)` | Join array into string |
| `format(value, pattern)` | Polymorphic date/number formatter |
| `filter(arr, predicate)` | Filter by lambda or shorthand |
| `reduce(arr, fn, init?)` | Fold array to a single value |

```json
"{{toUpperCase(name)}}"
"{{round(price * quantity, 2)}}"
"{{filter(items, 'active')}}"
"{{format(date, 'YYYY-MM-DD')}}"
"{{format(price, '#,##0.00')}}"
```

### 3.6.2 Shorthand Forms

- `filter(items, 'active')` — filter items whose `active` property is truthy.
- `filter(items, 'status', 'done')` — filter items whose `status` equals `'done'`.
- `reduce(items, 'price')` — sum the numeric `price` property across items.

### 3.6.3 Extended Built-ins (SHOULD)

| Function | Description |
|----------|-------------|
| `abs(n)` | Absolute value |
| `parseInt(s)` | Parse integer, `null` on failure |
| `parseDouble(s)` | Parse float, `null` on failure |
| `now()` | Current time as ISO 8601 string |
| `substring(s, start, end?)` | Extract substring |
| `calculateDuration(start, end, unit?)` | Duration between ISO timestamps |
| `map(list, property)` | Extract property from each item |
| `where(list, predicate)` | Alias for `filter` |

### 3.6.4 Method Call Form

Built-ins that take a primary argument may be called as methods: `name.toUpperCase()` is equivalent to `toUpperCase(name)`; `text.contains('x')` is equivalent to `contains(text, 'x')`.

### 3.6.5 Transforms

Transforms are named string operations applied through a `transform` key in structured binding definitions. They receive a resolved value and return a transformed value.

| Transform | Description |
|-----------|-------------|
| `uppercase` / `lowercase` / `capitalize` | String case |
| `round` / `floor` / `ceil` / `abs` / `truncate` | Numeric shaping |
| `currency` | Localized currency format |
| `percentage` | Percentage format |
| `date` / `time` | Date/time formatting |
| `json` | Serialize to JSON string |
| `padLeft` / `padRight` | String padding |

## 3.7 Reactivity Contract

- Every binding records the set of paths it reads. A state change to any of those paths invalidates the binding.
- Runtimes MUST recompute invalidated bindings before the next frame.
- A binding whose recomputed value is unchanged MUST NOT trigger a re-render of its hosting widget.
- State updates within a single frame are batched; only the final value drives re-render.

Normative performance targets are in [`18_Conformance.md`](18_Conformance.md) §18.2.11.

## 3.8 Computed Properties

A page or app `state` may declare `computed` values. Computed expressions auto-update when any referenced binding changes.

```json
{
  "state": {
    "initial": { "items": [], "filter": "all" },
    "computed": {
      "filteredItems": "{{filter == 'all' ? items : filter(items, 'status', filter)}}",
      "completedCount": "{{length(filter(items, 'completed'))}}",
      "totalPrice": "{{reduce(items, (acc, i) => acc + i.price * i.qty, 0)}}"
    }
  }
}
```

- Dependencies are detected automatically from the expression.
- Values are cached until a dependency changes.
- Computed properties may reference other computed properties.
- Circular dependencies are detected and produce a warning; the cyclic value resolves to `null`.

## 3.9 State Watchers

A watcher triggers actions when a state path changes.

```json
{
  "state": {
    "watchers": [
      {
        "watch": "user.isAuthenticated",
        "condition": "{{!user.isAuthenticated}}",
        "actions": [
          { "type": "navigation", "action": "push", "route": "/login" }
        ]
      }
    ]
  }
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `watch` | string | Yes | State path to observe |
| `condition` | string | No | Fire only when this expression evaluates truthy |
| `actions` | Action[] | Yes | Actions to run when the watcher fires |

## 3.10 Tool Response Auto-Merge

When a `tool` action succeeds, the runtime parses the response text as JSON and merges each top-level key into page state.

1. Top-level keys in the response become page state variables. Merged keys are available at the page root (e.g. `{{counter}}`) and equivalently via the explicit `page.*` prefix (e.g. `{{page.counter}}`) — see §3.5.
2. Nested objects replace existing values (no deep merge). **Lists in the response REPLACE existing lists; the runtime does not append.** Authors who need to accumulate items MUST use a `state` action with `action: append` (see [`04_Actions.md`](04_Actions.md) §4.2) inside `onSuccess`, not rely on auto-merge.
3. Widgets bound to changed keys re-render.
4. Auto-merge is skipped when the tool action specifies `bindResult`; the full result is stored at that path instead.
5. **Non-Map responses (list, scalar, or `null`) are not auto-merged.** If `bindResult` is set, the runtime stores the raw value at that path. If `bindResult` is absent, the runtime skips the merge silently and SHOULD emit a warning log; page state is unchanged. Authors that expect a non-Map response MUST declare `bindResult` (or transform the value in `onSuccess` via a `state` action).

## 3.11 StateChangeEvent

Every state mutation emits a StateChangeEvent to listeners.

```json
{
  "path": "counter",
  "oldValue": 4,
  "newValue": 5,
  "timestamp": "2026-04-18T10:30:00Z",
  "source": "action"
}
```

| `source` | Meaning |
|----------|---------|
| `action` | User-triggered via a `state` action |
| `tool` | Tool response auto-merge |
| `subscription` | Resource notification |
| `system` | Internal runtime update |
