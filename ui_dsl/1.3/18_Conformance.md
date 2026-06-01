# 18. Conformance

**Normative.** This document is the single source for conformance requirements. Other sections describe behavior; this file binds them to **MUST / SHOULD / MAY** tags per profile. Where this document and any other section disagree, this document governs.

Requirement verbs follow RFC 2119 / RFC 8174 semantics as summarised in [`00_Overview.md`](00_Overview.md) §0.6.

## 18.1 Conformance Model

A runtime implementation declares the set of **Profiles** it supports. Each Profile is a named subset of DSL capabilities with normative requirements. Profiles replace version-based conformance statements ("v1.2 compliant") because version numbers alone do not disambiguate which optional features a runtime implements.

### 18.1.1 Profile List

| Profile | Dependency | Scope |
|---------|------------|-------|
| **Core** | *(none)* | Required for all implementations. Application / Page / Widget tree, core widgets, binding engine, core actions, theme system, MCP protocol contract. |
| **Client** *(since v1.1)* | Core | Client-side `client.*` actions, permission system, bidirectional channels, `client://` resource protocol. |
| **Bundle** *(since v1.2)* | Core | ApplicationDefinition metadata, `bundle://` URI scheme, `ui://app/info` well-known resource, bundle adapters. |
| **Advanced** *(since v1.0)* | Core | Advanced widgets (chart, canvas, code editor, terminal, etc.). |
| **Template** *(since v1.1 / v1.3)* | Core | Template system — static invocation (v1.1), stateful + lifecycle + remote libraries (v1.3). |

### 18.1.2 Implication Rule

Claiming a dependent profile implies conformance to the base Profile. An implementation claiming "Client" MUST also conform to "Core".

### 18.1.3 Declaration

An implementation SHOULD publish its conformance claim in the machine-readable form described in §18.7. An implementation MUST NOT declare a Profile it does not fully satisfy.

### 18.1.4 Per-Profile Version Gating

Some Profile requirements are further gated by `since: vX.Y` on the underlying feature. A runtime claiming a Profile at a given DSL version MUST satisfy all requirements of that Profile at or below that version.

---

## 18.2 Core Profile

Required for every conformant implementation.

### 18.2.1 Required Widget Types

Runtimes MUST parse and render every widget in [`17_Naming.md`](17_Naming.md) §17.2.1 that is tagged *Core Profile*. This comprises:

- **Layout:** `box`, `linear`, `stack`, `center`, `align`, `padding`, `expanded`, `flexible`, `spacer`, `sizedBox`, `wrap`, `positioned`, `safeArea`, `visibility`, `conditional`, `margin`, `aspectRatio`, `constrained`, `fractionallySized`, `intrinsicHeight`, `intrinsicWidth`
- **Display:** `text`, `richText`, `image`, `icon`, `card`, `divider`, `verticalDivider`, `badge`, `chip`, `avatar`, `tooltip`, `placeholder`, `progressBar`, `banner`
- **Input:** `button`, `iconButton`, `textInput`, `toggle`, `select`, `checkbox`, `checkboxGroup`, `radio`, `radioGroup`, `slider`, `rangeSlider`, `numberField`, `form`, `rating`
- **Date/time input:** `dateField`, `timeField`, `datePicker`, `timePicker`, `dateRangePicker`, `colorPicker`, `segmentedControl`, `stepper`, `numberStepper`
- **List:** `list`, `grid`, `listItem`
- **Navigation:** `headerBar`, `bottomNavigation`, `tabBar`, `tabBarView`, `drawer`, `navigationRail`, `floatingActionButton`, `popupMenuButton`
- **Scroll:** `scrollView`, `singleChildScrollView`, `scrollBar`, `pageView`
- **Interaction:** `gestureDetector`, `inkWell`, `draggable`, `dragTarget`
- **Dialog:** `alertDialog`, `simpleDialog`, `customDialog`, `snackBar`, `bottomSheet`
- **Animation:** `animatedContainer`, `opacity` *(since v1.3)*, `transform` *(since v1.3)*, `lottieAnimation`
- **Utility:** `fittedBox`, `clipOval`, `clipRRect`, `decoration`, `accessibleWrapper`, `lazy`

