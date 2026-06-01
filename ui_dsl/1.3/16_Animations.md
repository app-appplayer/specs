# 16. Animations

**Status:** Normative.
This section defines how animations are declared, triggered, and composed in MCP UI DSL. The DSL distinguishes **implicit** animations (a property change animates automatically) from **explicit** animations (an action drives a property from one value to another).

Basic animation support is part of the Core Profile; shared-element transitions, gesture-linked drivers, and physics-based animations are optional capabilities.

---

## 16.1 Implicit Animations

An implicit animation runs whenever a property on the widget changes value. The runtime interpolates between the old and new values over `duration` along `curve`.

Widgets supporting implicit animation:

| Widget | `animated` flag | Default duration | Since |
|--------|-----------------|------------------|-------|
| `animatedContainer` | always implicit | 300 ms | v1.0 |
| `opacity` | optional (`animated: true`) | 300 ms | v1.3 |
| `transform` | optional (`animated: true`) | 300 ms | v1.3 |
| `animatedOpacity` | always implicit | 300 ms | v1.3 |
| `animatedAlign` | always implicit | 300 ms | v1.3 |
| `animatedPositioned` | always implicit | 300 ms | v1.3 |
| `animatedDefaultTextStyle` | always implicit | 300 ms | v1.3 |

The four `animated*` widgets (since v1.3) are dedicated implicit-animation wrappers — distinct from the `opacity` / `transform` widgets that gate animation behind `animated: true`. Each wraps a single property domain (opacity, alignment, edge offsets, default text style) and has no static-rendering mode. Use them when an animation is unconditional; use `opacity` / `transform` with `animated` toggled when an author needs to switch between animated and static.

### 16.1.1 Shape

```json
{
  "type": "opacity",
  "opacity": "{{state.isActive ? 1.0 : 0.3}}",
  "animated": true,
  "duration": 300,
  "curve": "easeInOut",
  "child": { "type": "card", "child": { "type": "text", "text": "{{title}}" } }
}
```

### 16.1.2 Common Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `animated` | boolean | No | `false` (on `opacity`, `transform`); always on for `animatedContainer` | Enable implicit animation |
| `duration` | number (ms) | No | 300 | Interpolation length |
| `curve` | string \| object | No | `"easeInOut"` | See §16.6 |

### 16.1.3 `animatedContainer`

`animatedContainer` implicitly animates every property that changes value: `width`, `height`, `padding`, `margin`, `alignment`, `decoration.backgroundColor`, `decoration.borderRadius`, `decoration.border`.

### 16.1.4 `opacity` *(since v1.3)*

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `opacity` | number (0.0–1.0) | Yes | Transparency; 0.0 = fully transparent, 1.0 = fully opaque |
| `child` | widget | Yes | Subtree to which opacity applies |
| `animated`, `duration`, `curve` | — | No | Implicit animation fields (§16.1.2) |

### 16.1.5 `transform` *(since v1.3)*

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `rotate` | number | No | Rotation angle in radians |
| `scale` | number \| `{ "x": number, "y": number }` | No | Uniform scale (number) or per-axis scale |
| `translate` | `{ "x": number, "y": number }` | No | Offset in logical pixels |
| `origin` | `{ "x": number, "y": number }` | No | Transform origin as fraction (0.0–1.0). Default: `{ "x": 0.5, "y": 0.5 }` (center) |
| `child` | widget | Yes | Subtree to which transform applies |
| `animated`, `duration`, `curve` | — | No | Implicit animation fields (§16.1.2) |

---

## 16.2 Explicit Animations via the `animation` Action

The `animation` action drives a named property on a target widget from one value to another.

```json
{
  "type": "animation",
  "target": "heroImage",
  "property": "opacity",
  "from": 1.0,
  "to": 0.0,
  "duration": 500,
  "curve": "easeOut",
  "onComplete": { "type": "navigation", "action": "pop" }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"animation"` | Yes | Action discriminator |
