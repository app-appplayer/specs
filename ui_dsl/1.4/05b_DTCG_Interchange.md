# 05b. DTCG JSON Interchange

The mcp_ui 1.3 theme exports and imports the [W3C Design Tokens Community Group draft][dtcg] JSON format, providing two-way compatibility with [Tokens Studio][ts], [Style Dictionary][sd], [Claude Design][cd], and Figma plugins.

[dtcg]: https://tr.designtokens.org/format/
[ts]: https://tokens.studio/
[sd]: https://amzn.github.io/style-dictionary/
[cd]: https://www.anthropic.com/news/claude-design-anthropic-labs

---

## B.1 DTCG core shape

The standard DTCG token shape:

```json
{
  "$type": "<type>",
  "$value": <value>,
  "$description": "<optional human description>"
}
```

Token groups nest as plain JSON objects.

---

## B.2 mcp_ui ↔ DTCG type mapping

| mcp_ui category | DTCG `$type` | `$value` form |
|---|---|---|
| Color (28 role + semantic + palette stop) | `color` | `#RRGGBB` / `#AARRGGBB` / `rgba(...)` |
| Spacing slot, Border width, Layout primitives | `dimension` | `"<n>px"` (string with unit) |
| Shape radius | `dimension` | `"<n>px"` |
| Elevation | `dimension` | `"<n>px"` (or `number` for unitless) |
| Motion duration | `duration` | `"<n>ms"` |
| Motion easing | `cubicBezier` | `[x1, y1, x2, y2]` |
| Typography role | `typography` | object — fontFamily / fontSize / fontWeight / lineHeight / letterSpacing |
| Font family | `fontFamily` | `string` or `string[]` |
| Font weight | `fontWeight` | `100`-`900` or alias |
| Opacity scale | `number` | 0.0 – 1.0 |
| Z-index | `number` | integer |
| Density | `number` | -4 to 4 |
| Breakpoints | `dimension` | `"<n>px"` |
| Component token | composite (free-form) | nested child tokens |

This spec uses 9 of the 13 standard DTCG draft types: `color`, `dimension`, `duration`, `cubicBezier`, `typography`, `fontFamily`, `fontWeight`, `number`, `string`.

---

## B.3 Export schema

The standard structure when mcp_ui 1.3 emits DTCG JSON:

