# 02. Widgets

**Status:** Normative.

> **SSOT:** The machine-readable widget registry at [`widgets/<category>/<type>.yaml`](widgets/) is authoritative. The prose rows in this file are regenerated from it. If YAML and prose disagree, YAML wins. Generated reference artifacts: [`generated/widgets.md`](generated/widgets.md), [`schema/widgets.schema.json`](schema/widgets.schema.json).

This section defines every widget in the Core Profile. The full canonical name list is also summarized in [`17_Naming.md`](17_Naming.md) §17.2.1; all aliases are registered in §17.3. Required widget sets are anchored in [`18_Conformance.md`](18_Conformance.md) §18.2.1. Advanced widgets are defined in [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md).

## 2.1 Widget Categories

- **Layout:** `box`, `linear`, `stack`, `center`, `align`, `padding`, `margin`, `expanded`, `flexible`, `spacer`, `wrap`, `positioned`, `safeArea`, `sizedBox`, `aspectRatio`, `constrained`, `fractionallySized`, `intrinsicHeight`, `intrinsicWidth`, `visibility`, `conditional`, `indexedStack`
- **Display:** `text`, `richText`, `image`, `icon`, `card`, `divider`, `verticalDivider`, `badge`, `chip`, `avatar`, `tooltip`, `placeholder`, `banner`, `progressBar`, `decoration`
- **Input:** `button`, `iconButton`, `textInput`, `toggle`, `select`, `checkbox`, `checkboxGroup`, `radio`, `radioGroup`, `slider`, `rangeSlider`, `numberField`, `dateField`, `timeField`, `datePicker`, `timePicker`, `dateRangePicker`, `colorPicker`, `segmentedControl`, `stepper`, `numberStepper`, `rating`, `form`
- **List:** `list`, `grid`, `listItem`
- **Navigation:** `headerBar`, `bottomNavigation`, `tabBar`, `tabBarView`, `drawer`, `navigationRail`, `floatingActionButton`, `popupMenuButton`
- **Scroll:** `scrollView`, `singleChildScrollView`, `scrollBar`, `pageView`
- **Interaction:** `gestureDetector`, `inkWell`, `draggable`, `dragTarget`
- **Dialog:** `alertDialog`, `simpleDialog`, `customDialog`, `snackBar`, `bottomSheet`
- **Animation:** `animatedContainer`, `opacity` *(since v1.3)*, `transform` *(since v1.3)*, `lottieAnimation`
- **Advanced:** see [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md)
- **Template:** `use`, `template` — see [`09_Templates.md`](09_Templates.md)
- **Accessibility:** `accessibleWrapper` — see [`13_Accessibility.md`](13_Accessibility.md)
- **Utility:** `lazy`, `fittedBox`, `clipOval`, `clipRRect`

## 2.2 Common Properties

Every widget accepts the following common properties:

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `type` | string | yes | — | Widget type discriminator. Canonical name only. |
| `child` | Widget | no | — | Single-child slot for single-child widgets. |
| `children` | Widget[] | no | — | Multi-child slot for multi-child widgets. |
| `visible` | boolean \| binding | no | `true` | Whether the widget is rendered. |
| `tooltip` | string \| binding | no | — | Hover / long-press tooltip text. Runtime wraps the widget in a `Tooltip` surface. |
| `click` | Action \| binding | no | — | Action fired when the widget is tapped. Runtime wraps the widget in a gesture surface and dispatches the action on tap. Accepts any action object form (see [`04_Actions.md`](04_Actions.md)); the action may be supplied inline or via binding. Widget-local activation surfaces (e.g. `button.onTap`, `iconButton.onTap`, `richText.spans[].onTap`) remain canonical for those widgets and are NOT replaced by `click`; `click` is the universal fallback for widgets that have no dedicated activation slot. |
| `accessibility` | object | no | — | Accessibility annotations; see [`13_Accessibility.md`](13_Accessibility.md). |
| `key` | string | no | — | Stable identity hint for reconciliation. |

Single-child widgets accept either a `child` object or the first element of a `children` array; multi-child widgets require `children`.

`click` is additive on top of any widget-local tap surface. When a widget already exposes its own activation (`button.onTap`, etc.) those remain the canonical author surface; `click` simply guarantees that **any** widget — including pure layout / decoration widgets like `box`, `card`, `linear`, `stack` — can be made tappable without nesting a `gestureDetector` wrapper. The runtime wraps the widget in a gesture surface only when `click` is present; bundles that omit it are unaffected.

Layout wrappers `padding`, `margin`, `align`, and `flex` may appear on any widget placed inside a `linear` or flex context.

## 2.3 Widget Schema Conventions

- Every widget's schema table lists property name, JSON type, whether it is required, default, and description.
- Widgets introduced after v1.0 include a `since: vX.Y` note in their heading.
- Legacy alias properties are **not** listed in widget schemas; they live only in [`17_Naming.md`](17_Naming.md) §17.3.
- All examples use canonical names only. Emitters MUST emit canonical names ([`17_Naming.md`](17_Naming.md) §17.5.2).

---

## 2.4 Layout Widgets

### 2.4.1 `box`

Rectangular region with padding, margin, border, decoration, and size constraints. Analogous to the CSS box model.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `width` | number \| object | no | — | Explicit width in logical pixels or `{value, unit}`. |
| `height` | number \| object | no | — | Explicit height in logical pixels or `{value, unit}`. |
| `minWidth` | number | no | — | Minimum width constraint. Honored independently of `width`. |
| `maxWidth` | number | no | — | Maximum width constraint; caps the child's width. |
| `minHeight` | number | no | — | Minimum height constraint. |
| `maxHeight` | number | no | — | Maximum height constraint. |
| `padding` | string \| EdgeInsets | no | — | Inner spacing. String form accepts an M3 spacing token (`xxs` / `xs` / `sm` / `md` / `lg` / `xl` / `2xl` / `3xl` / `4xl`, or any custom slot in `theme.spacing`) that resolves through `theme.spacing.<token>` to a uniform inset; object form is `{all}`, `{horizontal, vertical}`, `{top, right, bottom, left}`, or `{token: "md"}`. |
| `margin` | EdgeInsets | no | — | Outer spacing. |
| `alignment` | string | no | — | Alignment of the child within the box. |
| `decoration` | object | no | — | `color`, `borderRadius`, `border`, `boxShadow`, `gradient`. |
| `color` | string | no | — | Shorthand for `decoration.color`. |
| `child` | Widget | no | — | Single child. |

```json
{
  "type": "box",
  "width": 200,
  "height": 100,
  "padding": { "all": 16 },
  "margin": { "horizontal": 8 },
  "alignment": "center",
  "decoration": {
    "color": "#FFFFFF",
    "borderRadius": 8,
    "border": { "color": "#E0E0E0", "width": 1 }
  },
  "child": { "type": "text", "text": "Hello" }
}
```

### 2.4.2 `linear`

Linear (flex) layout along a single axis. Replaces separate `row` / `column` widgets — use `direction: "horizontal"` or `direction: "vertical"`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `direction` | string | yes | `"vertical"` | `"horizontal"` or `"vertical"`. |
| `alignment` | string | no | `"start"` | Cross-axis alignment: `start`, `center`, `end`, `stretch`. |
| `distribution` | string | no | `"start"` | Main-axis distribution: `start`, `center`, `end`, `spaceBetween`, `spaceAround`, `spaceEvenly`. |
| `spacing` | number | no | `0` | Gap between children in logical pixels. |
| `children` | Widget[] | yes | — | Child widgets arranged along `direction`. |

```json
{
  "type": "linear",
  "direction": "vertical",
  "alignment": "center",
  "distribution": "start",
  "spacing": 12,
  "children": [
    { "type": "text", "text": "Title" },
    { "type": "text", "text": "Subtitle" }
  ]
}
```

```json
{
  "type": "linear",
  "direction": "horizontal",
  "distribution": "spaceBetween",
  "children": [
    { "type": "text", "text": "Left" },
    { "type": "text", "text": "Right" }
  ]
}
```

### 2.4.3 `stack`

Overlapping children. Non-positioned children align per `alignment`; positioned children use explicit offsets (see `positioned`).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `alignment` | string | no | `"topStart"` | Alignment of non-positioned children: `topStart`, `topCenter`, `topEnd`, `centerStart`, `center`, `centerEnd`, `bottomStart`, `bottomCenter`, `bottomEnd`. |
| `fit` | string | no | `"loose"` | Sizing of non-positioned children: `loose`, `expand`, `passthrough`. |
| `children` | Widget[] | yes | — | Stacked children (rendered in order; later children render on top). |

```json
{
  "type": "stack",
  "alignment": "topStart",
  "children": [
    { "type": "image", "src": "bundle://hero.png" },
    { "type": "positioned", "top": 8, "right": 8,
      "child": { "type": "badge", "label": "NEW" } }
  ]
}
```

### 2.4.4 `center`

Centers a single child within available space.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Centered widget. |

```json
{ "type": "center", "child": { "type": "text", "text": "Centered" } }
```

### 2.4.5 `align`

Aligns a single child at a specified alignment.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `alignment` | string | yes | — | `topStart`, `topCenter`, `topEnd`, `centerStart`, `center`, `centerEnd`, `bottomStart`, `bottomCenter`, `bottomEnd`. |
| `child` | Widget | yes | — | Aligned widget. |

```json
{ "type": "align", "alignment": "bottomEnd",
  "child": { "type": "icon", "icon": "close" } }
```

### 2.4.6 `padding`

