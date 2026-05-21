# tui-yoga-layout: TUI Architecture Notes

## Overview

The Yoga layout engine in this TUI system is provided via a WebAssembly-compiled library (2218.js, 120 KB) with an emscripten-generated JS glue layer. The raw WASM bindings are wrapped by a high-level JavaScript API module (2219.js) that adds ergonomic method overloading for CSS-like style values (auto, percent, point units). The Ink rendering system consumes Yoga through an intermediate node tree (2239.js) where each DOM-like node holds a `yogaNode` property. A dedicated style-to-Yoga mapper module (2240.js) translates Ink component style props (flexDirection, margin, padding, width, etc.) into Yoga C API calls. The Ink renderer (2294.js) triggers layout calculation by setting the root node's width to the terminal width and calling `calculateLayout()`, after which the output renderer (2254.js) reads computed positions via `getComputedLeft()`, `getComputedTop()`, `getComputedWidth()`, and `getComputedHeight()`.

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 2218.js | 119.9 KB | Yoga WASM runtime: emscripten glue, embind type system, WASM instantiation | Vendor |
| 2219.js | ~8 KB | Yoga JS API wrapper: enum constants, method overloading, Node/Config facade | Vendor |
| 2239.js | ~2 KB | Ink node tree: node creation with yogaNode, parent-child Yoga tree management | Vendor |
| 2240.js | ~3 KB | Style-to-Yoga mapper: translates Ink style props to Yoga API calls | Vendor |
| 2254.js | ~3 KB | Output renderer: reads Yoga-computed positions, writes to terminal buffer | Vendor |
| 2294.js | ~5 KB | Ink renderer: triggers calculateLayout on root, manages terminal I/O | Vendor |

## Architecture

The Yoga layout system operates in a four-stage pipeline:

### Stage 1: WASM Initialization (2218.js)
The Yoga C library is compiled to WebAssembly. Module 2218.js is the emscripten-generated glue code that:
- Loads and instantiates the WASM binary via `WebAssembly.instantiateStreaming` with fallback to `ArrayBuffer`
- Provides the embind type system that bridges C++ classes (`YGNode`, `YGConfig`) to JavaScript
- Exposes typed memory views (HEAP8, HEAP16, HEAP32, HEAPU8, HEAPU32, HEAPF32, HEAPF64)
- Registers class constructors, methods, and property accessors through the embind `wasmTable`

### Stage 2: JS API Wrapper (2219.js)
Module 2219.js wraps the raw embind-exposed C API with ergonomic JavaScript:
- **Enum constants**: Defines all Yoga enums as JS objects (ALIGN_*, DISPLAY_*, EDGE_*, FLEX_DIRECTION_*, etc.)
- **Method overloading**: Patches `Node.prototype` methods (`setWidth`, `setHeight`, `setMargin`, `setPadding`, `setPosition`, `setFlexBasis`, `setGap`) to accept string ("50%"), number (50), or "auto" values, routing to the appropriate underlying C function
- **Node.create**: OpenDroid that calls `createDefault()` or `createWithConfig(config)` based on argument
- **calculateLayout**: Wrapper with default NaN width/height and LTR direction
- **MeasureCallback/DirtiedCallback**: Wraps JS functions as WASM callbacks for text measurement and dirty notification

### Stage 3: Style Application (2240.js)
When a React component's style props change, the reconciler (via host config 2245.js) calls `wdA(yogaNode, style)`. This function decomposes the style object into sub-mappers:

```
VIf(yogaNode, style):
  fIf  → position (absolute/relative)
  MIf  → margin (margin, marginX, marginY, marginLeft/Right/Top/Bottom)
  UIf  → padding (padding, paddingX, paddingY, paddingLeft/Right/Top/Bottom)
  _If  → flex (flexGrow, flexShrink, flexWrap, flexDirection, flexBasis, alignItems, alignSelf, justifyContent)
  PIf  → dimensions (width, height, minWidth, minHeight): supports number, "percent%", and auto
  BIf  → display (flex/none)
  WIf  → border (borderStyle → 1/0 border width on each edge)
  TIf  → gap (gap, columnGap, rowGap)
```

### Stage 4: Layout Calculation & Consumption (2294.js → 2254.js)
The Ink renderer's `calculateLayout` method:
1. Reads terminal width via `getTerminalWidth()`
2. Sets root yogaNode width: `rootNode.yogaNode.setWidth(terminalWidth)`
3. Calls `rootNode.yogaNode.calculateLayout(undefined, undefined, DIRECTION_LTR)`

The output renderer (`OwI()` in 2254.js) then traverses the node tree:
- For each node, reads `yogaNode.getComputedLeft()` and `yogaNode.getComputedTop()` for absolute position
- Reads `getComputedWidth()` and `getComputedHeight()` for clipping regions
- Reads `getComputedBorder(EDGE_*)` to offset content inside borders
- Writes text at computed coordinates to the terminal output buffer