Unknown widget types MUST NOT crash the runtime; the runtime MUST render an error placeholder and continue. See [`02_Widgets.md`](02_Widgets.md).

### 18.2.2 Required Action Types

Runtimes MUST handle every action type tagged *Core Profile* in [`17_Naming.md`](17_Naming.md) §17.2.2:

`state`, `navigation`, `tool`, `resource`, `dialog`, `batch`, `conditional`, `notification`, `parallel`, `sequence`, `cancel`, `animation`.

See [`04_Actions.md`](04_Actions.md) for behavior. Actions whose `type` is unknown MUST fail gracefully with a logged error; they MUST NOT crash the runtime.

### 18.2.3 Required Navigation Sub-Actions

Runtimes MUST support every navigation sub-action in [`17_Naming.md`](17_Naming.md) §17.2.3:

- `push`, `replace`, `pop`, `popToRoot`, `pushAndClear`, `setIndex` — core.
- `openApp`, `exitApp` *(since v1.3)* — runtimes claiming Core at v1.3+ MUST support both; `exitApp` MUST invoke the host-registered `onExit` callback as described in [`06_Runtime_Contract.md`](06_Runtime_Contract.md) §6.6.1.

### 18.2.4 Required State Sub-Actions

`set`, `increment`, `decrement`, `toggle`, `append`, `remove`, `push`, `pop`, `removeAt` per [`17_Naming.md`](17_Naming.md) §17.2.4.

### 18.2.5 Required Binding Features

- `{{...}}` expression syntax ([`03_Data_Binding.md`](03_Data_Binding.md) §3.2).
- Prefix resolution per [`03_Data_Binding.md`](03_Data_Binding.md) §3.4 for the Core prefix set: bare variable, `state.`, `page.`, `app.`, `route.params.`, `theme.`, `event.`, and the list-iteration context variables (`item`, `index`, `isFirst`, `isLast`, `isEven`, `isOdd`).
- The sixteen built-in expression functions ([`03_Data_Binding.md`](03_Data_Binding.md) §3.5).
- Type coercion rules ([`03_Data_Binding.md`](03_Data_Binding.md) §3.6).

Binding failures MUST NOT crash the runtime. Unknown prefixes MUST resolve to `null`.

### 18.2.6 Required MCP Protocol Support

Runtimes MUST implement the MCP contract described in [`06_Runtime_Contract.md`](06_Runtime_Contract.md):

- `resources/read` for `ui://app` and `ui://page/*`.
- `tools/call` dispatch.
- Resource subscription in *standard* mode (extended mode SHOULD).
- Consumption of `notifications/resources/updated`.
- Well-known resources per §6.2.

### 18.2.7 Required Theme Support

Runtimes MUST implement [`05_Theme.md`](05_Theme.md):

- `light`, `dark`, and `system` modes (with `system` dynamically tracking host brightness).
- **Color** — Material 3 28 roles (primary · onPrimary · primaryContainer · onPrimaryContainer · secondary · onSecondary · secondaryContainer · onSecondaryContainer · tertiary · onTertiary · tertiaryContainer · onTertiaryContainer · error · onError · errorContainer · onErrorContainer · surface · onSurface · surfaceVariant · onSurfaceVariant · surfaceTint · background · onBackground · outline · outlineVariant · inverseSurface · inverseOnSurface · inversePrimary · scrim · shadow) plus 6 semantic (success · onSuccess · warning · onWarning · info · onInfo).
- **Typography** — Material 3 15 roles (display/headline/title/body/label × Large/Medium/Small) plus 6 TextStyle fields (fontFamily · fontSize · fontWeight · lineHeight · letterSpacing · fontFeatureSettings).
- **Spacing** — 9 slots (xxs · xs · sm · md · lg · xl · 2xl · 3xl · 4xl) plus 4 layout primitives (screenPadding · cardPadding · sectionGap · inlineGap).
- **Shape** — Material 3 7 family (none · extraSmall · small · medium · large · extraLarge · full) plus per-corner override (topStart / topEnd / bottomStart / bottomEnd).
- **Elevation** — Material 3 6 levels (level0–level5).
- `theme.*` binding resolution plus page-level `themeOverride` (deep merge).
- DTCG W3C JSON import per [`05b_DTCG_Interchange.md`](05b_DTCG_Interchange.md) — the 9 categories in § B.3.

