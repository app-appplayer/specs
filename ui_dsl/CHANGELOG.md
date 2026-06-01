# MCP UI DSL — Changelog

## [1.3.4]

Content-app capability tier — five-phase enrichment that turns the spec from a utility-app builder into a content-app builder (book / album / magazine / portfolio). All phases ship at the spec layer in this release; runtime first-cut wraps the M3 standard surfaces.

### New shared primitives ([`configs/_primitive/`](1.3/configs/_primitive/))

New cross-schema directory `configs/_primitive/` — single source of truth for primitives referenced from widgets / app / page / theme schemas. Codegen embeds each primitive's body into every output schema's `$defs` so each schema stays self-contained.

- `AssetRef` — five accepted scheme prefixes: `bundle://`, `https?://`, `data:`, `assets/`, `client://`. Migrated from `configs/app/AssetRef.yaml`. image / icon / BackgroundImage / videoPlayer / fontRegistry all reference this.
- `TextStyle` — inline text-style superset (16 fields including shader, fontFeatures, shadows). Migrated from `configs/theme/TextStyle.yaml`. Used by `theme.typography.<role>` AND `text.style` / `richText.style`.
- `BorderRadius` — per-corner radius. **Directional canonical** (RTL-aware per M3): `topStart` / `topEnd` / `bottomStart` / `bottomEnd` + `all` shorthand. Replaces `configs/theme/ShapeCorner.yaml` (visual coords).
- `Alignment` — 2D rectangle alignment. **Directional canonical** plus numeric `{x, y}` form. Used by box / align / stack / fittedBox / image / indexedStack / BackgroundImage.alignment / Gradient.begin/end/center.
- `Dimension` — number OR `{value, unit}` object form.
- `Gradient` — `linear` / `radial` / `sweep` discriminated by `type`; colors[] + stops[] + per-type endpoints + tileMode.
- `BoxBorder` / `BorderSide` — uniform-side or per-side border (color / width / style).
- `BoxShadow` — drop shadow (color / offset / blurRadius / spreadRadius).
- `BackgroundImage` — image fill behind a box. AssetRef + fit + alignment + opacity + colorFilter (5-mode subset) + repeat.
- `BoxDecoration` — composition wrapper (color / gradient / image / border / borderRadius / boxShadow / shape / backdropBlur).
- `Binding` — `^\{\{.*\}\}$` pattern accepted as an alt branch on every primitive — any primitive value may be substituted with a binding expression.
- `AnimationCurve` — 12 canonical curves (CSS 4 + M3 standard 3 + M3 emphasized 3 + bounce 2). Used by every implicit-anim widget + RouteTransition.
- `RouteTransition` — per-route page transition. 6 styles (slide / fade / scale / cube / sharedAxis / fadeThrough) + duration / curve / axis / reverse.
- `NavigationStyle` — visual styling for nav surfaces (background / indicator / divider / labels / icons / selected colors / elevation).
- `Span` — richText inline span. Discriminated oneOf: TextSpan (text + style + onTap + nested children) OR WidgetSpan (widget + alignment + baseline).
- `DropCap` — typographic drop-cap (lines / glyph override / per-cap style).
- `Sliver` — scrollView's `slivers` array element. Discriminated oneOf of 5 sliver shapes (sliverAppBar / sliverPersistentHeader / sliverList / sliverGrid / sliverFixedExtentList).

### Phase 1 — Decoration substrate ([`02_Widgets.md`](1.3/02_Widgets.md))

- `box` — `decoration` typed as `BoxDecoration`.
- `decoration` widget — full BoxDecoration via `decoration:` OR flat shorthand on every constituent field.
- `text` — `style: TextStyle` (binding string accepted), `dropCap: DropCap`. `style.shader` paints rendered glyphs through a gradient via shader mask.
- `richText` — `spans: array<Span>` formalising TextSpan / WidgetSpan; paragraph-level `style` (TextStyle); `dropCap` on first TextSpan.
- `icon` — five source forms (named / codepoint / URL / bundle SVG / data URI); `shader: Gradient` mutually exclusive with `color`.

