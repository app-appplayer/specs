# Form — Document Template Specification — v1.0

**Language-neutral specification.** Form is a **schema-based document generation
engine**: fix a *template*, fill only the *content*, and you always get a
professionally structured document. Whoever fills it — an LLM, a knowledge
source, or a person — the structure never drifts.

This spec is the authority for **authoring templates** and **using the engine**.
Tools, agents, and LLMs build templates and calls against this document (and the
JSON Schema derived from it). Implemented by the Dart reference in the `mcp_form`
package.

> **Not an input form.** "Form" here is **document generation**, not interactive
> data collection (survey / signup / `submit`). *Interactive, reactive
> publishing* is produced by rendering to the **`uiDsl`** format (§3.2), which a
> UI DSL runtime displays — a separate surface, not the subject of this model.

Use cases: résumés, reports (daily / weekly / monthly), meeting minutes, work
logs, exam sheets, timetables, newsletters, papers, letterhead, invoices,
contracts, business cards, detail pages — any document whose **structure must
stay fixed**.

---

## 1. Model

### 1.1 Template (`FormTemplate`)

The distribution and versioning unit. Self-describing (carries its own manifest
with dependencies and a compatibility range).

| Field | Meaning |
|---|---|
| `templateId` · `version` | Identity + version (the unit history accumulates on) |
| `name` · `description` | Display |
| `schema` | Definition of the fields to fill (§1.2) |
| `layoutPolicy` | Page, margins, fonts, grid (§1.3) |
| `defaultSections` | Document body structure (§1.4) |
| `locale` · `i18nStrings` · `components` | Localization and reusable fragments |
| `manifest` | `dependencies` · `compatRange` · `version` — the template is itself a versioned, dependency-aware distributable |

### 1.2 Schema (`FormSchema` / `FormSchemaField`)

The contract for the data being filled. A field carries:

`name` · `type` · `required` · `label` · `placeholder` · `format` · `minValue` ·
`maxValue` · `pattern` (plus `FormSchemaRule` rules).

This schema is the source of the **Template→JSON Schema** export (§2.1) — an LLM
fills strictly *within* it.

### 1.3 Layout policy (`FormLayoutPolicy`)

`pageSize` (A4 / Letter / …) · `margins` · `fontFamily` · `fontPolicy` (default /
heading / body / minimum sizes) · `gridColumns` (design grid — not newspaper
columns) · `maxTableRows` · `maxLineLength` · `autoWrap` · `autoScale`.

### 1.4 Sections & blocks (body structure)

A `FormSection` is a group of blocks. `FormBlock` kinds:

`FormTextBlock` · `FormHeadingBlock` · `FormTableBlock` (columns with per-column
`width` as a proportional weight — default 1.0, equal — and `alignment`
left/center/right) · `FormChartBlock` · `FormImageBlock` (`maxWidth` /
`aspectRatio`) · `FormCanvasBlock` (a reference seam to a vector canvas; drawing
content is composed at the *app* layer via a separate canvas capability, not by
this engine) · `FormFieldBlock` (bound to a schema field) · `FormRepeatableBlock`
(array binding).

Rich presentation attaches through **carriers**, leaving the core model
unchanged: `FormBlock.style` (map keys — colSpan, height, alignment, `placement`
(§3.4), `pageBreak` (§3.5), `fit`, barcode, qr, math, change [redline], …),
`FormTextBlock.content`/`format` (rich inline), and a render-time
`FormStyleSheet`.

**Page size is arbitrary** — `layoutPolicy.pageSize {size, width, height}` (mm) +
`isLandscape` express any format: business card (90×50), invitation / ticket
(custom), A4 (210×297), A3 (297×420), portrait or landscape. All renderers size
from it.

### 1.5 Document (`FormDocument`) — the instance

The result of binding data into a template (i.e. a filled document):
`documentId` · `templateId` / `templateVersion` · `status` (workflow) ·
`version` · `sections` · `data` · `bindings` · `metadata` (author, timestamps).

A **snapshot (publication)** is a `FormDocument` frozen at a point in time — an
immutable official record, distinct from a live view that re-derives its
content. Corrections are new publications; the old snapshot is preserved as
issued.

---

## 2. Filling (LLM · knowledge · human)

### 2.1 Template → JSON Schema (C1)

`form.template_schema {templateId, version?, withCapacity?}` → a **JSON Schema
(draft 2020-12)**. Constraining an external LLM's structured output to this
schema means **the structure cannot be broken**. This is the core
differentiator: generative tools let structure wobble and data-merge already
requires structured data, whereas here unstructured intent/knowledge is filled
*structurally, under schema constraint*.

### 2.2 Capacity feed-forward (C2)

`form.capacity {templateId, version?, fieldId?}` → per-field capacity
(`maxChars` · `lines` · `charsPerLine`): roughly how much text a fixed box holds,
computed *before* generation. C1 folds this capacity into the schema as
`maxLength` by default, so the model "writes to fit".