### Pipeline Diagram

```
React Component (style={flexDirection:"row", margin:1})
        │
        ▼
Host Config (2245.js): commitUpdate → QdA(node, style) + wdA(yogaNode, style)
        │
        ▼
Style Mapper (2240.js): translates props → yogaNode.setFlexDirection(ROW), etc.
        │
        ▼
Ink Renderer (2294.js): calculateLayout():
    rootNode.yogaNode.setWidth(terminalWidth)
    rootNode.yogaNode.calculateLayout(undefined, undefined, LTR)
        │
        ▼
Yoga WASM Engine (2218.js): C++ flexbox algorithm on WASM heap
        │
        ▼
Output Renderer (2254.js): reads getComputedLeft/Top/Width/Height
        │
        ▼
Terminal Buffer: text written at computed x,y coordinates
```

## Key Components

### Module 2219.js: Yoga Enum Constants (lines 1-160)

The module defines all Yoga enum constants as JavaScript objects. Key enums:

```js
// Module 2219.js, lines 1-80
// Flex Direction enum
KrH = (function (H) {
  return (
    (H[(H.Column = 0)] = "Column"),
    (H[(H.ColumnReverse = 1)] = "ColumnReverse"),
    (H[(H.Row = 2)] = "Row"),
    (H[(H.RowReverse = 3)] = "RowReverse"),
    H
  );
})({}),

// Align enum
Rb = (function (H) {
  return (
    (H[(H.Auto = 0)] = "Auto"),
    (H[(H.FlexStart = 1)] = "FlexStart"),
    (H[(H.Center = 2)] = "Center"),
    (H[(H.FlexEnd = 3)] = "FlexEnd"),
    (H[(H.Stretch = 4)] = "Stretch"),
    (H[(H.Baseline = 5)] = "Baseline"),
    (H[(H.SpaceBetween = 6)] = "SpaceBetween"),
    (H[(H.SpaceAround = 7)] = "SpaceAround"),
    (H[(H.SpaceEvenly = 8)] = "SpaceEvenly"),
    H
  );
})({}),
```

### Module 2219.js: Node.create OpenDroid (line 296)

```js
// Module 2219.js, line 296
A(H.Node, "create", (I, D) => {
  return D ? H.Node.createWithConfig(D) : H.Node.createDefault();
}),
```

### Module 2219.js: calculateLayout Wrapper (line 305)

```js
// Module 2219.js, line 305
A(H.Node.prototype, "calculateLayout", function (I) {
  let D = arguments.length > 1 && arguments[1] !== void 0 ? arguments[1] : NaN,
    E = arguments.length > 2 && arguments[2] !== void 0 ? arguments[2] : NaN,
    f = arguments.length > 3 && arguments[3] !== void 0 ? arguments[3] : f7H.LTR;
  return I.call(this, D, E, f);
}),
```

### Module 2219.js: Unit-aware Method Overloading (lines 228-270)

```js
// Module 2219.js, lines 228-260
// Patches setPosition, setMargin, setFlexBasis, setWidth, setHeight, etc.
// to accept number (point), string with "%" (percent), or "auto"
for (let I of [
  "setPosition", "setMargin", "setFlexBasis", "setWidth", "setHeight",
  "setMinWidth", "setMinHeight", "setMaxWidth", "setMaxHeight",
  "setPadding", "setGap",
]) {
  let D = {
    [jZ.Point]: H.Node.prototype[I],           // e.g., setWidth(50)
    [jZ.Percent]: H.Node.prototype[`${I}Percent`], // e.g., setWidthPercent(50)
    [jZ.Auto]: H.Node.prototype[`${I}Auto`],    // e.g., setWidthAuto()
  };
  A(H.Node.prototype, I, function (E) {
    let P = M.pop(); // last arg = value
    if (P === "auto") ((B = jZ.Auto), (W = void 0));
    else if (typeof P === "object") ((B = P.unit), (W = P.valueOf()));
    else if (typeof P === "string" && P.endsWith("%")) ((B = jZ.Percent), (W = parseFloat(P)));
    else ((B = jZ.Point), (W = parseFloat(P)));
    return D[B].call(this, ...M, W);
  });
}
```

### Module 2239.js: Ink Node with Yoga Binding (lines 6-16)

```js
// Module 2239.js, lines 6-16
var hrH = (H) => {
  let A = {
    nodeName: H,
    style: {},
    attributes: {},
    childNodes: [],
    parentNode: void 0,
    yogaNode: H === "ink-virtual-text" ? void 0 : AD.Node.create(),
    internal_accessibility: {},
  };
  if (H === "ink-text") A.yogaNode?.setMeasureFunc(DIf.bind(null, A));
  return A;
},
```