### Phase 2 — Gallery layouts ([`02_Widgets.md`](1.3/02_Widgets.md) §2.7 / §2.9)

- `staggeredGrid` (since v1.3) — Pinterest-style masonry.
- `carousel` (since v1.3) — partial-viewport horizontal browser. viewportFraction + loop + autoPlay + transition (slide / fade / coverflow / depth) + indicatorPosition.
- `pageView` — `initialPage`, `loop`, `scrollPhysics`, `allowImplicitScrolling`.
- `scrollView` — sliver mode via `slivers` array (mutually exclusive with `child` / `children`).

### Phase 3 — Motion ([`16_Animations.md`](1.3/16_Animations.md))

- `hero` (since v1.3) — shared-element transition wrapper.
- `animatedOpacity` / `animatedAlign` / `animatedPositioned` / `animatedDefaultTextStyle` (since v1.3) — dedicated implicit-animation wrappers.
- `scrollAnimated` (since v1.3) — scroll-position-driven animation.
- `rive` (since v1.3) — Rive animation.
- `animatedContainer` — formalised `decoration: BoxDecoration`, `alignment: Alignment`, `curve: AnimationCurve`.
- `RouteValue` — wrapper form `{ page, transition }` carries a per-route `RouteTransition`. The prior un-grounded `pageTransition` and `sharedElementConfig` prose blocks are replaced (RouteTransition primitive + `hero` widget).

### Phase 4 — Media ([`02_Widgets.md`](1.3/02_Widgets.md) §2.5 / [`10_Advanced_Widgets.md`](1.3/10_Advanced_Widgets.md))

- `kenBurnsImage` (since v1.3) — image with slow zoom-and-pan animation.
- `imageFilter` (since v1.3) — colour / blur filter applied to a child subtree. 7 filter kinds (sepia / grayscale / blur / saturation / brightness / contrast / invert).
- `lightbox` (since v1.3) — full-screen modal image viewer with pinch-zoom and swipe.
- `mediaPlayer` — `source` / `poster` typed as `AssetRef`; new `waveform` boolean (audio mode only).

### Phase 5 — Theme & navigation polish ([`05_Theme.md`](1.3/05_Theme.md) / [`01_Core_Concepts.md`](1.3/01_Core_Concepts.md))

- `theme.preset` (since v1.3) — curated content-app theme bundle. 5 presets: `warm` / `cool` / `sepia` / `mono` / `highContrast`. Applied as base; other `theme.*` fields layer overrides.
- `theme.fonts` (since v1.3) — font asset registration. Per-family `weights` + `variableAxes` (`{tag, min, max, default}`) + `fallbacks`.
- `NavigationConfig.style` (since v1.3) — visual styling for the navigation surface (NavigationStyle).
- `NavItem.style` (since v1.3) — per-item override layered on top of NavigationConfig.style.

### Spec hygiene

- Cross-cutting primitives moved to `configs/_primitive/` from `configs/app/` (AssetRef) and `configs/theme/` (TextStyle / ShapeCorner). Single source of truth; codegen embeds body in every output schema.
- `BorderRadius` and `Alignment` adopt **directional** corner / token names (M3 canonical, RTL-aware). Visual aliases (`topLeft` / `topRight` / `bottomLeft` / `bottomRight`) accepted by the runtime only, not the schema.
- `17_Naming.md` — `| constrained | constrainedBox |` row removed and the catalog entry dropped. `constrained` / `constrainedBox` are pre-1.3 carryover with no spec presence; runtimes MAY retain them as backward-compat aliases of `box`.
- `clipRRect.borderRadius` typed as `BorderRadius` (was free `number | object`); description aligned to directional corner names.
- `AnimationCurve` named curves (§16.6.1) realigned with the 12-value primitive — replacing the prior ad-hoc 13-value list.
- Common widget `click` / `tooltip` field — every widget admits a `click: Action` (universal gesture surface) and `tooltip: string` (hover / long-press) at the common-property layer (§2.2 / §1.4.1). Additive — bundles that omit them are unaffected. Runtimes wrap the widget in a gesture / tooltip surface only when the field is present. `click` complements (does NOT replace) widget-local activation slots such as `button.onTap` / `iconButton.onTap` / `richText.spans[].onTap`; it is the canonical surface for making pure layout / decoration widgets (`box`, `card`, `linear`, `stack`, ...) tappable without nesting a `gestureDetector`.