Applies padding around a single child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `padding` | EdgeInsets | yes | — | Inset values. |
| `child` | Widget | yes | — | Inset widget. |

```json
{ "type": "padding", "padding": { "all": 16 },
  "child": { "type": "text", "text": "Padded" } }
```

### 2.4.7 `margin`

Applies outer margin around a single child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `margin` | EdgeInsets | yes | — | Outer margin. |
| `child` | Widget | yes | — | Child widget. |

```json
{ "type": "margin", "margin": { "all": 16 },
  "child": { "type": "text", "text": "With outer margin" } }
```

### 2.4.8 `expanded`

Flex child that expands to fill available space inside a `linear` parent.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `flex` | number | no | `1` | Flex weight. |
| `child` | Widget | yes | — | Expanded widget. |

```json
{ "type": "expanded", "flex": 2,
  "child": { "type": "box", "child": { "type": "text", "text": "Fills space" } } }
```

### 2.4.9 `flexible`

Flex child that may occupy up to its natural size.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `flex` | number | no | `1` | Flex weight. |
| `fit` | string | no | `"loose"` | `"loose"` or `"tight"`. |
| `child` | Widget | yes | — | Child widget. |

### 2.4.10 `spacer`

Flexible empty space inside a `linear` parent.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `flex` | number | no | `1` | Proportion of available space to occupy. |

```json
{ "type": "spacer", "flex": 1 }
```

### 2.4.11 `wrap`

Flow layout that wraps children to the next line when they exceed available width/height.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `direction` | string | no | `"horizontal"` | Primary flow direction. |
| `spacing` | number | no | `0` | Gap between children on the same run. |
| `runSpacing` | number | no | `0` | Gap between runs. |
| `alignment` | string | no | `"start"` | Alignment within a run. |
| `children` | Widget[] | yes | — | Children to wrap. |

### 2.4.12 `positioned`

Positions a child within a `stack` using offsets.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `top` / `right` / `bottom` / `left` | number | no | — | Distance from the corresponding edge. |
| `width` / `height` | number | no | — | Explicit dimensions. |
| `child` | Widget | yes | — | Positioned widget. |

### 2.4.13 `safeArea`

Insets children so they avoid system UI overlaps (notch, status bar, home indicator).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `top` / `bottom` / `left` / `right` | boolean | no | `true` | Whether to apply the corresponding inset. |
| `child` | Widget | yes | — | Child widget. |

```json
{ "type": "safeArea", "top": true, "bottom": true,
  "child": { "type": "text", "text": "Safe content" } }
```

### 2.4.14 `sizedBox`

Fixed-size box. Commonly used as a spacer with an explicit size.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `width` | number | no | — | Width in logical pixels. |
| `height` | number | no | — | Height in logical pixels. |
| `child` | Widget | no | — | Optional child. |

```json
{ "type": "sizedBox", "height": 24 }
```

### 2.4.15 `aspectRatio`

Sizes the child to a specific aspect ratio.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `aspectRatio` | number | yes | — | Width-to-height ratio (e.g., `1.5` for 3:2). |
| `child` | Widget | yes | — | Child widget. |

### 2.4.16 `fractionallySized`

Sizes the child as a fraction of the parent.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `widthFactor` | number | no | — | Width fraction in `0.0..1.0`. |
| `heightFactor` | number | no | — | Height fraction in `0.0..1.0`. |
| `child` | Widget | yes | — | Child widget. |

### 2.4.17 `intrinsicHeight`

Constrains a child to the intrinsic height required by its content.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Child widget. |

### 2.4.18 `intrinsicWidth`

Constrains a child to the intrinsic width required by its content.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Child widget. |

### 2.4.19 `visibility`

Shows or hides a child with optional state preservation and replacement content.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `visible` | boolean \| binding | yes | — | Whether the child is visible. |
| `maintainSize` | boolean | no | `false` | Keep space allocated when hidden. |
| `maintainState` | boolean | no | `false` | Preserve `child` state when hidden. Does not affect `replacement`, which is built fresh each time it is shown. |
| `replacement` | Widget | no | — | Widget shown in place of `child` when `visible` is `false`. When absent, the child is simply hidden in-place (respecting `maintainSize` / `maintainState`). |
| `child` | Widget | yes | — | Primary widget. |

```json
{
  "type": "visibility",
  "visible": "{{showControls}}",
  "maintainState": true,
  "child": { "type": "button", "label": "Hide me" }
}
```

### 2.4.20 `conditional`

Renders different branches based on an expression. Two forms are supported: then/else and multi-branch (`switch`/`cases`).

#### Then/else form

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `condition` | binding | yes | — | Boolean-producing expression. |
| `then` | Widget | yes | — | Rendered when `condition` is truthy. |
| `else` | Widget | no | — | Rendered when `condition` is falsy. |

```json
{
  "type": "conditional",
  "condition": "{{user.isAuthenticated}}",
  "then": { "type": "text", "text": "Welcome, {{user.name}}!" },
  "else": {
    "type": "button",
    "label": "Sign In",
    "onTap": { "type": "navigation", "action": "push", "route": "/login" }
  }
}
```

#### Multi-branch form

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `switch` | binding | yes | — | Expression whose value is compared against each case. |
| `cases` | array | yes | — | List of `{ value, child }` entries. |
| `default` | Widget | no | — | Rendered when no case matches. |

```json
{
  "type": "conditional",
  "switch": "{{status}}",
  "cases": [
    { "value": "loading", "child": { "type": "progressBar" } },
    { "value": "error",   "child": { "type": "text", "text": "Error: {{errorMessage}}" } },
    { "value": "empty",   "child": { "type": "text", "text": "No data available" } }
  ],
  "default": { "type": "text", "text": "Ready" }
}
```

### 2.4.21 `indexedStack`

Displays a single child selected by index. All children retain state.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `index` | number \| binding | no | `0` | Index of the child to display. |
| `alignment` | string | no | `"start"` | Alignment of the displayed child. |
| `children` | Widget[] | yes | — | Candidate children. |

```json
{
  "type": "indexedStack",
  "index": "{{currentStep}}",
  "children": [
    { "type": "text", "text": "Step 1" },
    { "type": "text", "text": "Step 2" },
    { "type": "text", "text": "Step 3" }
  ]
}
```

---

## 2.5 Display Widgets

### 2.5.1 `text`

Renders a string. The canonical content field is `text`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `text` | string | yes | — | Text content. Supports binding expressions. |
| `variant` | string | no | — | M3 typography role — one of `displayLarge` / `displayMedium` / `displaySmall` / `headlineLarge` / `headlineMedium` / `headlineSmall` / `titleLarge` / `titleMedium` / `titleSmall` / `bodyLarge` / `bodyMedium` / `bodySmall` / `labelLarge` / `labelMedium` / `labelSmall`. Resolves through `theme.typography.<variant>`; the inline `style` block layers on top. |
| `style` | string \| object | no | — | Inline `{ fontSize, fontWeight, color, fontFamily, letterSpacing, lineHeight, decoration }`. String form accepts a binding to a typography map (typically `"{{theme.typography.<role>}}"`, equivalent to `variant: "<role>"`). When both `variant` and an object `style` are set, `style` overrides individual fields on top of the variant base. |
| `maxLines` | number | no | — | Maximum rendered lines. |
| `overflow` | string | no | `"clip"` | `clip`, `ellipsis`, `fade`, `visible`. |
| `textAlign` | string | no | `"start"` | `start`, `center`, `end`, `justify`. |

```json
{ "type": "text", "text": "Title", "variant": "headlineLarge" }
```

```json
{
  "type": "text",
  "text": "Hello {{user.name}}",
  "style": { "fontSize": 16, "fontWeight": "bold", "color": "#333333" }
}
```

### 2.5.2 `richText`

Styled text composed of inline spans.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `spans` | Span[] | yes | — | Inline spans `{ text, style, onTap? }`. |
| `textAlign` | string | no | `"start"` | Alignment for the composed text. |

```json
{
  "type": "richText",
  "spans": [
    { "text": "Hello ", "style": { "color": "#333333" } },
    { "text": "{{user.name}}", "style": { "fontWeight": "bold" } }
  ]
}
```

### 2.5.3 `image`

Displays an image from a URL, bundle, or client resource.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `src` | string | yes | — | Image URL, `bundle://...`, or `client://...`. |
| `width` | number | no | — | Width in logical pixels. |
| `height` | number | no | — | Height in logical pixels. |
| `fit` | string | no | `"contain"` | `cover`, `contain`, `fill`, `none`, `scaleDown`, `fitHeight`, `fitWidth`. |
| `alignment` | string | no | `"center"` | Alignment within bounds. |

```json
{
  "type": "image",
  "src": "https://example.com/image.png",
  "width": 200,
  "height": 150,
  "fit": "cover"
}
```

### 2.5.4 `icon`

Displays a named icon from the runtime's icon set.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `icon` | string | yes | — | Icon name (e.g., `"home"`, `"settings"`), `http(s)://` URL, or codepoint object `{codepoint, fontFamily?, fontPackage?}`. |
| `size` | string \| number | no | — | Numeric dp, or an `AppIconSizes` token (`sm` / `md` / `lg` / `xl`) that scales with the active form factor. |
| `sizeToken` | string | no | — | Equivalent to `size` when given as a token; explicit form for tooling. |
| `color` | string | no | — | Icon color. |

```json
{ "type": "icon", "icon": "home", "sizeToken": "md", "color": "#2196F3" }
```

### 2.5.5 `card`

