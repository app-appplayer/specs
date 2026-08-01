# 09. Templates

**Status:** Normative.
**Profile:** Template Profile. See [`18_Conformance.md`](18_Conformance.md) §18.6 for required capabilities by profile level.
---

## 9.1 Overview

A **template** is a reusable, parameterized widget subtree. It is declared once and instantiated at zero or more sites via the `use` widget. Templates are the single mechanism for custom widgets in MCP UI DSL: a template definition is a component definition, and every component is a template built from other widgets.

A template:

- Accepts typed **parameters** (`params`).
- MAY expose named **slots** for caller-provided widget content.
- MAY declare **scoped styles** that do not leak across the template boundary.
- MAY declare **`stateDefaults`** — initial local state per instance *(since v1.3)*.
- MAY declare **`onMount`** / **`onUnmount`** lifecycle hooks *(since v1.3)*.
- MAY invoke other templates, recursively, up to a bounded depth.

Templates are registered at three scopes — page, application, and remote library — and resolved in that order (§9.10).

---

## 9.2 Template Definition

Templates are declared in the **map-key form**: as entries in a `templates` object on the application root.

### 9.2.1 Declaration form

Template definitions appear as entries in a `templates` object. The object key is the template name.

```json
{
  "templates": {
    "formField": {
      "params": {
        "label": { "type": "string", "required": true },
        "value": { "type": "any" }
      },
      "slots": {
        "trailing": { "required": false }
      },
      "content": {
        "type": "linear",
        "direction": "vertical",
        "children": [
          { "type": "text", "text": "{{label}}" },
          { "type": "textInput", "value": "{{value}}" }
        ]
      }
    }
  }
}
```

### 9.2.2 Definition fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `params` | object | no | Parameter declarations (§9.3) |
| `slots` | object | no | Slot declarations (§9.4) |
| `styles` | object | no | Scoped style definitions (§9.5) |
| `scopedStyles` | boolean | no | When `true`, styles declared in the template do not leak outside the template boundary. Default `false`. |
| `content` | widget | yes | Widget tree that is the template's output |
| `stateDefaults` | object | no | Instance-scoped initial local state *(since v1.3)* |
| `onMount` | action \| action[] | no | Runs once per instance mount, after `stateDefaults` initialization *(since v1.3)* |
| `onUnmount` | action \| action[] | no | Runs once per instance unmount *(since v1.3)* |
| `description` | string | no | Human-readable description |

---

## 9.3 Parameters

