# 12. Internationalization

This section defines the `i18n` field on `ApplicationDefinition` and the `{{i18n.*}}` binding family that resolves against it. Internationalization is a Core Profile feature — a runtime claiming Core Profile MUST implement at least the basic locale resolution and fallback rules defined here. Pluralization, number formatting, and date formatting are SHOULD-level capabilities (see §12.9).

## 12.1 `i18n` Field on ApplicationDefinition

```json
{
  "type": "application",
  "title": "My App",
  "i18n": {
    "defaultLocale": "en-US",
    "locales": ["en-US", "ko-KR", "ja-JP"],
    "text": {
      "en-US": {
        "greeting": "Hello",
        "farewell": "Goodbye"
      },
      "ko-KR": {
        "greeting": "안녕하세요",
        "farewell": "안녕히 가세요"
      },
      "ja-JP": {
        "greeting": "こんにちは",
        "farewell": "さようなら"
      }
    }
  },
  "routes": { }
}
```

### 12.1.1 Field Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `defaultLocale` | string (BCP 47) | Yes (when `i18n` is present) | Locale used when the active locale has no entry for a requested key. |
| `locales` | string[] (BCP 47) | No | Declared supported locales. Informational; runtime MUST NOT reject a locale omitted from this list if entries exist for it. |
| `text` | object | No | Map of locale → map of key → translated string. |
| `pluralization` | object | No | Map of locale → map of key → plural-category map. See §12.3. |
| `numberFormat` | object | No | Map of locale → map of format-name → number-format descriptor. See §12.4. |
| `dateFormat` | object | No | Map of locale → map of format-name → date/time format descriptor. See §12.5. |
| `textDirection` | object | No | Map of locale → `"ltr"` \| `"rtl"`. Overrides defaults for the locale. See §12.8. |

Locale tags follow BCP 47 (`en-US`, `ko-KR`, `ja-JP`, `ar-SA`, `he-IL`, `zh-Hans-CN`).

## 12.2 `{{i18n.*}}` Binding

The `i18n.` prefix resolves a localized value against the currently active locale.

Registered in [`17_Naming.md`](17_Naming.md) §17.2.5.

### 12.2.1 Plain Key

```json
{ "type": "text", "text": "{{i18n.greeting}}" }
```

- Resolves `i18n.text[<activeLocale>].greeting`.
- Falls back per §12.7 if the key is missing.

### 12.2.2 Key with Arguments

Arguments are passed using function-call syntax with a JSON object:

```json
{ "type": "text", "text": "{{i18n.itemCount({count: 5})}}" }
```

- Resolves against `pluralization` (§12.3), `numberFormat` (§12.4), or `dateFormat` (§12.5) descriptors registered under the same key, depending on which map contains it.
- A key registered simultaneously in more than one map is a definition error; runtimes SHOULD log a warning and prefer `pluralization` > `numberFormat` > `dateFormat` > `text`.

### 12.2.3 Explicit Locale Override

```json
{ "type": "text", "text": "{{i18n.greeting:en-US}}" }
```

- Resolves the value under `en-US` regardless of the active locale.
- Runtimes claiming Core Profile MUST accept this syntax. If the locale is not registered, fallback per §12.7 applies.

## 12.3 Pluralization

Plural categories follow CLDR: `zero`, `one`, `two`, `few`, `many`, `other`. `other` is required for every pluralized key.

```json
{
  "i18n": {
    "defaultLocale": "en-US",
    "pluralization": {
      "en-US": {
        "itemCount": {
          "zero": "No items",
          "one": "1 item",
          "other": "{count} items"
        }
      },
      "ko-KR": {
        "itemCount": {
          "other": "{count}개 항목"
        }
      }
    }
  }
}
```

Usage:

```json
{ "type": "text", "text": "{{i18n.itemCount({count: 5})}}" }
```

### 12.3.1 Selection Rules

- The active locale's CLDR plural rules select the category from the `count` argument. A runtime MUST implement CLDR plural selection for each locale it declares support for.
- `{count}` (and any other argument) within the selected template string is replaced with the formatted argument value. Numeric arguments use the locale's default `numberFormat` unless overridden (§12.4.2).
- If the selected category is absent, the runtime MUST fall back to `other`. If `other` is absent for the active locale, the runtime MUST fall back to the `defaultLocale` for the same key (§12.7).

## 12.4 Number Formatting

### 12.4.1 Descriptor

```json
{
  "i18n": {
    "defaultLocale": "en-US",
    "numberFormat": {
      "en-US": {
        "currency": { "style": "currency", "currency": "USD" },
        "percent": { "style": "percent", "maximumFractionDigits": 1 }
      },
      "ko-KR": {
        "currency": { "style": "currency", "currency": "KRW" }
      }
    }
  }
}
```

