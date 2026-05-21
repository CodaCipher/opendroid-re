# tui-ink-core: TUI Architecture Notes

## Overview

The Ink core render pipeline bridges React's reconciler (2228.js) to a terminal output layer. The architecture follows a **reconciler → host config → node tree → Yoga layout → output renderer** flow. React components are reconciled through a custom host config (2245.js) that creates lightweight ink DOM nodes (2239.js) backed by Yoga flexbox nodes. The main Ink renderer class (2294.js) manages the container lifecycle, FPS-throttled rendering, and terminal I/O. A context provider layer (2292.js) wraps the app tree with stdin/stdout/focus management contexts. The visual output renderer (2254.js) traverses the node tree using Yoga-computed positions and writes to a terminal output buffer. This is a standard React custom renderer architecture adapted for terminal UI, using mutation mode (not persistence or hydration).

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 2228.js | 231.3 KB | react-reconciler vendor bundle (factory function) | Vendor |
| 2294.js | ~5 KB | Ink renderer class: container lifecycle, FPS throttle, terminal I/O | Vendor |
| 2245.js | ~5 KB | Reconciler host config: createInstance, commitUpdate, scheduler wiring | Vendor |
| 2239.js | ~3 KB | Ink node tree: create/append/remove nodes, Yoga node management | Vendor |
| 2254.js | ~4 KB | Output renderer: tree → terminal output (screen reader + visual modes) | Vendor |
| 2292.js | ~4 KB | Context providers: stdin/stdout/focus/exit/error boundary | Vendor |

## Architecture

The Ink render pipeline operates in a clear layered architecture:

### Pipeline: Reconciler → Scheduler → Host Config → Node Tree → Yoga Layout → Output

```
React Component Tree
        │
        ▼
┌─────────────────┐
│  react-reconciler│  (2228.js): createContainer, updateContainer, flushSyncWork
│  (opendroid)       │
└────────┬────────┘
         │ calls host config methods
         ▼
┌─────────────────┐
│  Host Config     │  (2245.js): createInstance, commitUpdate, resetAfterCommit
│  (Ink renderer)  │  Wires scheduler (Am.unstable_scheduleCallback)
└────────┬────────┘
         │ creates/manages
         ▼
┌─────────────────┐
│  Ink Node Tree   │  (2239.js): hrH(create), jrH(appendChild), B7H(removeChild)
│  (DOM-like)      │  Each node has yogaNode for layout
└────────┬────────┘
         │ layout computed by
         ▼
┌─────────────────┐
│  Yoga Layout     │  (2218.js: see tui-yoga-layout)
│  (flexbox)       │  rootNode.yogaNode.setWidth(terminalWidth).calculateLayout()
└────────┬────────┘
         │ positions used by
         ▼
┌─────────────────┐
│  Output Renderer │  (2254.js): OwI() traverses tree, writes to output buffer
│  (visual+SR)     │  crH() for screen reader text output
└────────┬────────┘
         │ written to
         ▼
┌─────────────────┐
│  Terminal I/O    │  (2294.js): log.write(), throttled at maxFps (default 30)
│  (stdout)        │  Supports incremental rendering, kitty keyboard protocol
└─────────────────┘
```

### Render Flow (per update cycle)

1. **React reconciles**: `Oh.updateContainerSync(element, container)` or `Oh.updateContainer()` (concurrent mode)
2. **Host config mutations**: reconciler calls `createInstance`, `appendChild`, `commitUpdate` etc.
3. **resetAfterCommit**: triggers `rootNode.onComputeLayout()` → `calculateLayout()` → `rootNode.onRender()`
4. **calculateLayout**: sets Yoga root width to terminal width, calls `yogaNode.calculateLayout()`
5. **onRender**: calls `iwI(rootNode)` to generate output, writes to stdout via throttled log
6. **FPS throttle**: `GrH(onRender, throttleMs, { leading: true, trailing: true })` (lodash-style)