A parameter declaration is an object on `params.<name>`.

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"string"`, `"number"`, `"boolean"`, `"array"`, `"object"`, `"any"`, `"widget"`, `"action"` |
| `required` | boolean | Whether the caller MUST supply the parameter. Default `false`. |
| `default` | any | Value used when the parameter is omitted |
| `enum` | array | Allowed values |
| `validator` | string | Binding expression evaluated against `value`; falsy result rejects the value |
| `description` | string | Documentation |

Within the template `content`, parameter values resolve as bare identifiers (`{{label}}`) or through the `{{...}}` binding syntax. Parameters occupy position 3 in the binding resolution chain defined in [`03_Data_Binding.md`](03_Data_Binding.md) — after bare locals and `local.*`, before page state.

### 9.3.1 Validation

A runtime MUST:

- Reject a `use` invocation that omits a `required` parameter.
- Apply `default` when a non-required parameter is omitted.
- Reject values not listed in `enum` when `enum` is present.
- Evaluate `validator` and reject the value when the result is falsy.

---

## 9.4 Slots

A slot is a named insertion point filled by the caller.

```json
{
  "templates": {
    "card": {
      "params": { "title": { "type": "string", "required": true } },
      "content": {
        "type": "box",
        "children": [
          { "type": "text", "text": "{{title}}" },
          { "type": "slot", "name": "body", "required": true },
          { "type": "slot", "name": "footer", "fallback": null }
        ]
      }
    }
  }
}
```

Invocation supplies `slots`:

```json
{
  "type": "use",
  "template": "card",
  "params": { "title": "User" },
  "slots": {
    "body": { "type": "text", "text": "Hello" },
    "footer": {
      "type": "button",
      "label": "Close",
      "onTap": { "type": "navigation", "action": "pop" }
    }
  }
}
```

### 9.4.1 Slot fields (within the template content)

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Slot identifier, unique within the template |
| `required` | boolean | When `true`, the caller MUST provide the slot. Default `false`. |
| `fallback` | widget \| null | Rendered when the slot is omitted. `null` renders nothing. |

---

## 9.5 Scoped Styles

A template MAY declare `styles` — a named style map — and set `scopedStyles: true` to isolate them.

```json
{
  "templates": {
    "badge": {
      "scopedStyles": true,
      "styles": {
        "container": { "borderRadius": 12, "paddingHorizontal": 8, "paddingVertical": 4 },
        "label": { "fontSize": 10, "fontWeight": "bold" }
      },
      "params": {
        "text": { "type": "string", "required": true },
        "color": { "type": "string", "default": "{{theme.colorScheme.primary}}" }
      },
      "content": {
        "type": "box",
        "style": "{{styles.container}}",
        "children": [
          { "type": "text", "text": "{{text}}", "style": "{{styles.label}}" }
        ]
      }
    }
  }
}
```

### 9.5.1 Isolation rules

| Rule | Behavior |
|------|----------|
| Outward isolation | Styles defined inside a scoped template MUST NOT affect parent or sibling widgets. |
| Inward isolation | Styles from outside MUST NOT affect widgets inside, except inherited text properties. |
| Inherited properties | `fontFamily`, `fontSize`, `color`, `textAlign` inherit from the parent unless explicitly overridden. |
| Theme pass-through | `{{theme.*}}` bindings MUST resolve regardless of scoping. |
| Nested scopes | Each nested scoped template creates its own isolated scope. |

### 9.5.2 Style resolution order

1. Inline `style` on the widget
2. Named `styles` entries of the containing template
3. Inherited properties from the template root
4. Theme values via `{{theme.*}}`

When `scopedStyles` is `false` or absent, styles behave as inline widgets with no isolation boundary.

---

## 9.6 Template Invocation

A template is invoked through the `use` widget.

```json
{
  "type": "use",
  "template": "formField",
  "params": { "label": "Email", "value": "{{page.email}}" },
  "slots": {
    "trailing": { "type": "icon", "icon": "mail" }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"use"` |
| `template` | string | yes | Template name to instantiate |
| `params` | object | no | Parameter values; merged with declared `default` values |
| `slots` | object | no | Widget subtrees keyed by slot name |

### 9.6.1 `itemTemplate` in list widgets

Inside `list`, `grid`, and other iterating widgets, the per-item widget is declared under the canonical key **`itemTemplate`**. The key `template` is an accepted alias. See [`17_Naming.md`](17_Naming.md) §17.3.2.

```json
{
  "type": "list",
  "binding": "files",
  "itemTemplate": {
    "type": "listItem",
    "title": "{{item.name}}",
    "subtitle": "{{item.size}} bytes"
  }
}
```

---

## 9.7 Nested Templates

A template content MAY invoke other templates. Recursion is bounded: a runtime MUST enforce a maximum expansion depth (default **10** levels) and MUST terminate expansion with an error when the limit is exceeded. The limit guards against self-referential templates and mutual recursion.

```json
{
  "templates": {
    "labeledField": {
      "params": { "label": { "type": "string", "required": true } },
      "content": {
        "type": "use",
        "template": "formField",
        "params": { "label": "{{label}}" }
      }
    }
  }
}
```

---

## 9.8 Stateful Templates *(since v1.3)*

A template MAY declare `stateDefaults` — a map of initial values for instance-scoped state accessible via the `{{local.*}}` binding prefix.

```json
{
  "templates": {
    "accordion": {
      "params": { "title": { "type": "string", "required": true } },
      "stateDefaults": { "expanded": false },
      "content": {
        "type": "linear",
        "direction": "vertical",
        "children": [
          {
            "type": "linear",
            "direction": "horizontal",
            "children": [
              { "type": "text", "text": "{{title}}" },
              {
                "type": "icon",
                "icon": "{{local.expanded ? 'expandLess' : 'expandMore'}}",
                "onTap": {
                  "type": "state", "action": "toggle",
                  "binding": "local.expanded"
                }
              }
            ]
          },
          {
            "type": "conditional",
            "condition": "{{local.expanded}}",
            "then": { "type": "slot", "name": "body" }
          }
        ]
      }
    }
  }
}
```

### 9.8.1 Isolation

Each `use` instance receives its own `local.*` scope. Writes via `{"type": "state", "action": "...", "binding": "local.<name>"}` MUST NOT propagate to sibling instances, parent scopes, page state, or application state.

### 9.8.2 Local state actions

Local state accepts every state action type defined in [`04_Actions.md`](04_Actions.md):

| Action | Example |
|--------|---------|
| `set` | `{"type":"state","action":"set","binding":"local.count","value":5}` |
| `toggle` | `{"type":"state","action":"toggle","binding":"local.expanded"}` |
| `increment` / `decrement` | `{"type":"state","action":"increment","binding":"local.count"}` |
| `append` / `remove` | `{"type":"state","action":"append","binding":"local.items","value":"x"}` |
| `push` / `pop` / `removeAt` | `{"type":"state","action":"removeAt","binding":"local.items","index":2}` |

### 9.8.3 Binding resolution impact

`local.*` is populated by `stateDefaults` at instance-mount time. When `stateDefaults` is absent, `local.*` reads resolve to `undefined`. See [`03_Data_Binding.md`](03_Data_Binding.md) for the full resolution chain.

---

## 9.9 Template Lifecycle Hooks *(since v1.3)*

Top-level `onMount` and `onUnmount` on a template definition fire **per instance**. They are not instance-level properties of a `use` invocation; they are declared once on the definition and apply to every instantiation.

```json
{
  "templates": {
    "liveStatus": {
      "params": { "deviceId": { "type": "string", "required": true } },
      "stateDefaults": { "status": "loading", "data": null },
      "onMount": [
        {
          "type": "tool",
          "tool": "getDeviceStatus",
          "params": { "id": "{{deviceId}}" },
          "onSuccess": {
            "type": "state", "action": "set",
            "binding": "local.data", "value": "{{result}}"
          }
        },
        {
          "type": "channel", "action": "channel.start",
          "channel": "device.{{deviceId}}.status"
        }
      ],
      "onUnmount": [
        {
          "type": "channel", "action": "channel.stop",
          "channel": "device.{{deviceId}}.status"
        }
      ],
      "content": { "type": "text", "text": "Status: {{local.status}}" }
    }
  }
}
```

### 9.9.1 Execution guarantees

- `onMount` runs **once** per instance mount, **after** `stateDefaults` initialization.
- `onUnmount` runs **once** per instance unmount.
- When the hook is an array, its actions execute sequentially.
- `onUnmount` MUST run even when the instance is removed via conditional rendering, navigation, or parent disposal.

---

## 9.10 Template Resolution Order

A `use` invocation resolves the template name against the following registries, in order. The first match wins.

1. Page-level templates (local to the current page definition)
2. Application-level templates (the `templates` field on `ApplicationDefinition`)
3. Remote library templates (via `templateLibraries`, §9.11) *(since v1.3)*
4. Built-in templates shipped by the runtime (§9.12)

**Collision rule.** When the same name exists at multiple scopes, the more-local definition wins. Implementations MUST emit a **single** warning per colliding name per application session; repeated collisions for the same name do not produce additional warnings.

---

## 9.11 Remote Template Libraries *(since v1.3)*

An `ApplicationDefinition` MAY declare external template collections that are loaded at application initialization and registered at application scope.

```json
{
  "type": "application",
  "templateLibraries": [
    { "uri": "https://cdn.example.com/components/charts-v2.json" },
    { "uri": "bundle://components/design-system.json" }
  ],
  "templates": {
    "myLocalTemplate": { "params": {}, "content": {} }
  },
  "routes": {}
}
```

### 9.11.1 Entry schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uri` | string | yes | Library location |
| `version` | string | no | Expected library version; runtime MAY warn on mismatch |
| `integrity` | string | no | Subresource integrity hash (e.g., `sha256-…`) |

### 9.11.2 Acceptable URI schemes

- `https://` — fetched over HTTPS
- `bundle://` — resolved within the current bundle
- Any additional scheme the runtime's resource layer can resolve (e.g., `file://` for development)

### 9.11.3 Library document format

```json
{
  "name": "charts-v2",
  "version": "2.0.0",
  "templates": {
    "barChart":  { "params": {}, "content": {} },
    "lineChart": { "params": {}, "content": {} }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Library identifier |
| `version` | string | yes | Library version (semver recommended) |
| `templates` | object | yes | Map of template name to template definition |

### 9.11.4 Registration and collisions

Loaded templates register into the application's template registry at the **remote library** tier (§9.10, position 3). When a name collides with a local (page or application) template, the local definition wins and the runtime emits one warning per name per session.

---

## 9.12 Built-in Templates

Runtimes SHOULD register a baseline set of built-in templates at startup. They populate the lowest-priority tier of the resolution order (§9.10, position 4) and MAY be overridden by application-authored templates of the same name.

The minimum recommended set:

| Name | Purpose |
|------|---------|
| `card` | Elevated container with optional title, leading, trailing, and body slot |
| `listItem` | Row with optional leading, title, subtitle, trailing |
| `listItem.single` | Single-line list row variant |
| `listItem.double` | Two-line list row variant (title + subtitle) |
| `listItem.triple` | Three-line list row variant (title + subtitle + supporting text) |
| `formField` | Labeled input with optional helper text and error state |
| `section` | Titled grouping for forms and settings pages |
| `loadingIndicator` | Centered progress indicator with optional label |
| `errorMessage` | Icon + message for error states |

The exact built-in set is implementation-defined; a runtime MAY register additional templates. Authors can override any built-in locally by declaring a template with the same name.

---

## 9.13 Example: End-to-end Template Usage

```json
{
  "type": "application",
  "templateLibraries": [
    { "uri": "bundle://components/design-system.json" }
  ],
  "templates": {
    "statusBadge": {
      "params": {
        "status": {
          "type": "string", "required": true,
          "enum": ["online", "offline", "degraded"]
        }
      },
      "content": {
        "type": "chip",
        "label": "{{status}}",
        "variant": "filled"
      }
    },
    "deviceRow": {
      "params": {
        "name":   { "type": "string", "required": true },
        "status": { "type": "string", "required": true }
      },
      "stateDefaults": { "expanded": false },
      "content": {
        "type": "linear",
        "direction": "vertical",
        "children": [
          {
            "type": "listItem",
            "title": "{{name}}",
            "trailing": {
              "type": "use",
              "template": "statusBadge",
              "params": { "status": "{{status}}" }
            },
            "onTap": {
              "type": "state", "action": "toggle",
              "binding": "local.expanded"
            }
          },
          {
            "type": "conditional",
            "condition": "{{local.expanded}}",
            "then": { "type": "slot", "name": "details" }
          }
        ]
      }
    }
  },
  "routes": {
    "/devices": {
      "type": "page",
      "content": {
        "type": "list",
        "binding": "devices",
        "itemTemplate": {
          "type": "use",
          "template": "deviceRow",
          "params": { "name": "{{item.name}}", "status": "{{item.status}}" },
          "slots": {
            "details": {
              "type": "text",
              "text": "Last seen: {{item.lastSeen}}"
            }
          }
        }
      }
    }
  }
}
```

---

## 9.14 Conformance

Template Profile requirements are defined in [`18_Conformance.md`](18_Conformance.md) §18.6. Summary:

- **§18.6.1** (v1.1+): `use` / `template` invocation, parameter validation, slots, scoped styles, template registry.
- **§18.6.2** (v1.3+): `stateDefaults` with `local.*` isolation, `onMount` / `onUnmount` per-instance execution, `templateLibraries` loading, local-wins collision rule with single-warning runtime behavior.