### Schema / generated artifacts

- `schema/{widgets,app,page,theme}.schema.json`, `generated/widgets.md`, `generated/llm_prompt_card.md` regenerated.
- `widget_builders.g.dart` (generator) regrown to 134 builders (12 new). New helper methods on `MCPUIJsonGenerator`: `gradient` / `backgroundImage` / `navigationStyle` / `routeTransition`.
- `widgets/_common.yaml` — single yaml source of truth for common widget properties (currently `click` / `tooltip`). Both codegens load it at startup and merge every common property into each widget's effective property set (widget-declared same-named property wins). Result: every widget def in `widgets.schema.json` admits `click` / `tooltip`, and every `MCPUIWidgetBuilders.<widget>` exposes a typed `click` / `tooltip` parameter. Underscore prefix keeps it out of the per-widget yaml iteration.
- `tools/spec_codegen/bin/spec_codegen.dart` — loads `configs/_primitive/` + `configs/widget/` and embeds primitives in `widgets.schema.json $defs`.
- `tools/spec_codegen/bin/configs_codegen.dart` — seeds every config schema's `$defs` with the same shared primitives so each schema stays self-contained.
- `tools/spec_codegen/bin/drift_audit.dart` — runtime-only-legacy allow-list (`constrained`, `constrainedBox`); multi-class file scan (`*_factories.dart` accepted alongside `*_factory.dart`); regex fix for generic-return-type method extraction.

### Conformance

- 111 / 111 yaml example bundles validate.
- drift_audit reports **zero drift** across Sections B–J.

---

## [1.3.3]

- App / page / theme schema yaml-ised under `1.3/configs/<stem>/`; schemas now codegen from yaml ground truth.
- `ChannelDefinition` schema rebuilt to match § 8.6 prose; `ServiceDefinition` / `I18nConfig` / `TemplateLibraryRef` field names aligned with prose.
- `lazy.trigger` enum `viewport` → `visible`. `NavigationConfig` § 1.2.3 formalised.
- 7 dismissible widgets (dialog / drawer / banner / sheet / snackBar) gained `onClose` callback.
- New tools: `spec_consistency.dart` (spec internal audit), `configs_codegen.dart` (yaml → app/page/theme schema). `drift_audit.dart` excludes `.g.dart` mirrors from source corpus.

---

## [1.3.2]

### Widgets ([`02_Widgets.md`](1.3/02_Widgets.md))

- `box` — added `minWidth` / `maxWidth` / `minHeight` / `maxHeight` flat-form constraint properties so `box` is a true superset of the legacy `constrained` widget.
- Removed `constrained` from the canonical surface (yaml + schema + 02_Widgets section). Runtimes MAY retain it as a legacy alias of `box`.

### Theme tokens ([`widgets/`](1.3/widgets/))

- M3 token shorthand on `text.variant` (15 typography roles), `box.padding`, `card.shape` / `card.elevation`, `button.elevation`, `icon.size` / `sizeToken` — accept the M3 token name; resolved through `theme.<domain>.<token>`.

### Responsive ([`14_Responsive_Events.md`](1.3/14_Responsive_Events.md))

- Rewritten on M3 5-class FormFactor — `compact` / `medium` / `expanded` / `large` / `extraLarge` plus `embedded`. Earlier `xs` / `sm` / `md` / `lg` / `xl` labels removed.
- `{{runtime.breakpoint}}` renamed to `{{runtime.formFactor}}`.
- Per-form-factor property override map formalised (`{compact, medium, expanded, large, extraLarge, embedded, default}`) on every property.

### Schema / generated artifacts