```json
{
  "$description": "MCP UI 1.3 theme export",
  "color": {
    "primary":            { "$type": "color", "$value": "#3F51B5" },
    "onPrimary":          { "$type": "color", "$value": "#FFFFFF" },
    "primaryContainer":   { "$type": "color", "$value": "#DDE1FF" },
    "onPrimaryContainer": { "$type": "color", "$value": "#001A41" },
    "...": "...",
    "stateLayer": {
      "hover":    { "$type": "number", "$value": 0.08 },
      "focus":    { "$type": "number", "$value": 0.12 },
      "pressed":  { "$type": "number", "$value": 0.16 },
      "disabled": { "$type": "number", "$value": 0.38 }
    }
  },
  "typography": {
    "displayLarge": {
      "$type": "typography",
      "$value": {
        "fontFamily":    "Inter",
        "fontSize":      "57px",
        "fontWeight":    400,
        "lineHeight":    "64px",
        "letterSpacing": "-0.25px"
      }
    },
    "...": "..."
  },
  "spacing": {
    "xxs": { "$type": "dimension", "$value": "2px" },
    "xs":  { "$type": "dimension", "$value": "4px" },
    "sm":  { "$type": "dimension", "$value": "8px" },
    "md":  { "$type": "dimension", "$value": "16px" },
    "lg":  { "$type": "dimension", "$value": "24px" },
    "xl":  { "$type": "dimension", "$value": "32px" },
    "2xl": { "$type": "dimension", "$value": "48px" },
    "3xl": { "$type": "dimension", "$value": "64px" },
    "4xl": { "$type": "dimension", "$value": "96px" },
    "screenPadding": { "$type": "dimension", "$value": "16px" },
    "cardPadding":   { "$type": "dimension", "$value": "16px" },
    "sectionGap":    { "$type": "dimension", "$value": "24px" },
    "inlineGap":     { "$type": "dimension", "$value": "8px" }
  },
  "shape": {
    "none":       { "$type": "dimension", "$value": "0px" },
    "extraSmall": { "$type": "dimension", "$value": "4px" },
    "small":      { "$type": "dimension", "$value": "8px" },
    "medium":     { "$type": "dimension", "$value": "12px" },
    "large":      { "$type": "dimension", "$value": "16px" },
    "extraLarge": { "$type": "dimension", "$value": "28px" },
    "full":       { "$type": "dimension", "$value": "9999px" }
  },
  "elevation": {
    "level0": { "$type": "dimension", "$value": "0px" },
    "level1": { "$type": "dimension", "$value": "1px" },
    "level2": { "$type": "dimension", "$value": "3px" },
    "level3": { "$type": "dimension", "$value": "6px" },
    "level4": { "$type": "dimension", "$value": "8px" },
    "level5": { "$type": "dimension", "$value": "12px" }
  },
  "motion": {
    "duration": {
      "short1":  { "$type": "duration", "$value": "50ms" },
      "short2":  { "$type": "duration", "$value": "100ms" },
      "short3":  { "$type": "duration", "$value": "150ms" },
      "short4":  { "$type": "duration", "$value": "200ms" },
      "medium1": { "$type": "duration", "$value": "250ms" },
      "medium2": { "$type": "duration", "$value": "300ms" },
      "medium3": { "$type": "duration", "$value": "350ms" },
      "medium4": { "$type": "duration", "$value": "400ms" },
      "long1":   { "$type": "duration", "$value": "450ms" },
      "long2":   { "$type": "duration", "$value": "500ms" },
      "long3":   { "$type": "duration", "$value": "550ms" },
      "long4":   { "$type": "duration", "$value": "600ms" },
      "extraLong": { "$type": "duration", "$value": "1000ms" }
    },
    "easing": {
      "standard":   { "$type": "cubicBezier", "$value": [0.2, 0.0, 0.0, 1.0] },
      "emphasized": { "$type": "cubicBezier", "$value": [0.2, 0.0, 0.0, 1.0] },
      "decelerate": { "$type": "cubicBezier", "$value": [0.0, 0.0, 0.0, 1.0] },
      "accelerate": { "$type": "cubicBezier", "$value": [0.3, 0.0, 1.0, 1.0] }
    }
  },
  "density": {
    "comfortable": { "vertical": { "$type": "number", "$value": 0  }, "horizontal": { "$type": "number", "$value": 0  } },
    "standard":    { "vertical": { "$type": "number", "$value": -1 }, "horizontal": { "$type": "number", "$value": -1 } },
    "compact":     { "vertical": { "$type": "number", "$value": -2 }, "horizontal": { "$type": "number", "$value": -2 } }
  },
  "breakpoints": {
    "compact":    { "$type": "dimension", "$value": "0px" },
    "medium":     { "$type": "dimension", "$value": "600px" },
    "expanded":   { "$type": "dimension", "$value": "840px" },
    "large":      { "$type": "dimension", "$value": "1200px" },
    "extraLarge": { "$type": "dimension", "$value": "1600px" }
  },
  "border": {
    "width": {
      "hairline": { "$type": "dimension", "$value": "0.5px" },
      "thin":     { "$type": "dimension", "$value": "1px" },
      "normal":   { "$type": "dimension", "$value": "1.5px" },
      "thick":    { "$type": "dimension", "$value": "2px" },
      "heavy":    { "$type": "dimension", "$value": "4px" }
    }
  },
  "opacity": {
    "8":  { "$type": "number", "$value": 0.08 },
    "12": { "$type": "number", "$value": 0.12 },
    "...": "..."
  },
  "focusRing": {
    "color":  { "$type": "color",     "$value": "{color.primary}" },
    "width":  { "$type": "dimension", "$value": "2px" },
    "offset": { "$type": "dimension", "$value": "2px" },
    "radius": { "$type": "dimension", "$value": "8px" }
  },
  "zIndex": {
    "base":     { "$type": "number", "$value": 0 },
    "dropdown": { "$type": "number", "$value": 100 },
    "sticky":   { "$type": "number", "$value": 200 },
    "overlay":  { "$type": "number", "$value": 300 },
    "modal":    { "$type": "number", "$value": 400 },
    "popover":  { "$type": "number", "$value": 500 },
    "tooltip":  { "$type": "number", "$value": 600 },
    "toast":    { "$type": "number", "$value": 700 },
    "system":   { "$type": "number", "$value": 999 }
  },
  "component": {
    "button": {
      "background":  { "$type": "color",     "$value": "{color.primary}" },
      "foreground":  { "$type": "color",     "$value": "{color.onPrimary}" },
      "radius":      { "$type": "dimension", "$value": "{shape.full}" },
      "typography":  { "$type": "typography","$value": "{typography.labelLarge}" }
    },
    "card": {
      "background": { "$type": "color",     "$value": "{color.surface}" },
      "radius":     { "$type": "dimension", "$value": "{shape.medium}" },
      "elevation":  { "$type": "dimension", "$value": "{elevation.level1}" }
    }
  }
}
```

---

## B.4 Import — DSL author usage

### B.4.1 Author DTCG JSON directly

An application may write its `theme` field directly in the DTCG form above. The runtime interprets it equivalently.

### B.4.2 Tokens Studio / Figma export

Export from Figma via the Tokens Studio plugin and paste the result into the application's `theme` field. The mcp_ui 1.3 runtime imports it directly.

### B.4.3 Style Dictionary build

Use the mcp_ui DTCG JSON as a Style Dictionary source to emit CSS, Sass, iOS, or Android. One source produces multi-platform output.

### B.4.4 Claude Design import

Claude Design scans a codebase, extracts tokens, converts them to DTCG JSON, and imports as the mcp_ui `theme`.

---

## B.5 Reference syntax

The DTCG draft alias reference notation `{path.to.token}` is supported as-is. It is distinct from mcp_ui binding (`{{theme.x.y}}`):

| Syntax | Used in |
|---|---|
| `{path}` | Alias reference inside DTCG JSON (during export/import) |
| `{{theme.x.y}}` | Widget property binding in the DSL (at runtime) |

During runtime import, all `{path}` references are resolved into raw values or aliases.

---

## B.6 Conformance

- Runtime MUST: import the 9 categories listed in the B.3 export schema (`color`, `typography`, `spacing`, `shape`, `elevation`, `motion`, `density`, `breakpoints`, `component`).
- Runtime SHOULD: support all 13 DTCG types plus the `border`, `opacity`, `focusRing`, `zIndex` categories.
- Runtime MAY: import custom component tokens; ignore unknown `$type` values.

Normative detail is in [`18_Conformance.md`](18_Conformance.md) §18.2.7.

---

## B.7 Tool compatibility

| Tool | Compatibility |
|---|---|
| Tokens Studio (Figma) | ✓ DTCG draft (W3C) — direct export/import |
| Style Dictionary | ✓ DTCG → CSS / Sass / iOS / Android |
| Claude Design | ✓ codebase scan → DTCG JSON extraction |
| Figma Variables (native) | ✓ DTCG plugin |
| Penpot | ✓ DTCG export |
| MCP UI runtime | ✓ native |
