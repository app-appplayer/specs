# 05. Theme

Theme tokens define the design system primitives every MCP UI application uses to render. This document is the **canonical Theme spec for MCP UI DSL v1.3** and replaces the legacy minimal model in earlier drafts.

The mcp_ui 1.3 design system follows modern standards (Material 3 naming, DTCG W3C 3-tier, shadcn-style semantic, HCT seed) as canonical. Legacy aliases (h1-h6, headline1-6, subtitle, overline, M1 textOn*, etc.) are not included.

Normative requirements live in [`18_Conformance.md`](18_Conformance.md) §18.2.7. See [`05b_DTCG_Interchange.md`](05b_DTCG_Interchange.md) for DTCG JSON interchange detail.

---

## 5.1 Theme Shape

`theme` is a top-level field of `ApplicationDefinition`. 16 token domains:

```json
{
  "theme": {
    "mode": "system",
    "color":       { "...": "..." },
    "typography":  { "...": "..." },
    "spacing":     { "...": "..." },
    "shape":       { "...": "..." },
    "elevation":   { "...": "..." },
    "motion":      { "...": "..." },
    "density":     { "...": "..." },
    "breakpoints": { "...": "..." },
    "border":      { "...": "..." },
    "opacity":     { "...": "..." },
    "focusRing":   { "...": "..." },
    "zIndex":      { "...": "..." },
    "component":   { "...": "..." },
    "preset":      "warm",
    "fonts":       { "...": "..." },
    "light": { "...": "..." },
    "dark":  { "...": "..." }
  }
}
```

All tokens resolve through the `theme.*` binding prefix (§ 5.13).

---

## 5.2 Theme Mode

`theme.mode` selects the light or dark color scheme.

| Value | Behavior |
|-------|----------|
| `light` | Force the light scheme |
| `dark` | Force the dark scheme |
| `system` | Follow host brightness (default) |

In `system` mode the runtime MUST track host brightness and switch schemes without re-rendering the application shell. Desktop = system appearance, mobile = device dark-mode toggle, web = `prefers-color-scheme`.

`theme.mode` may be bound to application state (`{{app.themeMode}}`); changes trigger theme recomputation and re-render.

---

## 5.3 Color (Material 3)

`theme.color` consists of **Material 3 28 roles** plus a state layer plus semantic colors (success / warning / info). It also supports auto-generation of a tonal palette from a single HCT seed.

### 5.3.1 Color roles (28 + 6 semantic)

#### Primary family

| Role | Description |
|---|---|
| `primary` | Primary brand color |
| `onPrimary` | Foreground on primary |
| `primaryContainer` | Container variant of primary |
| `onPrimaryContainer` | Foreground on primaryContainer |

#### Secondary family

| Role | Description |
|---|---|
| `secondary` | Secondary brand color |
| `onSecondary` | Foreground on secondary |
| `secondaryContainer` | Container variant of secondary |
| `onSecondaryContainer` | Foreground on secondaryContainer |

#### Tertiary family

| Role | Description |
|---|---|
| `tertiary` | Tertiary brand color |
| `onTertiary` | Foreground on tertiary |
| `tertiaryContainer` | Container variant of tertiary |
| `onTertiaryContainer` | Foreground on tertiaryContainer |

#### Error family

| Role | Description |
|---|---|
| `error` | Error / destructive color |
| `onError` | Foreground on error |
| `errorContainer` | Container variant of error |
| `onErrorContainer` | Foreground on errorContainer |

#### Surface family

| Role | Description |
|---|---|
| `surface` | Default surface (cards, sheets, dialogs) |
| `onSurface` | Foreground on surface |
| `surfaceVariant` | Lower-emphasis surface |
| `onSurfaceVariant` | Foreground on surfaceVariant |
| `surfaceTint` | Tint applied at elevation (M3 elevation tinting) |

#### Background family

| Role | Description |
|---|---|
| `background` | Page background |
| `onBackground` | Foreground on background |

#### Outline family

| Role | Description |
|---|---|
| `outline` | Subtle border, divider |
| `outlineVariant` | Lower-emphasis border |

#### Inverse family