- `schema/widgets.schema.json`, `schema/app.schema.json`, `generated/widgets.md`, `generated/llm_prompt_card.md` regenerated.

---

## [1.3.1]

### Templates ([`09_Templates.md`](1.3/09_Templates.md))

- Rename template body field `body` → `content` for consistency with page-level `content`.
- Remove legacy inline-`name` definition form. Templates declare exclusively in the **map-key form** (`templates: { name: { ... } }`).
- Remove legacy invocation aliases. The `use` widget is the sole canonical invocation type; `template` / `useTemplate` are no longer accepted. `params` is the sole canonical field name; `arguments` / `overrides` are no longer accepted.

### Theme ([`05_Theme.md`](1.3/05_Theme.md) §5.3.6)

- Define mode-specific fallback policy for sparse `theme` blocks. A bundle that omits the `theme` block entirely MUST resolve to the M3 default **dark** scheme under dark host brightness (rather than re-tagging the light scheme). Common-only `theme.color` (no `light`/`dark` variant) MUST apply to both modes unchanged.

### Schema / generated artifacts

- `schema/widgets.schema.json`, `generated/widgets.md`, `generated/llm_prompt_card.md` regenerated from the updated `widgets/utility/use.yaml` (aliases removed).

---

## [1.3] — Modern Theme System

Adopts a modern design-system standard (Material 3 + DTCG + shadcn-style + HCT seed) as canonical. No legacy aliases.

### Theme spec rewrite ([`05_Theme.md`](1.3/05_Theme.md))

- **Color (28 + 6 semantic)** — Material 3 28 roles (primary / primaryContainer / onPrimaryContainer / secondary / secondaryContainer / tertiary / tertiaryContainer / error / errorContainer / surface / surfaceVariant / surfaceTint / background / outline / outlineVariant / inverseSurface / inverseOnSurface / inversePrimary / scrim / shadow plus each on*) plus 6 semantic (success / warning / info plus on*) plus state-layer opacity plus HCT seed auto-derivation.
- **Typography (15 roles)** — Material 3 naming (display / headline / title / body / label × Large/Medium/Small). Earlier names `headline1-6`, `subtitle1/2`, `body1/2`, `caption`, `button`, `overline` are removed.
- **Spacing (9 slots)** — `xxs / xs / sm / md / lg / xl / 2xl / 3xl / 4xl` (8pt grid) plus 4 layout primitives (screenPadding / cardPadding / sectionGap / inlineGap). Earlier `small / medium / large` naming removed.
- **Shape (M3 7 family)** — `none / extraSmall / small / medium / large / extraLarge / full` plus per-corner override (topStart / topEnd / bottomStart / bottomEnd, RTL-aware). `borderRadius` naming removed.
- **Elevation (M3 6 levels)** — `level0`–`level5` (0 / 1 / 3 / 6 / 8 / 12) plus tonal surface (shadow / tint separated). Earlier `small / medium / large` naming removed.
- **New domains** — Motion (M3 13 durations + 4 easings), Density (3-step), Breakpoints (5-class), Border (5 width aliases), Opacity (M3 11-step), FocusRing, Z-index (9 layers), Component tokens (button / input / card / dialog / menu / list).
- **DTCG W3C JSON interchange** — export/import compatible with Tokens Studio, Style Dictionary, and Claude Design ([`05b_DTCG_Interchange.md`](1.3/05b_DTCG_Interchange.md)).
- **3-tier token model** — reference / alias / component ([`05a_Tokens_Reference.md`](1.3/05a_Tokens_Reference.md)).

### Conformance ([`18_Conformance.md`](1.3/18_Conformance.md) §18.2.7)

- MUST — light/dark/system mode, M3 28 color roles, M3 15 typography, 9 spacing, 7 shape, 6 elevation, `theme.*` binding plus page override (deep merge), DTCG JSON import (9 categories).
- SHOULD — HCT seed auto-derivation, state layer, DTCG export, Motion, Density, Breakpoints, Border, Opacity, FocusRing, Z-index, 6 standard components.
- MAY — additional spacing slots, custom components / semantic roles.

### Schema