### Module 2239.js: Parent-Child Yoga Tree Management (lines 18-37)

```js
// Module 2239.js, lines 18-37
// appendChild: inserts into both JS childNodes array and Yoga tree
jrH = (H, A) => {
  if (A.parentNode) B7H(A.parentNode, A);
  if (((A.parentNode = H), H.childNodes.push(A), A.yogaNode))
    H.yogaNode?.insertChild(A.yogaNode, H.yogaNode.getChildCount());
  if (H.nodeName === "ink-text" || H.nodeName === "ink-virtual-text") SrH(H);
},

// removeChild: removes from both JS array and Yoga tree
B7H = (H, A) => {
  if (A.yogaNode) A.parentNode?.yogaNode?.removeChild(A.yogaNode);
  A.parentNode = void 0;
  let L = H.childNodes.indexOf(A);
  if (L >= 0) H.childNodes.splice(L, 1);
  if (H.nodeName === "ink-text" || H.nodeName === "ink-virtual-text") SrH(H);
},
```

### Module 2240.js: Flex Direction Mapping (lines 40-45)

```js
// Module 2240.js, lines 40-45
if ("flexDirection" in A) {
  if (A.flexDirection === "row") H.setFlexDirection(AD.FLEX_DIRECTION_ROW);
  if (A.flexDirection === "row-reverse") H.setFlexDirection(AD.FLEX_DIRECTION_ROW_REVERSE);
  if (A.flexDirection === "column") H.setFlexDirection(AD.FLEX_DIRECTION_COLUMN);
  if (A.flexDirection === "column-reverse")
    H.setFlexDirection(AD.FLEX_DIRECTION_COLUMN_REVERSE);
}
```

### Module 2240.js: Dimension Mapping (lines 76-82)

```js
// Module 2240.js, lines 76-82
if ("width" in A)
  if (typeof A.width === "number") H.setWidth(A.width);
  else if (typeof A.width === "string") H.setWidthPercent(Number.parseInt(A.width, 10));
  else H.setWidthAuto();
if ("height" in A)
  if (typeof A.height === "number") H.setHeight(A.height);
  else if (typeof A.height === "string") H.setHeightPercent(Number.parseInt(A.height, 10));
  else H.setHeightAuto();
```

### Module 2294.js: Layout Calculation Trigger (lines 122-126)

```js
// Module 2294.js, lines 122-126
calculateLayout = () => {
  let H = this.getTerminalWidth();
  (this.rootNode.yogaNode.setWidth(H),
    this.rootNode.yogaNode.calculateLayout(void 0, void 0, AD.DIRECTION_LTR));
};
```

### Module 2254.js: Yoga-Computed Position Reading (lines 57-86)

```js
// Module 2254.js, lines 57-86
OwI = (H, A, L) => {
  let { yogaNode: f } = H;
  if (f) {
    if (f.getDisplay() === AD.DISPLAY_NONE) return;
    let M = $ + f.getComputedLeft(),    // absolute X position
        U = I + f.getComputedTop(),      // absolute Y position
    // ... for boxes, compute border offsets:
    let X = W ? M + f.getComputedBorder(AD.EDGE_LEFT) : void 0,
        Q = W ? M + f.getComputedWidth() - f.getComputedBorder(AD.EDGE_RIGHT) : void 0,
        w = V ? U + f.getComputedBorder(AD.EDGE_TOP) : void 0,
        C = V ? U + f.getComputedHeight() - f.getComputedBorder(AD.EDGE_BOTTOM) : void 0;
    // clip region for overflow:hidden
    A.clip({ x1: X, x2: Q, y1: w, y2: C });
  }
}
```

## Theme / Style Tokens

Yoga itself does not carry theme tokens: it is a pure layout engine. However, the style mapper (2240.js) processes these layout-related style properties that theme tokens may feed into:

- **borderStyle**: Controls border visibility (1 = visible, 0 = hidden) per edge; theme borderColor is applied at the output rendering layer, not in Yoga
- **gap / columnGap / rowGap**: Spacing between flex children; numeric values in terminal cells
- **padding / margin**: Spacing tokens; numeric values in terminal cells
- **display**: "flex" or "none": controls visibility at layout level

Color and visual theming is handled by the theme system (tui-theme-system) and applied during output rendering (2254.js), not by Yoga.

## Keyboard / Input Handling

Not applicable. Yoga is a pure layout computation engine with no input handling. Keyboard/input events flow through the Ink context providers (2292.js) and are consumed by React components, not by the layout system.

## Integration Points

