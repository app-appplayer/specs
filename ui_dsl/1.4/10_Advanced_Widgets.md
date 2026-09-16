# 10. Advanced Widgets

**Profile:** Advanced Profile (optional). A Core-only runtime MAY skip every widget in this file. Conformance requirements for this profile are defined in [`18_Conformance.md`](18_Conformance.md) §18.5.

> **SSOT:** The machine-readable widget registry at [`widgets/advanced/<type>.yaml`](widgets/advanced/) is authoritative. Tables and property rows in this file are regenerated from it; if they disagree, YAML wins. Generated reference: [`generated/widgets.md`](generated/widgets.md), [`schema/widgets.schema.json`](schema/widgets.schema.json).

All examples use canonical names only. Binding expressions (`{{...}}`) follow [`03_Data_Binding.md`](03_Data_Binding.md). Aliases (if any) are registered in [`17_Naming.md`](17_Naming.md) §17.3.

## 10.1 Catalog

| Widget | Purpose | Since |
|--------|---------|-------|
| `chart` | Data visualization with `chartType`: line, bar, pie, donut, scatter, area | v1.0 |
| `table` | Tabular layout with row/column model | v1.0 |
| `dataTable` | Material-style sortable/selectable data table | v1.0 |
| `map` | Geographic map with markers, overlays | v1.0 |
| `mediaPlayer` | Audio/video player | v1.0 |
| `calendar` | Calendar view (month/week/day) | v1.0 |
| `timeline` | Chronological timeline | v1.0 |
| `gauge` | Analog/digital gauge | v1.0 |
| `heatmap` | 2D heatmap visualization | v1.0 |
| `tree` | Expandable tree view | v1.0 |
| `graph` | Node/edge graph visualization | v1.0 |
| `networkGraph` | Network topology graph | v1.0 |
| `codeEditor` | Syntax-highlighted code editing | v1.0 |
| `terminal` | ANSI terminal emulator | v1.0 |
| `fileExplorer` | File/directory browser | v1.0 |
| `markdown` | Markdown renderer | v1.0 |
| `webView` | Embedded web view | v1.0 |
| `signature` | Signature capture pad | v1.0 |
| `canvas` | General-purpose drawing canvas | v1.3 |
| `lightbox` | Full-screen image viewer with pinch-zoom + swipe | v1.3 |
| `qrCode` | QR code renderer | v1.4 |
| `barcode` | 1-D barcode renderer | v1.4 |
| `pdfViewer` | PDF document viewer with page navigation and zoom | v1.4 |
| `diffViewer` | Side-by-side or unified text comparison | v1.4 |
| `richTextEditor` | Formatted text entry; value is HTML or Markdown | v1.4 |
| `splitter` | Panes divided by draggable gutters | v1.4 |
| `resizable` | A box the user resizes by dragging its edges | v1.4 |
| `kanban` | Cards in columns, moved by dragging | v1.4 |
| `gantt` | Tasks on a time axis with dependencies | v1.4 |
| `spreadsheet` | Editable cell grid with optional formulas | v1.4 |

## 10.2 `chart` *(since v1.0)*

Data visualization widget with multiple chart types.