| `target` | string (widget id) | Yes | Widget whose property is animated |
| `property` | string | Yes | Animatable property (`opacity`, `translateX`, `translateY`, `scale`, `rotation`, `width`, `height`, `color`, ...) |
| `from` | any | No | Starting value; defaults to current value |
| `to` | any | Yes | Ending value |
| `duration` | number (ms) | No | Default 300 |
| `curve` | string \| object | No | See §16.6 |
| `delay` | number (ms) | No | Wait before starting |
| `repeat` | number \| `"infinite"` | No | Repeat count |
| `reverse` | boolean | No | Reverse direction on repeat (ping-pong) |
| `onComplete` | action | No | Fires when the animation ends |

---

## 16.3 Page Transitions

`ApplicationDefinition` MAY declare a default page transition, and individual `navigation` actions MAY override it per transition:

Page transitions are declared via the `RouteTransition` primitive (`configs/_primitive/RouteTransition.yaml`). Two attachment points:

**Per-route default** — wrap the route value in `{ page, transition }`. Every navigation to that route uses the declared transition.

```json
{
  "routes": {
    "/book/:id": {
      "page": "ui://book/details",
      "transition": {
        "style": "sharedAxis",
        "axis": "horizontal",
        "duration": 350,
        "curve": "emphasized"
      }
    }
  }
}
```

**Per-call override** — pass a `transition` field on the `navigate` action.

```json
{
  "type": "navigation",
  "action": "push",
  "route": "/detail/{{item.id}}",
  "transition": { "style": "fade", "duration": 250 }
}
```

### 16.3.1 Transition Styles

The `RouteTransition.style` enum carries six canonical styles (Material 3 motion-pattern aligned):

| Style | Effect |
|-------|--------|
| `slide` | Incoming page slides over the outgoing along `axis` (M3 default for forward navigation). |
| `fade` | Cross-fade between pages at the same position. |
| `scale` | Outgoing scales out, incoming scales in. |
| `cube` | Perspective-rotate as if turning a 3D cube. Needs a perspective-aware compositor — falls back to `slide` when unavailable. |
| `sharedAxis` | Coordinated slide+fade across an axis. M3 motion pattern for sibling-level navigation. |
| `fadeThrough` | Outgoing fades out while incoming fades in after a brief zero-opacity bridge. M3 motion pattern for top-level navigation. |

### 16.3.2 Transition Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `style` | enum | Yes | See §16.3.1. |
| `duration` | number (ms) | No | Defaults to the M3 motion-system value for the chosen style. |
| `curve` | `AnimationCurve` | No | Defaults to `emphasized`. See §16.6. |
| `axis` | `"horizontal"` \| `"vertical"` | No | Direction for `slide` / `sharedAxis`. Defaults to `horizontal`. |
| `reverse` | boolean | No | When true, run in reverse direction (back navigation). |

---

## 16.4 Shared Element Transitions

Wrap a child widget with the `hero` widget (§ 2.12.5) on BOTH the source page AND the destination page using the same `tag`; the runtime morphs the source widget's bounds → destination widget's bounds during the route transition. Pairs naturally with `RouteTransition` styles such as `fade`, `sharedAxis`, or `fadeThrough` — the hero morph runs concurrently with the page-level transition.

```json
{
  "type": "hero",
  "tag": "album-{{album.id}}",
  "child": {
    "type": "image",
    "src": "{{album.cover}}",
    "fit": "cover"
  }
}
```

### 16.4.1 Fields

See § 2.12.5 for the full field table. Tags MUST be unique within each page (a single page cannot host two `hero`s with the same tag).

### 16.4.2 Semantics

During navigation, if a `hero` on the destination page carries the same `tag` as one on the source, the runtime animates the source widget's geometry (position, size) to the destination widget's geometry. Multiple shared elements animate in parallel. Setting `transitionOnUserGestures: true` extends the morph to gesture-driven back navigation (iOS swipe-from-edge).

Shared-element transitions are optional. A runtime that does not support them MUST render both source and destination normally without a morph.

---

## 16.5 Physics-Based Animations