Elevated single-child container.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `elevation` | number \| string | no | `1` | Shadow elevation. String form accepts an M3 elevation token (`level0` … `level5`) that resolves through `theme.elevation.<token>.shadow`. |
| `margin` | EdgeInsets | no | — | Outer margin. |
| `shape` | object \| string | no | — | M3 shape family token (`extraSmall` / `small` / `medium` / `large` / `extraLarge` / `full` / `none`) that resolves through `theme.shape.<token>`, or the legacy object `{ type: "rounded", radius: 12 }`. |
| `child` | Widget | yes | — | Card content. |

```json
{
  "type": "card",
  "elevation": 2,
  "margin": { "all": 8 },
  "shape": { "type": "rounded", "radius": 12 },
  "child": { "type": "text", "text": "Card content" }
}
```

### 2.5.6 `divider`

Horizontal separator.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `thickness` | number | no | `1` | Line thickness. |
| `color` | string | no | — | Line color. |
| `indent` | number | no | — | Leading indent. |
| `endIndent` | number | no | — | Trailing indent. |

### 2.5.7 `verticalDivider`

Vertical separator, typically used in horizontal layouts.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `width` | number | no | — | Total width including spacing. |
| `thickness` | number | no | `1` | Line thickness. |
| `color` | string | no | — | Line color. |
| `indent` | number | no | — | Top indent. |
| `endIndent` | number | no | — | Bottom indent. |

### 2.5.8 `badge`

Small status indicator typically anchored to another widget.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Text content of the badge. |
| `color` | string | no | — | Badge background color. |
| `child` | Widget | no | — | Optional child the badge is attached to. |

```json
{ "type": "badge", "label": "3",
  "child": { "type": "icon", "icon": "notifications" } }
```

### 2.5.9 `chip`

Compact element representing an attribute, action, or filter.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | yes | — | Display text. |
| `avatar` | Widget | no | — | Leading widget (icon or image). |
| `selected` | boolean | no | `false` | Selected state. |
| `variant` | string | no | `"filled"` | `filled` or `outlined`. |
| `onDelete` | Action | no | — | Action when delete icon is tapped. |
| `onTap` | Action | no | — | Action when chip is tapped. |

```json
{
  "type": "chip",
  "label": "Technology",
  "avatar": { "type": "icon", "icon": "tag", "size": 16 },
  "variant": "filled",
  "onDelete": {
    "type": "state", "action": "remove",
    "binding": "selectedTags", "value": "Technology"
  }
}
```

### 2.5.10 `avatar`

Circular widget for user images or initials.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `src` | string | no | — | Image URL; if absent, falls back to `label`. |
| `label` | string | no | — | Text label (typically initials). |
| `size` | number | no | `40` | Diameter in logical pixels. |
| `color` | string | no | — | Background color when showing label. |

```json
{
  "type": "avatar",
  "src": "https://example.com/user.jpg",
  "label": "JD",
  "size": 40,
  "color": "#2196F3"
}
```

### 2.5.11 `tooltip`

Wrapper that reveals a tooltip on hover or long-press.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `message` | string | yes | — | Tooltip text. |
| `child` | Widget | yes | — | Wrapped widget. |

```json
{
  "type": "tooltip",
  "message": "Click to save your changes",
  "child": { "type": "button", "label": "Save" }
}
```

### 2.5.12 `placeholder`

Empty placeholder widget with optional dimensions and color.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `fallbackWidth` | number | no | — | Width when unconstrained. |
| `fallbackHeight` | number | no | — | Height when unconstrained. |
| `color` | string | no | — | Fill color. |
| `strokeWidth` | number | no | — | Line stroke width. |
| `child` | Widget | no | — | Optional child. |

### 2.5.13 `banner`

Persistent message banner at the top of a section or page.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `message` | string | yes | — | Banner text. |
| `severity` | string | no | `"info"` | `info`, `success`, `warning`, `error`. |
| `actions` | BannerAction[] | no | — | Action buttons `{ label, onTap }`. |

```json
{
  "type": "banner",
  "message": "Your trial expires in 3 days",
  "severity": "warning",
  "actions": [
    { "label": "Upgrade",
      "onTap": { "type": "navigation", "action": "push", "route": "/pricing" } },
    { "label": "Dismiss",
      "onTap": { "type": "state", "action": "set",
                 "binding": "showBanner", "value": false } }
  ]
}
```

### 2.5.14 `progressBar`

Progress indicator (linear or circular).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `value` | number \| binding | no | — | Progress in `0.0..1.0`; omit for indeterminate. |
| `indicatorType` | string | no | `"linear"` | `linear` or `circular`. |
| `color` | string | no | theme primary | Foreground color. |
| `backgroundColor` | string | no | — | Track color. |

```json
{
  "type": "progressBar",
  "value": 0.65,
  "indicatorType": "linear",
  "color": "#4CAF50",
  "backgroundColor": "#E0E0E0"
}
```

### 2.5.15 `decoration`

Wraps a child with a decoration box (color, gradient, border, shadow, image, blur) without otherwise affecting layout. Accepts the full `BoxDecoration` via `decoration:` OR any of its fields flat at the top level for ergonomic shorthand.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `decoration` | `BoxDecoration` | no | — | Full decoration object. Constituent fields may also appear flat. |
| `color` | Color | no | — | Flat shorthand for `decoration.color`. |
| `gradient` | `Gradient` | no | — | Flat shorthand for `decoration.gradient`. |
| `image` | `BackgroundImage` | no | — | Flat shorthand for `decoration.image`. |
| `border` | `BoxBorder` | no | — | Flat shorthand for `decoration.border`. |
| `borderRadius` | `BorderRadius` | no | — | Flat shorthand for `decoration.borderRadius`. |
| `boxShadow` | array\<`BoxShadow`\> | no | — | Flat shorthand for `decoration.boxShadow`. |
| `shape` | string | no | `"rectangle"` | `rectangle` or `circle`. |
| `backdropBlur` | number | no | — | Gaussian backdrop-filter sigma. |
| `child` | Widget | yes | — | Decorated widget. |

### 2.5.16 `kenBurnsImage` *(since v1.3)*

Image with an automatic slow zoom-and-pan animation (the "Ken Burns effect"). Used for cinematic stills in book chapter openings, landing pages, slideshow frames.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `src` | `AssetRef` | yes | — | Image source. |
| `duration` | number | no | `8000` | Total animation duration in ms. |
| `intensity` | number | no | `0.15` | Zoom amount (1.0 → 1.0 + intensity). Practical range `0.05` (subtle) .. `0.30` (dramatic). |
| `startAlignment` | `Alignment` | no | `"topStart"` | Pan start position. |
| `endAlignment` | `Alignment` | no | `"bottomEnd"` | Pan end position. |
| `loop` | boolean | no | `true` | Reverse and replay continuously when true. |
| `curve` | `AnimationCurve` | no | `"linear"` | Easing curve over the zoom/pan progression. |
| `width` / `height` | number | no | — | Render dimensions. |
| `fit` | string | no | `"cover"` | How the image scales to bounds before the Ken Burns animation overlays. |

### 2.5.17 `imageFilter` *(since v1.3)*

Wraps a child with a colour / blur filter. Filters the entire rendered subtree (distinct from `BoxDecoration.image.colorFilter` which colours an image background).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `filter` | string | yes | — | `sepia`, `grayscale`, `blur`, `saturation`, `brightness`, `contrast`, `invert`. |
| `intensity` | number | no | `1.0` | Filter strength. Range depends on `filter` (`0..1` for sepia/grayscale/invert; sigma-px for blur; multiplier for saturation/brightness/contrast). |
| `child` | Widget | yes | — | Filtered subtree. |

---

## 2.6 Input Widgets

### 2.6.0 Common input behavior

Every input widget in §2.6 (those that accept user-changeable values) MUST follow these shared wiring rules. The per-widget property tables below omit the shared rows; assume they apply to every input widget unless explicitly overridden.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `binding` | string | no | — | State path bound two-way to the widget's value. When set, the runtime reads the current value from the path and writes user input back to the same path without requiring an explicit `onChange` action. |
| `value` | (widget-specific) | no | — | One-way initial/display value. May be a literal or `{{path}}` expression. Used when `binding` is not set. |
| `enabled` | boolean | no | `true` | Whether the widget accepts user input. |
| `onChange` | Action | no | — | Fired when the widget's value changes. Inside handler `params`, the new value is available as `{{event.value}}`. |

**Precedence.** When both `binding` and `value`+`onChange` are present, the explicit `value`+`onChange` take precedence over the `binding` shorthand. This lets authors opt out for custom flows (e.g., debounced writes, side-effecting tools) while keeping the shorthand as the default.

**Missing state.** If `binding` resolves to a state path that does not yet exist, the runtime treats the current value as the widget's documented default and proceeds to write on first change.

**Conformance.** Runtimes MUST support `binding` on every input widget §2.6.3–§2.6.22 that has a user-changeable value, with the following exceptions:

- **`button`, `iconButton`** — no user-changeable value.
- **`form`** — pure container that manages aggregate state.
- **`radio` (§2.6.8)** — individual radio buttons are subcomponents; the selected value is bound by the enclosing `radioGroup` (§2.6.9). Standalone `radio` uses `groupValue`.
- **`dateRangePicker` (§2.6.17)** — uses two separate bindings (`startDate`, `endDate`) instead of a single `binding`; see that section.

For widgets whose primary value is an object (e.g., `rangeSlider` with `{start, end}`, `stepper` with the active step index via `currentStep`), `binding` applies to that object or primary field as documented in each section.

This is a normative requirement; factories that accept only `value`+`onChange` without `binding` (outside the exceptions above) are non-conformant.

