# 11. Expression Language

The bundle ships with an embedded **Expression Language (EL)** used in
- conditional fields (action `condition`, binding `condition`,
  `flow` step `condition`, navigation guard `condition`),
- transform expressions (`DataBinding.transform`,
  `ProfileContentSection.condition`),
- template interpolation strings (`${...}` inside text content).

The EL is evaluated by the host runtime; the bundle only carries the
expression strings.

References, in the `mcp_bundle` package:
- `lib/src/expression/lexer.dart`
- `lib/src/expression/parser.dart`
- `lib/src/expression/ast.dart`
- `lib/src/expression/evaluator.dart`
- `lib/src/expression/functions.dart`
- `lib/src/expression/context.dart`

## 11.1 Two Modes

The EL has two surface modes:

| Mode | Syntax | Use |
|------|--------|-----|
| **Interpolation** | `"prefix ${expr} suffix"` | Embed values in strings. The expression's value is converted to string. |
| **Condition** | A bare boolean expression | Truthiness gates a slot (action condition, binding condition, ...). |

The two modes share the same expression grammar; only the surface
differs. A condition expression is exactly one expression — no `${...}`
wrapper. An interpolation string can contain multiple `${expr}`
occurrences mixed with literal text.

## 11.2 Lexical Structure

```ebnf
literal     = string | number | boolean | null ;
string      = '"' character* '"' | "'" character* "'" ;
number      = integer | decimal ;
boolean     = 'true' | 'false' ;

identifier  = letter (letter | digit | '_')* ;
path        = identifier ('.' identifier | '[' index ']')* ;

operator    = comparison | logical | arithmetic | membership ;
comparison  = '==' | '!=' | '<' | '<=' | '>' | '>=' ;
logical     = '&&' | '||' | '!' | 'and' | 'or' | 'not' ;
arithmetic  = '+' | '-' | '*' | '/' | '%' | '**' ;
membership  = 'in' | 'matches' ;

ternary     = expr '?' expr ':' expr ;
nullish     = expr '??' expr ;
safe-nav    = '?.' ;

delimiter   = '(' | ')' | '[' | ']' | ',' ;
```

String escaping: `\\`, `\"`, `\'`, `\n`, `\t`, `\r`, `\$` (literal
dollar — prevents interpolation).

## 11.3 Operators

| Category | Operators | Notes |
|----------|-----------|-------|
| Arithmetic | `+ - * / % **` | `+` overloaded for `String` (concatenation) and `List` (concat). `int ** int` returns `int`; mixed returns `double`. |
| Comparison | `== != < <= > >=` | `==` is value-equality across primitives, structural for `Map` / `List`. |
| Logical | `&&` `||` `!` (also `and` `or` `not`) | Short-circuit evaluation. |
| Membership | `x in collection` | `collection` may be `List` (contains), `Map` (containsKey), or `String` (contains). |
| Pattern | `value matches regex` | `regex` is a Dart `RegExp` pattern string; throws on invalid pattern. |
| Ternary | `cond ? a : b` | Standard. |
| Nullish | `a ?? b` | Returns `a` when non-null; otherwise `b`. |
| Safe nav | `obj?.prop` `obj?.method()` | Returns `null` when receiver is `null`. |

## 11.4 Path Expressions

```
${state.user.name}
${state.users[0].email}
${state.config["nested-key"]}
```

| Form | Meaning |
|------|---------|
| `name` | Identifier — looked up in the context. |
| `obj.name` | Property access. |
| `obj[index]` | Bracket access — `int` for `List`, `String` for `Map`, `int` for `String` (returns one-character `String`). |
| `obj?.name` / `obj?.[i]` | Safe navigation — null receiver returns null. |
| `arr.length` | Built-in `length` property on `String` / `List` / `Map`. |

Negative indices are NOT supported by the runtime evaluator (the
`IndexExpr` evaluator emits "Index out of bounds" for negative `int`).
Use the `at(arr, -1)` built-in function for negative indexing where
needed.

## 11.5 Built-in Functions

The reference `ExpressionFunctions` registers 60+ built-ins:

| Group | Functions |
|-------|-----------|
| String | `length`, `upper`, `lower`, `trim`, `trimStart`, `trimEnd`, `substring`, `replace`, `replaceAll`, `split`, `join`, `startsWith`, `endsWith`, `contains`, `indexOf`, `padStart`, `padEnd` |
| Math | `abs`, `ceil`, `floor`, `round`, `min`, `max`, `sum`, `avg`, `pow`, `sqrt`, `log`, `sin`, `cos`, `tan`, `random`, `clamp` |
| Array | `first`, `last`, `at`, `slice`, `reverse`, `sort`, `unique`, `flatten`, `map`, `filter`, `reduce`, `find`, `findIndex`, `every`, `some`, `count`, `groupBy`, `sortBy`, `pluck`, `zip`, `range` |
| Object | `keys`, `values`, `entries` (additional functions in `functions.dart`). |

Calling an unknown function throws `ArgumentError: Unknown function:
<name>`.

Hosts MAY register additional functions by extending
`ExpressionFunctions.register(name, fn)`.

## 11.6 Evaluation Context

The host supplies an `EvaluationContext` exposing typed roots:

| Root | Description | Available in |
|------|-------------|--------------|
| `inputs` | Tool / skill input parameters. | tool execution, skill steps |
| `state` | Mutable state. | UI, skill, flow |
| `context` | Execution context (run id, agent id, ...). | always |
| `steps` | Previous step results. | skill, flow |
| `config` | Configuration. | always |
| `bundle` | Bundle metadata. | always |
| `facts` | FactGraph facts (when bound). | knowledge-aware contexts |
| `metrics` | Profile metrics. | profile-aware contexts |
| `claims` | Claims from skills / fact graph. | skill, fact graph |
| `pipeline` | Pipeline execution context. | pipeline runs |
| `workflow` | Workflow execution context. | workflow runs |

Roots not bound in a context throw `EvaluationException: Undefined
variable: <name>`.

## 11.7 Interpolation Rules

```
"Hello, ${state.user.name}!"
"Total: ${sum(state.cart.items.price)}"
"\$1.00 = ${1 + 0}.00"   ← literal $1.00 = 1.00
```

Interpolation:

- A `${` opens an expression. Matching `}` (respecting nested braces
  inside the expression) closes it.
- The expression value is converted to string via Dart `toString()`.
- `null` becomes the empty string.
- `\$` is a literal dollar sign (prevents interpolation).

Nested interpolation `"${state.${state.key}}"` is **not** supported.

## 11.8 Conditions

```
state.user.isAuthenticated
state.cart.items.length > 0
state.role == "admin" && state.feature.enabled
"@" in state.user.email
state.email matches "^[a-z0-9_.]+@[a-z0-9.]+$"
```

A condition expression evaluates to a value; the runtime applies
JavaScript-like truthiness:

- `false`, `null`, `0`, empty string, empty `List`, empty `Map` →
  falsy;
- everything else → truthy.

## 11.9 Errors

The evaluator throws `EvaluationException` with a human-readable
message on:

- division / modulo by zero;
- index out of bounds;
- type mismatch (`Cannot add String + Map`);
- safe-nav violation (`Cannot access property on null` when the
  source is non-optional);
- unknown identifier;
- unknown operator.

Hosts SHOULD catch evaluation errors at the slot boundary and treat
them as either a falsy condition (for `condition` slots) or an empty
string (for `${...}` interpolation), with a logged warning. The bundle
is malformed only if every evaluation of a given expression fails on
every render — the host need not refuse activation on first error.

## 11.10 Conformance

An EL implementation MUST:

1. Implement the operators, functions, and modes listed above with
   the listed semantics.
2. Honor short-circuit semantics for `&&` / `||`.
3. Implement safe-nav (`?.`) and nullish-coalesce (`??`) per their
   semantics.
4. Treat unknown enum / function names as errors (not silent
   fallbacks).

An EL implementation MAY:

- Add host-specific functions via the function registry.
- Cache compiled expression ASTs.
- Apply per-evaluation timeouts.

A bundle MAY rely only on the operators and functions defined here;
host-specific extensions reduce portability.