Physics-based animations parameterize motion by physical laws rather than fixed duration:

```json
{
  "type": "box",
  "animation": {
    "physics": {
      "type": "spring",
      "mass": 1.0,
      "stiffness": 200,
      "damping": 10
    }
  }
}
```

### 16.5.1 Physics Types

| Type | Fields |
|------|--------|
| `spring` | `mass`, `stiffness`, `damping` |
| `friction` | `friction` (coefficient) |
| `gravity` | `magnitude`, `direction` (`"up"`, `"down"`, `"left"`, `"right"`) |

Physics-based animations are optional. Runtimes that do not support them MAY fall back to a fixed-duration animation with a reasonable default.

---

## 16.6 Animation Curves

A curve is either a named string or a cubic Bezier object.

### 16.6.1 Named Curves

The `AnimationCurve` primitive (`configs/_primitive/AnimationCurve.yaml`) is the canonical 12-value enum used wherever a curve is referenced (`animatedContainer.curve`, `animatedOpacity.curve`, `RouteTransition.curve`, `scrollAnimated` per-binding curve, etc.).

| Name | Shape | Group |
|------|-------|-------|
| `linear` | Constant velocity | CSS-familiar |
| `easeIn` | Starts slow, accelerates | CSS-familiar |
| `easeOut` | Starts fast, decelerates | CSS-familiar |
| `easeInOut` | Slow at both ends | CSS-familiar |
| `standard` | M3 standard motion — minimal overshoot, clean cubic | M3 standard |
| `standardAccelerate` | M3 standard, accelerate-only | M3 standard |
| `standardDecelerate` | M3 standard, decelerate-only | M3 standard |
| `emphasized` | M3 emphasized — dramatic acceleration profile | M3 emphasized |
| `emphasizedAccelerate` | M3 emphasized, accelerate-only | M3 emphasized |
| `emphasizedDecelerate` | M3 emphasized, decelerate-only | M3 emphasized |
| `bounceIn` | Bounces at the start | Bounce |
| `bounceOut` | Bounces at the end | Bounce |

Use M3 standard for everyday transitions; M3 emphasized for hero / page transitions; bounce for attention micro-interactions. Runtimes MUST implement all 12.

### 16.6.2 Cubic Bezier

A custom curve MAY be specified as a cubic Bezier object:

```json
{ "curve": { "type": "cubic", "p1x": 0.25, "p1y": 0.1, "p2x": 0.25, "p2y": 1.0 } }
```

All four control-point fields are required. `p1x` and `p2x` MUST be in `[0.0, 1.0]`.

---

## 16.7 Gesture-Linked Animations

An animation MAY be driven by a user gesture rather than elapsed time:

```json
{
  "type": "box",
  "animation": {
    "driver": "gesture",
    "gesture": "horizontalDrag",
    "range": { "min": -200, "max": 0 },
    "properties": {
      "translateX": { "from": 0, "to": -200 },
      "opacity": { "from": 1, "to": 0 }
    },
    "onComplete": {
      "type": "tool",
      "tool": "deleteItem",
      "params": { "id": "{{item.id}}" }
    }
  }
}
```

Supported gestures: `horizontalDrag`, `verticalDrag`, `pinch`, `rotate`, `pull`. The normalized gesture progress in `[0, 1]` maps linearly across `range`, and each listed property interpolates between its `from` and `to`.

Gesture-linked animation is optional. Runtimes that do not support it render the target statically.

---

## 16.8 Scroll-Linked Animations

An animation MAY be driven by scroll position:

```json
{
  "type": "headerBar",
  "animation": {
    "driver": "scroll",
    "scrollTarget": "mainScrollView",
    "scrollRange": { "min": 0, "max": 200 },
    "properties": {
      "height": { "from": 200, "to": 56 },
      "opacity": { "from": 1, "to": 0.8 }
    }
  }
}
```

Scroll-linked animation is optional. Runtimes that do not support it render the target with its initial property values.

