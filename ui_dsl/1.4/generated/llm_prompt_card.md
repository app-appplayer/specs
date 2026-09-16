<!-- GENERATED: do not edit. Source: 1.4/widgets/*.yaml. -->
# MCP UI DSL — LLM Prompt Card

Authoritative catalog of every widget type recognised by the MCP UI DSL runtime. Use as context when generating or editing DSL. Widget types and property names listed here are the only sanctioned ones; anything else is either a legacy alias (§17.3) or invalid.

Format per widget:
- `type` — canonical widget type name (camelCase).
- `aliases` — legacy names accepted by the runtime.
- `properties` — supported top-level properties (`?` = optional).
- `children` — how children are passed, if any.
- `events` — widget-level event handlers (Action payload).

## Advanced

### `barcode` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `value: string | binding`, `format?: string`, `width?: number`, `height?: number`, `displayValue?: boolean`, `foregroundColor?: Color`, `backgroundColor?: Color`

### `calendar` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `eventColor?: Color`, `firstDayOfWeek?: number`, `onMonthChange?: Action`, `primaryColor?: Color`, `showHeader?: boolean`, `showWeekNumbers?: boolean`, `todayColor?: Color`, `height?: Dimension`, `selectedDate?: string | binding`, `events?: array<object>`, `firstDate?: string`, `lastDate?: string`, `view?: string`, `onChange?: Action`, `backgroundColor?: Color`, `change?: Action`, `selectedColor?: Color`, `width?: Dimension`

### `canvas` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `width?: Dimension`

### `chart` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `colors?: array<Color>`, `primaryColor?: Color`, `showGrid?: boolean`, `showLabels?: boolean`, `showLegend?: boolean`, `labelColor?: Color`, `chartType: string`, `data: object | array`, `data.datasets[].label?: string`, `data.datasets[].data: array<number>`, `data.datasets[].borderColor?: string`, `data.datasets[].backgroundColor?: string`, `options?: object`, `options.responsive?: boolean`, `options.animation.duration?: number`, `options.legend.position?: string`, `width?: number`, `height?: number`, `backgroundColor?: Color`, `gridColor?: Color`, `title?: string`

### `codeEditor` *(since v1.0)*
- aliases: `code`
- properties: `click?: Action`, `tooltip?: string`, `lineNumberColor?: Color`, `copyable?: boolean`, `expandAll?: boolean`, `code?: string | binding`, `language?: string`, `theme?: string`, `readOnly?: boolean`, `showLineNumbers?: boolean`, `fontSize?: number`, `lineHeight?: number`, `tabSize?: number`, `width?: number`, `height?: number`, `backgroundColor?: Color`, `textColor?: Color`, `onChange?: Action`, `binding?: string`, `change?: Action`

### `dataTable` *(since v1.0)*
- aliases: `dataGrid`
- properties: `click?: Action`, `tooltip?: string`, `editable?: boolean`, `filterable?: boolean`, `resizableColumns?: boolean`, `virtualScroll?: boolean`, `rowHeight?: number`, `columns: array<Column>`, `columns[].key: string`, `columns[].label: string`, `columns[].width?: number`, `columns[].sortable?: boolean`, `columns[].align?: string`, `rows: array<object> | binding`, `selectable?: boolean`, `sortColumn?: string`, `sortAscending?: boolean`, `onSort?: Action`, `onCellEdit?: Action`, `onRowTap?: Action`

### `diffViewer` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `oldValue: string | binding`, `newValue: string | binding`, `splitView?: boolean`, `language?: string`, `showLineNumbers?: boolean`, `contextLines?: number`, `highlightLines?: array<number>`

### `fileExplorer` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `iconColor?: Color`, `items?: array<object> | binding`, `rootPath?: string`, `files?: array<string>`, `directories?: array<string>`, `showIcons?: boolean`, `showHidden?: boolean`, `expandAll?: boolean`, `width?: number`, `height?: number`, `selectedColor?: Color`, `onSelect?: Action`, `onOpen?: Action`, `backgroundColor?: Color`

### `gantt` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `tasks: array<object> | binding`, `viewMode?: string`, `range?: object`, `editable?: boolean`, `showProgress?: boolean`, `showDependencies?: boolean`, `todayMarker?: boolean`, `rowHeight?: number`
- events: `onTaskChange`, `onTaskClick`

### `gauge` *(since v1.0)*
- aliases: `meter`
- properties: `click?: Action`, `tooltip?: string`, `value: number`, `min?: number`, `max?: number`, `segments?: array<Segment>`, `size?: number`, `strokeWidth?: number`, `backgroundColor?: Color`, `valueColor?: Color`, `showLabel?: boolean`, `labelFormat?: string`, `startAngle?: number`, `sweepAngle?: number`

### `graph` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `labelColor?: Color`, `data: array<Point>`, `chartType?: string`, `width?: number`, `height?: number`, `showGrid?: boolean`, `showLabels?: boolean`, `lineColor?: Color`, `fillColor?: Color`, `gridColor?: Color`, `strokeWidth?: number`

### `heatmap` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `cellGap?: Dimension`, `colorScheme?: string`, `columns?: number`, `maxValue?: number`, `minValue?: number`, `showLabels?: boolean`, `data: array | binding`, `columnLabels?: array<string>`, `rowLabels?: array<string>`, `cellSize?: number`, `colorRange?: { low, high }`, `colorRange.low?: Color`, `colorRange.high?: Color`, `showValues?: boolean`, `onCellTap?: Action`

### `kanban` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `columns: array<object> | binding`, `itemTemplate: Widget`, `itemKey?: string`, `draggable?: boolean`, `columnWidth?: Dimension`, `optimistic?: boolean`
- events: `onCardMove`, `onCardClick`

### `lightbox` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `images: array<AssetRef>`, `initialIndex?: number`, `allowZoom?: boolean`, `maxZoom?: number`, `allowSwipe?: boolean`, `backgroundColor?: Color`, `onIndexChanged?: Action`, `onClose?: Action`

### `map` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `markerColor?: Color`, `showCoordinates?: boolean`, `showGrid?: boolean`, `height?: Dimension`, `interactive?: boolean`, `center?: { latitude, longitude }`, `latitude?: number`, `longitude?: number`, `zoom?: number`, `mapType?: string`, `markers?: array<Marker>`, `markers[].id: string`, `markers[].latitude: number`, `markers[].longitude: number`, `markers[].label?: string`, `markers[].icon?: string`, `markers[].color?: string`, `overlays?: array<Overlay>`, `overlays[].type: string`, `overlays[].points?: array<object{ latitude: number, longitude: number }>`, `overlays[].fillColor?: string`, `overlays[].strokeColor?: string`, `overlays[].strokeWidth?: number`, `onMarkerTap?: Action`, `onMapTap?: Action`, `backgroundColor?: Color`, `gridColor?: Color`, `width?: Dimension`

### `markdown` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `text: string | binding`, `selectable?: boolean`, `width?: number`, `height?: number`, `fontSize?: number`, `textColor?: Color`, `backgroundColor?: Color`, `linkColor?: Color`, `codeBackgroundColor?: Color`, `onLinkTap?: Action`

### `mediaPlayer` *(since v1.0)*
- aliases: `video`, `audio`
- properties: `click?: Action`, `tooltip?: string`, `accentColor?: Color`, `controlsColor?: Color`, `onSeek?: Action`, `source: AssetRef`, `mediaType?: string`, `autoPlay?: boolean`, `loop?: boolean`, `muted?: boolean`, `volume?: number`, `controls?: boolean`, `poster?: AssetRef`, `waveform?: boolean`, `width?: number`, `height?: number`, `onPlay?: Action`, `onPause?: Action`, `onEnded?: Action`, `onTimeUpdate?: Action`, `onError?: Action`, `backgroundColor?: Color`, `duration?: Dimension`, `title?: string`

### `networkGraph` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `onNodeTap?: Action`, `edgeColor?: Color`, `edges?: array<object> | binding`, `layout?: string`, `nodeColor?: Color`, `nodes?: array<object> | binding`, `onEdgeTap?: Action`, `height?: Dimension`, `interactive?: boolean`, `labelColor?: Color`, `width?: Dimension`

### `pdfViewer` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `height?: Dimension`, `src: AssetRef`, `page?: number | binding`, `zoom?: number | binding`, `showToolbar?: boolean`, `showPageNav?: boolean`, `showZoom?: boolean`, `fit?: string`
- events: `onLoad`, `onPageChange`, `onError`

### `qrCode` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `value: string | binding`, `size?: number`, `errorCorrection?: string`, `foregroundColor?: Color`, `backgroundColor?: Color`, `margin?: boolean`, `logo?: AssetRef`

### `richTextEditor` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `format?: string`, `toolbar?: array<string>`, `placeholder?: string`, `minHeight?: number`, `maxLength?: number`, `binding?: string`, `onChange?: Action`

### `signature` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `onSignatureStart?: Action`, `borderWidth?: Dimension`, `binding?: binding`, `penColor?: Color`, `penWidth?: number`, `width?: number`, `height?: number`, `backgroundColor?: Color`, `borderColor?: Color`, `showClearButton?: boolean`, `showGuide?: boolean`, `onSignatureEnd?: Action`, `onClear?: Action`

### `spreadsheet` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `data: array<array> | binding`, `columns?: array<object>`, `rowHeaders?: boolean`, `columnHeaders?: boolean`, `editable?: boolean`, `formulas?: boolean`, `frozenRows?: number`, `frozenColumns?: number`
- events: `onChange`, `onCellSelect`

### `table` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `textBaseline?: string`, `rows: array<TableRow>`, `border?: { color, width }`, `defaultColumnWidth?: string | number`, `defaultVerticalAlignment?: string`, `columnWidths?: object`, `textDirection?: string`

### `terminal` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `lines?: array<string>`, `prompt?: string`, `showInput?: boolean`, `maxLines?: number`, `width?: number`, `height?: number`, `fontSize?: number`, `backgroundColor?: Color`, `textColor?: Color`, `promptColor?: Color`, `onCommand?: Action`, `theme?: string`

### `timeline` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `nodeSize?: Dimension`, `lineWidth?: Dimension`, `spacing?: Dimension`, `items: array<TimelineItem>`, `items[].title: string`, `items[].subtitle?: string`, `items[].icon?: string`, `items[].time?: string`, `items[].color?: string`, `orientation?: string`, `itemTemplate?: Widget`, `lineColor?: Color`

### `tree` *(since v1.0)*
- aliases: `treeView`
- properties: `click?: Action`, `tooltip?: string`, `checkable?: boolean`, `checkedKeys?: array<string> | binding`, `draggable?: boolean`, `data: array | binding`, `childrenKey?: string`, `indentation?: number`, `itemPadding?: EdgeInsets`, `itemTemplate?: Widget`, `expandable?: boolean`, `initiallyExpanded?: boolean`, `selectable?: boolean`, `showLines?: boolean`, `selectedColor?: Color`, `lineColor?: Color`, `width?: number`, `height?: number`, `onDrop?: Action`, `onNodeTap?: Action`, `onSelect?: Action`, `onExpand?: Action`, `onCollapse?: Action`

### `webView` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `url?: string`, `html?: string`, `allowNavigation?: boolean`, `enableJavaScript?: boolean`, `enableZoom?: boolean`, `width?: number`, `height?: number`, `onPageStarted?: Action`, `onPageFinished?: Action`, `onError?: Action`, `backgroundColor?: Color`

## Animation

### `animatedAlign` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `alignment: Alignment`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedContainer`
- properties: `click?: Action`, `tooltip?: string`, `foregroundDecoration?: BoxDecoration`, `transform?: array<number>`, `transformAlignment?: Alignment`, `clipBehavior?: string`, `constraints?: object`, `duration?: Dimension`, `curve?: AnimationCurve`, `width?: Dimension`, `height?: Dimension`, `padding?: EdgeInsets`, `margin?: EdgeInsets`, `alignment?: Alignment`, `decoration?: BoxDecoration`, `onEnd?: Action`, `child?: Widget`

### `animatedDefaultTextStyle` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `style: TextStyle`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedOpacity` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `opacity: number`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedPositioned` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `top?: Dimension`, `right?: Dimension`, `bottom?: Dimension`, `left?: Dimension`, `width?: Dimension`, `height?: Dimension`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `hero` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `tag: string`, `child: Widget`, `transitionOnUserGestures?: boolean`, `flightShuttleBuilder?: Widget`

### `lottieAnimation`
- properties: `click?: Action`, `tooltip?: string`, `fit?: string`, `onComplete?: Action`, `speed?: number`, `height?: Dimension`, `src: AssetRef`, `autoPlay?: boolean`, `loop?: boolean`, `backgroundColor?: Color`, `width?: Dimension`

### `opacity` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `opacity: number | binding`, `animated?: boolean`, `duration?: number`, `curve?: string`, `child: Widget`

### `rive` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `src: AssetRef`, `artboard?: string`, `animation?: string`, `stateMachine?: string`, `inputs?: object`, `fit?: string`, `alignment?: Alignment`, `width?: Dimension`, `height?: Dimension`

### `scrollAnimated` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `bindings: array<object>`, `child: Widget`

### `transform` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `rotate?: number`, `scale?: number | object`, `translate?: object`, `origin?: object`, `animated?: boolean`, `duration?: number`, `curve?: string`, `child: Widget`

## Dialog

### `alertDialog` *(since v1.0)*
- aliases: `alert`, `confirmDialog`
- properties: `click?: Action`, `tooltip?: string`, `insetPadding?: EdgeInsets`, `scrollable?: boolean`, `clipBehavior?: string`, `shadowColor?: Color`, `shape?: object`, `surfaceTintColor?: Color`, `title?: string`, `content?: string | Widget`, `dismissible?: boolean`, `onClose?: Action`, `actions?: array<object{ label: string, variant: string, primary: boolean, onTap: Action }>`, `alignment?: Alignment`, `backgroundColor?: Color`, `elevation?: Dimension`

### `bottomSheet`
- properties: `click?: Action`, `tooltip?: string`, `dragHandleColor?: Color`, `dragHandleSize?: object`, `onClosing?: Action`, `showDragHandle?: boolean`, `clipBehavior?: string`, `constraints?: object`, `shadowColor?: Color`, `child: Widget`, `isDismissible?: boolean`, `enableDrag?: boolean`, `backgroundColor?: Color`, `shape?: object`, `onClose?: Action`, `elevation?: Dimension`

### `customDialog`
- aliases: `modal`, `dialog`
- properties: `click?: Action`, `tooltip?: string`, `actions?: array<Widget>`, `insetPadding?: EdgeInsets`, `clipBehavior?: string`, `shadowColor?: Color`, `shape?: object`, `surfaceTintColor?: Color`, `child: Widget`, `dismissible?: boolean`, `onClose?: Action`, `alignment?: Alignment`, `backgroundColor?: Color`, `content?: string`, `elevation?: Dimension`, `title?: string`

### `popover` *(since v1.4)*
- aliases: `hoverCard`
- properties: `click?: Action`, `tooltip?: string`, `content: Widget`, `child: Widget`, `open?: boolean | binding`, `trigger?: string`, `placement?: string`, `openDelay?: number`, `closeDelay?: number`, `dismissOnOutside?: boolean`
- events: `onOpen`, `onClose`

### `simpleDialog`
- properties: `click?: Action`, `tooltip?: string`, `contentPadding?: EdgeInsets`, `titlePadding?: EdgeInsets`, `shape?: object`, `title?: string`, `options?: array<Option>`, `children?: array<Widget>`, `onSelect?: Action`, `onClose?: Action`, `backgroundColor?: Color`, `elevation?: Dimension`

### `snackBar`
- aliases: `toast`
- properties: `click?: Action`, `tooltip?: string`, `behavior?: string`, `closeIconColor?: Color`, `dismissDirection?: string`, `showCloseIcon?: boolean`, `margin?: EdgeInsets`, `padding?: EdgeInsets`, `shape?: object`, `content: string`, `duration?: number`, `action?: SnackBarAction`, `onClose?: Action`, `backgroundColor?: Color`, `elevation?: Dimension`, `textColor?: Color`, `width?: Dimension`

## Display

### `avatar`
- properties: `click?: Action`, `tooltip?: string`, `src?: AssetRef`, `label?: string`, `size?: number`, `color?: Color`, `backgroundImage?: BackgroundImage`, `foregroundColor?: Color`, `icon?: IconRef`, `radius?: number`

### `badge`
- properties: `click?: Action`, `tooltip?: string`, `isLabelVisible?: boolean`, `smallSize?: boolean`, `offset?: object`, `label?: string`, `color?: Color`, `child?: Widget`, `alignment?: Alignment`, `backgroundColor?: Color`, `textColor?: Color`

### `banner`
- properties: `click?: Action`, `tooltip?: string`, `message: string`, `severity?: string`, `actions?: array<BannerAction>`, `onClose?: Action`

### `card`
- properties: `click?: Action`, `tooltip?: string`, `semanticContainer?: boolean`, `clipBehavior?: string`, `shadowColor?: Color`, `surfaceTintColor?: Color`, `elevation?: string`, `margin?: EdgeInsets`, `shape?: string`, `color?: Color`, `child: Widget`

### `chip`
- aliases: `tag`
- properties: `click?: Action`, `tooltip?: string`, `deleteIcon?: IconRef`, `onDeleted?: Action`, `side?: object`, `padding?: EdgeInsets`, `shadowColor?: Color`, `shape?: object`, `label: string`, `avatar?: Widget`, `selected?: boolean`, `variant?: string`, `onDelete?: Action`, `onTap?: Action`, `backgroundColor?: Color`, `elevation?: Dimension`, `labelStyle?: TextStyle`, `onPressed?: Action`

### `decoration`
- properties: `click?: Action`, `tooltip?: string`, `position?: string`, `decoration?: BoxDecoration`, `color?: Color`, `borderRadius?: BorderRadius`, `border?: BoxBorder`, `gradient?: Gradient`, `image?: BackgroundImage`, `boxShadow?: array<BoxShadow>`, `shape?: string`, `backdropBlur?: number`, `child?: Widget`, `children?: array<Widget>`

### `divider`
- properties: `click?: Action`, `tooltip?: string`, `vertical?: boolean`, `height?: Dimension`, `thickness?: number`, `color?: Color`, `indent?: number`, `endIndent?: number`

### `icon`
- properties: `click?: Action`, `tooltip?: string`, `icon: IconRef`, `size?: string`, `sizeToken?: string`, `color?: Color`, `shader?: Gradient`

### `image`
- properties: `click?: Action`, `tooltip?: string`, `errorWidget?: string`, `fallbackBehavior?: string`, `fallbackUrl?: AssetRef`, `src: AssetRef`, `width?: number`, `height?: number`, `fit?: string`, `alignment?: Alignment`, `fallback?: Widget`, `loading?: Widget`, `placeholder?: string`

### `imageFilter` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `filter: string`, `intensity?: number`, `child: Widget`

### `kenBurnsImage` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `src: AssetRef`, `duration?: Dimension`, `intensity?: number`, `startAlignment?: Alignment`, `endAlignment?: Alignment`, `loop?: boolean`, `curve?: AnimationCurve`, `width?: Dimension`, `height?: Dimension`, `fit?: string`

### `placeholder`
- aliases: `skeleton`
- properties: `click?: Action`, `tooltip?: string`, `fallbackWidth?: number`, `fallbackHeight?: number`, `color?: Color`, `strokeWidth?: number`, `child?: Widget`

### `progressBar`
- aliases: `linearProgressIndicator`, `circularProgressIndicator`, `loadingIndicator`, `loading-indicator`, `progress-bar`, `progress`
- properties: `click?: Action`, `tooltip?: string`, `size?: Dimension`, `strokeWidth?: Dimension`, `value?: number | binding`, `indicatorType?: string`, `color?: Color`, `backgroundColor?: Color`

### `richText`
- properties: `click?: Action`, `tooltip?: string`, `textScaleFactor?: number`, `spans: array<Span>`, `style?: TextStyle`, `dropCap?: DropCap`, `textAlign?: string`, `textDirection?: string`, `maxLines?: number`, `overflow?: string`, `softWrap?: boolean`

### `text`
- aliases: `label`
- properties: `click?: Action`, `tooltip?: string`, `textTransform?: string`, `ariaLabel?: string`, `semanticsLabel?: string`, `softWrap?: boolean`, `textScaleFactor?: number`, `text: string`, `variant?: string`, `style?: TextStyle`, `dropCap?: DropCap`, `maxLines?: number`, `overflow?: string`, `textAlign?: string`, `textDirection?: string`, `value?: string | number | boolean | object | array`

### `tooltip`
- properties: `click?: Action`, `tooltip?: string`, `preferBelow?: boolean`, `richMessage?: array<object>`, `showDuration?: number`, `textStyle?: TextStyle`, `triggerMode?: string`, `verticalOffset?: Dimension`, `enableFeedback?: boolean`, `excludeFromSemantics?: boolean`, `height?: Dimension`, `margin?: EdgeInsets`, `padding?: EdgeInsets`, `message: string`, `child: Widget`, `decoration?: BoxDecoration`, `textAlign?: string`, `waitDuration?: number`

### `verticalDivider`
- properties: `click?: Action`, `tooltip?: string`, `width?: number`, `thickness?: number`, `color?: Color`, `indent?: number`, `endIndent?: number`

## Input

### `button`
- properties: `click?: Action`, `tooltip?: string`, `fullWidth?: boolean`, `iconPosition?: string`, `ariaLabel?: string`, `borderWidth?: Dimension`, `size?: string`, `label: string`, `variant?: string`, `elevation?: string`, `icon?: IconRef`, `enabled?: boolean`, `onTap?: Action`, `onDoubleTap?: Action`, `onLongPress?: Action`, `backgroundColor?: Color`, `borderColor?: Color`, `foregroundColor?: Color`, `loading?: boolean`, `onSubmit?: Action`, `submit?: Action`

### `checkbox`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `tristate?: boolean`, `change?: Action`

### `checkboxGroup`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `options: array<Option>`, `orientation?: string`, `change?: Action`, `direction?: string`

### `colorPicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `showAlpha?: boolean`, `showLabel?: boolean`, `pickerType?: string`, `enableHistory?: boolean`

### `combobox` *(since v1.4)*
- aliases: `autocomplete`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `options?: array<Option>`, `allowCustom?: boolean`, `onSearch?: Action`, `minChars?: number`, `debounceMs?: number`, `placeholder?: string`

### `dateField`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `errorText?: string`, `label?: string`, `format?: string`, `firstDate?: string`, `lastDate?: string`, `mode?: string`, `locale?: string`

### `datePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `initialDate?: string`, `label?: string`, `firstDate?: string`, `lastDate?: string`, `change?: Action`, `dateFormat?: string`, `icon?: IconRef`, `variant?: string`

### `dateRangePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `errorText?: string`, `startDate?: string`, `endDate?: string`, `label?: string`, `firstDate?: string`, `lastDate?: string`, `format?: string`, `locale?: string`, `change?: Action`

### `dateTimePicker` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `min?: string`, `max?: string`, `dateFormat?: string`, `timeFormat?: string`, `minuteInterval?: number`, `timeZone?: string`

### `fileInput` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `dragDrop?: boolean`, `maxFiles?: number`, `preview?: boolean`, `crop?: boolean`, `aspectRatio?: number`, `accept?: array<string>`, `multiple?: boolean`, `maxBytes?: number`, `label?: string`, `onError?: Action`

### `form`
- properties: `click?: Action`, `tooltip?: string`, `children: array<Widget>`, `showErrorsOn?: string`, `onSubmit?: Action`

### `iconButton`
- properties: `click?: Action`, `tooltip?: string`, `fontFamily?: string`, `disabledColor?: Color`, `enableFeedback?: boolean`, `highlightColor?: Color`, `iconSize?: Dimension`, `padding?: EdgeInsets`, `splashColor?: Color`, `splashRadius?: Dimension`, `icon: IconRef`, `size?: number`, `color?: Color`, `enabled?: boolean`, `onTap?: Action`, `alignment?: Alignment`

### `multiSelect` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `options: array<Option>`, `placeholder?: string`, `maxSelections?: number`, `showChips?: boolean`, `selectAll?: boolean`, `searchable?: boolean`

### `numberField`
- aliases: `numberInput`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `error?: string`, `showStepper?: boolean`, `label?: string`, `min?: number`, `max?: number`, `step?: number`, `decimalPlaces?: number`, `prefix?: string`, `suffix?: string`, `thousandSeparator?: boolean`, `change?: Action`, `format?: string`, `helperText?: string`, `hint?: string`

### `numberStepper`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `size?: string`, `label?: string`, `min?: number`, `max?: number`, `step?: number`, `change?: Action`, `color?: Color`

### `otpInput` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `length?: number`, `inputType?: string`, `onComplete?: Action`, `masked?: boolean`, `autofill?: boolean`

### `radio`
- properties: `click?: Action`, `tooltip?: string`, `activeColor?: Color`, `focusColor?: Color`, `hoverColor?: Color`, `splashRadius?: Dimension`, `value: any`, `groupValue: any | binding`, `label?: string`, `onChange?: Action`, `binding?: string`, `change?: Action`, `fillColor?: Color`

### `radioGroup`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `options: array<Option>`, `orientation?: string`, `direction?: string`

### `rangeSlider`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `labels?: array<string>`, `onChangeEnd?: Action`, `onChangeStart?: Action`, `activeColor?: Color`, `inactiveColor?: Color`, `min?: number`, `max?: number`, `divisions?: number`, `change?: Action`

### `rating`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `allowHalf?: boolean`, `emptyColor?: Color`, `readOnly?: boolean`, `size?: Dimension`, `max?: number`, `icon?: IconRef`, `color?: Color`, `change?: Action`

### `segmentedControl`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `options: array<Option>`, `variant?: string`

### `select`
- aliases: `dropdown`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `disabledHint?: string`, `isExpanded?: boolean`, `itemHeight?: Dimension`, `iconSize?: Dimension`, `style?: TextStyle`, `options: array<Option>`, `placeholder?: string`, `change?: Action`, `elevation?: Dimension`, `hint?: string`, `label?: string`

### `slider`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: number`, `enabled?: boolean`, `onChange?: Action`, `onChangeEnd?: Action`, `onChangeStart?: Action`, `thumbColor?: Color`, `activeColor?: Color`, `inactiveColor?: Color`, `label?: string`, `min?: number`, `max?: number`, `divisions?: number`, `change?: Action`

### `stepper`
- aliases: `steps`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `margin?: EdgeInsets`, `physics?: string`, `steps: array<Step>`, `currentStep?: number | binding`, `stepperType?: string`, `onStepTapped?: Action`, `onStepContinue?: Action`, `onStepCancel?: Action`

### `textInput` *(since v1.0)*
- aliases: `textField`, `textfield`, `textFormField`, `text-form-field`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | binding`, `enabled?: boolean`, `onChange?: Action`, `debounce?: number`, `textInputAction?: string`, `errorText?: string`, `style?: TextStyle`, `label?: string`, `placeholder?: string`, `helperText?: string`, `prefixIcon?: string`, `suffixIcon?: string`, `obscureText?: boolean`, `readOnly?: boolean`, `maxLines?: integer`, `maxLength?: integer`, `inputType?: string`, `showToggle?: boolean`, `defaultCountry?: string`, `validation?: ValidationConfig`, `error?: string | boolean | binding`
- events: `onChange`, `onSubmit`, `onFocus`, `onBlur`

### `timeField`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `errorText?: string`, `label?: string`, `format?: string`, `use24HourFormat?: boolean`, `mode?: string`

### `timePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `initialTime?: string`, `label?: string`, `use24HourFormat?: boolean`, `change?: Action`, `icon?: IconRef`, `timeFormat?: string`, `variant?: string`

### `toggle`
- aliases: `switch`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `change?: Action`, `label?: string`

### `voiceInput` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `language?: string`, `continuous?: boolean`, `interimResults?: boolean`, `maxDuration?: number`, `showWaveform?: boolean`
- events: `onStart`, `onResult`, `onEnd`, `onError`

## Interaction

### `contextMenu` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `items: array<object>`, `enabled?: boolean`
- events: `onSelect`

### `dragTarget`
- properties: `click?: Action`, `tooltip?: string`, `canDrop?: binding`, `builder: Widget`, `children?: array<Widget>`, `onDrop?: Action`, `onDragEnter?: Action`, `onDragLeave?: Action`

### `draggable`
- properties: `click?: Action`, `tooltip?: string`, `affinity?: string`, `axis?: string`, `dragAnchorStrategy?: string`, `onDragCompleted?: Action`, `onDragEnd?: Action`, `onDragStarted?: Action`, `onDraggableCanceled?: Action`, `data: any | binding`, `feedback?: Widget`, `childWhenDragging?: Widget`, `child: Widget`

### `gestureDetector`
- properties: `click?: Action`, `tooltip?: string`, `onScaleUpdate?: Action`, `child: Widget`, `onTap?: Action`, `onDoubleTap?: Action`, `onLongPress?: Action`, `onPanStart?: Action`, `onPanUpdate?: Action`, `onPanEnd?: Action`

### `inkWell`
- properties: `click?: Action`, `tooltip?: string`, `onHighlightChanged?: Action`, `onHover?: Action`, `onTapCancel?: Action`, `onTapDown?: Action`, `onTapUp?: Action`, `autofocus?: boolean`, `canRequestFocus?: boolean`, `customBorder?: object`, `enableFeedback?: boolean`, `excludeFromSemantics?: boolean`, `focusColor?: Color`, `highlightColor?: Color`, `hoverColor?: Color`, `overlayColor?: Color`, `splashColor?: Color`, `splashRadius?: Dimension`, `child: Widget`, `borderRadius?: number`, `onTap?: Action`, `onLongPress?: Action`, `onDoubleTap?: Action`

## Layout

### `accordion` *(since v1.4)*
- aliases: `collapsible`
- properties: `click?: Action`, `tooltip?: string`, `panels: array<object>`, `allowMultiple?: boolean`, `expandedIds?: array<string>`, `bordered?: boolean`, `icon?: IconRef`
- events: `onChange`

### `align`
- properties: `click?: Action`, `tooltip?: string`, `heightFactor?: number`, `widthFactor?: number`, `alignment?: Alignment`, `child: Widget`

### `aspectRatio`
- properties: `click?: Action`, `tooltip?: string`, `aspectRatio?: number`, `child: Widget`

### `box` *(since v1.0)*
- aliases: `container`, `constrained`, `decoratedBox`, `constrainedBox`
- properties: `click?: Action`, `tooltip?: string`, `constraints?: object`, `width?: Dimension`, `height?: Dimension`, `minWidth?: number`, `maxWidth?: number`, `minHeight?: number`, `maxHeight?: number`, `padding?: BoxSpacing`, `margin?: BoxSpacing`, `alignment?: Alignment`, `color?: Color`, `decoration?: BoxDecoration`
- children: single (key: `child`)

### `center`
- properties: `click?: Action`, `tooltip?: string`, `heightFactor?: number`, `widthFactor?: number`, `child: Widget`

### `conditional`
- properties: `click?: Action`, `tooltip?: string`, `condition?: boolean | binding`, `then?: Widget`, `else?: Widget`, `switch?: binding`, `cases?: array`, `default?: Widget`

### `expanded`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`, `child: Widget`

### `flexible`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`, `fit?: string`, `child: Widget`

### `fractionallySized`
- properties: `click?: Action`, `tooltip?: string`, `widthFactor?: number`, `heightFactor?: number`, `child: Widget`, `alignment?: Alignment`

### `indexedStack`
- properties: `click?: Action`, `tooltip?: string`, `sizing?: string`, `clipBehavior?: string`, `index?: number | binding`, `alignment?: Alignment`, `children: array<Widget>`, `textDirection?: string`

### `intrinsicHeight`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `intrinsicWidth`
- properties: `click?: Action`, `tooltip?: string`, `stepHeight?: Dimension`, `stepWidth?: Dimension`, `child: Widget`

### `linear`
- aliases: `row`, `column`
- properties: `click?: Action`, `tooltip?: string`, `wrap?: boolean`, `padding?: EdgeInsets`, `mainAxisSize?: string`, `direction: string`, `alignment?: string`, `distribution?: string`, `spacing?: number`, `children: array<Widget>`

### `margin`
- properties: `click?: Action`, `tooltip?: string`, `margin: EdgeInsets`, `child: Widget`

### `padding`
- properties: `click?: Action`, `tooltip?: string`, `padding: EdgeInsets`, `child: Widget`

### `positioned`
- properties: `click?: Action`, `tooltip?: string`, `height?: Dimension`, `child: Widget`, `bottom?: Dimension`, `left?: Dimension`, `right?: Dimension`, `top?: Dimension`, `width?: Dimension`

### `resizable` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `width?: number | binding`, `height?: number | binding`, `minWidth?: number`, `maxWidth?: number`, `minHeight?: number`, `maxHeight?: number`, `handles?: array<string>`, `keepAspectRatio?: boolean`
- events: `onResize`, `onResizeEnd`

### `safeArea`
- properties: `click?: Action`, `tooltip?: string`, `maintainBottomViewPadding?: boolean`, `minimum?: EdgeInsets`, `child: Widget`, `bottom?: boolean`, `left?: boolean`, `right?: boolean`, `top?: boolean`

### `sizedBox`
- properties: `click?: Action`, `tooltip?: string`, `width?: number`, `height?: number`, `child?: Widget`

### `spacer`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`

### `splitter` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `children: array<Widget>`, `orientation?: string`, `sizes?: array<number> | binding`, `minSizes?: array<number>`, `gutterSize?: number`, `collapsible?: array<boolean>`
- events: `onDragEnd`

### `stack`
- properties: `click?: Action`, `tooltip?: string`, `clipBehavior?: string`, `alignment?: Alignment`, `fit?: string`, `children: array<Widget>`, `textDirection?: string`

### `visibility`
- properties: `click?: Action`, `tooltip?: string`, `maintainAnimation?: boolean`, `maintainInteractivity?: boolean`, `visible?: boolean | binding`, `maintainSize?: boolean`, `maintainState?: boolean`, `replacement?: Widget`, `child?: Widget`, `children?: array<Widget>`

### `wrap`
- properties: `click?: Action`, `tooltip?: string`, `runAlignment?: string`, `verticalDirection?: string`, `clipBehavior?: string`, `direction?: string`, `spacing?: number`, `runSpacing?: number`, `alignment?: string`, `children: array<Widget>`, `crossAxisAlignment?: string`, `textDirection?: string`

## List

### `carousel` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `items?: array | binding`, `itemTemplate?: Widget`, `children?: array<Widget>`, `scrollDirection?: string`, `viewportFraction?: number`, `loop?: boolean`, `autoPlay?: boolean | number | binding`, `initialIndex?: number`, `transition?: string`, `indicatorPosition?: string`, `onPageChanged?: Action`

### `grid`
- aliases: `gridview`
- properties: `click?: Action`, `tooltip?: string`, `mainAxisExtent?: Dimension`, `maxCrossAxisExtent?: Dimension`, `padding?: EdgeInsets`, `physics?: string`, `shrinkWrap?: boolean`, `spacing?: Dimension`, `items?: array | binding`, `itemTemplate?: Widget`, `children?: array<Widget>`, `columns: number | object`, `rowGap?: number`, `columnGap?: number`, `itemAspectRatio?: number`, `reverse?: boolean`, `scrollDirection?: string`, `template?: object`

### `list`
- aliases: `listView`, `listview`
- properties: `click?: Action`, `tooltip?: string`, `itemBuilder?: Widget`, `itemCount?: number`, `scrollCacheExtent?: Dimension`, `padding?: EdgeInsets`, `physics?: string`, `shrinkWrap?: boolean`, `virtual?: boolean`, `itemHeight?: number`, `overscan?: number`, `items?: array | binding`, `itemTemplate?: Widget`, `children?: array<Widget>`, `spacing?: number`, `orientation?: string`, `emptyMessage?: string`, `itemExtent?: number`, `reverse?: boolean`, `scrollDirection?: string`, `template?: object`

### `listItem`
- aliases: `listTile`, `list-tile`
- properties: `click?: Action`, `tooltip?: string`, `contentPadding?: EdgeInsets`, `dense?: boolean`, `isThreeLine?: boolean`, `selectedTileColor?: Color`, `tileColor?: Color`, `focusColor?: Color`, `hoverColor?: Color`, `iconColor?: Color`, `shape?: object`, `title?: string | Widget`, `subtitle?: string | Widget`, `leading?: Widget`, `trailing?: Widget`, `onTap?: Action`, `selected?: boolean`, `enabled?: boolean`, `onLongPress?: Action`, `textColor?: Color`

### `staggeredGrid` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `items?: array | binding`, `itemTemplate?: Widget`, `children?: array<Widget>`, `columns: number | object`, `mainAxisSpacing?: number`, `crossAxisSpacing?: number`, `padding?: EdgeInsets`, `scrollDirection?: string`

## Navigation

### `bottomNavigation`
- aliases: `bottomNav`, `bottomnavigationbar`
- properties: `click?: Action`, `tooltip?: string`, `fixedColor?: Color`, `selectedFontSize?: Dimension`, `selectedIconTheme?: object`, `selectedItemColor?: Color`, `selectedLabelStyle?: TextStyle`, `showSelectedLabels?: boolean`, `showUnselectedLabels?: boolean`, `unselectedFontSize?: Dimension`, `unselectedIconTheme?: object`, `unselectedItemColor?: Color`, `unselectedLabelStyle?: TextStyle`, `enableFeedback?: boolean`, `iconSize?: Dimension`, `selectedIndex?: number | binding`, `items: array<NavItem>`, `onChange?: Action`, `backgroundColor?: Color`, `change?: Action`, `elevation?: Dimension`, `onTap?: Action`, `currentIndex?: number`

### `breadcrumb` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `items: array<object>`, `separator?: string`, `maxItems?: number`
- events: `onClick`

### `drawer`
- properties: `click?: Action`, `tooltip?: string`, `semanticLabel?: string`, `shadowColor?: Color`, `shape?: object`, `surfaceTintColor?: Color`, `items?: array<DrawerItem>`, `children?: array<Widget>`, `header?: Widget`, `onSelect?: Action`, `onClose?: Action`, `backgroundColor?: Color`, `elevation?: Dimension`, `width?: Dimension`

### `floatingActionButton`
- properties: `click?: Action`, `tooltip?: string`, `disabledElevation?: Dimension`, `focusElevation?: Dimension`, `heroTag?: string`, `highlightElevation?: Dimension`, `hoverElevation?: Dimension`, `isExtended?: boolean`, `materialTapTargetSize?: string`, `mini?: boolean`, `autofocus?: boolean`, `clipBehavior?: string`, `focusColor?: Color`, `hoverColor?: Color`, `shape?: object`, `splashColor?: Color`, `icon?: IconRef`, `label?: string`, `onTap?: Action`, `backgroundColor?: Color`, `elevation?: Dimension`, `foregroundColor?: Color`, `onLongPress?: Action`

### `headerBar`
- aliases: `appbar`
- properties: `click?: Action`, `tooltip?: string`, `automaticallyImplyLeading?: boolean`, `bottomHeight?: Dimension`, `bottomOpacity?: number`, `flexibleSpace?: Widget`, `toolbarHeight?: Dimension`, `toolbarOpacity?: number`, `shadowColor?: Color`, `shape?: object`, `title?: string | Widget`, `leading?: Widget`, `actions?: array<Widget>`, `exitButton?: ExitButtonConfig | boolean`, `backgroundColor?: Color`, `elevation?: number`, `centerTitle?: boolean`, `bottom?: Dimension`, `foregroundColor?: Color`

### `link` *(since v1.4)*
- aliases: `navLink`
- properties: `click?: Action`, `tooltip?: string`, `label: string`, `route?: string`, `params?: object`, `url?: string`, `target?: string`, `activeWhen?: string | binding`, `underline?: string`, `icon?: IconRef`, `child?: Widget`
- events: `onClick`

### `menu` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `items: array<object>`, `selectedKey?: string | binding`, `openKeys?: array<string>`, `mode?: string`, `collapsed?: boolean | binding`
- events: `onSelect`

### `navigationRail`
- properties: `click?: Action`, `tooltip?: string`, `extended?: boolean`, `groupAlignment?: number`, `labelType?: string`, `minExtendedWidth?: Dimension`, `minWidth?: Dimension`, `selectedIconTheme?: object`, `selectedLabelTextStyle?: TextStyle`, `unselectedIconTheme?: object`, `unselectedLabelTextStyle?: TextStyle`, `selectedIndex?: number | binding`, `items: array<NavItem>`, `onChange?: Action`, `backgroundColor?: Color`, `change?: Action`, `elevation?: Dimension`, `leading?: Widget`, `onSelect?: Action`, `trailing?: Widget`

### `pagination` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `current?: number`, `binding?: string`, `total: number`, `pageSize?: number`, `siblingCount?: number`, `showSizeChanger?: boolean`, `pageSizeOptions?: array<number>`, `showTotal?: boolean`
- events: `onChange`

### `popupMenuButton`
- aliases: `dropdownMenu`
- properties: `click?: Action`, `tooltip?: string`, `onCanceled?: Action`, `onOpened?: Action`, `iconSize?: Dimension`, `offset?: object`, `padding?: EdgeInsets`, `shadowColor?: Color`, `shape?: object`, `splashRadius?: Dimension`, `surfaceTintColor?: Color`, `icon?: IconRef`, `items: array<MenuItem>`, `onSelect?: Action`, `change?: Action`, `color?: Color`, `elevation?: Dimension`, `onChange?: Action`

### `tabBar`
- properties: `click?: Action`, `tooltip?: string`, `indicator?: object`, `indicatorPadding?: EdgeInsets`, `indicatorSize?: string`, `indicatorWeight?: Dimension`, `isScrollable?: boolean`, `labelPadding?: EdgeInsets`, `mouseCursor?: string`, `unselectedLabelColor?: Color`, `unselectedLabelStyle?: TextStyle`, `enableFeedback?: boolean`, `labelColor?: Color`, `overlayColor?: Color`, `padding?: EdgeInsets`, `physics?: string`, `selectedIndex?: number | binding`, `tabs: array<Tab>`, `onChange?: Action`, `change?: Action`, `indicatorColor?: Color`, `labelStyle?: TextStyle`, `onTap?: Action`

### `tabBarView`
- properties: `click?: Action`, `tooltip?: string`, `dragStartBehavior?: string`, `physics?: string`, `selectedIndex?: number | binding`, `children: array<Widget>`

## Scroll

### `pageView`
- properties: `click?: Action`, `tooltip?: string`, `padEnds?: boolean`, `pageSnapping?: boolean`, `clipBehavior?: string`, `direction?: string`, `children: array<Widget>`, `initialPage?: number`, `loop?: boolean`, `scrollPhysics?: string`, `allowImplicitScrolling?: boolean`, `onPageChanged?: Action`, `onChange?: Action`, `reverse?: boolean`, `scrollDirection?: string`

### `scrollBar`
- properties: `click?: Action`, `tooltip?: string`, `thumbVisibility?: boolean`, `trackVisibility?: boolean`, `thickness?: number`, `radius?: number`, `child?: Widget`, `children?: array<Widget>`

### `scrollView`
- aliases: `scrollArea`
- properties: `click?: Action`, `tooltip?: string`, `primary?: boolean`, `direction?: string`, `padding?: EdgeInsets`, `scrollPhysics?: string`, `child?: Widget`, `children?: array<Widget>`, `slivers?: array<Sliver>`, `reverse?: boolean`, `scrollDirection?: string`

### `singleChildScrollView`
- properties: `click?: Action`, `tooltip?: string`, `clipBehavior?: string`, `physics?: string`, `primary?: boolean`, `direction?: string`, `padding?: EdgeInsets`, `child?: Widget`, `children?: array<Widget>`, `reverse?: boolean`, `scrollDirection?: string`

## Utility

### `accessibleWrapper`
- properties: `click?: Action`, `tooltip?: string`, `announceNavigation?: boolean`, `announceOnChange?: boolean`, `autoFocus?: boolean`, `focusGroup?: string`, `focusOrder?: number`, `liveRegion?: string`, `navigationMessage?: string`, `watchPath?: string`, `child: Widget`, `accessibility?: object`

### `baseline`
- properties: `click?: Action`, `tooltip?: string`, `baseline: number`, `baselineType?: string`, `child: Widget`

### `clipOval`
- properties: `click?: Action`, `tooltip?: string`, `clipBehavior?: string`, `child: Widget`

### `clipRRect`
- properties: `click?: Action`, `tooltip?: string`, `clipBehavior?: string`, `borderRadius?: BorderRadius`, `child: Widget`

### `dashboard` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `content?: Widget`, `refreshInterval?: number`, `onTap?: Action`

### `errorBoundary`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `fallback?: Widget`, `onError?: Action`

### `errorRecovery`
- properties: `click?: Action`, `tooltip?: string`, `child?: Widget`, `children?: array<Widget>`, `fallback?: Widget`, `handlers?: object`, `onError?: Action`, `showDetails?: boolean`

### `fittedBox`
- properties: `click?: Action`, `tooltip?: string`, `clipBehavior?: string`, `fit?: string`, `alignment?: Alignment`, `child: Widget`

### `flow`
- properties: `click?: Action`, `tooltip?: string`, `children: array<Widget>`, `direction?: string`, `spacing?: number`, `alignment?: string`

### `layoutBuilder`
- properties: `click?: Action`, `tooltip?: string`, `breakpoints?: object`, `layouts?: object`, `default?: Widget`

### `lazy` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `delay?: number`, `placeholder?: Widget`, `content?: Widget | object`, `child?: Widget`, `children?: array<Widget>`, `trigger?: string`, `onLoad?: Action`, `onError?: Action`

### `limitedBox`
- properties: `click?: Action`, `tooltip?: string`, `maxWidth?: number`, `maxHeight?: number`, `child: Widget`

### `mediaQuery`
- properties: `click?: Action`, `tooltip?: string`, `condition?: object | binding`, `then?: Widget`, `else?: Widget`, `breakpoints?: object`, `defaultChild?: Widget`

### `offlineFallback`
- properties: `click?: Action`, `tooltip?: string`, `online?: Widget`, `offline?: Widget`, `message?: string`, `icon?: IconRef`, `showRetry?: boolean`, `onRetry?: Action`, `isOnline?: boolean | binding`

### `permissionPrompt`
- properties: `click?: Action`, `tooltip?: string`, `permissions?: array<string>`, `permissionType?: string`, `style?: string`, `title?: string`, `description?: string`, `icon?: IconRef`, `allowPartial?: boolean`, `onAllow?: Action`, `onDeny?: Action`

### `use`
- properties: `click?: Action`, `tooltip?: string`, `template: string`, `params?: object`, `slots?: object`

### `view` *(since v1.4)*
- properties: `click?: Action`, `tooltip?: string`, `source: DefinitionSource`, `props?: object`, `fallback?: Widget`, `loading?: Widget`, `onError?: Action`, `theme?: string`