Runtimes SHOULD implement:

- HCT seed → tonal palette auto-generation (Material color utilities algorithm).
- State layer opacity (hover 0.08 / focus 0.12 / pressed 0.16 / disabled 0.38).
- DTCG W3C JSON export.
- **Motion** — Material 3 13 durations plus 4 easings.
- **Density** — comfortable / standard / compact (3-step).
- **Breakpoints** — Material 3 5-class (compact · medium · expanded · large · extraLarge).
- **Border** (5 width aliases · style), **Opacity** (M3 11-step), **FocusRing** (color/width/offset/radius), **Z-index** (9 layers).
- **Component tokens** — button · input · card · dialog · menu · list (6 standard) — applied per recognized component.

Runtimes MAY implement:

- Additional spacing slots (`5xl`, `6xl`, etc.).
- Additional component tokens (chip · tab · switch, etc.).
- Author-defined custom semantic roles.

Earlier-draft naming — `theme.colorScheme.<mode>.<slot>` (textOnPrimary, divider, etc.), `theme.typography.headline1-6` / `subtitle1/2` / `overline`, `theme.spacing.small/medium/large`, `theme.borderRadius.*`, `theme.elevation.small/medium/large` — is **removed in 1.3**. Only the new naming is valid.

### 18.2.8 Required Security Behaviors

Runtimes MUST implement [`07_Security.md`](07_Security.md):

- Expression sandbox with time and memory limits (§7.1).
- Input validation (§7.2) — invalid JSON is rejected; unknown widget types render as error placeholders.
- State isolation per application, per page, and per template instance (§7.4).
- XSS and injection prevention for user input rendering (§7.5).
- Bundle integrity verification when the Bundle Profile is also claimed (§7.6).

### 18.2.9 Required Accessibility

Runtimes MUST implement [`13_Accessibility.md`](13_Accessibility.md):

- Accept the structured `accessibility.label`, `accessibility.hint`, `accessibility.role`, `accessibility.live` properties.
- Accept the `semanticLabel` shorthand.
- Accept the ARIA-legacy aliases `ariaLabel` / `aria-label` / `ariaHint` / `aria-hint` / `ariaRole` / `aria-role` / `role` / `ariaLive` / `aria-live` per [`17_Naming.md`](17_Naming.md) §17.3.2.
- Enforce a minimum 48×48 dp hit region for interactive widgets (§13.6).
- Honor the default role-per-widget table (§13.3.1).

### 18.2.10 Required Legacy Alias Acceptance

Runtimes MUST accept every alias registered in [`17_Naming.md`](17_Naming.md) §17.3. When a canonical key and an alias key are both present on the same object, the canonical key wins (§17.5.3). Runtimes MAY log a deprecation warning on alias use; they MUST NOT reject the input.

### 18.2.11 Performance (SHOULD)

- Widget-tree render SHOULD complete in < 100 ms for a tree of 1000 widgets.
- Binding resolution SHOULD complete in < 10 ms for 1000 expressions.
- State-update propagation SHOULD complete in < 16 ms to preserve 60 fps.
- Expression parse results SHOULD be cached.
- Large lists SHOULD be rendered with lazy/viewport strategies.

---

## 18.3 Client Profile *(since v1.1)*