The `scrollAnimated` widget (since v1.3, § 2.12.10) is the dedicated wrapper for scroll-linked animation: each binding maps a scroll-offset window to a property tween (`opacity` / `scale` / `translateX` / `translateY` / `rotate`). Use it for parallax mastheads, reveal-on-scroll, and shrinking titles without writing the inline `animation: { driver: "scroll" }` block on every target.

```json
{
  "type": "scrollAnimated",
  "bindings": [
    { "property": "translateY", "fromOffset": 0, "toOffset": 240, "fromValue": 0, "toValue": -120 }
  ],
  "child": { "type": "image", "src": "{{cover}}", "fit": "cover" }
}
```

---

## 16.9 Lottie / Rive Integration

Two widgets cover authored animation files:

- `lottieAnimation` — for After Effects-derived Lottie JSON. Time-driven motion graphics.
- `rive` (since v1.3, § 2.12.11) — for Rive `.riv` files. Supports interactive state machines, vector deformation, and smaller binary footprints than Lottie.

### `lottieAnimation`

```json
{
  "type": "lottieAnimation",
  "src": "client://workspace/animations/loading.json",
  "width": 200,
  "height": 200,
  "autoPlay": true,
  "loop": true
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `src` | URI string | required | Lottie JSON source |
| `width`, `height` | number | intrinsic | Render size |
| `autoPlay` | boolean | `true` | Start playing on mount |
| `loop` | boolean | `true` | Repeat indefinitely |
| `speed` | number | 1.0 | Playback speed multiplier |

### `rive`

```json
{
  "type": "rive",
  "src": "bundle://avatars/dragon.riv",
  "stateMachine": "Idle",
  "inputs": {
    "happy": "{{user.online}}",
    "wave":  "{{justGreeted}}"
  },
  "width": 120,
  "height": 120
}
```

State-machine `inputs` accept `boolean` / `number` / `trigger` types matching the input declared in the Rive file. `trigger` inputs fire when set to `true`. See § 2.12.11 for the full property table.

Support is optional. Runtimes without a Lottie / Rive engine SHOULD render a placeholder box of the requested size.

---

## 16.10 Animation Composition

Multiple animations MAY be composed:

```json
{
  "animation": {
    "compose": "parallel",
    "animations": [
      { "property": "opacity", "from": 0, "to": 1, "duration": 300 },
      { "property": "translateY", "from": 20, "to": 0, "duration": 400, "curve": "easeOut" }
    ]
  }
}
```

### 16.10.1 Composition Modes

| Mode | Behavior |
|------|----------|
| `parallel` | All animations run simultaneously |
| `sequence` | Animations run one after another, in array order |
| `stagger` | Sequential with a fixed `staggerDelay` (ms) between starts |

### 16.10.2 Conflict Resolution

When two animations target the same property on the same widget:

- Within a single compose block, definitions later in the array override earlier ones for the overlapping time range.
- Across multiple concurrent animation sources, the most recently started animation wins (last-start-wins).

---

## 16.11 Conformance

Core Profile runtimes MUST:

- Render `animatedContainer` with implicit animation of its documented properties.
- Render `opacity` *(since v1.3)* and `transform` *(since v1.3)* widgets, including the `animated` / `duration` / `curve` fields.
- Implement the `animation` action with `target`, `property`, `from`, `to`, `duration`, `curve`, `onComplete`.
- Implement all named curves in §16.6.1 and cubic Bezier per §16.6.2.
- Render page transitions of types `slide`, `fade`, `scale`, and `none`.
- Accept `slideFade`, `rotation`, `custom` transition types (MAY fall back to `fade` if not implemented).

Core Profile runtimes SHOULD:

- Implement animation composition modes per §16.10.1.
- Honor `repeat` and `reverse` on explicit animations.

Shared-element transitions (§16.4), physics-based animations (§16.5), gesture-linked animations (§16.7), scroll-linked animations (§16.8), and Lottie/Rive integration (§16.9) are optional. Runtimes that do not support an optional capability MUST degrade per the rule stated in that section rather than raising an error.