- **tui-ink-core**: Yoga is the layout backbone of the Ink render pipeline. The reconciler host config (2245.js) calls `wdA(yogaNode, style)` on every commitUpdate. The renderer (2294.js) calls `calculateLayout()` before each render pass. The output renderer (2254.js) reads computed positions from Yoga nodes. See `tui-ink-core.md` Architecture section for the full pipeline diagram.
- **tui-react-runtime**: React components define style props that flow through the reconciler to the Yoga style mapper. useState/useEffect changes trigger re-renders which re-apply styles via 2240.js.
- **tui-ansi-terminal-utils**: The terminal width detection (used by `getTerminalWidth()` in 2294.js) determines the root Yoga node's width constraint. String width calculation (`MdA` in 2239.js) is used by Yoga's measure function for text nodes.
- **tui-theme-system**: Theme tokens for spacing, padding, and gap values feed into the style objects that 2240.js translates to Yoga calls. Border visibility (borderStyle) is a Yoga concern, but border color comes from the theme system.
- **tui-renderer-main-loop**: The main render loop coordinates terminal resize events → layout recalculation → output rendering, using Yoga as the core layout engine.

## Implementation Notes

For porting the Yoga-based layout system to OpenDroid:

1. **Dependency**: yoga-layout-prebuilt (npm) provides the same WASM + JS wrapper. Alternatively, use `yoga-layout` which ships pre-built WASM.
2. **Node creation pattern**: Each UI element needs a `yogaNode` created via `Node.create()`. Virtual text nodes (no layout) skip this.
3. **Style mapping**: The style-to-Yoga mapper (2240.js pattern) is straightforward: map CSS-like string values to Yoga enum constants + method calls. This pattern is reusable as-is.
4. **Text measurement**: Text nodes need a `setMeasureFunc` callback that returns `{width, height}` for text measurement. This requires integrating with a string-width utility (e.g., `string-width` npm package for terminal cells).
5. **Layout trigger**: Call `rootNode.setWidth(terminalWidth)` then `rootNode.calculateLayout()` before each render. Height is typically unconstrained (NaN) for terminal apps that grow vertically.
6. **Position reading**: After layout, traverse the node tree and read `getComputedLeft()`, `getComputedTop()`, `getComputedWidth()`, `getComputedHeight()` for each node to determine its position in the terminal grid.

## Module Reference

- **2218.js** (lines 1-1524): Yoga WASM runtime: emscripten glue code, embind type system, WASM instantiation, memory management
  - Lines 1-80: Module wrapper, WASM setup, UTF-8 decoder, memory views
  - Lines 140-280: Handle management (emval), type registration, class registration
  - Lines 700-770: Pointer management (clone, delete, isDeleted)
  - Lines 1000-1100: Constructor/method registration (embind `p` and `a` functions)
  - Lines 1400-1524: WASM instantiation, runtime initialization, exported cleanup functions
- **2219.js** (lines 1-314): Yoga JS API: enum definitions, method overloading, Node/Config facade
  - Lines 1-160: Enum definitions (FlexDirection, Align, Display, Edge, Wrap, Position, etc.)
  - Lines 160-225: Constant map (ALIGN_*, DISPLAY_*, FLEX_DIRECTION_*, etc.)
  - Lines 228-295: Method overloading for unit-aware setters (point/percent/auto)
  - Lines 296-314: Node.create, calculateLayout, free, freeRecursive wrappers
- **2239.js** (lines 1-68): Ink node tree with Yoga integration
  - Lines 6-16: Node creation (hrH): creates yogaNode via AD.Node.create()
  - Lines 18-37: Parent-child management (jrH/append, B7H/remove): keeps Yoga tree in sync
  - Lines 38-68: Text measurement (DIf), ancestor yoga lookup (r2I), dirty marking (SrH)
- **2240.js** (lines 1-100): Style-to-Yoga property mapper
  - Lines 10-12: Position type mapping (fIf)
  - Lines 14-21: Margin mapping (MIf)
  - Lines 23-30: Padding mapping (UIf)
  - Lines 32-69: Flex properties mapping (_If): flexGrow, flexShrink, flexWrap, flexDirection, flexBasis, alignItems, alignSelf, justifyContent
  - Lines 71-90: Dimension mapping (PIf): width, height, minWidth, minHeight
  - Lines 92-94: Display mapping (BIf)
  - Lines 96-103: Border mapping (WIf)
  - Lines 105-108: Gap mapping (TIf)
  - Lines 110-112: Composite mapper (VIf): calls all sub-mappers
- **2254.js** (lines 1-100): Output renderer that reads Yoga-computed positions
  - Lines 7-17: Root node positioning (pIf): getComputedLeft/Top
  - Lines 19-50: Screen reader text renderer (crH): tree traversal
  - Lines 52-100: Visual output renderer (OwI): reads computed positions, writes to buffer
- **2294.js** (lines 122-126): Ink renderer calculateLayout method