```json
{
  "type": "chart",
  "chartType": "line",
  "data": {
    "labels": ["Jan", "Feb", "Mar", "Apr"],
    "datasets": [
      {
        "label": "Sales",
        "data": [100, 200, 150, 300],
        "borderColor": "#2196F3",
        "backgroundColor": "rgba(33, 150, 243, 0.2)"
      }
    ]
  },
  "options": {
    "responsive": true,
    "animation": { "duration": 1000 },
    "legend": { "position": "top" }
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `chartType` | enum | required | `line`, `bar`, `pie`, `donut`, `scatter`, `area`, `radar`, `polar`, `bubble` |
| `data` | object \| array | required | `{ labels: string[], datasets: Dataset[] }`, or a bare series array |
| `data.datasets[].label` | string | null | Dataset label shown in legend |
| `data.datasets[].data` | number[] | required | Data points |
| `data.datasets[].borderColor` | string | theme | Line/border color |
| `data.datasets[].backgroundColor` | string | theme | Fill color |
| `options` | object | `{}` | Rendering options |
| `options.responsive` | boolean | `true` | Resize with container |
| `options.animation.duration` | number | 1000 | Animation duration (ms) |
| `options.legend.position` | enum | `top` | `top`, `bottom`, `left`, `right`, `none` |
| `width` | number | null | Fixed width (logical pixels) |
| `height` | number | null | Fixed height (logical pixels) |

Unknown `chartType` values MUST be rejected at parse time.

## 10.3 `table` *(since v1.0)*

Layout table for arranging widgets in rows and columns. Not data-bound; each cell is a widget.

```json
{
  "type": "table",
  "border": { "color": "#E0E0E0", "width": 1 },
  "defaultColumnWidth": "flex",
  "defaultVerticalAlignment": "middle",
  "rows": [
    {
      "cells": [
        { "type": "text", "text": "Name" },
        { "type": "textInput", "value": "{{name}}",
          "onChange": { "type": "state", "action": "set", "binding": "name", "value": "{{event.value}}" } }
      ]
    },
    {
      "cells": [
        { "type": "text", "text": "Email" },
        { "type": "textInput", "value": "{{email}}",
          "onChange": { "type": "state", "action": "set", "binding": "email", "value": "{{event.value}}" } }
      ]
    }
  ]
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `rows` | `TableRow[]` | required | Row definitions — each carries the widgets for its cells (`{ cells: Widget[] }`). A row with no cells is accepted and skipped at render, so a document carrying one still opens. |
| `border` | `{ color, width }` | null | Optional cell border |
| `defaultColumnWidth` | string \| number | `flex` | `flex`, `intrinsic`, or a number (fixed px) |
| `defaultVerticalAlignment` | enum | `middle` | `top`, `middle`, `bottom`, `baseline` |
| `columnWidths` | object | null | Map `columnIndex` → width override |

## 10.4 `dataTable` *(since v1.0)*

Material-style sortable, selectable data table bound to a row array.

```json
{
  "type": "dataTable",
  "columns": [
    { "key": "id", "label": "ID", "width": 100 },
    { "key": "name", "label": "Name", "sortable": true },
    { "key": "status", "label": "Status", "align": "center" }
  ],
  "rows": "{{items}}",
  "selectable": true,
  "sortColumn": "{{sortColumn}}",
  "sortAscending": "{{sortAscending}}",
  "onSort": {
    "type": "batch",
    "actions": [
      { "type": "state", "action": "set", "binding": "sortColumn", "value": "{{event.column}}" },
      { "type": "state", "action": "set", "binding": "sortAscending", "value": "{{event.ascending}}" }
    ]
  },
  "onRowTap": {
    "type": "navigation",
    "action": "push",
    "route": "/detail/{{event.row.id}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `columns` | `Column[]` | required | Column definitions |
| `columns[].key` | string | required | Row field key |
| `columns[].label` | string | required | Header label |
| `columns[].width` | number | null | Fixed column width |
| `columns[].sortable` | boolean | `false` | Whether column supports sorting |
| `columns[].align` | enum | `start` | `start`, `center`, `end` |
| `rows` | `object[]` \| binding | required | Array of row objects — a literal array or a binding to one |
| `selectable` | boolean | `false` | Whether rows are selectable |
| `sortColumn` | binding | null | Current sort column key |
| `sortAscending` | binding | `true` | Current sort direction |
| `editable` | boolean | `false` | Allow in-place cell editing. Edits report through `onCellEdit`; the widget does not mutate `rows` on its own |
| `filterable` | boolean | `false` | Per-column filter row under the header |
| `resizableColumns` | boolean | `false` | Drag column edges to resize |
| `virtualScroll` | boolean | `false` | Build rows on demand. Requires `rowHeight` |
| `rowHeight` | number | — | Fixed row height in logical pixels |
| `onSort` | Action | null | Fired on header tap of a sortable column |
| `onCellEdit` | Action | null | Fired when an edit is committed; `{ row, column, value, previous }` |
| `onRowTap` | Action | null | Fired on row tap; `event.row` is the row object |

**Two render paths, and which one a document gets.** `editable`, `virtualScroll` or
`resizableColumns` selects a laid-out grid; without any of them the table is a Material
`DataTable`. The distinction is not cosmetic: a `DataTable`'s cells are text, so `editable`
alone drawing one gave a table marked editable with no editable cell. Both paths honour
`columns[].align`, the header tap of a `sortable` column, and the declared `sortColumn` /
`sortAscending`.

Sorting compares two numbers as numbers whatever their subtype. An `int` and a `double` in
one column are one kind of value; comparing them as strings puts `617.5` after `2160`.

An editable cell is keyed by its **row's identity**, not its position, so a re-sort moves each
field with its row. Keyed by position, the field at row *n* keeps the text it was built with
while the event names the row now at *n* — the number typed lands visually in one line and is
reported for another.

With editable cells a tap focuses the field, so `onRowTap` fires only on the row's padding.
That is inherent to the composition: a document that needs both puts the row action elsewhere.

## 10.5 `map` *(since v1.0)*

Geographic map with markers and overlays.

**Canonical coordinate fields:** `latitude`, `longitude`. No `lat` / `lng` shorthand and no `properties['latitude']` nested fallback.

```json
{
  "type": "map",
  "center": { "latitude": 37.5665, "longitude": 126.9780 },
  "zoom": 13,
  "mapType": "standard",
  "markers": [
    {
      "id": "m1",
      "latitude": 37.5665,
      "longitude": 126.9780,
      "label": "Seoul City Hall",
      "icon": "location_pin",
      "color": "#F44336"
    },
    {
      "id": "m2",
      "latitude": 37.5512,
      "longitude": 126.9882,
      "label": "Namsan Tower"
    }
  ],
  "overlays": [
    {
      "type": "polygon",
      "points": [
        { "latitude": 37.570, "longitude": 126.975 },
        { "latitude": 37.570, "longitude": 126.985 },
        { "latitude": 37.560, "longitude": 126.985 }
      ],
      "fillColor": "rgba(33,150,243,0.3)",
      "strokeColor": "#2196F3",
      "strokeWidth": 2
    }
  ],
  "onMarkerTap": {
    "type": "state", "action": "set",
    "binding": "selectedMarker", "value": "{{event.id}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `center` | `{ latitude, longitude }` | no | — | Map center coordinates. Required unless flat `latitude` / `longitude` are given. |
| `zoom` | number | 13 | Zoom level (0–22) |
| `mapType` | enum | `standard` | `standard`, `satellite`, `terrain`, `hybrid` |
| `markers` | `Marker[]` | `[]` | Point markers |
| `markers[].id` | string | required | Unique marker ID |
| `markers[].latitude` | number | required | Marker latitude |
| `markers[].longitude` | number | required | Marker longitude |
| `markers[].label` | string | null | Marker label |
| `markers[].icon` | string | null | Icon name |
| `markers[].color` | string | null | Marker color |
| `overlays` | `Overlay[]` | `[]` | Polygon / polyline / circle overlays |
| `overlays[].type` | enum | required | `polygon`, `polyline`, `circle` |
| `overlays[].points` | `{ latitude, longitude }[]` | required for polygon/polyline | Vertex list |
| `overlays[].fillColor` | string | null | Fill color (polygon/circle) |
| `overlays[].strokeColor` | string | null | Stroke color |
| `overlays[].strokeWidth` | number | 1 | Stroke width |
| `onMarkerTap` | Action | null | Fired on marker tap; `event.id` is the marker ID |
| `onMapTap` | Action | null | Fired on map tap; `event.latitude` / `event.longitude` |

> Legacy `lat` / `lng` field shorthands are **not** canonical. Emitters MUST emit `latitude` / `longitude`. Runtimes MAY accept shorthand forms as a best-effort compatibility behavior but MUST NOT require them.

## 10.6 `mediaPlayer` *(since v1.0)*

Audio / video player widget.

```json
{
  "type": "mediaPlayer",
  "source": "bundle://assets/intro.mp4",
  "mediaType": "video",
  "autoPlay": false,
  "loop": false,
  "muted": false,
  "volume": 0.8,
  "controls": true,
  "poster": "bundle://assets/intro_poster.png",
  "width": 640,
  "height": 360,
  "onPlay": { "type": "state", "action": "set", "binding": "isPlaying", "value": true },
  "onPause": { "type": "state", "action": "set", "binding": "isPlaying", "value": false },
  "onEnded": { "type": "state", "action": "set", "binding": "isPlaying", "value": false },
  "onTimeUpdate": {
    "type": "state", "action": "set", "binding": "currentTime", "value": "{{event.currentTime}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `source` | `AssetRef` | required | Media asset reference. |
| `mediaType` | enum | inferred | `audio`, `video` |
| `autoPlay` | boolean | `false` | Start playing automatically |
| `loop` | boolean | `false` | Loop at end |
| `muted` | boolean | `false` | Start muted |
| `volume` | number | 1.0 | Volume 0.0–1.0 |
| `controls` | boolean | `true` | Show native controls |
| `poster` | `AssetRef` | — | Preview image rendered before playback (video only). |
| `id` | string | — | Names this player so §4.9b media actions can drive it. Required to build a custom transport (`controls: false`). |
| `waveform` | boolean | `false` | Audio mode only — render the audio's amplitude waveform above the transport controls, advancing with playback. A runtime whose host does not supply amplitude data MUST report the capability absent through `onError` (§6.13.2) rather than accept the property and draw nothing. |
| `width` | number | null | Display width |
| `height` | number | null | Display height |
| `onPlay` | Action | null | Fired when playback starts |
| `onPause` | Action | null | Fired when playback pauses |
| `onEnded` | Action | null | Fired when media ends |
| `onTimeUpdate` | Action | null | Fired on time update; `event.currentTime` in seconds |
| `onError` | Action | null | Fired on playback error; `event.error` |

## 10.7 `calendar` *(since v1.0)*

Calendar view for date selection and event display.

```json
{
  "type": "calendar",
  "selectedDate": "{{selectedDate}}",
  "events": "{{calendarEvents}}",
  "firstDate": "2020-01-01",
  "lastDate": "2030-12-31",
  "view": "month",
  "onChange": {
    "type": "state", "action": "set",
    "binding": "selectedDate", "value": "{{event.value}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `selectedDate` | string \| binding (ISO 8601) | null | Currently selected date |
| `events` | binding | null | Array of `{ date, title, color? }` |
| `firstDate` | string (ISO 8601) | null | Earliest selectable date |
| `lastDate` | string (ISO 8601) | null | Latest selectable date |
| `view` | enum | `month` | `month`, `week`, `day` |
| `onChange` | Action | null | Fired on date selection; `event.value` is ISO date |

## 10.8 `timeline` *(since v1.0)*

Chronological timeline of events.

```json
{
  "type": "timeline",
  "orientation": "vertical",
  "items": [
    { "title": "Order Placed", "subtitle": "Your order has been confirmed",
      "icon": "shopping_cart", "time": "2025-01-15T09:00:00Z" },
    { "title": "Processing", "subtitle": "Preparing your order",
      "icon": "inventory", "time": "2025-01-15T10:30:00Z" },
    { "title": "Shipped", "subtitle": "On its way",
      "icon": "local_shipping", "time": "2025-01-15T14:00:00Z" }
  ]
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `items` | `TimelineItem[]` | required | Timeline entries |
| `items[].title` | string | required | Entry title |
| `items[].subtitle` | string | null | Secondary text |
| `items[].icon` | string | null | Icon name |
| `items[].time` | string (ISO 8601) | null | Event time |
| `items[].color` | string | theme | Marker color |
| `orientation` | enum | `vertical` | `vertical`, `horizontal` |

## 10.9 `gauge` *(since v1.0)*

Radial gauge for a value within a range.

```json
{
  "type": "gauge",
  "value": 72,
  "min": 0,
  "max": 100,
  "segments": [
    { "from": 0, "to": 30, "color": "#F44336" },
    { "from": 30, "to": 70, "color": "#FF9800" },
    { "from": 70, "to": 100, "color": "#4CAF50" }
  ],
  "size": 200,
  "strokeWidth": 20,
  "showLabel": true,
  "labelFormat": "{value}%",
  "startAngle": 135,
  "sweepAngle": 270
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `value` | number | required | Current value |
| `min` | number | 0 | Range minimum |
| `max` | number | 100 | Range maximum |
| `segments` | `Segment[]` | null | Color segments with `from`, `to`, `color` |
| `size` | number | 200 | Diameter (logical px) |
| `strokeWidth` | number | 10 | Arc thickness in logical pixels |
| `backgroundColor` | string | `#E0E0E0` | Track color |
| `valueColor` | string | theme primary | Value arc color (when no segments) |
| `showLabel` | boolean | `true` | Show numeric label |
| `labelFormat` | string | `{value}%` | Label pattern; `{value}` is the reading |
| `startAngle` | number | -220 | Start angle in degrees |
| `sweepAngle` | number | 260 | Sweep span in degrees |

## 10.10 `heatmap` *(since v1.0)*

Two-dimensional heatmap visualization.

```json
{
  "type": "heatmap",
  "data": "{{heatmapData}}",
  "columnLabels": ["Mon", "Tue", "Wed", "Thu", "Fri"],
  "rowLabels": ["Morning", "Afternoon", "Evening"],
  "cellSize": 40,
  "colorRange": { "low": "#E3F2FD", "high": "#1565C0" },
  "showValues": true
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `data` | array \| binding | required | 2D numeric array (rows × columns) |
| `columnLabels` | string[] | null | Horizontal axis labels |
| `rowLabels` | string[] | null | Vertical axis labels |
| `cellSize` | number | 40 | Cell size (logical px) |
| `colorRange` | `{ low, high }` | `{ "#E3F2FD", "#1565C0" }` | Hex gradient endpoints |
| `showValues` | boolean | `true` | Render numeric value inside each cell |
| `onCellTap` | Action | null | Fired on cell tap; `event.row`, `event.column`, `event.value` |

## 10.11 `tree` *(since v1.0)*

Hierarchical tree view with expandable nodes.

```json
{
  "type": "tree",
  "data": "{{fileStructure}}",
  "expandable": true,
  "childrenKey": "children",
  "itemTemplate": {
    "type": "listItem",
    "title": "{{item.name}}",
    "leading": { "type": "icon", "icon": "{{item.icon}}" }
  },
  "onNodeTap": {
    "type": "state", "action": "set",
    "binding": "selectedNodeId", "value": "{{event.id}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `data` | array \| binding | required | Hierarchical data; each node MAY carry a `children` array |
| `childrenKey` | string | `children` | Field name of the children array |
| `itemTemplate` | Widget | no | — | Template rendered per node; `{{item}}` is node data. Omitted, each node draws its label. |
| `expandable` | boolean | `true` | Allow expand/collapse |
| `initiallyExpanded` | boolean | `false` | Expand all nodes on mount |
| `selectable` | boolean | `false` | Tapping selects the node |
| `selectedColor` | Color | theme primary @ 20% | Selection highlight |
| `checkable` | boolean | `false` | Draw a checkbox per node |
| `checkedKeys` | string[] \| binding | `[]` | Checked node ids |
| `draggable` | boolean | `false` | Allow reordering and reparenting by drag. Reports through `onDrop`; the widget does not mutate the tree |
| `showLines` | boolean | `true` | Draw guide lines |
| `lineColor` | Color | theme outlineVariant | Guide line colour |
| `indentation` | Dimension | `24` | Indent per depth level |
| `itemPadding` | EdgeInsets | `{top:4, bottom:4, right:8}` | Padding inside every row; the vertical component is row density |
| `width` / `height` | number | — | Fixed widget size |
| `onNodeTap` | Action | null | Fired on node tap |
| `onSelect` | Action | null | Fired on selection; requires `selectable` |
| `onDrop` | Action | null | Fired when a dragged node is released on another |
| `onExpand` | Action | null | Fired on node expand; `event.id` |
| `onCollapse` | Action | null | Fired on node collapse; `event.id` |

**`onDrop` carries where it landed.** `event.item` is the node that moved, `event.target` the
node it was released on, and `event.position` one of `before` / `inside` / `after` — which edge
of the target it landed on. A move that cannot say where it landed is a move the document
cannot apply.

Every node is a drag source and a drop target, **expandable nodes included**: dropping on a
group's row with `position: "inside"` is the reparenting this event exists for, and a tree
where only leaves accept a drop cannot express it. The edge is measured against the target
**row**, not the widget, and a node is refused as a target for itself or anything in its own
subtree — that move has no consistent result.

## 10.12 `graph` *(since v1.0)*

Time-series / numeric data graph (line, bar, area, scatter) drawn on a single set of axes. For node/edge topology visualization use `networkGraph` (§10.13).

```json
{
  "type": "graph",
  "chartType": "line",
  "data": [
    { "x": 0, "y": 12 },
    { "x": 1, "y": 19 },
    { "x": 2, "y": 8 }
  ],
  "width": 300,
  "height": 200,
  "showGrid": true,
  "lineColor": "#2196F3"
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `data` | `Point[]` \| binding | required | Array of `{ x, y }` points (or `{ label, value }`). |
| `chartType` | enum | `line` | `line`, `bar`, `area`, `scatter`. Legacy alias: `type` (kept for backward compatibility; avoid — collides with the widget-type discriminator). |
| `width` | number | `300` | Render width in logical px. |
| `height` | number | `200` | Render height. |
| `showGrid` | boolean | `true` | Draw background grid. |
| `showLabels` | boolean | `true` | Draw axis labels. |
| `lineColor` | Color | `#2196F3` | Line / bar color. |
| `fillColor` | Color | — | Area fill color. |
| `gridColor` | Color | light grey | Grid color. |
| `strokeWidth` | number | `2` | Line stroke thickness. |

## 10.13 `networkGraph` *(since v1.0)*

Network topology graph. Same node/edge model as `graph`, with topology-oriented defaults (hierarchical layout, directed edges).

```json
{
  "type": "networkGraph",
  "nodes": "{{graphNodes}}",
  "edges": "{{graphEdges}}",
  "layout": "hierarchical",
  "directed": true
}
```

Properties are identical to `graph` (§10.12). A runtime MAY implement `networkGraph` as a configured preset of `graph`.

## 10.14 `codeEditor` *(since v1.0)*

Syntax-highlighted code editor.

```json
{
  "type": "codeEditor",
  "code": "{{sourceCode}}",
  "language": "javascript",
  "theme": "vsDark",
  "readOnly": false,
  "showLineNumbers": true,
  "fontSize": 14,
  "lineHeight": 1.5,
  "tabSize": 2,
  "width": 600,
  "height": 400,
  "onChange": {
    "type": "state", "action": "set",
    "binding": "sourceCode", "value": "{{event.value}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `code` | string \| binding | no | — | Code content. Required unless `binding` supplies it. |
| `language` | enum | `plaintext` | `plaintext`, `javascript`, `typescript`, `dart`, `python`, `java`, `kotlin`, `swift`, `go`, `rust`, `c`, `cpp`, `csharp`, `ruby`, `php`, `sql`, `json`, `yaml`, `xml`, `html`, `css`, `markdown`, `shell` |
| `theme` | enum | `vsDark` | `vsLight`, `vsDark`, `monokai`, `solarizedLight`, `solarizedDark`, `github`, `dracula` (legacy `light` / `dark` still read) |
| `readOnly` | boolean | `false` | Disable editing (syntax highlighting still applies) |
| `showLineNumbers` | boolean | `true` | Show gutter line numbers |
| `fontSize` | number | null | Font size (logical px) |
| `lineHeight` | number | null | Line height multiplier |
| `tabSize` | number | 2 | Tab width in spaces |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `backgroundColor` | string | null | Override theme background |
| `textColor` | string | null | Override theme default text color |
| `onChange` | Action | null | Fired on code change; `event.value` is current code |

Unknown `language` or `theme` values fall back to `plaintext` / `vsLight` respectively, with a warning.

## 10.15 `terminal` *(since v1.0)*

ANSI-capable terminal emulator.

```json
{
  "type": "terminal",
  "lines": "{{terminalHistory}}",
  "prompt": "$ ",
  "showInput": true,
  "maxLines": 1000,
  "width": 600,
  "height": 400,
  "fontSize": 14,
  "backgroundColor": "#1E1E1E",
  "textColor": "#D4D4D4",
  "promptColor": "#4EC9B0",
  "onCommand": {
    "type": "tool",
    "tool": "executeCommand",
    "params": { "command": "{{event.value}}" }
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `lines` | binding | null | Output lines array (ANSI-escaped strings accepted) |
| `prompt` | string | `$ ` | Input prompt prefix |
| `showInput` | boolean | `true` | Render an input line |
| `maxLines` | number | null | Scrollback retention cap |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `fontSize` | number | null | Font size (logical px) |
| `backgroundColor` | string | null | Terminal background |
| `textColor` | string | null | Default text color |
| `promptColor` | string | null | Prompt color |
| `onCommand` | Action | null | Fired on Enter; `event.value` is the submitted command |

## 10.16 `fileExplorer` *(since v1.0)*

File and directory browser.

```json
{
  "type": "fileExplorer",
  "items": "{{fileTree}}",
  "showIcons": true,
  "showHidden": false,
  "expandAll": false,
  "width": 300,
  "height": 500,
  "selectedColor": "#E3F2FD",
  "onSelect": {
    "type": "state", "action": "set",
    "binding": "selectedFile", "value": "{{event.value}}"
  },
  "onOpen": {
    "type": "tool",
    "tool": "openFile",
    "params": { "path": "{{event.value}}" }
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `items` | `object[]` \| binding | no | — | Hierarchical `{ name, path, type, children? }` tree — a literal array or a binding to one. Omitted, the explorer renders empty. |
| `showIcons` | boolean | `true` | Show file / folder icons |
| `showHidden` | boolean | `false` | Show entries whose name starts with `.` |
| `expandAll` | boolean | `false` | Expand all folders by default |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `selectedColor` | string | null | Background color for selected entry |
| `onSelect` | Action | null | Fired on selection; `event.value` is the selected path |
| `onOpen` | Action | null | Fired on activation (double-tap / Enter); `event.value` is the path |

## 10.17 `markdown` *(since v1.0)*

Markdown renderer.

```json
{
  "type": "markdown",
  "text": "# Hello World\n\nThis is **markdown** content with [a link](https://example.com).",
  "selectable": true,
  "width": 600,
  "fontSize": 16,
  "textColor": "#212121",
  "linkColor": "#1976D2",
  "codeBackgroundColor": "#F5F5F5",
  "onLinkTap": {
    "type": "navigation", "action": "push",
    "route": "{{event.url}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `text` | string \| binding | required | Markdown source |
| `selectable` | boolean | `false` | Allow text selection |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `fontSize` | number | null | Base font size |
| `textColor` | string | null | Default text color |
| `backgroundColor` | string | null | Surface behind the rendered document |
| `linkColor` | string | null | Hyperlink color |
| `codeBackgroundColor` | string | null | Code-block background |
| `onLinkTap` | Action | null | Fired on link tap; `event.url` is the target |

> The content field for `markdown` is `text` (canonical, matching the text widget per [`17_Naming.md`](17_Naming.md) §17.3.2). `content` is a legacy alias from v1.0.

## 10.18 `webView` *(since v1.0)*

Embedded web view.

```json
{
  "type": "webView",
  "url": "https://example.com",
  "allowNavigation": true,
  "enableJavaScript": true,
  "enableZoom": true,
  "width": 600,
  "height": 400,
  "onPageStarted": {
    "type": "state", "action": "set",
    "binding": "webViewLoading", "value": true
  },
  "onPageFinished": {
    "type": "state", "action": "set",
    "binding": "webViewLoading", "value": false
  },
  "onError": {
    "type": "state", "action": "set",
    "binding": "webViewError", "value": "{{event.error}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `url` | string | null (if `html` set) | URL to load |
| `html` | string | null (if `url` set) | Raw HTML content — alternative to `url` |
| `allowNavigation` | boolean | `true` | Allow link navigation within the view |
| `enableJavaScript` | boolean | `true` | Enable JavaScript |
| `enableZoom` | boolean | `true` | Allow pinch-to-zoom |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `onPageStarted` | Action | null | Fired when loading starts |
| `onPageFinished` | Action | null | Fired when loading finishes |
| `onError` | Action | null | Fired on load error; `event.error` |

Exactly one of `url` or `html` MUST be present. If both are present, `url` wins.

## 10.19 `signature` *(since v1.0)*

Signature capture pad.

```json
{
  "type": "signature",
  "binding": "{{signatureData}}",
  "penColor": "#000000",
  "penWidth": 2.0,
  "width": 400,
  "height": 200,
  "backgroundColor": "#FFFFFF",
  "borderColor": "#E0E0E0",
  "showClearButton": true,
  "showGuide": true,
  "onSignatureEnd": {
    "type": "state", "action": "set",
    "binding": "signatureData", "value": "{{event.value}}"
  },
  "onClear": {
    "type": "state", "action": "set",
    "binding": "signatureData", "value": null
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `binding` | binding | no | Target binding for the signature. Written as a `data:image/png;base64,…` URI once a stroke completes, and `null` once the pad is cleared. Omitted, the pad still draws; the stroke goes nowhere. |
| `penColor` | string | `#000000` | Stroke color |
| `penWidth` | number | 2.0 | Stroke width |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `backgroundColor` | string | null | Pad background |
| `borderColor` | string | null | Pad border |
| `showClearButton` | boolean | `true` | Show a clear-signature button |
| `showGuide` | boolean | `true` | Show a signing guide line |
| `onSignatureEnd` | Action | null | Fired when a stroke completes. `event.value` carries the same `data:` URI the binding receives; `event.strokes` carries the stroke coordinates for a document that wants the vector rather than the picture; `event.strokeCount` and `event.hasSignature` describe what is on the pad. Fired **after** the encode, so it is asynchronous with respect to the gesture. |
| `onClear` | Action | null | Fired when the signature is cleared |

## 10.20 `canvas` *(since v1.3)*

General-purpose vector drawing via an ordered command array. All numeric and color properties support binding expressions, enabling data-driven graphics (progress rings, custom indicators, composable chart primitives).

```json
{
  "type": "canvas",
  "width": 300,
  "height": 200,
  "backgroundColor": "#FFFFFF",
  "commands": [
    { "op": "rect", "x": 10, "y": 10, "width": 100, "height": 80,
      "fill": "#FF5722", "cornerRadius": 8 },
    { "op": "circle", "cx": 200, "cy": 100, "radius": 40,
      "fill": "{{app.color}}", "stroke": "#333333", "strokeWidth": 2 },
    { "op": "arc", "cx": 60, "cy": 150, "radius": 30,
      "startAngle": 3.14, "endAngle": "{{3.14 + (app.progress / 100) * 3.14}}",
      "stroke": "#4CAF50", "strokeWidth": 4, "strokeCap": "round" },
    { "op": "line", "x1": 10, "y1": 190, "x2": 290, "y2": 190,
      "stroke": "#E0E0E0", "strokeWidth": 1 },
    { "op": "path", "d": "M 10 10 L 90 10 L 90 90 Z",
      "fill": "rgba(33,150,243,0.2)", "stroke": "#2196F3", "strokeWidth": 1 },
    { "op": "text", "text": "{{app.label}}", "x": 10, "y": 180,
      "fontSize": 14, "color": "#333333", "textAlign": "start" },
    { "op": "image", "src": "bundle://icons/logo.png",
      "x": 250, "y": 10, "width": 40, "height": 40, "opacity": 0.9 }
  ]
}
```

### 10.20.1 Canvas Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `width` | number | yes | Canvas width (logical px) |
| `height` | number | yes | Canvas height (logical px) |
| `commands` | `Command[]` | yes | Ordered list of drawing commands |
| `backgroundColor` | string | no | Canvas background color |

### 10.20.2 Command `op` Enumeration

The `op` field is **strictly enumerated**. Allowed values:

| `op` | Purpose |
|------|---------|
| `rect` | Rectangle (filled and/or stroked, optional rounded corners) |
| `circle` | Circle |
| `arc` | Arc or partial circle (filled and/or stroked) |
| `line` | Straight line segment |
| `path` | SVG-style path data string |
| `text` | Text label |
| `image` | Embedded image |

A command whose `op` is not in the list above MUST be **skipped** by the runtime and SHOULD produce a warning in the runtime log. A skipped command MUST NOT abort rendering of subsequent commands.

### 10.20.3 Command Schemas

**`rect`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `x`, `y` | number | required | Top-left corner |
| `width`, `height` | number | required | Rectangle size |
| `fill` | string | null | Fill color (null = no fill) |
| `stroke` | string | null | Stroke color (null = no stroke) |
| `strokeWidth` | number | 1 | Stroke width |
| `cornerRadius` | number | 0 | Corner radius |

**`circle`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `cx`, `cy` | number | required | Center |
| `radius` | number | required | Radius |
| `fill` | string | null | Fill color |
| `stroke` | string | null | Stroke color |
| `strokeWidth` | number | 1 | Stroke width |

**`arc`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `cx`, `cy` | number | required | Center |
| `radius` | number | required | Radius |
| `startAngle` | number | required | Start angle (radians) |
| `endAngle` | number | required | End angle (radians) |
| `fill` | string | null | Fill color (for filled arc / pie slice) |
| `stroke` | string | null | Stroke color |
| `strokeWidth` | number | 1 | Stroke width |
| `strokeCap` | enum | `butt` | `butt`, `round`, `square` |

**`line`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `x1`, `y1`, `x2`, `y2` | number | required | Endpoint coordinates |
| `stroke` | string | theme | Stroke color |
| `strokeWidth` | number | 1 | Stroke width |
| `strokeCap` | enum | `butt` | `butt`, `round`, `square` |

**`path`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `d` | string | required | SVG path data (`M`, `L`, `C`, `Q`, `A`, `Z` commands) |
| `fill` | string | null | Fill color |
| `stroke` | string | null | Stroke color |
| `strokeWidth` | number | 1 | Stroke width |

**`text`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `text` | string | required | Text content |
| `x`, `y` | number | required | Baseline anchor |
| `fontSize` | number | 14 | Font size |
| `fontWeight` | enum | `normal` | `normal`, `bold`, or numeric `100`–`900` |
| `color` | string | theme | Text color |
| `textAlign` | enum | `start` | `start`, `center`, `end` |

**`image`**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `src` | string | required | Image URL (`bundle://`, `https://`, `client://`) |
| `x`, `y` | number | required | Top-left corner |
| `width`, `height` | number | required | Rendered size |
| `opacity` | number | 1.0 | Rendering opacity 0.0–1.0 |

### 10.20.4 Canvas in Template Example

```json
{
  "type": "template",
  "name": "progressRing",
  "stateDefaults": {},
  "params": {
    "value": { "type": "number", "required": true },
    "max": { "type": "number", "default": 100 },
    "color": { "type": "string", "default": "#4CAF50" },
    "size": { "type": "number", "default": 60 }
  },
  "content": {
    "type": "canvas",
    "width": "{{size}}",
    "height": "{{size}}",
    "commands": [
      { "op": "arc", "cx": "{{size / 2}}", "cy": "{{size / 2}}", "radius": "{{size / 2 - 4}}",
        "startAngle": 0, "endAngle": 6.28,
        "stroke": "#E0E0E0", "strokeWidth": 4 },
      { "op": "arc", "cx": "{{size / 2}}", "cy": "{{size / 2}}", "radius": "{{size / 2 - 4}}",
        "startAngle": -1.57, "endAngle": "{{-1.57 + (value / max) * 6.28}}",
        "stroke": "{{color}}", "strokeWidth": 4, "strokeCap": "round" },
      { "op": "text", "text": "{{value}}%", "x": "{{size / 2}}", "y": "{{size / 2 + 5}}",
        "fontSize": "{{size * 0.25}}", "textAlign": "center", "color": "#333333" }
    ]
  }
}
```

## 10.21 `lightbox` *(since v1.3)*

Full-screen modal image viewer with pinch-zoom and swipe-between-images. Used as a tap target on a thumbnail — opening the lightbox transitions the thumbnail into the full-screen view. Pairs naturally with `hero` for the cover-to-detail morph.

```json
{
  "type": "lightbox",
  "images": "{{photos}}",
  "initialIndex": "{{tappedIndex}}",
  "allowZoom": true,
  "maxZoom": 6,
  "onClose": {
    "type": "state", "action": "set",
    "binding": "lightboxOpen", "value": false
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `images` | array\<`AssetRef`\> | required | Image asset list. |
| `initialIndex` | number | `0` | Image shown first. |
| `allowZoom` | boolean | `true` | Enable pinch-zoom and double-tap-to-zoom. |
| `maxZoom` | number | `4.0` | Maximum zoom factor (1.0 = fit-bounds). |
| `allowSwipe` | boolean | `true` | Enable swipe-between-images. |
| `backgroundColor` | Color | `"#FF000000"` | Backdrop color. |
| `onIndexChanged` | Action | — | Fires when the user swipes to a new image. `event.index` carries the new index. |
| `onClose` | Action | — | Fires when the user dismisses the lightbox. |

## 10.22 `lazy` *(since v1.0)*

Defer rendering of an expensive subtree until it enters the viewport (or until explicitly loaded). `lazy` belongs to the Utility group in [`02_Widgets.md`](02_Widgets.md) and is listed here because its schema pairs conceptually with the heavy Advanced widgets above.

```json
{
  "type": "lazy",
  "placeholder": {
    "type": "box",
    "height": 200,
    "child": { "type": "progressBar" }
  },
  "content": {
    "source": "ui://pages/heavy-component"
  }
}
```

Inline-defined content variant:

```json
{
  "type": "lazy",
  "placeholder": {
    "type": "box",
    "height": 120,
    "child": { "type": "text", "text": "Loading chart…" }
  },
  "content": {
    "type": "chart",
    "chartType": "line",
    "data": "{{salesData}}"
  }
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `placeholder` | Widget | null | Rendered while `content` is not yet materialized |
| `content` | Widget \| object | — | Inline widget, or `{ source: "ui://..." }` naming a fragment to fetch — resolved the same way `view` resolves a `DefinitionSource` (§2.13.1). Required when `children` / `child` are omitted. |
| `trigger` | enum | `visible` | `visible` (render when the widget becomes visible to the user), `immediate` (render on mount), `manual` (render when a `load()` signal is received) |
| `onLoad` | Action | null | Fired after `content` is materialized |
| `onError` | Action | null | Fired if `content` fetch fails; `event.error` |


## 10.23 `qrCode` *(since v1.4)*

Renders a QR code for `value`. Cannot be composed — the module grid is computed, not laid out. The definition is small in exchange, and it pairs with the scanning side the platform already has: a code produced here is the same artefact [`08_Client_Extensions.md`](08_Client_Extensions.md) §8.9 reads on arrival.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `value` | string \| binding | required | Encoded payload — a URL, a `ui://` route, or an entry token |
| `size` | number | `200` | Edge length in logical pixels; the grid is square |
| `errorCorrection` | enum | `medium` | `low` (7%), `medium` (15%), `quartile` (25%), `high` (30%). Higher survives damage at the cost of density — use `high` when a logo overlays the centre. The QR standard's own `L`/`M`/`Q`/`H` are legacy spellings a runtime MAY accept (§17.3); they are not declared values. |
| `foregroundColor` | Color | — | Module color. Contrast must stay scannable; a runtime SHOULD refuse to render below it rather than emit an unreadable code |
| `backgroundColor` | Color | — | Quiet-zone and gap color |
| `margin` | boolean | `true` | Include the quiet zone. Omitting it breaks scanning against busy backgrounds |
| `logo` | AssetRef | — | Image centred over the code. Requires a higher `errorCorrection` (`quartile` or `high`) to stay scannable |

## 10.24 `barcode` *(since v1.4)*

Renders a 1-D barcode. Like `qrCode` the bar pattern is computed; unlike it, each `format` constrains what `value` may contain — an EAN-13 payload is thirteen digits with a check digit, not arbitrary text.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `value` | string \| binding | required | Payload. MUST satisfy `format`; a runtime reports a violation rather than rendering an unscannable image |
| `format` | enum | `code128` | `code128`, `code39`, `ean13`, `ean8`, `upcA`, `upcE`, `itf`, `codabar` |
| `width` | number | intrinsic | Omitted means the natural width for the module count, which is what stays scannable |
| `height` | number | `80` | Bar height |
| `displayValue` | boolean | `true` | Print the payload under the bars |
| `foregroundColor` / `backgroundColor` | Color | — | Colors |

## 10.25 `pdfViewer` *(since v1.4)*

Renders a PDF with page navigation and zoom. `webView` can display a PDF where the host has a plugin, and that is the substitution to avoid: the document then belongs to the browser, so page position, zoom, and search sit outside the DSL's reach and outside the app's theme.

`src` is an `AssetRef`, so the same document works from a bundle, a URL, a picked file (`fileInput` writes a `data:` URI), or a server resource.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `src` | AssetRef | required | Document source |
| `page` | number \| binding | — | Two-way bound current page, 1-based |
| `zoom` | number \| binding | — | Two-way bound zoom; `1.0` is fit-width |
| `showToolbar` | boolean | `true` | Built-in toolbar. False leaves navigation to bindings |
| `showPageNav` / `showZoom` | boolean | `true` | Toolbar controls |
| `fit` | enum | `width` | `width`, `height`, `page` |

Events: `onLoad` (carries page count), `onPageChange` (however it changed), `onError`.

## 10.26 `diffViewer` *(since v1.4)*

Comparison of two texts. Separate from `codeEditor` rather than a mode on it, because the input is **two documents**: a `mode` flag on a single-value widget would leave one of the two with nowhere to bind. Read-only by design — editing a diff means editing one side, which is `codeEditor`'s job.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `oldValue` / `newValue` | string \| binding | required | Base and changed text |
| `splitView` | boolean | `true` | Two columns; false is unified |
| `language` | string | — | Highlighting, same vocabulary as `codeEditor.language` |
| `showLineNumbers` | boolean | `true` | Gutter line numbers |
| `contextLines` | number | — | Unchanged lines kept around each change; omitted shows the whole document |
| `highlightLines` | number[] | — | Extra emphasis on new-side lines |

## 10.27 `richTextEditor` *(since v1.4)*

Formatted text entry. Shared rows per §2.6.0.

**The bound value is HTML** (or Markdown via `format`), and that choice is why this needed a decision rather than a definition: an editor's value format is a contract every consumer of the document inherits, and a proprietary delta model would make the content unreadable to anything but the editor that produced it. A host that cannot edit can still display HTML.

A runtime MUST sanitise on the way in and on the way out — [`07_Security.md`](07_Security.md) §7.5 applies to this value exactly as to any other author-supplied markup.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `format` | enum | `html` | `html`, `markdown`. HTML is limited to inline marks, block elements, and `img` whose `src` is an `AssetRef`; anything else is **stripped, not escaped** |
| `toolbar` | string[] | runtime default | Controls shown. A control absent here MUST also be unreachable by shortcut, or the toolbar lies about what the document can contain |
| `placeholder` | string | — | Shown while empty |
| `minHeight` | number | — | Minimum height |
| `maxLength` | number | — | Ceiling on text content, not markup |

## 10.28 `splitter` *(since v1.4)*

Panes separated by draggable gutters. Distinct from `resizable`, which resizes one box against the layout around it: this divides a **fixed area** between siblings, so dragging takes space from one pane and gives it to the next and the total never changes.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `children` | Widget[] | required | Panes in order; gutters appear between them |
| `orientation` | enum | `horizontal` | Direction panes lie along |
| `sizes` | number[] \| binding | even | Fractions summing to 1. Two-way bindable so a layout can be restored |
| `minSizes` | number[] | — | Per-pane minimums. A drag stops at them rather than collapsing a pane not marked collapsible |
| `gutterSize` | number | `8` | Thickness. Keep it hittable with a finger where touch is possible |
| `collapsible` | boolean[] | — | Per-pane: whether dragging past the minimum collapses it |

## 10.29 `resizable` *(since v1.4)*

A box the user resizes by dragging its edges or corners. Distinct from `splitter`: this changes one widget's own size against whatever surrounds it.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `child` | Widget | required | Content being sized |
| `width` / `height` | number \| binding | — | Two-way bound; bind to persist |
| `minWidth` / `maxWidth` / `minHeight` / `maxHeight` | number | — | Bounds |
| `handles` | string[] | `[bottomEnd]` | Draggable edges/corners |
| `keepAspectRatio` | boolean | `false` | Constrain to the starting ratio |

Events: `onResize` (per frame), `onResizeEnd` (on release — prefer this for persistence).

## 10.30 `kanban` *(since v1.4)*

Cards in columns, moved between columns by dragging.

The pieces for a composition exist (`grid` + `draggable` + `dragTarget`) and that composition is where the work actually is: drop targets must be the **gaps between cards** rather than the cards, the column must auto-scroll while a card hovers near its edge, and a drop must report *where* in the destination the card landed. Authors composing this get a board that looks right and reorders wrongly.

The widget owns presentation and gesture. It does **not** own the move: `onCardMove` reports intent and the author's action decides, so a server-rejected move is not silently already applied on screen.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `columns` | object[] \| binding | required | `{ key, title, items, limit?, color? }`. `limit` refuses an over-capacity drop **at the gesture**, not after |
| `itemTemplate` | Widget | required | Card body, rendered per item with `item` in scope |
| `itemKey` | string | `"id"` | Field identifying a card — stable identity is what makes a move addressable |
| `draggable` | boolean | `true` | False renders a read-only board |
| `columnWidth` | Dimension | — | Fixed width; omitted distributes available width |
| `height` | Dimension | — | Board height. Omitted, the board fills its parent, which must then be bounded (§2.15) |
| `optimistic` | boolean | `false` | Move on screen before `onCardMove` resolves. False keeps the board as the truth the server confirmed |

Events: `onCardMove` (`{ item, from: {column, index}, to: {column, index} }` — the destination index matters, since a board without ordering is a list of columns), `onCardClick`.

**The board cannot size to its content** — each column scrolls its own cards — so it needs a
height from `height` or from a bounded parent. No default is invented when neither is given:
a made-up height is a layout that looks deliberate and is not.

**A drop lands anywhere in a column**: in the gaps between cards, on a card (its upper half
inserts before it, its lower half after), and in the space below the last card. Gaps alone
leave a column with cards in it mostly not a drop target.

**`optimistic` holds while the bound data is unchanged**, and is replaced when the data
changes. The board compares incoming `columns` by content: compared by identity, the state
write that `onCardMove` itself provokes reads as new data and undoes the move on the next
frame. One consequence for a document: a server refusal that leaves the data identical is
"nothing changed" and the card stays, so a refusal that must show sends the data the server
holds.

## 10.31 `gantt` *(since v1.4)*

Tasks on a time axis, with dependencies and progress.

What makes this a widget rather than a composition is the **axis**. Bars are positioned by time, not by index, so the row and the header must share one scale, stay aligned through zoom and horizontal scroll, and place a task whose span is shorter than a pixel without dropping it. Composing it from `grid` gives rows that drift from the header as soon as either scrolls.

Dependencies are drawn, not enforced: the widget renders the arrows and reports an edit; whether a move is legal is the server's answer.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `tasks` | object[] \| binding | required | `{ id, label, start, end, progress?, dependsOn?, group?, color? }`; times are ISO-8601 |
| `viewMode` | enum | `day` | `hour`, `day`, `week`, `month`, `quarter`, `year` |
| `range` | object | fits tasks | `{ start, end }` window shown |
| `editable` | boolean | `false` | Drag bars to move, drag edges to reschedule |
| `showProgress` | boolean | `true` | Render each task's `progress` as a fill: the completed fraction in the bar's colour, the remainder in a lighter tint of it |
| `showDependencies` | boolean | `true` | Draw dependency arrows, from the end of each task a row depends on to the start of that row's task |
| `todayMarker` | boolean | `true` | Mark the current instant on the axis |
| `rowHeight` | number | — | Task row height |

Events: `onTaskChange` (`{ id, start, end }` — the **proposed** schedule, not an applied one), `onTaskClick`.

**The header labels by granularity.** A unit of a day or longer names a *span*, so its label is
centred in the cell between two ticks, over the bar it dates — the reading every calendar and
gantt trains. A unit shorter than a day names a *point*, so its label sits beside the tick and
reads the time, with the date where the day turns. Ticks stay on cell boundaries in both. A
label that would overlap the previous one is dropped rather than drawn over it, so a dense
scale shows fewer dates instead of unreadable ones.

## 10.32 `spreadsheet` *(since v1.4)*

Editable grid of cells addressed by row and column, with optional formulas.

Distinct from `dataTable`, which presents **records**: a table's unit is a row with typed fields, a spreadsheet's unit is a **cell with a coordinate**. That is why one cannot be a mode of the other — selection, editing, and paste all address different things.

**Formulas are evaluated by the runtime's expression engine under [`07_Security.md`](07_Security.md) §7.1**, not by a spreadsheet language of the DSL's own. A runtime that cannot meet that sandbox MUST render formulas as their last computed value rather than evaluating them loosely: a cell that runs arbitrary text is the one place this widget could become an injection surface.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `data` | array[] \| binding | required | Rows of cell values. A cell is a scalar, or `{ value, formula?, format?, style? }` |
| `columns` | object[] | derived | `{ key?, label?, width?, type?, readOnly? }`; omitted labels them A, B, C… |
| `rowHeaders` / `columnHeaders` | boolean | `true` | Gutter and header |
| `editable` | boolean | `true` | Allow cell editing |
| `formulas` | boolean | `false` | Evaluate `formula` fields. Off by default — a document that needs no computation should not carry an evaluator |
| `frozenRows` / `frozenColumns` | number | `0` | Panes pinned while scrolling |

Events: `onChange` (`{ row, column, value, previous }`), `onCellSelect`.