| Role | Description |
|---|---|
| `inverseSurface` | Inverse of surface (snackbars, etc.) |
| `inverseOnSurface` | Foreground on inverseSurface |
| `inversePrimary` | Inverse of primary |

#### Misc

| Role | Description |
|---|---|
| `scrim` | Modal scrim (dialog backdrop) |
| `shadow` | Shadow color base |

#### Semantic (additions beyond M3)

| Role | Description |
|---|---|
| `success` | Success state |
| `onSuccess` | Foreground on success |
| `warning` | Warning state |
| `onWarning` | Foreground on warning |
| `info` | Informational state |
| `onInfo` | Foreground on info |

### 5.3.2 Seed (HCT)

When `theme.color.seed` is specified, the runtime MUST derive all 28 roles plus dark-mode variants automatically using the [HCT (Hue · Chroma · Tone)][m3color] algorithm. Any explicitly specified roles override only those roles (deep merge).

```json
{
  "color": {
    "seed": "#3F51B5",
    "primary": "#5C6BC0"
  }
}
```

### 5.3.3 State layer

Interactive surface hover / focus / pressed / disabled state colors are composed by overlaying a state layer at standard opacity onto the base color.

| State | Opacity (default) |
|---|---|
| `hover` | 0.08 |
| `focus` | 0.12 |
| `pressed` | 0.16 |
| `disabled` (foreground) | 0.38 |
| `disabled` (background) | 0.12 |

Override via `theme.color.stateLayer.<state>`.

### 5.3.4 Color format

| Format | Pattern | Example |
|---|---|---|
| 6-digit hex | `#RRGGBB` | `#2196F3` |
| 8-digit hex | `#AARRGGBB` | `#80000000` |
| Functional rgba | `rgb(r, g, b)` / `rgba(r, g, b, a)` | `rgba(33, 150, 243, 0.5)` |

Hex MUST. Functional SHOULD. Named CSS-keyword colors are not canonical.

### 5.3.5 Light / Dark schemes

Define mode-independent defaults in `theme.color`, then override per mode under `theme.light.color` / `theme.dark.color`. When a seed is present both light and dark are auto-derived, so explicit overrides are unnecessary.

### 5.3.6 Mode-specific fallback

The runtime MUST resolve a theme for both `light` and `dark` even when the bundle's `theme` block is sparse, so the `system` toggle always produces a sensible scheme.