### Node Types

| Node Name | Purpose | Yoga Node? | Notes |
|-----------|---------|------------|-------|
| `ink-root` | Root container | Yes | Created in Ink renderer constructor |
| `ink-box` | Flexbox container | Yes | Maps to `<Box>` component |
| `ink-text` | Text container | Yes | Has measureFunc for text wrapping |
| `ink-virtual-text` | Nested text span | No | Created when `<Text>` nests inside `<Text>` |
| `#text` | Text leaf | No | Actual text content |

### Concurrent vs Legacy Mode

- `concurrent: true` → `UoH.ConcurrentRoot` + `Oh.updateContainer()` (async)
- `concurrent: false` (default) → `UoH.LegacyRoot` + `Oh.updateContainerSync()` + `Oh.flushSyncWork()` (sync)

## Key Components

### Module 2294.js: Ink Renderer Class `_oH`

The central orchestrator. Key constructor logic:

```js
// Module 2294.js, line 21-50
constructor(H) {
    this.rootNode = hrH("ink-root");
    this.rootNode.onComputeLayout = this.calculateLayout;
    let A = H.debug || this.isScreenReaderEnabled,
        L = H.maxFps ?? 30,
        $ = L > 0 ? Math.max(1, Math.ceil(1000 / L)) : 0;
    if (A) {
        this.rootNode.onRender = this.onRender;
    } else {
        let D = GrH(this.onRender, $, { leading: true, trailing: true });
        this.rootNode.onRender = D;
    }
    this.rootNode.onImmediateRender = this.onRender;
    this.log = DKI.create(H.stdout, { incremental: H.incrementalRendering });
    // ...
    let I = H.concurrent ? UoH.ConcurrentRoot : UoH.LegacyRoot;
    this.container = Oh.createContainer(
        this.rootNode, I, null, false, null, "id",
        () => {}, () => {}, () => {}, () => {}
    );
}
```

Layout calculation triggers Yoga:

```js
// Module 2294.js, line 93-97
calculateLayout = () => {
    let H = this.getTerminalWidth();
    this.rootNode.yogaNode.setWidth(H);
    this.rootNode.yogaNode.calculateLayout(void 0, void 0, AD.DIRECTION_LTR);
};
```

Render lifecycle:

```js
// Module 2294.js, line 98-116
onRender = () => {
    if (this.isUnmounted) return;
    let H = performance.now();
    let { output: A, outputHeight: L, staticOutput: $ } = iwI(this.rootNode, this.isScreenReaderEnabled);
    this.options.onRender?.({ renderTime: performance.now() - H });
    // ... handles debug mode, screen reader mode, TTY output with eraseLines
};
```

### Module 2245.js: Reconciler Host Config

The host config factory call wires the reconciler to Ink's DOM:

```js
// Module 2245.js, line 33-40
resetAfterCommit(H) {
    if (typeof H.onComputeLayout === "function") H.onComputeLayout();
    if (H.isStaticDirty) {
        H.isStaticDirty = false;
        if (typeof H.onImmediateRender === "function") H.onImmediateRender();
        return;
    }
    if (typeof H.onRender === "function") H.onRender();
},
```

Instance creation maps React elements to ink nodes:

```js
// Module 2245.js, line 46-60
createInstance(H, A, L, $) {
    if ($.isInsideText && H === "ink-box")
        throw Error("<Box> can't be nested inside <Text> component");
    let I = H === "ink-text" && $.isInsideText ? "ink-virtual-text" : H,
        D = hrH(I);
    for (let [E, f] of Object.entries(A)) {
        if (E === "children") continue;
        if (E === "style") {
            QdA(D, f);
            if (D.yogaNode) wdA(D.yogaNode, f);
            continue;
        }
        // ... handles internal_transform, internal_static, other attributes
    }
    return D;
},
```

Scheduler wiring:

```js
// Module 2245.js, line 94-98
isPrimaryRenderer: true,
supportsMutation: true,
supportsPersistence: false,
scheduleMicrotask: queueMicrotask,
scheduleCallback: Am.unstable_scheduleCallback,
cancelCallback: Am.unstable_cancelCallback,
shouldYield: Am.unstable_shouldYield,
now: Am.unstable_now,
```

Commit update:

```js
// Module 2245.js, line 118-140
commitUpdate(H, A, L, $) {
    if (qdA && H.internal_static) qdA.isStaticDirty = true;
    let I = e2I(L, $),   // diff props
        D = e2I(L.style, $.style);  // diff styles
    if (!I && !D) return;
    if (I) {
        for (let [E, f] of Object.entries(I)) {
            if (E === "style") { QdA(H, f); continue; }
            if (E === "internal_transform") { H.internal_transform = f; continue; }
            FdA(H, E, f);
        }
    }
    if (D && H.yogaNode) wdA(H.yogaNode, D);  // apply style diffs to yoga
},
```

### Module 2239.js: Ink Node Tree Management

Node creation creates a Yoga node per ink element:

```js
// Module 2239.js, line 3-11
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
};
```

Append child inserts into both JS tree and Yoga tree:

```js
// Module 2239.js, line 12-18
var jrH = (H, A) => {
    if (A.parentNode) B7H(A.parentNode, A);
    A.parentNode = H;
    H.childNodes.push(A);
    if (A.yogaNode)
        H.yogaNode?.insertChild(A.yogaNode, H.yogaNode.getChildCount());
    if (H.nodeName === "ink-text" || H.nodeName === "ink-virtual-text") SrH(H);
};
```

Text measure function for Yoga:

```js
// Module 2239.js, line 28-36
var DIf = function (H, A) {
    let L = H.nodeName === "#text" ? H.nodeValue : P7H(H),
        $ = MdA(L);
    if ($.width <= A) return $;
    let I = H.style?.textWrap ?? "wrap",
        D = RrH(L, A, I);
    return MdA(D);
};
```

### Module 2254.js: Output Renderer

Visual output traversal with Yoga positions:

```js
// Module 2254.js, line 34-60
var OwI = (H, A, L) => {
    let { offsetX: $ = 0, offsetY: I = 0, transformers: D = [] } = L;
    if (H.internal_static) return;
    let { yogaNode: f } = H;
    if (f) {
        if (f.getDisplay() === AD.DISPLAY_NONE) return;
        let M = $ + f.getComputedLeft(),
            U = I + f.getComputedTop(),
            P = D;
        if (typeof H.internal_transform === "function") P = [H.internal_transform, ...D];
        if (H.nodeName === "ink-text") {
            let W = P7H(H);
            if (W.length > 0) {
                let V = aa(W), X = $wI(f);
                if (V > X) {
                    let Q = H.style.textWrap ?? "wrap";
                    W = RrH(W, X, Q);
                }
                W = pIf(H, W);  // apply indentation from Yoga child position
                A.write(M, U, W, { transformers: P });
            }
            return;
        }
        // ... handles ink-box border rendering, overflow clipping
        if (H.nodeName === "ink-root" || H.nodeName === "ink-box") {
            for (let W of H.childNodes)
                OwI(W, A, { offsetX: M, offsetY: U, transformers: P, ... });
        }
    }
};
```

### Module 2292.js: Context Provider Layer

The `xKI` component wraps the app tree with contexts:

```js
// Module 2292.js, line 15-30
function xKI({ children, stdin, stdout, stderr, writeToStdout, writeToStderr,
               exitOnCtrlC, onExit, setCursorPosition }) {
    let [U, P] = nE.useState(true);       // isFocusEnabled
    let [B, W] = nE.useState(void 0);     // activeId
    let [, V] = nE.useState([]);          // focusStack
    // ...
    // Focus management callbacks
    let l = nE.useCallback(() => { /* focusNext via Tab */ }, [g]);
    let LH = nE.useCallback(() => { /* focusPrevious via Shift+Tab */ }, [e]);
    // ...
    return nE.default.createElement(
        arH.Provider, { value: { exit: q } },        // useApp context
        nE.default.createElement(
            srH.Provider, { value: { stdin, setRawMode: v, ... } }, // useStdin context
            nE.default.createElement(
                erH.Provider, { value: { stdout, write: I } },  // useStdout context
                nE.default.createElement(
                    edA.Provider, { value: { stderr, write: D } }, // useStderr context
                    nE.default.createElement(
                        HoH.Provider, { value: focusManager },  // useFocusManager context
                        nE.default.createElement(
                            AlA.Provider, { value: { setCursorPosition: M } },
                            nE.default.createElement(EoH, { onError: q }, H),
                        ),
                    ),
                ),
            ),
        ),
    );
}
```

Focus cycling with Tab / Shift+Tab:

```js
// Module 2292.js, line 84-90
nE.useEffect(() => {
    let OH = (bH) => {
        if (!U || X.current === 0) return;
        if (bH === xDf) l();      // "\t" → focusNext
        if (bH === vDf) LH();     // "\x1B[Z" → focusPrevious
    };
    w.current.on("input", OH);
}, [U, l, LH]);
```

### Module 2228.js: React Reconciler (Vendor)

The reconciler is a standard react-reconciler bundle (v19.2.0). Key exports used by Ink:

```js
// Module 2228.js, line 12593-12595
SE.createContainer = function (J, K, R, r, fH, _H, jH, cH, TA, xA) {
    return nS(J, K, false, null, R, r, _H, null, jH, cH, TA, xA);
};
// Module 2228.js, line 12956-12959
SE.updateContainer = function (J, K, R, r) {
    var fH = K.current, _H = oV(fH);
    return (qLH(fH, _H, J, K, R, r), _H);
};
// Module 2228.js, line 12818
SE.flushSyncWork = Q3;
// Module 2228.js, line 12961
SE.updateContainerSync = ui;
```

## Theme / Style Tokens

Ink-core does not define theme tokens directly. Style props are passed through as `{ style: { ... } }` from React components to ink nodes. The `wdA()` function (referenced in 2245.js, defined in another module within the ink-core cluster) applies style objects to Yoga nodes. Concrete theme tokens (colors, borders) are handled by the **tui-theme-system** feature (3773.js, 0939.js, etc.).

Key style properties observed in the pipeline:
- `flexDirection`: "row" | "row-reverse" | "column" | "column-reverse"
- `overflow` / `overflowX` / `overflowY`: "hidden" triggers clipping
- `textWrap`: "wrap" (default) controls text wrapping behavior
- `display`: `AD.DISPLAY_NONE` / `AD.DISPLAY_FLEX` for hide/unhide

## Keyboard / Input Handling

Ink-core provides the **focus management** infrastructure (2292.js), not specific key bindings:
- **Tab** (`\t`): focusNext
- **Shift+Tab** (`\x1B[Z`): focusPrevious
- **Ctrl+C** (`\x03`): triggers exit when `exitOnCtrlC` is true
- **Escape** (`\x1B`): handled but currently a no-op for focus

Raw mode is managed per stdin: `A.setRawMode(true)` on focus enable, `A.setRawMode(false)` on disable.

Specific key handlers for panels (arrow keys, Enter, etc.) are in the **tui-renderer-main-loop** and **tui-panel-*** features.

## Integration Points

1. **tui-yoga-layout** (2218.js): Ink-core directly uses `AD.Node.create()`, `AD.DIRECTION_LTR`, `AD.DISPLAY_NONE`, `AD.DISPLAY_FLEX`, `AD.EDGE_*` from the Yoga layout engine. The `calculateLayout()` call on ink-root triggers the full Yoga layout pass.

