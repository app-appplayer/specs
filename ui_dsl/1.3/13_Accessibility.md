# 13. Accessibility

**Status:** Normative.
**Conformance:** [`18_Conformance.md`](18_Conformance.md) §18.2.9.
Accessibility is a Core-Profile concern. Every runtime MUST surface the semantic properties declared here to its host platform's accessibility tree (UIAccessibility on iOS, TalkBack/AccessibilityNode on Android, ARIA on Web, UIA on Windows, NSAccessibility on macOS).

---

## 13.1 The `accessibility` Object

Every widget MAY carry an `accessibility` object. All sub-fields are optional.

```json
{
  "type": "button",
  "label": "Save",
  "accessibility": {
    "label": "Save current document",
    "hint": "Saves your changes to disk",
    "role": "button",
    "live": "polite",
    "atomic": true,
    "focusable": true,
    "isHeader": false,
    "excludeFromSemantics": false,
    "focusOrder": 1,
    "focusTraversalOrder": "default",
    "focusTrap": false,
    "initialFocus": false,
    "restoreFocus": true
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `label` | string | Short name announced to assistive technology. |
| `hint` | string | Usage hint announced after the label. |
| `role` | string | Semantic role. See §13.3 for the enum. |
| `live` | string | Live-region politeness: `off` (default), `polite`, `assertive`. See §13.5. |
| `atomic` | boolean | When true, a live-region update announces the entire region rather than only changed children. Default `false`. |
| `focusable` | boolean | Whether the widget can receive keyboard focus. Defaults to the widget's natural interactivity. |
| `isHeader` | boolean | Marks the widget as a section header for screen-reader navigation. Default `false`. |
| `excludeFromSemantics` | boolean | Hides the widget from the accessibility tree. Default `false`. |
| `focusOrder` | number | Explicit tab-order index within the parent focus scope. Lower values focus first. |
| `focusTraversalOrder` | string | Traversal policy: `default` (document order), `numeric` (follow `focusOrder`), `spatial` (2D layout-aware). Default `default`. |
| `focusTrap` | boolean | Confines Tab/Shift-Tab cycling to the subtree. Used for dialogs. Default `false`. |
| `initialFocus` | boolean \| string | When `true`, this widget receives focus on mount. When a string, the widget whose `testId` equals the string receives focus. |
| `restoreFocus` | boolean | On unmount, restore focus to the previously focused widget. Default `true` inside focus traps, `false` otherwise. |

---

## 13.2 Shorthand: `semanticLabel`

The top-level property `semanticLabel` is a shorthand for `accessibility.label`.

```json
{ "type": "image", "src": "bundle://chart.png", "semanticLabel": "Q3 sales chart showing 25% growth" }
```

The two forms are semantically equivalent. When both are present, the object form wins per [`17_Naming.md`](17_Naming.md) §17.5.3 collision rule.

---

## 13.3 Roles

`accessibility.role` accepts the following values. Names follow the ARIA convention (kebab-case retained where ARIA uses it):

`button`, `checkbox`, `radio`, `switch`, `slider`, `spinbutton`, `textbox`, `combobox`, `listbox`, `menu`, `menuitem`, `tab`, `tablist`, `tabpanel`, `dialog`, `alert`, `alertdialog`, `tooltip`, `heading`, `link`, `img`, `list`, `listitem`, `navigation`, `region`, `status`, `timer`, `progressbar`, `scrollbar`, `separator`, `toolbar`, `banner`, `main`, `complementary`, `form`, `grid`, `gridcell`, `article`, `option`.

### 13.3.1 Default Role per Widget

Runtimes MUST assign these default roles when `accessibility.role` is not specified:

| Widget | Default role | Notes |
|--------|--------------|-------|
| `button` | `button` | Toggle buttons additionally expose a pressed state. |
| `iconButton` | `button` | |
| `textInput` | `textbox` | Multi-line variants expose a multiline flag. |
| `checkbox` | `checkbox` | Exposes checked state. |
| `radio` | `radio` | Within a `radioGroup` container. |
| `toggle` | `switch` | Exposes checked state. |
| `slider` | `slider` | Exposes min, max, current value. |
| `select` | `listbox` | Options expose role `option`. |
| `list` | `list` | Children expose `listitem`. |
| `grid` | `grid` | Children expose `gridcell`. |
| `image` | `img` | Requires `accessibility.label` or `semanticLabel`. |
| `text` | *(none)* | Headings expose `heading` with level. |
| `card` | `article` | |
| `tabBar` | `tablist` | Tabs expose `tab`; panels expose `tabpanel`. |
| `headerBar` | `banner` | |
| `drawer` | `navigation` | |
| `form` | `form` | |
| `alertDialog` | `alertdialog` | |
| `progressBar` | `progressbar` | Determinate bars expose current value. |

---

## 13.4 Keyboard Navigation

Runtimes MUST support keyboard traversal of all interactive widgets.

### 13.4.1 Tab Order

Tab and Shift-Tab traverse focusable widgets. Order resolves in this priority:

1. `accessibility.focusOrder` when set (ascending).
2. Otherwise, document order (widget-tree traversal).

A focus scope (any widget with `focusTraversalOrder` declared) contains its own ordering; Tab leaves the scope only after all scoped widgets are visited.

### 13.4.2 Focus Trap

A widget with `accessibility.focusTrap: true` confines Tab/Shift-Tab to its subtree. Typical use: dialogs.

```json
{
  "type": "alertDialog",
  "accessibility": {
    "focusTrap": true,
    "initialFocus": "confirm-button",
    "restoreFocus": true
  }
}
```

On dismiss, focus returns to the widget that opened the dialog when `restoreFocus: true`.

### 13.4.3 Per-Widget Key Bindings

Runtimes MUST honor these default key bindings:

| Widget | Keys | Behavior |
|--------|------|----------|
| `button`, `iconButton` | Enter, Space | Activate `onTap`. |
| `textInput` | Tab / Shift-Tab | Move focus. |
| `checkbox`, `toggle` | Space | Toggle state. |
| `radio` | Arrow Up / Down (or Left / Right) | Move selection within group. |
| `select` | Arrow Up / Down, Enter | Navigate options, confirm. |
| `slider`, `rangeSlider` | Arrow Left / Right | Decrement / increment. |
| `tabBar` | Arrow Left / Right | Switch tabs. |
| `list`, `grid` | Arrow keys | Move between items. |
| Any dialog | Escape | Dismiss. |
| `drawer` | Escape | Close. |

---

## 13.5 Live Regions

`accessibility.live` controls how dynamic content is announced.

| Value | Behavior |
|-------|----------|
| `off` | No live announcement. Default. |
| `polite` | Announce when assistive technology is idle. |
| `assertive` | Announce immediately, interrupting current speech. |

`accessibility.atomic: true` causes the region to announce its entire content rather than only the changed subtree.

### 13.5.1 Recommended Usage

| Context | `live` | `atomic` |
|---------|--------|----------|
| Error / validation messages | `assertive` | `false` |
| Status / toast notifications | `polite` | `true` |
| Counters and badges | `polite` | `true` |
| Timers | `off` (announce on demand) | — |
| Dialog openings | *(use role `alertdialog` instead)* | — |

---

## 13.6 Touch Targets

All interactive widgets MUST render with a minimum hit region of **48×48dp** (density-independent pixels). Where the visual element is smaller, the runtime MUST expand the gesture area invisibly to reach the minimum; the visual appearance is unchanged.

This applies to `button`, `iconButton`, `checkbox`, `radio`, `toggle`, `listItem`, `tab`, and any widget that receives tap/click gestures.

---

## 13.7 `accessibleWrapper` Widget

`accessibleWrapper` attaches accessibility properties to a subtree that does not natively accept an `accessibility` object. The wrapper introduces no layout of its own.

```json
{
  "type": "accessibleWrapper",
  "accessibility": {
    "label": "Product image: Blue Widget",
    "hint": "Activates the details view",
    "role": "button",
    "live": "polite",
    "isHeader": false,
    "excludeFromSemantics": false,
    "focusable": true,
    "focusOrder": 3
  },
  "child": { "type": "image", "src": "bundle://widget.png" }
}
```

All fields from §13.1 are accepted. The `semanticLabel` shorthand is also accepted.

---

## 13.8 Semantics Inheritance

- `accessibility.excludeFromSemantics: true` hides the widget and its descendants from the accessibility tree unless a descendant explicitly sets `excludeFromSemantics: false`.
- `accessibility.isHeader: true` promotes the widget to a heading entry point. Screen readers MAY offer heading-level navigation based on these markers.
- Parent `focusTraversalOrder` scopes the `focusOrder` of descendants. Nested scopes are independent.

---

## 13.9 Legacy Forms

The following non-canonical names are accepted as aliases of the corresponding canonical `accessibility.*` fields:

`ariaLabel`, `aria-label`, `ariaHint`, `aria-hint`, `ariaRole`, `aria-role`, `ariaLive`, `aria-live`, `role` (as a top-level widget property), `semanticLabel` (as a top-level shorthand for `accessibility.label`).

The canonical form is the structured `accessibility` object. See [`17_Naming.md`](17_Naming.md) §17.3.2 for the alias registry. Generators MUST emit only the canonical form.

When both the canonical form and a legacy form appear on the same widget, the canonical form wins per [`17_Naming.md`](17_Naming.md) §17.5.3.

---

## 13.10 Conformance

Per [`18_Conformance.md`](18_Conformance.md) §18.2.9, Core-Profile runtimes MUST:

- Accept `accessibility.label`, `accessibility.hint`, and `accessibility.role`.
- Accept the `semanticLabel` shorthand.
- Accept the legacy aliases registered in [`17_Naming.md`](17_Naming.md) §17.3.2.
- Enforce the 48×48dp minimum hit region on interactive widgets.
- Surface declared accessibility properties to the host platform's accessibility tree.

Runtimes SHOULD additionally accept the remaining `accessibility.*` sub-fields (`live`, `atomic`, `focusable`, `isHeader`, `excludeFromSemantics`, `focusOrder`, `focusTraversalOrder`, `focusTrap`, `initialFocus`, `restoreFocus`) and honor their declared semantics.