Descriptor fields follow the ECMA-402 `Intl.NumberFormat` options subset:

| Field | Type | Description |
|-------|------|-------------|
| `style` | `"decimal"` \| `"currency"` \| `"percent"` | Format style. Default `"decimal"`. |
| `currency` | string (ISO 4217) | Required when `style` is `"currency"`. |
| `minimumFractionDigits` | number | — |
| `maximumFractionDigits` | number | — |
| `useGrouping` | boolean | — |

### 12.4.2 Usage

```json
{ "type": "text", "text": "{{i18n.currency(price)}}" }
```

- The single positional argument is the numeric value.
- Returns the locale-formatted string for the active locale.

## 12.5 Date and Time Formatting

### 12.5.1 Descriptor

```json
{
  "i18n": {
    "defaultLocale": "en-US",
    "dateFormat": {
      "en-US": {
        "shortDate": { "year": "numeric", "month": "2-digit", "day": "2-digit" },
        "longDate": { "year": "numeric", "month": "long", "day": "numeric", "weekday": "long" }
      },
      "ko-KR": {
        "shortDate": { "year": "numeric", "month": "2-digit", "day": "2-digit" }
      }
    }
  }
}
```

Descriptor fields follow the ECMA-402 `Intl.DateTimeFormat` options subset (`year`, `month`, `day`, `weekday`, `hour`, `minute`, `second`, `hour12`, `timeZone`).

A pattern string (`"MM/dd/yyyy"`) MAY be used in place of the descriptor for simple cases; runtimes MUST accept both forms.

### 12.5.2 Usage

```json
{ "type": "text", "text": "{{i18n.shortDate(createdAt)}}" }
```

- The positional argument MAY be an ISO 8601 string or a numeric epoch (milliseconds).
- The formatter resolves against the active locale.

## 12.6 Runtime Locale Switching

Runtimes MUST expose a mechanism to set the active locale at runtime. When the active locale changes:

- All `{{i18n.*}}` bindings MUST re-resolve immediately.
- All widgets affected by `textDirection` (§12.8) MUST re-layout.
- `numberFormat` and `dateFormat` outputs MUST use the new locale.
- State, route, and other non-i18n bindings are unaffected.

The initial active locale is `i18n.defaultLocale`. The runtime MAY expose a host-level preference (e.g., OS language) that overrides the default on app start.

## 12.7 Fallback

Resolution order for any `{{i18n.<key>}}` or argumented form:

1. Active locale's entry for `<key>`.
2. If the resolver is pluralization and the selected category is missing, `other` under the active locale.
3. `defaultLocale` equivalent (same resolution map, then `other` for pluralization).
4. If still missing: the literal key string prefixed with a warning marker (e.g., `!!greeting`). Runtimes MUST log a warning the first time each missing key is observed per locale.

Explicit-locale syntax (`{{i18n.key:xx-YY}}`) applies the same chain, substituting `xx-YY` for step 1.

## 12.8 RTL Support

Locales whose script is right-to-left (e.g., `ar`, `he`, `fa`, `ur`) cause the runtime to flip default horizontal layout direction:

- `linear` with `direction: "horizontal"` MUST render children right-to-left.
- `textAlign: "start"` resolves to the right edge; `"end"` to the left edge.
- Iconography and padding/margin start/end resolve accordingly.

### 12.8.1 Direction Source

The runtime determines direction in this order:

1. Widget-level `textDirection` property (explicit `"ltr"` or `"rtl"`) — overrides locale defaults on any `linear`, `text`, `richText`, or any widget that accepts it.
2. `i18n.textDirection[<activeLocale>]` if defined.
3. Built-in locale default (LTR for most locales; RTL for `ar*`, `he*`, `fa*`, `ur*`, `yi*`).

### 12.8.2 Mirroring

Runtimes MUST mirror start/end-relative properties (`paddingStart`, `paddingEnd`, `marginStart`, `marginEnd`, `alignment: "start"`, `alignment: "end"`) in RTL contexts. Absolute `left`/`right` properties are not mirrored.

## 12.9 Conformance

Core Profile MUST:

- Accept the `i18n` field on `ApplicationDefinition`.
- Resolve `{{i18n.<key>}}` against `text` with active-locale and default-locale fallback (§12.7).
- Accept `{{i18n.<key>:<locale>}}` explicit-locale syntax.
- Re-resolve all `{{i18n.*}}` bindings on locale change (§12.6).
- Apply RTL layout for built-in RTL locales (§12.8).

Core Profile SHOULD:

- Implement CLDR-accurate `pluralization` selection (§12.3).
- Implement `numberFormat` (§12.4) and `dateFormat` (§12.5) against ECMA-402 option semantics.
- Honor `i18n.textDirection` overrides.

See [`18_Conformance.md`](18_Conformance.md) for profile-bound normative statements.
