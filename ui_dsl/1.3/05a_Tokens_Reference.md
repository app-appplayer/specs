# 05a. Tokens — Reference / Alias / Component (3-tier)

A reference document that re-frames the token model in [`05_Theme.md`](05_Theme.md) as the W3C DTCG **3-tier** layout (reference → alias → component). The 3-tier view exists for serialization (DTCG JSON) and traceability; runtime binding (`{{theme.x.y}}`) is unchanged.

---

## A.1 3-tier overview

```
┌─────────────────────────────────────────────────────────────┐
│ Tier 3 — Component                                          │
│   button.background = {alias.color.primary}                 │
│   card.elevation    = {alias.elevation.raised}              │
└────────────┬────────────────────────────────────────────────┘
             │ references
┌────────────▼────────────────────────────────────────────────┐
│ Tier 2 — Alias (semantic role)                              │
│   color.primary       = {ref.palette.indigo.500}            │
│   color.surface       = {ref.palette.neutral.99}            │
│   elevation.raised    = {ref.elevation.level2}              │
└────────────┬────────────────────────────────────────────────┘
             │ references
┌────────────▼────────────────────────────────────────────────┐
│ Tier 1 — Reference (raw values)                             │
│   palette.indigo.500  = #3F51B5                             │
│   palette.neutral.99  = #FFFCF5                             │
│   elevation.level2    = 3                                   │
└─────────────────────────────────────────────────────────────┘
```

DSL authors write only alias and component tiers; the reference tier (palette and primitive values) is auto-derived from an HCT seed (M3).

---

## A.2 Tier 1 — Reference

### A.2.1 Color palette

When an HCT seed is specified, a 13-stop tonal palette is auto-generated:

| Tone | 0 | 10 | 20 | 30 | 40 | 50 | 60 | 70 | 80 | 90 | 95 | 99 | 100 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

Each stop appears across 6 families (primary / secondary / tertiary / error / neutral / neutralVariant), yielding 78 reference colors.

- `ref.palette.primary.<tone>`
- `ref.palette.secondary.<tone>`
- `ref.palette.tertiary.<tone>`
- `ref.palette.error.<tone>`
- `ref.palette.neutral.<tone>`
- `ref.palette.neutralVariant.<tone>`

To use an explicit color, declare a reference directly:

```json
{
  "ref": {
    "palette": {
      "indigo": { "500": "#3F51B5", "700": "#303F9F" }
    }
  }
}
```

### A.2.2 Dimension primitives

- `ref.dimension.spacing.<n>` — 0, 2, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 96 (8pt grid)
- `ref.dimension.radius.<n>` — 0, 4, 8, 12, 16, 24, 28, 9999
- `ref.dimension.elevation.<level>` — 0, 1, 3, 6, 8, 12

### A.2.3 Duration / curve primitives

- `ref.duration.<n>` — 50, 100, 150, 200, 250, 300, 350, 400, 450, 500, 550, 600, 1000
- `ref.curve.standard` / `emphasized` / `decelerate` / `accelerate`

---

## A.3 Tier 2 — Alias (semantic role)

The layer application authors write directly. The DSL JSON `theme.color`, `theme.typography`, `theme.spacing`, `theme.shape`, `theme.elevation`, and `theme.motion` are all aliases.

| Alias | Default mapping (light / dark) |
|---|---|
| `color.primary` | `ref.palette.primary.40` / `ref.palette.primary.80` |
| `color.onPrimary` | `ref.palette.primary.100` / `ref.palette.primary.20` |
| `color.primaryContainer` | `ref.palette.primary.90` / `ref.palette.primary.30` |
| `color.onPrimaryContainer` | `ref.palette.primary.10` / `ref.palette.primary.90` |
| `color.surface` | `ref.palette.neutral.99` / `ref.palette.neutral.10` |
| `color.onSurface` | `ref.palette.neutral.10` / `ref.palette.neutral.90` |
| `color.outline` | `ref.palette.neutralVariant.50` / `ref.palette.neutralVariant.60` |
| `color.scrim` | `ref.palette.neutral.0 @ 32%` / same |

(The full 28-role mapping follows the M3 spec and is applied automatically by the runtime.)

### A.3.1 Typography alias

| Alias | Default reference |
|---|---|
| `typography.titleLarge` | `{ size: 22, weight: medium, lineHeight: 28, letterSpacing: 0 }` |
| ... | The remaining 14 roles — see [`05_Theme.md`](05_Theme.md) § 5.4.3 table |

---

## A.4 Tier 3 — Component

Token bundles for common components. References aliases for DRY:

```json
{
  "component": {
    "button": {
      "background":      "{alias.color.primary}",
      "foreground":      "{alias.color.onPrimary}",
      "padding":         { "horizontal": "{alias.spacing.md}", "vertical": "{alias.spacing.sm}" },
      "radius":          "{alias.shape.full}",
      "typography":      "{alias.typography.labelLarge}",
      "stateLayer": {
        "hover":    "{alias.color.onPrimary @ 0.08}",
        "focus":    "{alias.color.onPrimary @ 0.12}",
        "pressed":  "{alias.color.onPrimary @ 0.16}",
        "disabled": "{alias.color.onSurface @ 0.38}"
      }
    },
    "card": {
      "background": "{alias.color.surface}",
      "radius":     "{alias.shape.medium}",
      "elevation":  "{alias.elevation.level1}",
      "padding":    "{alias.spacing.cardPadding}"
    }
  }
}
```

A runtime SHOULD recognize the 6 standard components — button, input, card, dialog, menu, list. Applications may register additional components (e.g. `chip`, `tab`, `switch`) freely.

---

## A.5 Token interchange (DTCG JSON)

The 3-tier model serializes via DTCG draft `$type` markers. See [`05b_DTCG_Interchange.md`](05b_DTCG_Interchange.md) for details.

Abbreviated example:

```json
{
  "ref": {
    "palette": {
      "primary": {
        "40": { "$type": "color", "$value": "#3F51B5" },
        "80": { "$type": "color", "$value": "#9FA8DA" }
      }
    }
  },
  "alias": {
    "color": {
      "primary":   { "$type": "color", "$value": "{ref.palette.primary.40}" },
      "onPrimary": { "$type": "color", "$value": "#FFFFFF" }
    }
  },
  "component": {
    "button": {
      "background": { "$type": "color", "$value": "{alias.color.primary}" }
    }
  }
}
```

The `{ref.palette.primary.40}` syntax is the DTCG draft alias notation.

---

## A.6 Authoring rules

1. Application authors write only **alias / component** (reference is generated from the seed).
2. If an explicit reference is needed, add a `ref` block.
3. Component-token values reference aliases (not raw values) for DRY and consistent theme propagation.
4. Runtimes detect and reject cycles in the reference → alias → component chain.
5. Missing aliases fall back to runtime defaults (M3 spec).

---

## A.7 Traceability

Each token's origin must be traceable:

- DSL author uses `{{theme.color.primary}}`.
- Runtime resolves `alias.color.primary`'s `$value` to `{ref.palette.primary.40}`.
- Chain follows through to the final hex value.
- Change-impact scope can be calculated automatically (compatible with Claude Design / Tokens Studio).

This traceability is the prerequisite for AI design tools to safely modify tokens.