### 2.6.1 `button`

Interactive button. The canonical label field is `label`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | yes | — | Button text. |
| `variant` | string | no | `"elevated"` | `elevated`, `filled`, `outlined`, `text`, `icon`. |
| `elevation` | number \| string | no | — | Shadow elevation. String form accepts an M3 elevation token (`level0` … `level5`) that resolves through `theme.elevation.<token>.shadow`. Honored only by the `elevated` variant; ignored by other variants. |
| `icon` | string | no | — | Optional leading icon name. |
| `enabled` | boolean | no | `true` | Whether the button is interactive. |
| `onTap` | Action | no | — | Tap handler. |
| `onDoubleTap` | Action | no | — | Double-tap handler. |
| `onLongPress` | Action | no | — | Long-press handler. |

```json
{
  "type": "button",
  "label": "Submit",
  "variant": "elevated",
  "onTap": { "type": "tool", "tool": "submitForm" }
}
```

### 2.6.2 `iconButton`

Icon-only button for compact actions.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `icon` | string | yes | — | Icon name. |
| `size` | number | no | — | Icon size; uses theme default if omitted. |
| `color` | string | no | — | Icon color. |
| `enabled` | boolean | no | `true` | Whether the button is interactive. |
| `onTap` | Action | no | — | Tap handler. |

```json
{
  "type": "iconButton",
  "icon": "favorite",
  "color": "#F44336",
  "onTap": { "type": "state", "action": "toggle", "binding": "isFavorite" }
}
```

### 2.6.3 `textInput`

Single-line text entry. Canonical placeholder field is `placeholder`. See §2.6.0 for shared `binding` / `value` / `enabled` / `onChange` semantics.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Field label. |
| `placeholder` | string | no | — | Placeholder text. |
| `helperText` | string | no | — | Helper text shown below the field. |
| `prefixIcon` | string | no | — | Icon displayed at the start of the field. |
| `suffixIcon` | string | no | — | Icon displayed at the end of the field. |
| `obscureText` | boolean | no | `false` | Hide input (passwords). |
| `readOnly` | boolean | no | `false` | Whether the field is read-only. |
| `maxLines` | number | no | `1` | Maximum number of lines. |
| `maxLength` | number | no | — | Maximum character length. |
| `inputType` | string | no | `"text"` | `text`, `number`, `email`, `phone`, `url`, `multiline`. |
| `validation` | ValidationConfig | no | — | Input constraints enforced before writing to state; see [`07_Security.md`](07_Security.md) §7.2.1. |
| `onSubmit` | Action | no | — | Fired when the input action button is invoked. |
| `onFocus` | Action | no | — | Fired when the field gains focus. |
| `onBlur` | Action | no | — | Fired when the field loses focus. |

```json
{
  "type": "textInput",
  "label": "Email",
  "placeholder": "Enter your email",
  "binding": "form.email",
  "inputType": "email",
  "validation": [
    { "rule": "required", "message": "Email is required" },
    { "rule": "email", "message": "Invalid email format" }
  ]
}
```

### 2.6.4 `toggle`

Binary on/off control. Shared `binding` / `value` / `enabled` / `onChange` per §2.6.0; `value` is `boolean`.

```json
{
  "type": "toggle",
  "binding": "settings.darkMode"
}
```

### 2.6.5 `select`

Single-value dropdown selection. Shared rows per §2.6.0.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `options` | Option[] | yes | — | `{ value, label, icon? }` entries. |
| `placeholder` | string | no | — | Placeholder shown when nothing is selected. |

```json
{
  "type": "select",
  "binding": "settings.language",
  "options": [
    { "value": "en", "label": "English" },
    { "value": "es", "label": "Español" },
    { "value": "fr", "label": "Français" }
  ]
}
```

### 2.6.6 `checkbox`

Boolean checkbox. Shared rows per §2.6.0; `value` is `boolean`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Optional label. |

```json
{
  "type": "checkbox",
  "label": "Receive notifications",
  "binding": "settings.notifications"
}
```

### 2.6.7 `checkboxGroup`

Multi-selection checkbox group. Shared rows per §2.6.0; `binding` holds an `array` of selected values (top-level, not per-option).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `options` | Option[] | yes | — | `{ value, label }` entries. |
| `orientation` | string | no | `"vertical"` | `vertical` or `horizontal`. |

```json
{
  "type": "checkboxGroup",
  "binding": "form.interests",
  "options": [
    { "value": "sports", "label": "Sports" },
    { "value": "music",  "label": "Music" }
  ]
}
```

### 2.6.8 `radio`

Single radio button. The group's selected value is bound via `groupValue` (or the enclosing `radioGroup`'s `binding`).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `value` | any | yes | — | This button's value. |
| `groupValue` | any \| binding | yes | — | Currently selected value in the group. |
| `label` | string | no | — | Optional label. |
| `onChange` | Action | no | — | Fired when selected. |

### 2.6.9 `radioGroup`

Single-selection radio group. Shared rows per §2.6.0.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `options` | Option[] | yes | — | `{ value, label }` entries. |
| `orientation` | string | no | `"vertical"` | `vertical` or `horizontal`. |

```json
{
  "type": "radioGroup",
  "binding": "settings.language",
  "options": [
    { "value": "en", "label": "English" },
    { "value": "es", "label": "Español" }
  ]
}
```

### 2.6.10 `slider`

Continuous single-value selection. Shared rows per §2.6.0; `value` is `number`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `min` | number | no | `0` | Minimum value. |
| `max` | number | no | `1` | Maximum value. |
| `divisions` | number | no | — | Number of discrete steps. |

```json
{
  "type": "slider",
  "binding": "settings.volume",
  "min": 0, "max": 100, "divisions": 10
}
```

### 2.6.11 `rangeSlider`

Range selection with two thumbs. Shared rows per §2.6.0; `value` is an object `{ start, end }`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `min` | number | no | `0` | Minimum value. |
| `max` | number | no | `1` | Maximum value. |
| `divisions` | number | no | — | Number of discrete steps. |

### 2.6.12 `numberField`

Specialized input for numeric values. Shared rows per §2.6.0; `value` is `number`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Field label. |
| `min` | number | no | — | Minimum value. |
| `max` | number | no | — | Maximum value. |
| `step` | number | no | `1` | Increment size. |
| `decimalPlaces` | number | no | `0` | Decimal precision. |
| `prefix` | string | no | — | Leading display text (e.g., `"$"`). |
| `suffix` | string | no | — | Trailing display text. |
| `thousandSeparator` | string | no | — | Thousands separator for display. |

### 2.6.13 `dateField`

Date input with calendar/text entry modes. Shared rows per §2.6.0; `value` is an ISO date string.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Field label. |
| `format` | string | no | `"yyyy-MM-dd"` | Display format. |
| `firstDate` | string | no | — | Earliest allowed date (ISO). |
| `lastDate` | string | no | — | Latest allowed date (ISO). |
| `mode` | string | no | `"calendar"` | `calendar`, `input`, `both`. |
| `locale` | string | no | — | Locale identifier. |

### 2.6.14 `timeField`

Time input. Shared rows per §2.6.0; `value` is a time string (e.g., `"14:30"`).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `label` | string | no | — | Field label. |
| `format` | string | no | `"HH:mm"` | Display format. |
| `use24HourFormat` | boolean | no | `true` | 24-hour clock. |
| `mode` | string | no | `"spinner"` | `spinner`, `input`, `dial`. |

### 2.6.15 `datePicker`

Standalone date picker surface. Shared rows per §2.6.0; `value` is an ISO date string.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `firstDate` | string | no | — | Earliest allowed date. |
| `lastDate` | string | no | — | Latest allowed date. |

### 2.6.16 `timePicker`

Standalone time picker surface. Shared rows per §2.6.0; `value` is a time string.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `use24HourFormat` | boolean | no | `false` | 24-hour clock. |

### 2.6.17 `dateRangePicker`

Date range selection. Exception to §2.6.0: instead of a single `binding`, the range is bound via **two separate properties** — `startDate` and `endDate` — each independently following §2.6.0 two-way semantics (each accepts a literal ISO string, a `{{path}}` expression for read-only display, or a plain state-path string for two-way binding). `firstDate`/`lastDate` remain one-way constraints on the allowed range.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `startDate` | string | yes | — | State path bound two-way to the range's start (ISO date). |
| `endDate` | string | yes | — | State path bound two-way to the range's end (ISO date). |
| `firstDate` | string | no | — | Earliest allowed date (constraint, one-way). |
| `lastDate` | string | no | — | Latest allowed date (constraint, one-way). |
| `format` | string | no | `"yyyy-MM-dd"` | Display format. |
| `locale` | string | no | — | Locale identifier. |
| `enabled` | boolean | no | `true` | Whether the picker accepts user input. |
| `onChange` | Action | no | — | Fired when the range changes. Receives `{{event.value}}` as `{ start, end }`. |

```json
{
  "type": "dateRangePicker",
  "startDate": "filter.from",
  "endDate": "filter.to",
  "firstDate": "2026-01-01",
  "lastDate": "2026-12-31"
}
```

### 2.6.18 `colorPicker`

Color selection. Shared rows per §2.6.0; `value` is a hex color string.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `showAlpha` | boolean | no | `false` | Enable alpha channel. |
| `showLabel` | boolean | no | `true` | Show hex label. |
| `pickerType` | string | no | `"wheel"` | `wheel`, `palette`, `both`. |
| `enableHistory` | boolean | no | `false` | Show recent colors. |

### 2.6.19 `segmentedControl`

