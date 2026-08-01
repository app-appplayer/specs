# MCP UI DSL — Widget Registry

**Status:** Phase 1 (proof of concept).

This directory is the **single source of truth** for every widget type in the MCP UI DSL. One YAML file per widget. The prose/tables in `../02_Widgets.md` and `../10_Advanced_Widgets.md` will be **generated from these files** once migration completes (see `../CHANGELOG.md`, "Widget Registry (Phase 1+)" entry). Until migration is complete, both sources may coexist; when they disagree, this registry wins.

## Why

Prose specs are ambiguous, cannot be consumed by validators or LLMs directly, and require human interpretation. A machine-readable registry enables:

- **JSON Schema generation** for IDE autocomplete and offline DSL validation.
- **Runtime load-time validation** with precise error paths.
- **Factory-schema conformance tests** that fail if a runtime factory ignores a declared property.
- **LLM prompt cards** that prevent widget-type and property-name hallucination.
- **Doc generation** that stays in sync with implementation.

## File layout

```
widgets/
├── README.md                    (this file)
├── _common.yaml                 (shared input contract per §2.6.0; imported by category)
├── layout/
│   ├── box.yaml
│   ├── linear.yaml
│   └── ...
├── display/
├── input/
├── list/
├── navigation/
├── scroll/
├── interactive/
├── dialog/
├── animation/
├── utility/
└── advanced/
```

One widget per file. The filename is the canonical `type` + `.yaml`. Aliases do NOT get their own files — they live in the `aliases:` field of the canonical file.

## Required fields

```yaml
type: <canonical_widget_type>     # must match §17.3 naming registry
category: <section>               # layout | display | input | list | navigation | scroll | interactive | dialog | animation | utility | advanced
profile: <conformance_profile>    # Core | Client | Bundle | Advanced | Template
since: <version>                  # "v1.0" | "v1.1" | "v1.2" | "v1.3" (informational; not normative)
description: |
  One paragraph. Plain prose. No markdown tables.
```

## Optional fields

### `aliases`

Legacy type-name strings accepted by the runtime. Each alias routes to this canonical factory.

```yaml
aliases: [alert]
```

### `properties`

Map of property name → property definition. Each definition:

```yaml
properties:
  <name>:
    type: <type_expr>         # see Type Expressions below
    required: <bool>          # default false
    default: <value>          # optional; must match `type`
    enum: [a, b, c]           # optional; restricts string type
    description: <str>        # optional
    aliases: [legacyName]     # legacy property-name aliases
    deprecated: <bool>        # optional; future warning path
```

#### Type expressions

| Expression | Meaning |
|---|---|
| `string` / `number` / `boolean` / `integer` | JSON primitives |
| `binding` | A `"{{path}}"` expression; resolves to runtime state |
| `Widget` | Nested widget definition (recursive) |
| `Action` | Handler per `04_Actions.md` |
| `EdgeInsets` | `{all} | {horizontal, vertical} | {top, right, bottom, left}` |
| `Color` | Hex `"#RRGGBB"` or `"#AARRGGBB"` |
| `ValidationConfig` | Shape A or B per `07_Security.md §7.2.1` |
| `<A> \| <B>` | Union — value may match either |
| `array<T>` | Array of `T` |
| `object{ field1: T1, field2: T2 }` | Inline object shape |

### `children`

Describes how the widget receives child widget(s).

```yaml
children:
  kind: none | single | list | template
  # single:   key = "child"            (configurable via `key`)
  # list:     key = "children"
  # template: items binding + itemTemplate Widget
  key: child                  # override default key name if needed
  items_key: items            # only for kind: template
  template_key: itemTemplate  # only for kind: template
```

### `events`

Widget-level event handlers. These are Actions fired by the widget itself (not child actions like `onTap` on a button inside `actions[]`).

```yaml
events:
  onChange:
    description: Fired when the bound value changes.
    payload:
      value: <type_expr>      # what {{event.value}} resolves to
      type: change            # constant
    aliases: [onChanged, change]
```

### `examples`

```yaml
examples:
  - name: basic
    description: Minimal valid usage
    dsl: |
      {
        "type": "box",
        "decoration": { "color": "#2196F3" }
      }
  - name: missing_required
    dsl: |
      { "type": "box" }
    expect: validation_error
    error_path: "#/oneOf"
```

Positive examples (no `expect` field) MUST pass schema validation and runtime render tests. Negative examples MUST fail schema validation with the specified `error_path` (JSON Pointer into the validator output).

## Shared contracts

Some contracts span multiple widgets (e.g., §2.6.0 input widget binding). Declare them once in `_common.yaml` and reference via `include`:

```yaml
# box.yaml
include:
  - _common/layout_base   # common layout properties
properties:
  decoration: ...
```

The codegen inlines included fragments before emitting artifacts.

## Conformance with §17 naming

The `type` field MUST match the canonical name in `../17_Naming.md §17.3.1`. The `aliases` field MUST be a subset of the legacy aliases listed in that registry. Codegen verifies this on every run.

## Editing rules

- **One widget per file.** Never group multiple widgets in one YAML.
- **Prose edits to `02_Widgets.md` / `10_Advanced_Widgets.md` are frozen** once the generated equivalents land. Edit the YAML, re-run codegen, commit the diff.
- **Never hand-edit `schema/*.json` or `generated/*.md`.** Those are build outputs.
- New widget = new YAML file + spec decision recorded in `../CHANGELOG.md`. Codegen must pass before merge.
