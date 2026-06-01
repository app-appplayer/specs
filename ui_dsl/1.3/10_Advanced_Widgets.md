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
| `data` | object | required | `{ labels: string[], datasets: Dataset[] }` |
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
| `rows` | `{ cells: Widget[] }[]` | required | Row definitions |
| `border` | `{ color, width }` | null | Optional cell border |
| `defaultColumnWidth` | enum | `flex` | `flex`, `intrinsic`, or a number (fixed px) |
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
| `rows` | binding | required | Array of row objects |
| `selectable` | boolean | `false` | Whether rows are selectable |
| `sortColumn` | binding | null | Current sort column key |
| `sortAscending` | binding | `true` | Current sort direction |
| `onSort` | Action | null | Fired on header tap of a sortable column |
| `onRowTap` | Action | null | Fired on row tap; `event.row` is the row object |

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
| `center` | `{ latitude, longitude }` | required | Map center coordinates |
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
| `waveform` | boolean | `false` | Audio mode only — render the audio's amplitude waveform above the transport controls, advancing with playback. |
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
| `selectedDate` | binding (ISO 8601) | null | Currently selected date |
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
| `strokeWidth` | number | 20 | Arc thickness |
| `backgroundColor` | string | `#E0E0E0` | Track color |
| `valueColor` | string | theme primary | Value arc color (when no segments) |
| `showLabel` | boolean | `true` | Show numeric label |
| `labelFormat` | string | `{value}` | Label format pattern |
| `startAngle` | number | 135 | Start angle in degrees |
| `sweepAngle` | number | 270 | Sweep span in degrees |

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
| `data` | binding | required | 2D numeric array (rows × columns) |
| `columnLabels` | string[] | null | Horizontal axis labels |
| `rowLabels` | string[] | null | Vertical axis labels |
| `cellSize` | number | 40 | Cell size (logical px) |
| `colorRange` | `{ low, high }` | `{ "#E3F2FD", "#1565C0" }` | Hex gradient endpoints |
| `showValues` | boolean | `false` | Render numeric value inside each cell |
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
| `data` | binding | required | Hierarchical data; each node MAY carry a `children` array |
| `childrenKey` | string | `children` | Field name of the children array |
| `itemTemplate` | Widget | required | Template rendered per node; `{{item}}` is node data |
| `expandable` | boolean | `true` | Allow expand/collapse |
| `initiallyExpanded` | boolean | `false` | Expand all nodes on mount |
| `onNodeTap` | Action | null | Fired on node tap |
| `onExpand` | Action | null | Fired on node expand; `event.id` |
| `onCollapse` | Action | null | Fired on node collapse; `event.id` |

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
| `code` | binding | required | Code content |
| `language` | enum | `plaintext` | `plaintext`, `javascript`, `typescript`, `dart`, `python`, `java`, `kotlin`, `swift`, `go`, `rust`, `c`, `cpp`, `csharp`, `ruby`, `php`, `sql`, `json`, `yaml`, `xml`, `html`, `css`, `markdown`, `shell` |
| `theme` | enum | `vsLight` | `vsLight`, `vsDark`, `monokai`, `solarizedLight`, `solarizedDark`, `github`, `dracula` |
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
| `items` | binding | required | Hierarchical `{ name, path, type, children? }` tree |
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
| `binding` | binding | required | Target binding for signature data (base64 PNG or SVG path) |
| `penColor` | string | `#000000` | Stroke color |
| `penWidth` | number | 2.0 | Stroke width |
| `width` | number | null | Widget width |
| `height` | number | null | Widget height |
| `backgroundColor` | string | null | Pad background |
| `borderColor` | string | null | Pad border |
| `showClearButton` | boolean | `true` | Show a clear-signature button |
| `showGuide` | boolean | `false` | Show a signing guide line |
| `onSignatureEnd` | Action | null | Fired when a stroke completes |
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
| `content` | Widget \| `{ source }` | required | Inline widget, or `{ source: "ui://..." }` to fetch a remote page fragment |
| `trigger` | enum | `viewport` | `viewport` (render when scrolled into view), `immediate` (render on mount), `manual` (render when `load()` signal is received) |
| `onLoad` | Action | null | Fired after `content` is materialized |
| `onError` | Action | null | Fired if `content` fetch fails; `event.error` |