Segmented selection, styled as tabs or buttons. Shared rows per §2.6.0.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `options` | Option[] | yes | — | `{ value, label, icon? }` entries. |
| `variant` | string | no | `"segmented"` | `segmented`, `tabs`, `buttons`. |

```json
{
  "type": "segmentedControl",
  "binding": "viewMode",
  "options": [
    { "value": "list", "label": "List", "icon": "list" },
    { "value": "grid", "label": "Grid", "icon": "grid_view" }
  ]
}
```

### 2.6.20 `stepper`

Step-by-step wizard. Shared `binding` / `value` / `enabled` / `onChange` per §2.6.0 — the bound value is the **active step index** (integer). `binding` and `currentStep` are aliases: when `binding` is set, it takes effect; otherwise `currentStep` is read as a one-way property.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `steps` | Step[] | yes | — | Each step: `{ title, subtitle?, state?, content, isActive? }`. |
| `currentStep` | number \| binding | no | `0` | One-way legacy property. Use §2.6.0 `binding` for two-way behavior. |
| `stepperType` | string | no | `"vertical"` | `vertical` or `horizontal`. |
| `onStepTapped` | Action | no | — | Fired when a step header is tapped. Receives `{{event.index}}`. |
| `onStepContinue` | Action | no | — | Fired when the continue button is pressed. |
| `onStepCancel` | Action | no | — | Fired when the cancel button is pressed. |

```json
{
  "type": "stepper",
  "binding": "wizard.currentStep",
  "steps": [
    { "title": "Account", "content": { "type": "text", "text": "..." } },
    { "title": "Profile", "content": { "type": "text", "text": "..." } }
  ]
}
```

### 2.6.21 `numberStepper`

Incremental numeric input with plus/minus buttons. Shared rows per §2.6.0; `value` is `number`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `min` | number | no | — | Minimum value. |
| `max` | number | no | — | Maximum value. |
| `step` | number | no | `1` | Increment size. |

### 2.6.22 `rating`

Discrete rating control (e.g., star rating). Shared rows per §2.6.0; `value` is `number`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `max` | number | no | `5` | Maximum rating value. |
| `icon` | string | no | `"star"` | Icon name for each unit. |
| `color` | string | no | — | Icon color. |

### 2.6.23 `form`

Container that manages validation state for its child inputs.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `children` | Widget[] | yes | — | Form field widgets. |
| `showErrorsOn` | string | no | `"submit"` | `submit`, `change`, `blur`. |
| `onSubmit` | Action | no | — | Fired on form submission. |

```json
{
  "type": "form",
  "showErrorsOn": "submit",
  "onSubmit": {
    "type": "tool", "tool": "submitRegistration", "params": "{{form.data}}"
  },
  "children": [
    { "type": "textInput", "label": "Email", "value": "{{form.email}}" },
    { "type": "textInput", "label": "Password",
      "value": "{{form.password}}", "obscureText": true },
    { "type": "button", "label": "Submit",
      "onTap": { "type": "form", "action": "submit" } }
  ]
}
```

---

## 2.7 List Widgets

### 2.7.1 `list`

Scrollable linear collection rendered from an array binding.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `items` | binding | yes | — | Array source. |
| `itemTemplate` | Widget | yes | — | Template rendered per item. Iteration variables `item`, `index`, `isFirst`, `isLast`, `isEven`, `isOdd` are in scope. |
| `spacing` | number | no | `0` | Gap between items. |
| `orientation` | string | no | `"vertical"` | `vertical` or `horizontal`. |
| `emptyMessage` | string | no | — | Displayed when the list is empty. |
| `itemExtent` | number | no | — | Fixed item size for performance. |

```json
{
  "type": "list",
  "items": "{{users}}",
  "spacing": 8,
  "itemTemplate": {
    "type": "listItem",
    "title": "{{item.name}}",
    "subtitle": "{{item.email}}"
  }
}
```

### 2.7.2 `grid`

Scrollable two-dimensional collection.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `items` | binding | yes | — | Array source. |
| `itemTemplate` | Widget | yes | — | Template rendered per item. |
| `columns` | number \| object | yes | — | Column count; may use responsive `{default, sm, md, lg}`. |
| `rowGap` | number | no | `0` | Gap between rows. |
| `columnGap` | number | no | `0` | Gap between columns. |
| `itemAspectRatio` | number | no | — | Fixed aspect ratio for each item. |

```json
{
  "type": "grid",
  "items": "{{products}}",
  "columns": 2,
  "rowGap": 12,
  "columnGap": 12,
  "itemAspectRatio": 0.75,
  "itemTemplate": {
    "type": "card",
    "child": {
      "type": "linear", "direction": "vertical",
      "children": [
        { "type": "image", "src": "{{item.image}}", "fit": "cover" },
        { "type": "text",  "text": "{{item.name}}" }
      ]
    }
  }
}
```

### 2.7.3 `listItem`

Single row in a list with optional leading/trailing widgets. Replaces Material `listTile` naming.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `title` | string | no | — | Primary text. |
| `subtitle` | string | no | — | Secondary text. |
| `leading` | Widget | no | — | Leading widget (icon, avatar). |
| `trailing` | Widget | no | — | Trailing widget. |
| `onTap` | Action | no | — | Tap handler. |
| `selected` | boolean | no | `false` | Selected state. |
| `enabled` | boolean | no | `true` | Whether the item is interactive. |

```json
{
  "type": "listItem",
  "title": "{{item.name}}",
  "subtitle": "{{item.description}}",
  "leading": { "type": "avatar", "label": "{{item.initials}}" },
  "onTap": {
    "type": "navigation", "action": "push",
    "route": "/detail/{{item.id}}"
  }
}
```

### 2.7.4 `staggeredGrid`

Pinterest-style masonry layout — items keep their intrinsic height and pack by column without uniform row alignment. Use for content shelves where covers have different aspect ratios. Distinct from `grid` (uniform rows) and `carousel` (single-axis scrolling).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `items` | binding | no | — | Array source. Required when `children` is omitted. |
| `itemTemplate` | Widget | no | — | Template rendered per bound item. Required with `items`. |
| `children` | Widget[] | no | — | Static cells. Mutually exclusive with `items` + `itemTemplate`. |
| `columns` | number \| object | yes | — | Column count; may use responsive `{default, sm, md, lg}`. |
| `mainAxisSpacing` | number | no | `0` | Gap along the scroll axis. |
| `crossAxisSpacing` | number | no | `0` | Gap across the scroll axis. |
| `padding` | EdgeInsets | no | — | Inner padding around the grid. |
| `scrollDirection` | string | no | `"vertical"` | `vertical` or `horizontal`. |

```json
{
  "type": "staggeredGrid",
  "items": "{{albums}}",
  "columns": { "default": 2, "md": 3, "lg": 4 },
  "mainAxisSpacing": 12,
  "crossAxisSpacing": 12,
  "itemTemplate": {
    "type": "card",
    "child": {
      "type": "image", "src": "{{item.cover}}", "fit": "cover"
    }
  }
}
```

### 2.7.5 `carousel`

Horizontally scrolling browser with optional partial-viewport framing (cover-flow / album-shelf style). Distinct from `pageView` (which always snaps a full viewport per page) — `carousel` lets authors set `viewportFraction < 1` so adjacent items peek in.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `items` | binding | no | — | Array source. Required when `children` is omitted. |
| `itemTemplate` | Widget | no | — | Per-item template. |
| `children` | Widget[] | no | — | Static slides. |
| `scrollDirection` | string | no | `"horizontal"` | `horizontal` or `vertical`. |
| `viewportFraction` | number | no | `1.0` | Slide width as a fraction of carousel width. `0.85` leaves both neighbours peeking. |
| `loop` | boolean | no | `false` | Wrap around — last → first → last. |
| `autoPlay` | number | no | — | Advance every `autoPlay` ms. Pair with `loop: true`. |
| `initialIndex` | number | no | `0` | Slide rendered first. |
| `transition` | string | no | `"slide"` | `slide`, `fade`, `coverflow`, `depth`. `coverflow`/`depth` need a perspective compositor (fall back to `slide`). |
| `indicatorPosition` | string | no | `"bottom"` | `bottom`, `top`, `none`. |
| `onPageChanged` | Action | no | — | Fires after the active slide settles. `event.page` carries the new index. |

```json
{
  "type": "carousel",
  "items": "{{books}}",
  "viewportFraction": 0.7,
  "loop": true,
  "autoPlay": 4000,
  "transition": "coverflow",
  "itemTemplate": {
    "type": "card",
    "child": { "type": "image", "src": "{{item.cover}}", "fit": "cover" }
  }
}
```

---

## 2.8 Navigation Widgets

### 2.8.1 `headerBar`

Application header / toolbar at the top of a page.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `title` | string \| Widget | no | — | Header title. |
| `leading` | Widget | no | — | Leading widget (hamburger, back arrow). |
| `actions` | Widget[] | no | — | Trailing action widgets. |
| `exitButton` | ExitButtonConfig \| boolean | no | — | Override the host-inserted close button on the root route; see below. |
| `backgroundColor` | string | no | — | Header background. |
| `elevation` | number | no | `1` | Shadow elevation. |
| `centerTitle` | boolean | no | `false` | Whether to center the title. |

**Host close button.** When the host registers an `onExit` callback and the app is on its **root route**, the runtime automatically appends a close button to `actions` at the **trailing (rightmost) edge**, after every app-defined action. Tapping it invokes the `exitApp` navigation action (see [`04_Actions.md`](04_Actions.md) §4.3.2). On inner routes, `leading` is the automatic back button and the close button is not rendered. When no `onExit` is registered, the close button is not rendered.