| Bundle declares | `mode: light` resolves to | `mode: dark` resolves to |
|---|---|---|
| Full theme (`color`, `light`, `dark`) | `theme + theme.light` (deep-merged) | `theme + theme.dark` (deep-merged) |
| Common `color` only (no `light` / `dark` variant) | `theme` | `theme` (the bundle's common colors apply to both modes) |
| `seed` only | M3 light scheme derived from seed | M3 dark scheme derived from seed |
| **No `theme` block at all** | **M3 default light scheme (HCT-derived)** | **M3 default dark scheme (HCT-derived)** |

The "no `theme` block" row is the key contract: a bundle that omits the `theme` block entirely MUST still render a usable dark scheme when the host brightness is dark — the runtime falls back to the M3 default dark scheme rather than re-tagging the light scheme as dark. Conversely, a bundle that supplies `theme.color` (common-level) without per-mode variants signals that the same colors apply to both modes; the runtime MUST NOT replace those with M3 defaults.

[m3color]: https://m3.material.io/styles/color/the-color-system/key-colors-tones

---

## 5.4 Typography (Material 3 — 15 role)

`theme.typography` consists of **M3 5 families × 3 sizes = 15 roles**. Legacy h1-h6 / subtitle / overline are not included.

### 5.4.1 Roles (15)

| Family | Sizes |
|---|---|
| `display` | `displayLarge` · `displayMedium` · `displaySmall` |
| `headline` | `headlineLarge` · `headlineMedium` · `headlineSmall` |
| `title` | `titleLarge` · `titleMedium` · `titleSmall` |
| `body` | `bodyLarge` · `bodyMedium` · `bodySmall` |
| `label` | `labelLarge` · `labelMedium` · `labelSmall` |

### 5.4.2 TextStyle fields

| Field | Type | Description |
|---|---|---|
| `fontFamily` | `string \| string[]` | Font family or fallback chain |
| `fontSize` | `number` | Logical px |
| `fontWeight` | `number \| 'regular' \| 'medium' \| 'bold'` | 100-900 or alias |
| `lineHeight` | `number` | Multiplier (e.g. 1.5) or absolute px (≥ 16 treated as px) |
| `letterSpacing` | `number` | Logical px |
| `fontFeatureSettings` | `string[]` | OpenType features (e.g. `["liga", "kern"]`) |

### 5.4.3 M3 default scale

| Role | size | weight | lineHeight | letterSpacing |
|---|---:|---:|---:|---:|
| displayLarge | 57 | regular | 64 | -0.25 |
| displayMedium | 45 | regular | 52 | 0 |
| displaySmall | 36 | regular | 44 | 0 |
| headlineLarge | 32 | regular | 40 | 0 |
| headlineMedium | 28 | regular | 36 | 0 |
| headlineSmall | 24 | regular | 32 | 0 |
| titleLarge | 22 | medium | 28 | 0 |
| titleMedium | 16 | medium | 24 | 0.15 |
| titleSmall | 14 | medium | 20 | 0.1 |
| bodyLarge | 16 | regular | 24 | 0.5 |
| bodyMedium | 14 | regular | 20 | 0.25 |
| bodySmall | 12 | regular | 16 | 0.4 |
| labelLarge | 14 | medium | 20 | 0.1 |
| labelMedium | 12 | medium | 16 | 0.5 |
| labelSmall | 11 | medium | 16 | 0.5 |

```json
{
  "typography": {
    "displayLarge": { "fontSize": 57, "fontWeight": "regular", "lineHeight": 64, "letterSpacing": -0.25 },
    "bodyLarge":    { "fontSize": 16, "fontWeight": "regular", "lineHeight": 24, "letterSpacing": 0.5 },
    "labelLarge":   { "fontSize": 14, "fontWeight": "medium",  "lineHeight": 20, "letterSpacing": 0.1 }
  }
}
```

When `fontFamily` is `string[]`, the last entry SHOULD be a generic family (`sans-serif` / `serif` / `monospace`).

---

## 5.5 Spacing (8pt grid · 9 slots)

```json
{
  "spacing": {
    "xxs": 2,
    "xs":  4,
    "sm":  8,
    "md":  16,
    "lg":  24,
    "xl":  32,
    "2xl": 48,
    "3xl": 64,
    "4xl": 96
  }
}
```

Values are logical px. The 9 slots are the standard set; additional slots may be declared (`5xl`, `6xl`, etc.). All slots bind via `{{theme.spacing.md}}` form.

### 5.5.1 Layout primitives

| Token | Default | Use |
|---|---:|---|
| `screenPadding` | `md (16)` | Screen outer padding |
| `cardPadding` | `md (16)` | Card inner padding |
| `sectionGap` | `lg (24)` | Gap between sections |
| `inlineGap` | `sm (8)` | Gap between inline elements |

Primitives bind to a spacing slot (e.g. `md`) or a raw number.

```json
{
  "spacing": {
    "screenPadding": 16,
    "cardPadding":   16,
    "sectionGap":    24,
    "inlineGap":     8
  }
}
```

---

## 5.6 Shape (Material 3 — 7 family + per-corner)

`theme.shape` is the M3 7-family corner-radius scale.

```json
{
  "shape": {
    "none":       0,
    "extraSmall": 4,
    "small":      8,
    "medium":     12,
    "large":      16,
    "extraLarge": 28,
    "full":       9999
  }
}
```

### 5.6.1 Per-corner override

For per-corner radii, use the object form:

```json
{
  "shape": {
    "topSheet": {
      "topStart":    16,
      "topEnd":      16,
      "bottomStart":  0,
      "bottomEnd":    0
    }
  }
}
```

`topStart` / `topEnd` / `bottomStart` / `bottomEnd` auto-flip under RTL (LTR `topStart` = topLeft; RTL `topStart` = topRight).

---

## 5.7 Elevation (Material 3 — 6 level + tonal surface)

```json
{
  "elevation": {
    "level0": 0,
    "level1": 1,
    "level2": 3,
    "level3": 6,
    "level4": 8,
    "level5": 12
  }
}
```

### 5.7.1 Tonal surface

M3 elevation is expressed as shadow plus surface tint. The surface tint composites `color.surfaceTint` over the surface at elevation-proportional opacities (level1 5%, level2 8%, level3 11%, level4 12%, level5 14%).

`elevation.shadow` and `elevation.tint` may be split explicitly (optional):

```json
{
  "elevation": {
    "level0": { "shadow": 0, "tint": 0 },
    "level3": { "shadow": 6, "tint": 0.11 }
  }
}
```

The shorthand (single number) implies tonal surface auto.

---

## 5.8 Motion (Material 3 — 13 duration + 4 easing)

```json
{
  "motion": {
    "duration": {
      "short1": 50, "short2": 100, "short3": 150, "short4": 200,
      "medium1": 250, "medium2": 300, "medium3": 350, "medium4": 400,
      "long1": 450, "long2": 500, "long3": 550, "long4": 600,
      "extraLong": 1000
    },
    "easing": {
      "standard":    "cubic-bezier(0.2, 0.0, 0.0, 1.0)",
      "emphasized":  "cubic-bezier(0.2, 0.0, 0.0, 1.0)",
      "decelerate":  "cubic-bezier(0.0, 0.0, 0.0, 1.0)",
      "accelerate":  "cubic-bezier(0.3, 0.0, 1.0, 1.0)"
    }
  }
}
```

Durations are in ms. Easings are a cubic-bezier string or a named alias (`standard`, etc.).

---

## 5.9 Density

```json
{
  "density": {
    "comfortable": { "vertical": 0,  "horizontal": 0 },
    "standard":    { "vertical": -1, "horizontal": -1 },
    "compact":     { "vertical": -2, "horizontal": -2 }
  }
}
```

Values are Material `VisualDensity` vertical / horizontal (range -4 to 4). `standard` suits desktop, `comfortable` suits mobile, `compact` suits dense data tables.

The active density is selected via `theme.density.active` (`comfortable` / `standard` / `compact`). Auto-selection by form factor is a runtime policy.

---

## 5.10 Breakpoints (Material 3 — 5 class)

```json
{
  "breakpoints": {
    "compact":      0,
    "medium":     600,
    "expanded":   840,
    "large":     1200,
    "extraLarge": 1600
  }
}
```

Values are logical-px width thresholds, bindable via `{{theme.breakpoints.medium}}`. Runtime layout branching uses the `FormFactor.of(context)` resolver.

---

## 5.11 Border / Opacity / FocusRing / Z-index

### 5.11.1 Border

```json
{
  "border": {
    "width": {
      "hairline": 0.5,
      "thin":     1,
      "normal":   1.5,
      "thick":    2,
      "heavy":    4
    },
    "style": "solid"
  }
}
```

### 5.11.2 Opacity scale

```json
{
  "opacity": {
    "0":   0,
    "4":   0.04,
    "8":   0.08,
    "12":  0.12,
    "16":  0.16,
    "24":  0.24,
    "38":  0.38,
    "50":  0.50,
    "64":  0.64,
    "87":  0.87,
    "100": 1.0
  }
}
```

### 5.11.3 Focus ring

```json
{
  "focusRing": {
    "color":  "{{theme.color.primary}}",
    "width":  2,
    "offset": 2,
    "radius": 8
  }
}
```

`color` is a color value or a binding. The ring is drawn around the widget's outer edge on keyboard focus.

### 5.11.4 Z-index

```json
{
  "zIndex": {
    "base":     0,
    "dropdown": 100,
    "sticky":   200,
    "overlay":  300,
    "modal":    400,
    "popover":  500,
    "tooltip":  600,
    "toast":    700,
    "system":   999
  }
}
```

Values are stacking-context integers (compatible with CSS z-index).

---

## 5.12 Component tokens

Component-level token bundles. Common-component shared properties live in one place, allowing layered override.

```json
{
  "component": {
    "button": {
      "height":     40,
      "padding":    { "horizontal": 16, "vertical": 8 },
      "radius":     "{{theme.shape.full}}",
      "elevation":  "{{theme.elevation.level0}}",
      "typography": "{{theme.typography.labelLarge}}"
    },
    "input": {
      "height":     56,
      "radius":     "{{theme.shape.extraSmall}}",
      "borderWidth": "{{theme.border.width.thin}}",
      "typography": "{{theme.typography.bodyLarge}}"
    },
    "card": {
      "radius":    "{{theme.shape.medium}}",
      "elevation": "{{theme.elevation.level1}}",
      "padding":   "{{theme.spacing.cardPadding}}"
    },
    "dialog": {
      "radius":    "{{theme.shape.extraLarge}}",
      "elevation": "{{theme.elevation.level3}}",
      "padding":   "{{theme.spacing.lg}}"
    },
    "menu": {
      "radius":    "{{theme.shape.extraSmall}}",
      "elevation": "{{theme.elevation.level2}}",
      "padding":   "{{theme.spacing.xs}}"
    },
    "list": {
      "itemHeight":  56,
      "denseHeight": 40
    }
  }
}
```

Component tokens are free-form, and unspecified components may be added. A runtime SHOULD recognize the 6 standard components shown above.

---

## 5.13 Theme binding

All tokens resolve through the `theme.*` binding prefix:

```json
{
  "type": "box",
  "backgroundColor": "{{theme.color.surface}}",
  "borderRadius":    "{{theme.shape.medium}}",
  "padding":         "{{theme.spacing.md}}",
  "elevation":       "{{theme.elevation.level1}}"
}
```

```json
{
  "type": "text",
  "text": "Title",
  "style": "{{theme.typography.titleLarge}}",
  "color": "{{theme.color.onSurface}}"
}
```

When `mode` is `system`, `{{theme.color.primary}}` automatically resolves to the light or dark scheme value based on host brightness.

---

## 5.14 Page-level overrides

A `PageDefinition`'s `themeOverride` deep-merges over the application theme.

```json
{
  "type": "page",
  "themeOverride": {
    "color": { "primary": "#4CAF50" },
    "typography": { "titleLarge": { "fontSize": 24 } }
  },
  "content": { "...": "..." }
}
```

### 5.14.1 Resolution order

When resolving `{{theme.X}}`:

1. Page `themeOverride`
2. Application `theme`
3. Runtime defaults (M3 / DTCG default)

---

## 5.15 Client theme *(host environment exposure)*

The host (OS / embedding app) theme is exposed read-only via the `client.theme.*` prefix.

| Binding | Type | Description |
|---|---|---|
| `client.theme.mode` | `'light' \| 'dark'` | Host brightness |
| `client.theme.color.<role>` | string | All 28 M3 roles exposed |
| `client.theme.typography.<role>` | object | The host's default text style |

Use when the application wants the host look-and-feel. Otherwise use `theme.*`.

---

## 5.16 Theme preset *(since v1.3)*

`theme.preset` selects a curated content-app theme bundle that ships with the runtime. The preset is applied first as a base; individual `theme.*` fields then layer overrides on top — authors pick a mood and tweak only one or two tokens rather than building the entire theme from scratch.

| Preset | Mood | Color tone | Typography character | Spacing |
|---|---|---|---|---|
| `warm` | Book-shelf paper tones | Warm neutrals (cream, terracotta) | Serif | Cozy |
| `cool` | Album / streaming ambient | Cool blues, deep neutrals | Sans-serif | Standard |
| `sepia` | Long-form reading | Aged-paper background | Serif, tall line-height | Narrow content column |
| `mono` | Typography-led, neutral grays | Generous line-height, minimal accents | Sans-serif | Open |
| `highContrast` | Accessibility-first (WCAG AAA) | Maximum-contrast pairs | Bold weights, enlarged hit targets | Standard |

```json
{
  "theme": {
    "preset": "sepia",
    "color": { "primary": "#8B4513" }
  }
}
```

The above starts with the `sepia` bundle and overrides only `color.primary`.

---

## 5.17 Font registry *(since v1.3)*

`theme.fonts` registers font assets the runtime should make available for `text.style.fontFamily`. Map of family name → declaration. Each declaration carries:

- **`weights`** — map of weight value (`100`..`900` or named `regular`/`bold`/etc.) to `AssetRef`. Set per file you want to ship.
- **`variableAxes`** — list of variable-font axis tunings. Each entry is `{ tag, min, max, default }` for axes like `wght` (weight), `wdth` (width), `opsz` (optical size), `ital` (italic), `slnt` (slant). When set, `weights` becomes optional — the runtime drives the `wght` axis from `text.style.fontWeight`.
- **`fallbacks`** — ordered family-name list used when a glyph isn't in the primary font (CJK, emoji, math).

```json
{
  "theme": {
    "fonts": {
      "Inter": {
        "variableAxes": [
          { "tag": "wght", "min": 100, "max": 900, "default": 400 },
          { "tag": "opsz", "min": 14,  "max": 32,  "default": 16  }
        ],
        "fallbacks": ["NotoSansKR", "AppleSDGothicNeo"]
      },
      "Caveat": {
        "weights": {
          "400": "bundle://fonts/Caveat-Regular.ttf",
          "700": "bundle://fonts/Caveat-Bold.ttf"
        }
      }
    }
  }
}
```

---

## 5.18 DTCG JSON interchange

The mcp_ui 1.3 theme exports and imports the [W3C Design Tokens Community Group draft][dtcg] JSON format.

```json
{
  "$description": "MCP UI theme — exported via mcp_ui 1.3",
  "color": {
    "primary":   { "$type": "color", "$value": "#3F51B5" },
    "onPrimary": { "$type": "color", "$value": "#FFFFFF" }
  },
  "spacing": {
    "md": { "$type": "dimension", "$value": "16px" }
  },
  "typography": {
    "titleLarge": {
      "$type": "typography",
      "$value": {
        "fontFamily": "Inter",
        "fontSize":   "22px",
        "fontWeight": 500,
        "lineHeight": "28px",
        "letterSpacing": "0px"
      }
    }
  }
}
```

Detailed spec and mapping live in [`05b_DTCG_Interchange.md`](05b_DTCG_Interchange.md). The reference / alias / component 3-tier model is in [`05a_Tokens_Reference.md`](05a_Tokens_Reference.md).

[dtcg]: https://tr.designtokens.org/format/

---

## 5.17 Migration from earlier drafts

Earlier draft naming (`theme.colorScheme.light.primary`, `theme.spacing.small`, `theme.typography.headline1`, etc.) is **removed in 1.3**. Applications switch to the new names:

| Earlier draft | 1.3 |
|---|---|
| `theme.colorScheme.light.primary` | `theme.color.primary` (or light override) |
| `theme.colorScheme.*.textOnPrimary` | `theme.color.onPrimary` |
| `theme.colorScheme.*.divider` | `theme.color.outline` |
| `theme.typography.headline1` | `theme.typography.displaySmall` (or designer's choice) |
| `theme.typography.subtitle1` | `theme.typography.titleMedium` |
| `theme.typography.button` | `theme.typography.labelLarge` |
| `theme.typography.overline` | `theme.typography.labelSmall` |
| `theme.spacing.small/medium/large` | `theme.spacing.sm/md/lg` |
| `theme.borderRadius.*` | `theme.shape.*` |
| `theme.elevation.small/medium/large` | `theme.elevation.level1/level2/level3` |

Migration is a one-time switch at 1.3 publish. No legacy reader is provided.

---

## 5.18 Conformance summary

Normative requirements live in [`18_Conformance.md`](18_Conformance.md) §18.2.7. Summary:

- Runtime MUST: M3 28 color roles, 15 typography roles, 9 spacing slots, 7 shape families, 6 elevation levels, dynamic tracking of `system` mode, DTCG JSON import.
- Runtime SHOULD: HCT seed auto-derivation, state layer, DTCG JSON export, Motion, Density, Border / Opacity / FocusRing / Z-index, Component tokens.
- Runtime MAY: additional spacing slots, additional components, custom semantic roles.