- New [`schema/theme.schema.json`](1.3/schema/theme.schema.json) — JSON Schema 2020-12, all 14 token domains covered.

### Migration

- All earlier-draft names (h1-h6, headline1-6, subtitle1/2, button, overline, textOnPrimary, divider, borderRadius, spacing.small, etc.) are removed in 1.3. No legacy reader is provided.
- Mapping table: see [`05_Theme.md`](1.3/05_Theme.md) §5.17.

### Other 1.3 additions

- Stateful templates (`stateDefaults` on template definitions).
- Template lifecycle hooks (`onMount`, `onUnmount`).
- Remote template libraries (`templateLibraries` with `bundle://` or `https://` URIs).
- Dashboard rendering mode (`dashboard` field on ApplicationDefinition).
- Canvas widget (drawing commands: `rect`, `circle`, `arc`, `line`, `path`, `text`, `image`).
- Opacity widget (with implicit animation).
- Transform widget (scale / rotate / translate with implicit animation).
- `openApp` navigation action (dashboard → full app transition).
- `exitApp` navigation action (host-inserted close button on root route).

---

## [1.2]

- ApplicationDefinition metadata fields: `id`, `description`, `icon`, `splash`, `category`, `publisher`, `timestamps`, `screenshots`.
- `TimestampInfo` structured type for creation / update timestamps.
- Well-known resource `ui://app/info` for lightweight metadata retrieval.
- `bundle://` URI scheme for referencing bundle-internal assets.
- Bundle serving: read / write adapters implementing the UI port.
- Online serving path (MCP server transparent to client) and local serving path (direct bundle load).
- `PublisherInfo`, `SplashConfig` types.

---

## [1.1]

- Client-side resources: `client.selectFile`, `client.readFile`, `client.writeFile`, `client.saveFile`, `client.listFiles`.
- Client network: `client.httpRequest`.
- Client system: `client.getSystemInfo`, `client.clipboard`, `client.exec`.
- Client notifications: `client.notification`.
- Permission system: file / network / system permissions with trust levels.
- Client data bindings: `client.workingDirectory`, `client.userName`, `client.platform`, `client.locale`, `client.theme.*`, `client.env.*`, `client.file.*`, `client.system.*`.
- Bidirectional channels: `client.watchFile`, `client.watchDirectory`, `client.systemMonitor`, `client.poll` with lifecycle management.
- Enhanced image widget with `client://` path and fallback URL.
- Parallel / sequence / cancel actions.
- `permission.revoke` action.
- `resources.*` binding prefix.
- Template system (static): parameters, slots, scoped styles.
- Responsive layout: breakpoints, MediaQuery, platform detection.
- Event propagation: capture / target / bubble phases, stopPropagation, delegation.
- Plugin system with lifecycle hooks.
- Offline queue and sync with conflict resolution strategies.
- Animation extensions: page transitions, shared element transitions, physics-based.
- Validation system: required, minLength / maxLength, pattern, email, match, async.

---

## [1.0]

Initial public release.

- Application / Page definition structure.
- Multi-page routing with parameterized routes (`route.params.*`).
- 40+ widget types across 8 categories.
- Unified `linear` layout replacing row / column.
- Data binding with expression language (variables, operators, conditionals, functions).
- 16 built-in expression functions.
- Type coercion rules.
- List iteration context (`item`, `index`, `isFirst`, `isLast`, `isEven`, `isOdd`).
- Action system: state / navigation / tool / resource / dialog / batch / conditional / notification.
- Tool response auto-merge into state.
- Resource subscription: standard and extended modes.
- Theme system: light/dark/system, color scheme, typography, spacing, radius, elevation.
- Theme binding via `{{theme.*}}`.
- Page lifecycle: `onInit`, `onDestroy`.
- MCP protocol integration: `resources/read`, `tools/call`, subscriptions, notifications.
- Security: input validation, resource access verification, state isolation, expression sandboxing.
- Accessibility: roles, keyboard navigation, touch targets, live regions.
- Internationalization: `{{i18n.key}}`, runtime language switching.