`exitButton` customization:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `icon` | string | `"close"` | Icon name for the button. |
| `tooltip` | string | `"Close"` | Hover / long-press tooltip. |
| `color` | string | — | Icon color; inherits AppBar foreground if omitted. |

Setting `exitButton: false` explicitly suppresses the host close button even when `onExit` is registered (for apps that provide their own exit affordance inside `actions`).

```json
{
  "type": "headerBar",
  "title": "Dashboard",
  "leading": { "type": "iconButton", "icon": "menu",
               "onTap": { "type": "navigation", "action": "openDrawer" } },
  "actions": [
    { "type": "iconButton", "icon": "search",
      "onTap": { "type": "navigation", "action": "push", "route": "/search" } }
  ]
}
```

### 2.8.2 `bottomNavigation`

Bottom navigation bar. Each item's text field is `label`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `selectedIndex` | number \| binding | yes | — | Currently selected index. |
| `items` | NavItem[] | yes | — | `{ icon, label, route? }` entries. |
| `onChange` | Action | no | — | Fired when selection changes. |

```json
{
  "type": "bottomNavigation",
  "selectedIndex": "{{currentTab}}",
  "items": [
    { "icon": "home",     "label": "Home" },
    { "icon": "search",   "label": "Search" },
    { "icon": "person",   "label": "Profile" }
  ],
  "onChange": {
    "type": "state", "action": "set",
    "binding": "currentTab", "value": "{{event.index}}"
  }
}
```

### 2.8.3 `tabBar`

Horizontal tab selector. Each tab's text field is `label`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `selectedIndex` | number \| binding | yes | — | Currently selected index. |
| `tabs` | Tab[] | yes | — | `{ label, icon? }` entries. |
| `onChange` | Action | no | — | Fired when selection changes. |

```json
{
  "type": "tabBar",
  "selectedIndex": "{{activeTab}}",
  "tabs": [
    { "label": "Overview" },
    { "label": "Activity" },
    { "label": "Settings" }
  ],
  "onChange": {
    "type": "state", "action": "set",
    "binding": "activeTab", "value": "{{event.index}}"
  }
}
```

### 2.8.4 `tabBarView`

Content area that displays widgets corresponding to the currently selected tab.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `selectedIndex` | number \| binding | no | — | Displayed index (usually bound to the same state as a `tabBar`). |
| `children` | Widget[] | yes | — | One child per tab. |

```json
{
  "type": "tabBarView",
  "selectedIndex": "{{activeTab}}",
  "children": [
    { "type": "text", "text": "Overview content" },
    { "type": "text", "text": "Activity content" },
    { "type": "text", "text": "Settings content" }
  ]
}
```

### 2.8.5 `drawer`

Side navigation drawer. Each item's text field is `label`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `items` | DrawerItem[] | yes | — | `{ icon, label, route? }` entries. |
| `header` | Widget | no | — | Drawer header widget. |
| `onSelect` | Action | no | — | Fired when an item is selected. |

```json
{
  "type": "drawer",
  "items": [
    { "icon": "dashboard", "label": "Dashboard", "route": "/dashboard" },
    { "icon": "settings",  "label": "Settings",  "route": "/settings" },
    { "icon": "person",    "label": "Profile",   "route": "/profile" }
  ],
  "onSelect": {
    "type": "navigation", "action": "push", "route": "{{event.route}}"
  }
}
```

### 2.8.6 `navigationRail`

Vertical navigation rail for tablet/desktop layouts. Each item's text field is `label`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `selectedIndex` | number \| binding | no | — | Currently selected item. |
| `items` | NavItem[] | yes | — | `{ icon, label, route? }` entries. |
| `onChange` | Action | no | — | Fired when selection changes. |

```json
{
  "type": "navigationRail",
  "selectedIndex": "{{selectedNavIndex}}",
  "items": [
    { "icon": "dashboard", "label": "Dashboard", "route": "/dashboard" },
    { "icon": "people",    "label": "Users",     "route": "/users" },
    { "icon": "settings",  "label": "Settings",  "route": "/settings" }
  ],
  "onChange": {
    "type": "navigation", "action": "push", "route": "{{event.route}}"
  }
}
```

### 2.8.7 `floatingActionButton`

Floating action button (FAB).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `icon` | string | no | — | Icon name. |
| `label` | string | no | — | Extended FAB label. |
| `onTap` | Action | yes | — | Tap handler. |

```json
{
  "type": "floatingActionButton",
  "icon": "add",
  "label": "New Item",
  "onTap": { "type": "navigation", "action": "push", "route": "/create" }
}
```

### 2.8.8 `popupMenuButton`

Button that reveals a popup menu of options.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `icon` | string | no | `"more_vert"` | Trigger icon. |
| `items` | MenuItem[] | yes | — | `{ value, label, icon?, enabled? }` entries. |
| `onSelect` | Action | no | — | Fired when an item is selected. |

```json
{
  "type": "popupMenuButton",
  "icon": "more_vert",
  "items": [
    { "value": "edit",   "label": "Edit" },
    { "value": "delete", "label": "Delete" }
  ],
  "onSelect": {
    "type": "state", "action": "set",
    "binding": "selectedAction", "value": "{{event.value}}"
  }
}
```

---

## 2.9 Scroll Widgets

### 2.9.1 `scrollView`

Scrollable viewport. Two layout modes — pick one per instance: linear mode (`child` or `children`) or sliver mode (`slivers` array).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `direction` | string | no | `"vertical"` | `vertical` or `horizontal`. |
| `padding` | EdgeInsets | no | — | Inner padding around the scrollable content. |
| `scrollPhysics` | string | no | `"clamping"` | `bouncing`, `clamping`, `neverScrollable`. |
| `child` | Widget | no | — | Single scrolled child. Mutually exclusive with `children` and `slivers`. |
| `children` | Widget[] | no | — | Multiple widgets wrapped in an implicit linear column along `direction`. |
| `slivers` | array\<Sliver\> | no | — | Sliver entries (sliverAppBar / sliverPersistentHeader / sliverList / sliverGrid / sliverFixedExtentList). Mutually exclusive with `child` and `children`. |

`Sliver` is one of five discriminated shapes — see `Sliver` in `configs/widget/Sliver.yaml`. Sliver mode unlocks collapsing app bars, sticky section headers, parallax mastheads, and mixing list/grid sections in one viewport.

```json
{
  "type": "scrollView",
  "slivers": [
    {
      "type": "sliverAppBar",
      "expandedHeight": 240,
      "pinned": true,
      "stretch": true,
      "title": { "type": "text", "text": "Library" },
      "background": { "type": "image", "src": "{{cover}}", "fit": "cover" }
    },
    {
      "type": "sliverList",
      "items": "{{books}}",
      "itemTemplate": { "type": "listItem", "title": "{{item.title}}" }
    }
  ]
}
```

### 2.9.2 `singleChildScrollView`

Lightweight scrollable wrapper for a single child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `direction` | string | no | `"vertical"` | Scroll direction. |
| `padding` | EdgeInsets | no | — | Inner padding. |
| `child` | Widget | yes | — | Scrolled content. |

```json
{
  "type": "singleChildScrollView",
  "direction": "vertical",
  "padding": { "all": 16 },
  "child": {
    "type": "linear", "direction": "vertical",
    "children": [
      { "type": "text", "text": "First section" },
      { "type": "text", "text": "Second section" }
    ]
  }
}
```

### 2.9.3 `scrollBar`

Wraps a scrollable child with a visible scroll bar.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `thumbVisibility` | boolean | no | — | Whether the scroll thumb is always visible. |
| `trackVisibility` | boolean | no | — | Whether the scroll track is always visible. |
| `thickness` | number | no | — | Scroll bar thickness. |
| `radius` | number | no | — | Scroll bar corner radius. |
| `child` | Widget | yes | — | Scrollable child. |

### 2.9.4 `pageView`

Full-viewport paged scroll view. Snaps one page per swipe; ideal for book pages, photo galleries, onboarding flows. Distinct from `carousel` (partial-viewport) — pageView is always one full page.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `direction` | string | no | `"horizontal"` | `horizontal` or `vertical`. |
| `children` | Widget[] | yes | — | One child per page. |
| `initialPage` | number | no | `0` | Page rendered first. |
| `loop` | boolean | no | `false` | Wrap around — last → first. |
| `scrollPhysics` | string | no | `"clamping"` | `bouncing`, `clamping`, `neverScrollable`. |
| `allowImplicitScrolling` | boolean | no | `false` | Pre-render adjacent pages off-screen for instant subsequent swipes. |
| `onPageChanged` | Action | no | — | Fires after the active page settles. `event.page` carries the new index. |

```json
{
  "type": "pageView",
  "direction": "horizontal",
  "children": [
    { "type": "center", "child": { "type": "text", "text": "Page 1" } },
    { "type": "center", "child": { "type": "text", "text": "Page 2" } }
  ],
  "onPageChanged": {
    "type": "state", "action": "set",
    "binding": "currentPage", "value": "{{event.page}}"
  }
}
```

---

## 2.10 Interaction Widgets

### 2.10.1 `gestureDetector`

Detects touch and pointer gestures on its child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Detected widget. |
| `onTap` | Action | no | — | Tap. |
| `onDoubleTap` | Action | no | — | Double tap. |
| `onLongPress` | Action | no | — | Long press. |
| `onPanStart` | Action | no | — | Pan start. |
| `onPanUpdate` | Action | no | — | Pan update. |
| `onPanEnd` | Action | no | — | Pan end. |