2. **tui-react-runtime** (2206.js, 2313.js): Ink-core imports React (`LL()`) for `createElement`, `PureComponent`, `useState`, `useEffect`, `useCallback`, `useRef`, `useMemo`, `createContext`. The reconciler uses the React dispatcher (`ML`).

3. **tui-theme-system** (3773.js, 0939.js): Theme tokens flow into Ink as style props on `<Box>` and `<Text>` components. The `wdA()` function applies style objects to Yoga nodes.

4. **tui-ansi-terminal-utils** (2539.js, etc.): Ink's output layer uses ANSI escape codes for cursor positioning (`Hs`, `Lm`), line erasing (`dB.eraseLines`), and terminal clearing (`dB.clearTerminal`). String width calculation (`aa()`, `MdA()`) wraps text correctly in terminal.

5. **tui-renderer-main-loop**: The main entry point calls Ink's `render()` function and provides stdin/stdout configuration. Kitty keyboard protocol support is initialized in the Ink renderer.

6. **tui-logo-animation** (4065.js): Uses the `<FullScreenAnimation>` component which leverages Ink's `internal_static` node feature for overlay rendering.

7. **External: scheduler** (`DdA()` module): The `Am.unstable_scheduleCallback`, `Am.unstable_cancelCallback`, `Am.unstable_shouldYield`, `Am.unstable_now` functions are used by the reconciler host config for scheduling work.

## Implementation Notes

To port this Ink core render pipeline to OpenDroid:

1. **Keep the reconciler architecture**: React custom renderer pattern is well-established. The host config (2245.js pattern) is the primary customization point.

2. **Replace Yoga**: Swap `AD.Node.create()` and Yoga API calls with OpenDroid's layout engine. The interface needed: `setWidth`, `calculateLayout`, `getComputedLeft/Top/Width/Height`, `setMeasureFunc`, `insertChild/removeChild`, `markDirty`.

3. **Replace terminal output**: The `OwI()` renderer (2254.js) writes to a terminal buffer. Replace with OpenDroid's render target. The position-based write API (`A.write(x, y, text)`) is the abstraction point.

4. **Keep context providers**: The stdin/stdout/focus management contexts (2292.js) are reusable. Replace stdin handling with OpenDroid's input system.

5. **FPS throttling**: The `GrH()` throttle pattern (lodash-style) with `maxFps: 30` default can be adjusted for OpenDroid's refresh rate.

6. **Node types map directly**:
   - `ink-root` → OpenDroid root container
   - `ink-box` → OpenDroid flex container
   - `ink-text` → OpenDroid text element
   - `ink-virtual-text` → OpenDroid inline text span

7. **Key modules to reuse as-is** (with Yoga/output swaps):
   - 2245.js (host config): change `createInstance` to create OpenDroid nodes
   - 2239.js (tree management): change `hrH()` to create OpenDroid nodes
   - 2292.js (contexts): change stdin/raw mode to OpenDroid input

## Module Reference

| Module | Lines Read | Key Functions/Classes |
|--------|------------|----------------------|
| 2228.js | 1-80, 12580-13001 | `createContainer()`, `updateContainer()`, `updateContainerSync()`, `flushSyncWork()` |
| 2294.js | Full (120 lines) | `_oH` class: constructor, `calculateLayout()`, `onRender()`, `render()`, `unmount()`, `waitUntilExit()` |
| 2245.js | Full (165 lines) | Host config: `createInstance()`, `commitUpdate()`, `resetAfterCommit()`, `getChildHostContext()`, scheduler wiring |
| 2239.js | Full (50 lines) | `hrH()` (createNode), `jrH()` (appendChild), `GdA()` (insertBefore), `B7H()` (removeChild), `DIf()` (measureFunc), `SrH()` (markDirty) |
| 2254.js | Full (70 lines) | `OwI()` (visual output traversal), `crH()` (screen reader output), `pIf()` (indentation) |
| 2292.js | Full (120 lines) | `xKI` context provider component, focus management (Tab/Shift+Tab), raw mode, error boundary `EoH` |