Required for runtimes that expose client-side capabilities (file system, network, system info, channels). Collected from distributed MUSTs across [`08_Client_Extensions.md`](08_Client_Extensions.md).

### 18.3.1 Required Client Actions

Runtimes claiming the Client Profile MUST support the *file*, *network*, *system*, and *storage* client actions in [`17_Naming.md`](17_Naming.md) §17.2.2 and [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.2:

`client.selectFile`, `client.readFile`, `client.writeFile`, `client.saveFile`, `client.listFiles`, `client.httpRequest`, `client.getSystemInfo`, `client.clipboard`, `client.notification`, `client.storage.get`, `client.storage.set`, `client.storage.remove`.

All client actions MUST return the unified result envelope described in §8.2.5.

### 18.3.2 SHOULD and MAY Client Actions

- `client.exec` SHOULD be supported. It requires the `system.exec` permission; runtimes MAY refuse to implement `client.exec` in sandboxed or security-restricted deployments.

### 18.3.3 Required Permission System

Runtimes MUST implement [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.4:

- The seven permission categories (§8.4.1).
- Page-level permission declaration parsing (§8.4.2).
- The four trust levels — `untrusted`, `basic`, `elevated`, `full` — and their default capability mapping (§8.4.3).
- Permission request flow with user consent (§8.4.4).
- `permissions.*` binding resolution (§8.4.5).
- `permission.revoke` action (§8.4.6, §8.7).

Runtimes MUST reject `client.*` action calls whose required permission has not been granted, returning the standard error envelope.

### 18.3.4 Required Channel Support

Runtimes MUST implement [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.6:

- Channel declaration parsing (§8.6.1).
- All channel types (§8.6.2).
- Channel lifecycle actions `channel.start`, `channel.stop`, `channel.restart`, `channel.toggle`, `channel.send` (§8.6.3).
- Both the canonical action shape and the v1.1 legacy shape per [`17_Naming.md`](17_Naming.md) §17.3.4.
- Channel callbacks `onMessage`, `onError`, `onConnect`, `onDisconnect` (§8.6.4).
- Lifecycle configuration including auto-start and restart policy (§8.6.5).
- `channels.*` binding resolution (§8.6.6).
- Message envelope parsing (§8.6.7).

### 18.3.5 Required `client://` Protocol

Runtimes MUST resolve the `client://file`, `client://workspace`, `client://temp`, `client://cache` resource URIs per [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.3. Path traversal outside the declared scope MUST be rejected (§8.3.3).

### 18.3.6 Required Client Bindings

Runtimes MUST resolve the `client.`, `client.theme.`, `client.env.`, `client.file.`, `client.system.`, `permissions.`, `channels.`, `resources.` prefixes per [`17_Naming.md`](17_Naming.md) §17.2.5 and [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.5. `client.env.*` MUST restrict access to a safe allowlist and MUST NOT expose variables that may contain secrets.

---

## 18.4 Bundle Profile *(since v1.2)*

Required for runtimes that load packaged MCP applications with metadata, icons, splashes, or bundled assets.

### 18.4.1 Required

Runtimes claiming the Bundle Profile MUST:

- Accept all ApplicationDefinition metadata fields in [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.1. Every field is optional; absence MUST NOT produce a parse error.
- Parse the `PublisherInfo` (§11.2), `TimestampInfo` (§11.3), and `SplashConfig` (§11.4) structures.
- Resolve `bundle://` URIs within the loaded bundle per §11.5.
- Serve or consume the `ui://app/info` well-known resource per §11.6.

### 18.4.2 SHOULD

- Implement `BundleUiReadAdapter` / `BundleUiWriteAdapter` against the `UiPort` contract described in [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.8.
- Verify bundle integrity via manifest checksum when the bundle declares one ([`07_Security.md`](07_Security.md) §7.6).
- Display app metadata (icon, category, publisher) when available.
- Request `ui://app/info` for app listings before loading the full ApplicationDefinition.

### 18.4.3 Backward Compatibility

- All v1.2 metadata fields are additive. A Core-only runtime (no Bundle Profile) MUST ignore unknown v1.2 fields gracefully and render the application normally.
- A Bundle-Profile runtime MUST accept ApplicationDefinitions with or without v1.2 metadata fields.
- The `dashboard` ApplicationDefinition field *(since v1.3)* is separate from visual widgets (see [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.9). Runtimes claiming Bundle at v1.3+ SHOULD support both `renderApp()` and `renderDashboard()` rendering modes.

---

## 18.5 Advanced Profile *(since v1.0)*

Required for runtimes that expose advanced visualization or editor widgets.

### 18.5.1 Required

Runtimes claiming the Advanced Profile MUST parse and render every widget in [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md) §10.1, subject to the version gating in §18.5.2:

`chart`, `table`, `dataTable`, `map`, `mediaPlayer`, `calendar`, `timeline`, `gauge`, `heatmap`, `tree`, `graph`, `networkGraph`, `codeEditor`, `terminal`, `fileExplorer`, `markdown`, `webView`, `signature`.

### 18.5.2 Version-Gated Widgets

- `canvas` — required only for runtimes claiming Advanced at v1.3+ (see [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md) §10.3). A runtime claiming Advanced at v1.0 / v1.1 / v1.2 MAY omit `canvas`.

Note: `opacity` and `transform` *(since v1.3)* are Core-Profile animation widgets, not Advanced — see §18.2.1 and [`16_Animations.md`](16_Animations.md) §16.11.

### 18.5.3 Unsupported Widget Handling

When an advanced widget is absent (either because the runtime does not claim the Advanced Profile, or because a specific widget is version-gated away), the runtime MUST follow the unknown-widget rule from §18.2.1 — render an error placeholder; do not crash.

---

## 18.6 Template Profile

### 18.6.1 Required (Template Profile at v1.1+)

Runtimes claiming the Template Profile at v1.1 or later MUST implement [`09_Templates.md`](09_Templates.md):

- The `use` and `template` widget types (§9.1, §9.2, §9.6).
- Map-key template-definition form (§9.2.1).
- Parameter validation via `TemplateParamDefinition.validate()` (§9.3.1).
- Slot resolution (§9.4).
- Scoped styles with the isolation rules in §9.5.1 and the resolution order in §9.5.2.
- Template registry operations (`registerFromJson`, `registerBuiltIns`) (§9.12).
- `itemTemplate` in list widgets (§9.6.1).
- Template safety constraints from [`07_Security.md`](07_Security.md) §7.7.

### 18.6.2 Required (Template Profile at v1.3+)

Additionally, runtimes claiming the Template Profile at v1.3 or later MUST implement:

- Stateful templates — `stateDefaults`, `local.*` scope, isolation per instance (§9.8).
- Template lifecycle hooks `onMount`, `onUnmount`, including the execution guarantees in §9.9.1.
- Remote template libraries — `templateLibraries` entry schema (§9.11.1), acceptable URI schemes (§9.11.2), library document format (§9.11.3), and registration / collision rules (§9.11.4) where **local definitions win** on name collision.
- The template resolution order in §9.10.
- Runtimes MUST execute `onUnmount` when a template instance is unmounted and MUST NOT share `local.*` state between instances.

### 18.6.3 Without the Template Profile

A runtime that does not claim the Template Profile MUST treat `"type": "use"` and `"type": "template"` as unknown widget types per §18.2.1. Applications that rely on templates but are loaded by a Core-only runtime degrade to error placeholders; they MUST NOT crash.

---

## 18.7 Conformance Claims

Implementations SHOULD publish their conformance claim in a machine-readable form. The canonical shape is YAML or an equivalent JSON object:

```yaml
mcp_ui_dsl:
  profiles: [Core, Client, Template]
  version: 1.3
  implementation: flutter_mcp_ui_runtime
  version_implementation: 0.9.0
  optional:
    canvas: true
    client.exec: false
    extendedSubscriptions: true
```

Fields:

| Field | Purpose |
|-------|---------|
| `profiles` | List of Profile names satisfied by the implementation. |
| `version` | Highest DSL version level at which the declared Profiles are claimed. |
| `implementation` | Implementation name (free-form). |
| `version_implementation` | Implementation's own semver (independent of DSL version per D10). |
| `optional` | Map of version-gated or SHOULD-level features and whether they are supported. |

An implementation MUST NOT list a Profile it does not fully satisfy at the declared `version`.

---

## 18.8 Conformance Testing

Each Profile has an associated test suite. Test IDs are prefixed:

| Prefix | Profile | Scope |
|--------|---------|-------|
| `CORE-*` | Core | Widgets, actions, bindings, protocol, theme, security, accessibility, legacy aliases. |
| `CLI-*` | Client | Client actions, permissions, channels, `client://` protocol, client bindings. |
| `BND-*` | Bundle | Metadata parsing, `bundle://` resolution, `ui://app/info`, adapters, dashboard. |
| `ADV-*` | Advanced | Each advanced widget; canvas at v1.3+. |
| `TPL-*` | Template | Static invocation (v1.1) and stateful / lifecycle / remote libraries (v1.3). |

A runtime passes a Profile suite when every MUST-level test returns green and SHOULD-level tests either pass or are documented as deliberate deviations. A runtime claiming multiple Profiles MUST pass every suite it claims.

Test suite authoring follows the canonical vocabulary in [`17_Naming.md`](17_Naming.md) §17.2. Test inputs MAY exercise legacy aliases to verify §18.2.10.

---

## 18.9 Informative: Feature-to-Profile Map

Quick lookup of which profile each feature section belongs to.

| Feature section | Profile |
|-----------------|---------|
| [`01_Core_Concepts.md`](01_Core_Concepts.md) — Application / Page / Widget tree | Core |
| [`02_Widgets.md`](02_Widgets.md) — Core widget catalog | Core |
| [`03_Data_Binding.md`](03_Data_Binding.md) — Binding syntax, prefixes, functions | Core |
| [`04_Actions.md`](04_Actions.md) — Core action catalog | Core |
| [`05_Theme.md`](05_Theme.md) — Theme system | Core |
| [`06_Runtime_Contract.md`](06_Runtime_Contract.md) — MCP protocol contract | Core |
| [`07_Security.md`](07_Security.md) §§7.1–7.5, 7.9 | Core |
| [`07_Security.md`](07_Security.md) §7.6 — Bundle integrity | Bundle |
| [`07_Security.md`](07_Security.md) §7.7 — Template safety | Template |
| [`07_Security.md`](07_Security.md) §7.8 — Offline encryption | Client |
| [`08_Client_Extensions.md`](08_Client_Extensions.md) | Client |
| [`09_Templates.md`](09_Templates.md) §§9.1–9.7, 9.12, 9.13 | Template (v1.1+) |
| [`09_Templates.md`](09_Templates.md) §§9.8–9.11 | Template (v1.3+) |
| [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md) | Advanced |
| [`10_Advanced_Widgets.md`](10_Advanced_Widgets.md) §10.3 — `canvas` | Advanced (v1.3+) |
| [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §§11.1–11.8 | Bundle |
| [`11_Bundle_Metadata.md`](11_Bundle_Metadata.md) §11.9 — Dashboard | Bundle (v1.3+) |
| [`12_Internationalization.md`](12_Internationalization.md) | Core |
| [`13_Accessibility.md`](13_Accessibility.md) | Core |
| [`14_Responsive_Events.md`](14_Responsive_Events.md) | Core |
| [`15_Offline_Sync.md`](15_Offline_Sync.md) | Client |
| [`16_Animations.md`](16_Animations.md) | Core (all animation widgets + `animation` action; optional advanced drivers degrade per §16.11) |
| [`17_Naming.md`](17_Naming.md) — Canonical vocabulary and alias registry | Core (alias acceptance is a Core MUST) |