```json
{
  "type": "gestureDetector",
  "onTap": { "type": "state", "action": "set",
             "binding": "tapped", "value": true },
  "onLongPress": {
    "type": "dialog",
    "dialog": { "type": "alertDialog", "title": "Context",
                "content": "Long press detected" }
  },
  "child": {
    "type": "box", "padding": { "all": 16 },
    "child": { "type": "text", "text": "Tap or long-press me" }
  }
}
```

### 2.10.2 `inkWell`

Material-style touch feedback with ripple effect.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Tappable widget. |
| `borderRadius` | number | no | — | Ripple clip radius. |
| `onTap` | Action | no | — | Tap handler. |
| `onLongPress` | Action | no | — | Long-press handler. |

### 2.10.3 `draggable`

Drag source widget that produces drag data and feedback visuals.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `data` | any \| binding | yes | — | Payload emitted on drop. |
| `feedback` | Widget | no | — | Widget shown while dragging. |
| `childWhenDragging` | Widget | no | — | Replacement for the source while dragging. |
| `child` | Widget | yes | — | Source widget. |

```json
{
  "type": "draggable",
  "data": "{{item.id}}",
  "feedback": {
    "type": "card",
    "child": { "type": "text", "text": "{{item.name}}" }
  },
  "child": {
    "type": "listItem",
    "title": "{{item.name}}"
  }
}
```

### 2.10.4 `dragTarget`

Drop area that accepts dragged data.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `canDrop` | binding | no | — | Expression controlling acceptance. |
| `builder` | Widget | yes | — | Widget rendered as the drop surface. |
| `onDrop` | Action | no | — | Fired on successful drop; `{{event.data}}` is the payload. |
| `onDragEnter` | Action | no | — | Fired when a draggable enters. |
| `onDragLeave` | Action | no | — | Fired when a draggable leaves. |

---

## 2.11 Dialog Widgets

### 2.11.1 `alertDialog`

Modal alert with title, content, and action buttons.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `title` | string | no | — | Dialog title. |
| `content` | string \| Widget | no | — | Dialog body. |
| `actions` | DialogAction[] | no | — | `{ label, variant?, primary?, onTap }` entries. |
| `dismissible` | boolean | no | `true` | Whether tapping outside dismisses. |

```json
{
  "type": "alertDialog",
  "title": "Confirm Action",
  "content": "Are you sure you want to proceed?",
  "actions": [
    { "label": "Cancel", "variant": "text",
      "onTap": { "type": "navigation", "action": "pop" } },
    { "label": "Confirm", "variant": "elevated", "primary": true,
      "onTap": {
        "type": "batch",
        "actions": [
          { "type": "tool", "tool": "deleteItem",
            "params": { "id": "{{item.id}}" } },
          { "type": "navigation", "action": "pop" }
        ]
      } }
  ]
}
```

### 2.11.2 `simpleDialog`

Dialog presenting a list of options.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `title` | string | no | — | Dialog title. |
| `options` | Option[] | yes | — | `{ value, label, icon? }` entries. |
| `onSelect` | Action | no | — | Fired when an option is chosen. |

### 2.11.3 `customDialog`

Dialog with arbitrary widget content.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Dialog body. |
| `dismissible` | boolean | no | `true` | Whether tapping outside dismisses. |

### 2.11.4 `snackBar`

Transient notification at the bottom of the screen.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `content` | string | yes | — | Message text. |
| `duration` | number | no | `4000` | Display duration in milliseconds. |
| `action` | SnackBarAction | no | — | `{ label, onTap }` optional action. |

```json
{
  "type": "snackBar",
  "content": "Item deleted successfully",
  "duration": 4000,
  "action": {
    "label": "Undo",
    "onTap": { "type": "tool", "tool": "undoDelete",
               "params": { "id": "{{deletedItemId}}" } }
  }
}
```

### 2.11.5 `bottomSheet`

Modal bottom sheet with swipeable handle.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Sheet content. |
| `isDismissible` | boolean | no | `true` | Allow dismiss by tapping scrim. |
| `enableDrag` | boolean | no | `true` | Allow drag to dismiss. |
| `backgroundColor` | string | no | — | Sheet background. |
| `shape` | object | no | — | `{ type: "rounded", radius: { top: 16 } }`. |

---

## 2.12 Animation Widgets

### 2.12.1 `animatedContainer`

Container with implicit animations on property changes.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `duration` | number | no | `300` | Animation duration in milliseconds. |
| `curve` | `AnimationCurve` | no | `"easeInOut"` | Easing curve. |
| `width` / `height` | number | no | — | Size; changes are animated. |
| `padding` / `margin` | EdgeInsets | no | — | Spacing; changes are animated. |
| `alignment` | `Alignment` | no | — | Child alignment; changes are animated. |
| `decoration` | `BoxDecoration` | no | — | Decoration (color/gradient/border/shadow/image); changes are animated. |
| `onEnd` | Action | no | — | Fired when the animation completes. |
| `child` | Widget | no | — | Child widget. |

```json
{
  "type": "animatedContainer",
  "duration": 500,
  "curve": "easeInOut",
  "width": "{{expanded ? 300 : 100}}",
  "height": "{{expanded ? 200 : 50}}",
  "padding": { "all": 16 },
  "decoration": {
    "color": "{{expanded ? '#2196F3' : '#E0E0E0'}}",
    "borderRadius": 8
  },
  "onEnd": {
    "type": "state", "action": "set",
    "binding": "animationComplete", "value": true
  },
  "child": { "type": "center",
             "child": { "type": "text", "text": "Animated" } }
}
```

### 2.12.2 `opacity` *(since v1.3)*

Wraps a child with opacity, with optional animation.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `opacity` | number \| binding | yes | — | Opacity in `0.0..1.0`. |
| `animated` | boolean | no | `false` | Animate opacity changes. |
| `duration` | number | no | `300` | Animation duration in milliseconds (when `animated: true`). |
| `curve` | string | no | `"easeInOut"` | Animation curve (when `animated: true`). |
| `child` | Widget | yes | — | Child widget. |

```json
{
  "type": "opacity",
  "opacity": "{{item.isActive ? 1.0 : 0.3}}",
  "animated": true,
  "duration": 200,
  "child": {
    "type": "card",
    "child": { "type": "text", "text": "{{item.name}}" }
  }
}
```

### 2.12.3 `transform` *(since v1.3)*

Geometric transformation (rotate, scale, translate) applied to a child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `rotate` | number | no | `0` | Rotation in radians. |
| `scale` | number \| object | no | `1` | Uniform scale, or `{ x, y }` non-uniform scale. |
| `translate` | object | no | — | `{ x, y }` offset in logical pixels. |
| `origin` | object | no | `{ "x": 0.5, "y": 0.5 }` | Transform origin as fraction. |
| `animated` | boolean | no | `false` | Animate transform changes. |
| `duration` | number | no | `300` | Animation duration in milliseconds. |
| `curve` | string | no | `"easeInOut"` | Animation curve. |
| `child` | Widget | yes | — | Child widget. |

```json
{
  "type": "transform",
  "rotate": "{{local.isFlipped ? 3.14 : 0}}",
  "scale": "{{local.isHovered ? 1.05 : 1.0}}",
  "animated": true,
  "duration": 300,
  "child": {
    "type": "card",
    "child": { "type": "text", "text": "{{title}}" }
  }
}
```

### 2.12.4 `lottieAnimation`

Embedded Lottie/JSON animation playback.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `src` | string | yes | — | Animation source (`bundle://`, `client://`, URL). |
| `width` / `height` | number | no | — | Dimensions. |
| `autoPlay` | boolean | no | `true` | Play on mount. |
| `loop` | boolean | no | `true` | Loop playback. |

### 2.12.5 `hero` *(since v1.3)*

Shared-element transition wrapper. Wrap a child on the source page AND the destination page with the same `tag`; the runtime morphs the source bounds → destination bounds during the route transition (cover → detail magnification). Pairs naturally with `RouteTransition` styles such as `fade` or `sharedAxis`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `tag` | string | yes | — | Identifier shared between source and destination. Both sides MUST set the same tag. Unique within each page. |
| `child` | Widget | yes | — | The widget that morphs across the route boundary. |
| `transitionOnUserGestures` | boolean | no | `false` | Also morph during gesture-driven back navigation. |
| `flightShuttleBuilder` | Widget | no | — | Optional intermediate widget rendered during the morph. |

```json
{ "type": "hero", "tag": "album-{{album.id}}",
  "child": { "type": "image", "src": "{{album.cover}}", "fit": "cover" } }
```

### 2.12.6 `animatedOpacity` *(since v1.3)*

Implicitly animates the child's `opacity` whenever it changes.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `opacity` | number | yes | — | Target opacity in `[0, 1]`. Changes are tweened. |
| `duration` | number | no | `300` | Duration in ms. |
| `curve` | `AnimationCurve` | no | `"easeInOut"` | Easing. |
| `onEnd` | Action | no | — | Fires when the animation completes. |
| `child` | Widget | yes | — | Faded child. |

### 2.12.7 `animatedAlign` *(since v1.3)*

Implicitly animates a child's `alignment` within its bounds.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `alignment` | `Alignment` | yes | — | Target alignment. |
| `duration` | number | no | `300` | Duration in ms. |
| `curve` | `AnimationCurve` | no | `"easeInOut"` | Easing. |
| `onEnd` | Action | no | — | Completion handler. |
| `child` | Widget | yes | — | Aligned child. |