### 2.3 Binding

`FormFieldBlock` ← schema field; `FormRepeatableBlock` ← array (dotted-path
binding, min/max). Values live in `FormDocument.data`.

---

## 3. Rendering

### 3.1 Formats (6)

| format | Character |
|---|---|
| `pdf` | Print-fidelity (pages, boxes, multilingual TrueType embedding, PDF/A, PDF/UA tagging) |
| `html` | Web |
| `docx` | Flat OPC (MS Word) |
| `markdown` | Plain text |
| `uiDsl` | **Interactive / reactive** — a JSON widget tree rendered by a UI DSL runtime |
| `image` / `png` | **Pure-Dart raster** — a PNG, drawn with no browser or platform canvas. Text is rasterized from the injected font's own glyph outlines (glyf contours → bezier flatten → even-odd scanline fill), so multilingual/CJK renders given `embeddedFont`. Covers headings, text, fields, tables (weighted widths + alignment), images, background and `style.placement`. Paged output is the **full page box** (A4/A3/card aspect, not content-cropped); continuous grows to content. Charts / math / columns are a follow-up. |

`form.render {documentId, format, options?}` → bytes (base64) + `pageCount`.
`form.export` is equivalent.

### 3.2 `uiDsl` = interactive publishing

Rendering the same template/document to **`uiDsl` produces an interactive
publication** (as opposed to a static PDF). A UI DSL runtime turns the widget
tree into a real reactive UI. So "publish a document interactively" = the
document engine + the `uiDsl` render format + a UI DSL runtime. That is where
rich-text / interactive output lands.

### 3.3 RenderOptions

The `options` object (every key optional, with defaults): `pageFlow` (paged /
continuous) · `columnCount` / `columnGap` (multi-column) · `headerText` /
`footerText` · `showPageNumbers` / `pageNumberTemplate` · `compress` ·
`fillableFields` (AcroForm) · `pdfA` · `taggedPdf` (PDF/UA) · watermark (text /
color / opacity / size / angle) · `pageBorder` (+ `pageBorderColor` /
`pageBorderWidth` / `pageBorderRadius` — an opt-in page frame for official
documents / certificates, drawn on every page) · `includeMetadata`.

### 3.4 Absolute placement & backgrounds (`style.placement`)

A block carries `style.placement` to leave the flow and sit at page coordinates
measured from the paper corner (margins ignored) — a seal, a footer company
name, a full-page background:

```
{ anchor: top-left | top-right | top-center | bottom-left | bottom-right | bottom-center | center,
  x, y (mm),
  width?, height? (mm, or 'full' for full-bleed),
  z?: 'back' (behind the flow) | 'front' (default),
  fit?: 'cover' | 'contain' (images),
  repeat?: true (every page; default is the last page only) }
```

Applies to image **and** text blocks (PDF · HTML · image). `z: 'back'` +
`fit: 'cover'` + `width/height: 'full'` is a full-bleed background image.

### 3.5 Page breaks (report structure)

`style.pageBreak: 'before'` (PDF also `'after'`) forces a new page in paged mode —
the primitive for a cover page, a TOC page, the body, then a back page.
Continuous flow ignores it; HTML emits a print page break.

### 3.6 Font & style injection

Multilingual PDF/image and global styling are **host-injected**: the render
assembly accepts `styleSheet` · `embeddedFont` · `fallbackFonts` (loading fonts
is the host's job). A default assembly must register all six renderers — an empty
registry reports every format as unsupported.

---

## 4. Tool surface (`form.*`, 14 verbs)

| Group | Verbs |
|---|---|
| Template management | `save_template` · `get_template` · `delete_template` · `list_templates` · `get_template_versions` |
| LLM filling | `template_schema` (C1) · `capacity` (C2) |
| Document | `create_document` · `patch` · `validate` · `get_document` (typed JSON, patches applied) · `get_status` |
| Publishing | `render` · `export` |

A host exposes this surface as a **capability** (`form.*`) so a bundle can call
`form.render` and the rest declaratively, portable across hosts.

---

## 5. Persistence (injection seam)

The engine is **stateless**. Storage of templates, documents, and history is
externalized through **`FormTemplatePort` / `FormPort` injection** — a host
supplies the backing store (durable, per-tenant / per-project, or in-memory for
convenience and tests). History is a version timeline; a correction supersedes
rather than mutates, and prior snapshots stay as issued.

---

## 6. Boundaries

- Out of scope: OpenType GSUB/GPOS, inline-flow math, full PDF/A ICC output
  intents, a veraPDF-validated PDF/UA pass, and (in the image renderer) charts /
  math / multi-column / composite glyphs.
- This spec is the *contract and authoring reference*. Detailed render
  algorithms, copy-fit, and shaping are documented by the `mcp_form` reference
  implementation.

## License

MIT — see [../../LICENSE](../../LICENSE).
