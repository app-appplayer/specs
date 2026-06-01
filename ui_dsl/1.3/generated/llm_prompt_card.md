<!-- GENERATED: do not edit. Source: 1.3/widgets/*.yaml. -->
# MCP UI DSL — LLM Prompt Card

Authoritative catalog of every widget type recognised by the MCP UI DSL runtime. Use as context when generating or editing DSL. Widget types and property names listed here are the only sanctioned ones; anything else is either a legacy alias (§17.3) or invalid.

Format per widget:
- `type` — canonical widget type name (camelCase).
- `aliases` — legacy names accepted by the runtime.
- `properties` — supported top-level properties (`?` = optional).
- `children` — how children are passed, if any.
- `events` — widget-level event handlers (Action payload).

## Advanced

### `calendar` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `selectedDate?: string | binding`, `events?: binding`, `firstDate?: string`, `lastDate?: string`, `view?: string`, `onChange?: Action`

### `canvas` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`

### `chart` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `chartType: string`, `data: object | array`, `data.datasets[].label?: string`, `data.datasets[].data: number[]`, `data.datasets[].borderColor?: string`, `data.datasets[].backgroundColor?: string`, `options?: object`, `options.responsive?: boolean`, `options.animation.duration?: number`, `options.legend.position?: string`, `width?: number`, `height?: number`

### `codeEditor` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `code: string | binding`, `language?: string`, `theme?: string`, `readOnly?: boolean`, `showLineNumbers?: boolean`, `fontSize?: number`, `lineHeight?: number`, `tabSize?: number`, `width?: number`, `height?: number`, `backgroundColor?: string`, `textColor?: string`, `onChange?: Action`

### `dataTable` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `columns: `Column[]``, `columns[].key: string`, `columns[].label: string`, `columns[].width?: number`, `columns[].sortable?: boolean`, `columns[].align?: string`, `rows: binding`, `selectable?: boolean`, `sortColumn?: binding`, `sortAscending?: binding`, `onSort?: Action`, `onRowTap?: Action`

### `fileExplorer` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `items?: binding`, `rootPath?: string`, `files?: string[]`, `directories?: string[]`, `showIcons?: boolean`, `showHidden?: boolean`, `expandAll?: boolean`, `width?: number`, `height?: number`, `selectedColor?: string`, `onSelect?: Action`, `onOpen?: Action`

### `gauge` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `value: number`, `min?: number`, `max?: number`, `segments?: `Segment[]``, `size?: number`, `strokeWidth?: number`, `backgroundColor?: string`, `valueColor?: string`, `showLabel?: boolean`, `labelFormat?: string`, `startAngle?: number`, `sweepAngle?: number`

### `graph` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `data?: `Point[]` `, `chartType?: string`, `width?: number`, `height?: number`, `showGrid?: boolean`, `showLabels?: boolean`, `lineColor?: Color`, `fillColor?: Color`, `gridColor?: Color`, `strokeWidth?: number`

### `heatmap` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `data: array | binding`, `columnLabels?: string[]`, `rowLabels?: string[]`, `cellSize?: number`, `colorRange?: `{ low, high }``, `showValues?: boolean`, `onCellTap?: Action`

### `lazy` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `placeholder?: Widget`, `content?: Widget | object`, `children?: Widget[]`, `trigger?: string`, `onLoad?: Action`, `onError?: Action`

### `lightbox` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `images: array<AssetRef>`, `initialIndex?: number`, `allowZoom?: boolean`, `maxZoom?: number`, `allowSwipe?: boolean`, `backgroundColor?: Color`, `onIndexChanged?: Action`, `onClose?: Action`

### `map` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `center?: `{ latitude, longitude }``, `latitude?: number`, `longitude?: number`, `zoom?: number`, `mapType?: string`, `markers?: `Marker[]``, `markers[].id: string`, `markers[].latitude: number`, `markers[].longitude: number`, `markers[].label?: string`, `markers[].icon?: string`, `markers[].color?: string`, `overlays?: `Overlay[]``, `overlays[].type: string`, `overlays[].points?: `{ latitude, longitude }[]``, `overlays[].fillColor?: string`, `overlays[].strokeColor?: string`, `overlays[].strokeWidth?: number`, `onMarkerTap?: Action`, `onMapTap?: Action`

### `markdown` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `text?: string `, `selectable?: boolean`, `width?: number`, `height?: number`, `fontSize?: number`, `textColor?: string`, `linkColor?: string`, `codeBackgroundColor?: string`, `onLinkTap?: Action`

### `mediaPlayer` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `source: AssetRef`, `mediaType?: string`, `autoPlay?: boolean`, `loop?: boolean`, `muted?: boolean`, `volume?: number`, `controls?: boolean`, `poster?: AssetRef`, `waveform?: boolean`, `width?: number`, `height?: number`, `onPlay?: Action`, `onPause?: Action`, `onEnded?: Action`, `onTimeUpdate?: Action`, `onError?: Action`

### `networkGraph` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`

### `signature` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `binding?: binding`, `penColor?: string`, `penWidth?: number`, `width?: number`, `height?: number`, `backgroundColor?: string`, `borderColor?: string`, `showClearButton?: boolean`, `showGuide?: boolean`, `onSignatureEnd?: Action`, `onClear?: Action`

### `table` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `rows: `{ cells: Widget[] }[]``, `border?: `{ color, width }``, `defaultColumnWidth?: string | number`, `defaultVerticalAlignment?: string`, `columnWidths?: object`

### `terminal` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `lines?: binding`, `prompt?: string`, `showInput?: boolean`, `maxLines?: number`, `width?: number`, `height?: number`, `fontSize?: number`, `backgroundColor?: string`, `textColor?: string`, `promptColor?: string`, `onCommand?: Action`

### `timeline` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `items: `TimelineItem[]``, `items[].title: string`, `items[].subtitle?: string`, `items[].icon?: string`, `items[].time?: string`, `items[].color?: string`, `orientation?: string`

### `tree` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `data: array | binding`, `childrenKey?: string`, `indentation?: number`, `itemPadding?: EdgeInsets`, `itemTemplate?: Widget`, `expandable?: boolean`, `initiallyExpanded?: boolean`, `selectable?: boolean`, `showLines?: boolean`, `selectedColor?: Color`, `lineColor?: Color`, `width?: number`, `height?: number`, `onNodeTap?: Action`, `onSelect?: Action`, `onExpand?: Action`, `onCollapse?: Action`

### `webView` *(since v1.0)*
- properties: `click?: Action`, `tooltip?: string`, `url?: string`, `html?: string`, `allowNavigation?: boolean`, `enableJavaScript?: boolean`, `enableZoom?: boolean`, `width?: number`, `height?: number`, `onPageStarted?: Action`, `onPageFinished?: Action`, `onError?: Action`

## Animation

### `animatedAlign` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `alignment: Alignment`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedContainer`
- properties: `click?: Action`, `tooltip?: string`, `duration?: Dimension`, `curve?: AnimationCurve`, `width?: Dimension`, `height?: Dimension`, `padding?: EdgeInsets`, `margin?: EdgeInsets`, `alignment?: Alignment`, `decoration?: BoxDecoration`, `onEnd?: Action`, `child?: Widget`

### `animatedDefaultTextStyle` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `style: TextStyle`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedOpacity` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `opacity: number`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `animatedPositioned` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `top?: Dimension`, `right?: Dimension`, `bottom?: Dimension`, `left?: Dimension`, `width?: Dimension`, `height?: Dimension`, `duration?: Dimension`, `curve?: AnimationCurve`, `onEnd?: Action`, `child: Widget`

### `hero` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `tag: string`, `child: Widget`, `transitionOnUserGestures?: boolean`, `flightShuttleBuilder?: Widget`

### `lottieAnimation`
- properties: `click?: Action`, `tooltip?: string`, `src: string`, `autoPlay?: boolean`, `loop?: boolean`

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
- aliases: `alert`
- properties: `click?: Action`, `tooltip?: string`, `title?: string`, `content?: string | Widget`, `dismissible?: boolean`, `onClose?: Action`, `actions?: array<object{ label: string, variant: string, primary: boolean, onTap: Action }>`

### `bottomSheet`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `isDismissible?: boolean`, `enableDrag?: boolean`, `backgroundColor?: string`, `shape?: object`, `onClose?: Action`

### `customDialog`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `dismissible?: boolean`, `onClose?: Action`

### `simpleDialog`
- properties: `click?: Action`, `tooltip?: string`, `title?: string`, `options?: Option[]`, `children?: Widget[]`, `onSelect?: Action`, `onClose?: Action`

### `snackBar`
- properties: `click?: Action`, `tooltip?: string`, `content: string`, `duration?: number`, `action?: SnackBarAction`, `onClose?: Action`

## Display

### `avatar`
- properties: `click?: Action`, `tooltip?: string`, `src?: string`, `label?: string`, `size?: number`, `color?: string`

### `badge`
- properties: `click?: Action`, `tooltip?: string`, `label?: string`, `color?: string`, `child?: Widget`

### `banner`
- properties: `click?: Action`, `tooltip?: string`, `message: string`, `severity?: string`, `actions?: BannerAction[]`, `onClose?: Action`

### `card`
- properties: `click?: Action`, `tooltip?: string`, `elevation?: string`, `margin?: EdgeInsets`, `shape?: string`, `color?: Color`, `child: Widget`

### `chip`
- properties: `click?: Action`, `tooltip?: string`, `label: string`, `avatar?: Widget`, `selected?: boolean`, `variant?: string`, `onDelete?: Action`, `onTap?: Action`

### `decoration`
- properties: `click?: Action`, `tooltip?: string`, `decoration?: BoxDecoration`, `color?: Color`, `borderRadius?: BorderRadius`, `border?: BoxBorder`, `gradient?: Gradient`, `image?: BackgroundImage`, `boxShadow?: array<BoxShadow>`, `shape?: string`, `backdropBlur?: number`, `child?: Widget`, `children?: Widget[]`

### `divider`
- properties: `click?: Action`, `tooltip?: string`, `thickness?: number`, `color?: string`, `indent?: number`, `endIndent?: number`

### `icon`
- properties: `click?: Action`, `tooltip?: string`, `icon: string | object`, `size?: string`, `sizeToken?: string`, `color?: Color`, `shader?: Gradient`

### `image`
- properties: `click?: Action`, `tooltip?: string`, `src: string`, `width?: number`, `height?: number`, `fit?: string`, `alignment?: Alignment`

### `imageFilter` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `filter: string`, `intensity?: number`, `child: Widget`

### `kenBurnsImage` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `src: AssetRef`, `duration?: Dimension`, `intensity?: number`, `startAlignment?: Alignment`, `endAlignment?: Alignment`, `loop?: boolean`, `curve?: AnimationCurve`, `width?: Dimension`, `height?: Dimension`, `fit?: string`

### `placeholder`
- properties: `click?: Action`, `tooltip?: string`, `fallbackWidth?: number`, `fallbackHeight?: number`, `color?: string`, `strokeWidth?: number`, `child?: Widget`

### `progressBar`
- aliases: `linearProgressIndicator`, `loadingIndicator`, `loading-indicator`, `progress-bar`, `progress`
- properties: `click?: Action`, `tooltip?: string`, `value?: number | binding`, `indicatorType?: string`, `color?: string`, `backgroundColor?: string`

### `richText`
- properties: `click?: Action`, `tooltip?: string`, `spans: array<Span>`, `style?: TextStyle`, `dropCap?: DropCap`, `textAlign?: string`, `textDirection?: string`, `maxLines?: number`, `overflow?: string`, `softWrap?: boolean`

### `text`
- properties: `click?: Action`, `tooltip?: string`, `text: string`, `variant?: string`, `style?: TextStyle`, `dropCap?: DropCap`, `maxLines?: number`, `overflow?: string`, `textAlign?: string`

### `tooltip`
- properties: `click?: Action`, `tooltip?: string`, `message: string`, `child: Widget`

### `verticalDivider`
- properties: `click?: Action`, `tooltip?: string`, `width?: number`, `thickness?: number`, `color?: string`, `indent?: number`, `endIndent?: number`

## Input

### `button`
- properties: `click?: Action`, `tooltip?: string`, `label: string`, `variant?: string`, `elevation?: string`, `icon?: string`, `enabled?: boolean`, `onTap?: Action`, `onDoubleTap?: Action`, `onLongPress?: Action`

### `checkbox`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`

### `checkboxGroup`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `options: Option[]`, `orientation?: string`

### `colorPicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `showAlpha?: boolean`, `showLabel?: boolean`, `pickerType?: string`, `enableHistory?: boolean`

### `dateField`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `format?: string`, `firstDate?: string`, `lastDate?: string`, `mode?: string`, `locale?: string`

### `datePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `firstDate?: string`, `lastDate?: string`

### `dateRangePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `startDate?: string`, `endDate?: string`, `label?: string`, `firstDate?: string`, `lastDate?: string`, `format?: string`, `locale?: string`

### `form`
- properties: `click?: Action`, `tooltip?: string`, `children: Widget[]`, `showErrorsOn?: string`, `onSubmit?: Action`

### `iconButton`
- properties: `click?: Action`, `tooltip?: string`, `icon: string`, `size?: number`, `color?: string`, `enabled?: boolean`, `onTap?: Action`

### `numberField`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `min?: number`, `max?: number`, `step?: number`, `decimalPlaces?: number`, `prefix?: string`, `suffix?: string`, `thousandSeparator?: string`

### `numberStepper`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `min?: number`, `max?: number`, `step?: number`

### `radio`
- properties: `click?: Action`, `tooltip?: string`, `value: any`, `groupValue: any | binding`, `label?: string`, `onChange?: Action`

### `radioGroup`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `options: Option[]`, `orientation?: string`

### `rangeSlider`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `min?: number`, `max?: number`, `divisions?: number`

### `rating`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `max?: number`, `icon?: string`, `color?: string`

### `segmentedControl`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `options: Option[]`, `variant?: string`

### `select`
- aliases: `dropdown`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `options: Option[]`, `placeholder?: string`

### `slider`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: number`, `enabled?: boolean`, `onChange?: Action`, `min?: number`, `max?: number`, `divisions?: number`

### `stepper`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `steps: Step[]`, `currentStep?: number | binding`, `stepperType?: string`, `onStepTapped?: Action`, `onStepContinue?: Action`, `onStepCancel?: Action`

### `textInput` *(since v1.0)*
- aliases: `textField`, `textfield`, `textFormField`, `text-form-field`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | binding`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `placeholder?: string`, `helperText?: string`, `prefixIcon?: string`, `suffixIcon?: string`, `obscureText?: boolean`, `readOnly?: boolean`, `maxLines?: integer`, `maxLength?: integer`, `inputType?: string`, `validation?: ValidationConfig`
- events: `onChange`, `onSubmit`, `onFocus`, `onBlur`

### `timeField`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `label?: string`, `format?: string`, `use24HourFormat?: boolean`, `mode?: string`

### `timePicker`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`, `use24HourFormat?: boolean`

### `toggle`
- aliases: `switch`
- properties: `click?: Action`, `tooltip?: string`, `binding?: string`, `value?: string | number | boolean | object | array`, `enabled?: boolean`, `onChange?: Action`

## Interaction

### `dragTarget`
- properties: `click?: Action`, `tooltip?: string`, `canDrop?: binding`, `builder?: Widget`, `children?: Widget[]`, `onDrop?: Action`, `onDragEnter?: Action`, `onDragLeave?: Action`

### `draggable`
- properties: `click?: Action`, `tooltip?: string`, `data: any | binding`, `feedback?: Widget`, `childWhenDragging?: Widget`, `child: Widget`

### `gestureDetector`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `onTap?: Action`, `onDoubleTap?: Action`, `onLongPress?: Action`, `onPanStart?: Action`, `onPanUpdate?: Action`, `onPanEnd?: Action`

### `inkWell`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `borderRadius?: number`, `onTap?: Action`, `onLongPress?: Action`

## Layout

### `align`
- properties: `click?: Action`, `tooltip?: string`, `alignment?: Alignment`, `child: Widget`

### `aspectRatio`
- properties: `click?: Action`, `tooltip?: string`, `aspectRatio?: number`, `child: Widget`

### `box` *(since v1.0)*
- aliases: `container`
- properties: `click?: Action`, `tooltip?: string`, `width?: number`, `height?: number`, `minWidth?: number`, `maxWidth?: number`, `minHeight?: number`, `maxHeight?: number`, `padding?: string`, `margin?: EdgeInsets`, `alignment?: Alignment`, `color?: Color`, `decoration?: BoxDecoration`
- children: single (key: `child`)

### `center`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `conditional`
- properties: `click?: Action`, `tooltip?: string`, `condition?: boolean | binding`, `then?: Widget`, `else?: Widget`, `switch?: binding`, `cases?: array`, `default?: Widget`

### `expanded`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`, `child: Widget`

### `flexible`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`, `fit?: string`, `child: Widget`

### `fractionallySized`
- properties: `click?: Action`, `tooltip?: string`, `widthFactor?: number`, `heightFactor?: number`, `child: Widget`

### `indexedStack`
- properties: `click?: Action`, `tooltip?: string`, `index?: number | binding`, `alignment?: Alignment`, `children: Widget[]`

### `intrinsicHeight`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `intrinsicWidth`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `linear`
- aliases: `row`, `column`
- properties: `click?: Action`, `tooltip?: string`, `mainAxisSize?: string`, `direction: string`, `alignment?: string`, `distribution?: string`, `spacing?: number`, `children: Widget[]`

### `margin`
- properties: `click?: Action`, `tooltip?: string`, `margin: EdgeInsets`, `child: Widget`

### `padding`
- properties: `click?: Action`, `tooltip?: string`, `padding: EdgeInsets`, `child: Widget`

### `positioned`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `safeArea`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `sizedBox`
- properties: `click?: Action`, `tooltip?: string`, `width?: number`, `height?: number`, `child?: Widget`

### `spacer`
- properties: `click?: Action`, `tooltip?: string`, `flex?: number`

### `stack`
- properties: `click?: Action`, `tooltip?: string`, `alignment?: Alignment`, `fit?: string`, `children: Widget[]`

### `visibility`
- properties: `click?: Action`, `tooltip?: string`, `visible?: boolean | binding`, `maintainSize?: boolean`, `maintainState?: boolean`, `replacement?: Widget`, `child?: Widget`, `children?: Widget[]`

### `wrap`
- properties: `click?: Action`, `tooltip?: string`, `direction?: string`, `spacing?: number`, `runSpacing?: number`, `alignment?: string`, `children: Widget[]`

## List

### `carousel` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `items?: binding`, `itemTemplate?: Widget`, `children?: Widget[]`, `scrollDirection?: string`, `viewportFraction?: number`, `loop?: boolean`, `autoPlay?: number`, `initialIndex?: number`, `transition?: string`, `indicatorPosition?: string`, `onPageChanged?: Action`

### `grid`
- aliases: `gridview`
- properties: `click?: Action`, `tooltip?: string`, `items?: binding`, `itemTemplate?: Widget`, `children?: Widget[]`, `columns: number | object`, `rowGap?: number`, `columnGap?: number`, `itemAspectRatio?: number`

### `list`
- aliases: `listView`, `listview`
- properties: `click?: Action`, `tooltip?: string`, `items?: binding`, `itemTemplate?: Widget`, `children?: Widget[]`, `spacing?: number`, `orientation?: string`, `emptyMessage?: string`, `itemExtent?: number`

### `listItem`
- aliases: `listTile`, `list-tile`
- properties: `click?: Action`, `tooltip?: string`, `title?: string | Widget`, `subtitle?: string | Widget`, `leading?: Widget`, `trailing?: Widget`, `onTap?: Action`, `selected?: boolean`, `enabled?: boolean`

### `staggeredGrid` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `items?: binding`, `itemTemplate?: Widget`, `children?: Widget[]`, `columns: number | object`, `mainAxisSpacing?: number`, `crossAxisSpacing?: number`, `padding?: EdgeInsets`, `scrollDirection?: string`

## Navigation

### `bottomNavigation`
- aliases: `bottomNav`, `bottomnavigationbar`
- properties: `click?: Action`, `tooltip?: string`, `selectedIndex?: number | binding`, `items: NavItem[]`, `onChange?: Action`

### `drawer`
- properties: `click?: Action`, `tooltip?: string`, `items?: DrawerItem[]`, `children?: Widget[]`, `header?: Widget`, `onSelect?: Action`, `onClose?: Action`

### `floatingActionButton`
- properties: `click?: Action`, `tooltip?: string`, `icon?: string`, `label?: string`, `onTap?: Action`

### `headerBar`
- aliases: `appbar`
- properties: `click?: Action`, `tooltip?: string`, `title?: string | Widget`, `leading?: Widget`, `actions?: Widget[]`, `exitButton?: ExitButtonConfig | boolean`, `backgroundColor?: string`, `elevation?: number`, `centerTitle?: boolean`

### `navigationRail`
- properties: `click?: Action`, `tooltip?: string`, `selectedIndex?: number | binding`, `items: NavItem[]`, `onChange?: Action`

### `popupMenuButton`
- properties: `click?: Action`, `tooltip?: string`, `icon?: string`, `items: MenuItem[]`, `onSelect?: Action`

### `tabBar`
- properties: `click?: Action`, `tooltip?: string`, `selectedIndex?: number | binding`, `tabs: Tab[]`, `onChange?: Action`

### `tabBarView`
- properties: `click?: Action`, `tooltip?: string`, `selectedIndex?: number | binding`, `children: Widget[]`

## Scroll

### `pageView`
- properties: `click?: Action`, `tooltip?: string`, `direction?: string`, `children: Widget[]`, `initialPage?: number`, `loop?: boolean`, `scrollPhysics?: string`, `allowImplicitScrolling?: boolean`, `onPageChanged?: Action`

### `scrollBar`
- properties: `click?: Action`, `tooltip?: string`, `thumbVisibility?: boolean`, `trackVisibility?: boolean`, `thickness?: number`, `radius?: number`, `child?: Widget`, `children?: Widget[]`

### `scrollView`
- properties: `click?: Action`, `tooltip?: string`, `direction?: string`, `padding?: EdgeInsets`, `scrollPhysics?: string`, `child?: Widget`, `children?: Widget[]`, `slivers?: array<Sliver>`

### `singleChildScrollView`
- properties: `click?: Action`, `tooltip?: string`, `direction?: string`, `padding?: EdgeInsets`, `child?: Widget`, `children?: Widget[]`

## Utility

### `accessibleWrapper`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `accessibility?: object`

### `baseline`
- properties: `click?: Action`, `tooltip?: string`, `baseline: number`, `baselineType?: string`, `child: Widget`

### `clipOval`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `clipRRect`
- properties: `click?: Action`, `tooltip?: string`, `borderRadius?: BorderRadius`, `child: Widget`

### `dashboard` *(since v1.3)*
- properties: `click?: Action`, `tooltip?: string`, `content?: Widget`, `refreshInterval?: number`, `onTap?: Action`

### `errorBoundary`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`, `fallback?: Widget`, `onError?: Action`

### `errorRecovery`
- properties: `click?: Action`, `tooltip?: string`, `child?: Widget`, `children?: Widget[]`, `fallback?: Widget`, `handlers?: object`, `onError?: Action`, `showDetails?: boolean`

### `fittedBox`
- properties: `click?: Action`, `tooltip?: string`, `fit?: string`, `alignment?: Alignment`, `child: Widget`

### `flow`
- properties: `click?: Action`, `tooltip?: string`, `children: Widget[]`, `direction?: string`, `spacing?: number`, `alignment?: string`

### `layoutBuilder`
- properties: `click?: Action`, `tooltip?: string`, `breakpoints?: object`, `layouts?: object`, `default?: Widget`

### `lazy`
- properties: `click?: Action`, `tooltip?: string`, `child: Widget`

### `limitedBox`
- properties: `click?: Action`, `tooltip?: string`, `maxWidth?: number`, `maxHeight?: number`, `child: Widget`

### `mediaQuery`
- properties: `click?: Action`, `tooltip?: string`, `condition?: object | binding`, `then?: Widget`, `else?: Widget`, `breakpoints?: object`, `defaultChild?: Widget`

### `offlineFallback`
- properties: `click?: Action`, `tooltip?: string`, `online?: Widget`, `offline?: Widget`, `message?: string`, `icon?: string`, `showRetry?: boolean`, `onRetry?: Action`, `isOnline?: boolean | binding`

### `permissionPrompt`
- properties: `click?: Action`, `tooltip?: string`, `permissions?: string[]`, `permissionType?: string`, `style?: string`, `title?: string`, `description?: string`, `icon?: string`, `allowPartial?: boolean`, `onAllow?: Action`, `onDeny?: Action`

### `use`
- properties: `click?: Action`, `tooltip?: string`, `template: string`, `params?: object`, `slots?: object`