### 2.12.8 `animatedPositioned` *(since v1.3)*

Implicitly animates a child's edge offsets inside a `stack`. Same contract as `positioned` but the offset/size changes are tweened. Only valid as a `stack` child.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `top` / `right` / `bottom` / `left` | number | no | — | Edge offsets. |
| `width` / `height` | number | no | — | Size. |
| `duration` | number | no | `300` | Duration in ms. |
| `curve` | `AnimationCurve` | no | `"easeInOut"` | Easing. |
| `onEnd` | Action | no | — | Completion handler. |
| `child` | Widget | yes | — | Positioned child. |

### 2.12.9 `animatedDefaultTextStyle` *(since v1.3)*

Implicitly animates the default `TextStyle` applied to descendant text widgets. Each style field tweens independently.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `style` | `TextStyle` | yes | — | Target style. |
| `duration` | number | no | `300` | Duration in ms. |
| `curve` | `AnimationCurve` | no | `"easeInOut"` | Easing. |
| `child` | Widget | yes | — | Subtree whose default text style is tweened. |

### 2.12.10 `scrollAnimated` *(since v1.3)*

Animation driven by scroll position rather than time. Wraps a child and reads from an ancestor scroll view's offset; each `binding` maps a scroll-offset window to a property tween. Common uses: parallax masthead, reveal-on-scroll, shrinking title.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `bindings` | array\<object\> | yes | — | Per-property scroll bindings: `{property, fromOffset, toOffset, fromValue, toValue, curve?}`. `property` is one of `opacity`, `scale`, `translateX`, `translateY`, `rotate`. |
| `child` | Widget | yes | — | Animated subtree. |

### 2.12.11 `rive` *(since v1.3)*

Plays a Rive animation (`.riv` asset) — alternative to `lottieAnimation` when authored in Rive. State-machine inputs are driven via `inputs`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `src` | `AssetRef` | yes | — | Rive asset path. |
| `artboard` | string | no | — | Named artboard (defaults to primary). |
| `animation` | string | no | — | Named timeline animation. Mutually exclusive with `stateMachine`. |
| `stateMachine` | string | no | — | Named state machine. Mutually exclusive with `animation`. |
| `inputs` | object | no | — | Map of state-machine input name → value (`boolean` / `number` / `trigger`). |
| `fit` | string | no | `"contain"` | Scale within bounds. |
| `alignment` | `Alignment` | no | `"center"` | Alignment within bounds. |
| `width` / `height` | number | no | — | Render dimensions. |

---

## 2.13 Utility Widgets

### 2.13.1 `lazy`

Defers construction of a child until it is first rendered.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Deferred child. |

### 2.13.2 `fittedBox`

Scales and aligns its child to fit available space.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `fit` | string | no | `"contain"` | `cover`, `contain`, `fill`, `scaleDown`, `none`. |
| `alignment` | string | no | `"center"` | Alignment within the fitted bounds. |
| `child` | Widget | yes | — | Child widget. |

### 2.13.3 `clipOval`

Clips a child to an oval.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Clipped child. |

### 2.13.4 `clipRRect`

Clips a child to a rounded rectangle.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `borderRadius` | `BorderRadius` | no | `0` | Corner radius — uniform number or directional `{topStart, topEnd, bottomStart, bottomEnd, all}`. |
| `child` | Widget | yes | — | Clipped child. |

### 2.13.5 `flow`

Lightweight flowing layout that places each child along the main axis with fixed `spacing`, falling back to a new run when space is exhausted. A simpler alternative to `wrap` (§2.4.11) for spacer-style chains.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `children` | Widget[] | yes | — | Flow children. |
| `direction` | string | no | `"horizontal"` | `horizontal` or `vertical`. |
| `spacing` | number | no | `8` | Gap between children in logical pixels. |
| `alignment` | string | no | `"start"` | `start`, `center`, `end`. |

### 2.13.6 `baseline`

Aligns a child by its baseline.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `baseline` | number | yes | — | Distance from the top to the child's baseline. |
| `baselineType` | string | no | `"alphabetic"` | `alphabetic` or `ideographic`. |
| `child` | Widget | yes | — | Baselined child. |

### 2.13.7 `limitedBox`

Limits a child's size only when it would otherwise be unbounded.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `maxWidth` | number | no | `double.infinity` | Upper bound when width is unbounded. |
| `maxHeight` | number | no | `double.infinity` | Upper bound when height is unbounded. |
| `child` | Widget | yes | — | Bounded child. |

### 2.13.8 `layoutBuilder`

Runtime-only responsive container: picks a child by matching the current constraints against a breakpoint map.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `breakpoints` | object | no | — | Map of breakpoint name → threshold (e.g., `{ "sm": 480, "md": 768 }`). |
| `layouts` | object | no | — | Map of breakpoint name → Widget; matched against `breakpoints`. |
| `default` | Widget | no | — | Fallback when no breakpoint matches. Legacy alias: `child`. |

### 2.13.9 `mediaQuery`

Responsive widget that inspects the current layout context and picks a child based on media-query conditions. Supports a `then`/`else` expression form and a `breakpoints` map form. The binding engine exposes `{{media.width}}`, `{{media.height}}`, `{{media.orientation}}` inside `condition`.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `condition` | binding | no | — | Boolean expression; selects `then` when truthy. |
| `then` | Widget | no | — | Rendered when `condition` is truthy. |
| `else` | Widget | no | — | Rendered when `condition` is falsy. Legacy alias: `orElse`. |
| `breakpoints` | object | no | — | Map of breakpoint name → Widget (e.g., `{ "sm": ..., "md": ... }`). |
| `defaultChild` | Widget | no | — | Rendered when no breakpoint matches. |

### 2.13.10 `accessibleWrapper`

Wraps a child with accessibility annotations (ARIA-style). See [`13_Accessibility.md`](13_Accessibility.md) for the full property set.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Wrapped subtree. |
| `accessibility` | object | no | — | `{ label, hint, role, live }` — mirrors §13. |

### 2.13.11 `errorBoundary`

Catches render and build errors thrown by descendants and displays a fallback UI. See [`06_Runtime_Contract.md`](06_Runtime_Contract.md) for the exact recovery lifecycle.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Protected subtree. |
| `fallback` | Widget | no | — | UI shown when `child` throws. Defaults to a generic error indicator. |
| `onError` | Action | no | — | Fired with `{{event.error}}` / `{{event.stack}}` when the boundary captures an exception. |

### 2.13.12 `errorRecovery`

Retry-oriented variant of `errorBoundary`. The `handlers` map may route specific error types to distinct fallbacks.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `child` | Widget | yes | — | Protected subtree. |
| `fallback` | Widget | no | — | Default UI shown when `child` throws. |
| `handlers` | object | no | — | Map of error-type → fallback Widget. |
| `onError` | Action | no | — | Fired on capture with `{{event.error}}` / `{{event.stack}}`. |
| `showDetails` | boolean | no | `false` | Whether to render stack/error detail. |

### 2.13.13 `offlineFallback`

Swaps between an online widget and an offline fallback based on connectivity tracked by the runtime. Connectivity state may be pinned by an `isOnline` binding for tests.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `online` | Widget | no | — | Rendered while connectivity is available. |
| `offline` | Widget | no | — | Rendered while offline. |
| `message` | string | no | — | Message shown inside the offline state. |
| `icon` | string | no | — | Icon rendered in the default offline state. |
| `showRetry` | boolean | no | `true` | Whether to show a retry affordance. |
| `onRetry` | Action | no | — | Fired when the user taps retry. |
| `isOnline` | boolean \| binding | no | — | Explicit connectivity override. |

### 2.13.14 `permissionPrompt`

Presents a prompt for one or more client capabilities (e.g., clipboard, filesystem). See [`07_Security.md`](07_Security.md) for the permission vocabulary.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `permissions` | string[] | yes | — | Permission identifiers (e.g., `["client.clipboard"]`). |
| `style` | string | no | `"inline"` | `inline`, `dialog`, `banner`. |
| `title` | string | no | — | Prompt title. |
| `description` | string | no | — | Prompt body. |
| `icon` | string | no | — | Icon shown alongside the prompt. |
| `allowPartial` | boolean | no | `false` | Allow the user to grant a subset. |
| `onAllow` | Action | no | — | Fired when the user grants (all / partial per `allowPartial`). |
| `onDeny` | Action | no | — | Fired when the user denies. |

### 2.13.15 `dashboard` *(since v1.3)*

Renders a compact dashboard tile — the widget equivalent of the `ApplicationDefinition.dashboard` field (see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11). Used to embed a dashboard-mode tile inside a page (launcher / app grid scenarios).

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `content` | Widget | yes | — | Dashboard tile content tree. |
| `refreshInterval` | number | no | — | Auto-refresh period in milliseconds. Omit for static tiles. |
| `onTap` | Action | no | — | Fired when the tile is tapped (commonly wired to `openApp` — see [`04_Actions.md`](04_Actions.md) §4.3). |

---

## 2.14 Template, Accessibility, and Advanced

- **Template widgets** (`use`, `template`) — see [`09_Templates.md`](09_Templates.md).
- **`accessibleWrapper`** — see [`13_Accessibility.md`](13_Accessibility.md).
- **Advanced widgets** (`chart`, `table`, `dataTable`, `map`, `mediaPlayer`, `calendar`, `timeline`, `gauge`, `heatmap`, `tree`, `graph`, `networkGraph`, `codeEditor`, `terminal`, `fileExplorer`, `markdown`, `webView`, `signature`, `canvas`) — see [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md). Core-only runtimes MAY skip these.
