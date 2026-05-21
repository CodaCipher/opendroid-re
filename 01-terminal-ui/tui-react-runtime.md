# tui-react-runtime: TUI Architecture Notes

## Overview

The React runtime in this TUI system consists of two core vendor modules: **2206.js** (React 19.2.3 production runtime, 21.5 KB) providing the full hooks API, component base classes, and element creation; and **2313.js** (JSX runtime + Ink input/focus provider, 10.5 KB) containing the JSX dev runtime and the `KeypressProvider` context that bridges terminal stdin to React's context system. Supporting vendor modules provide PropTypes validation (3893.js with deps 3889/3890/3891/3892) and React type introspection (3888.js / `react-is`). The architecture follows a standard React custom renderer pattern: 2206.js exports all hooks as facades that delegate to `__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE`, which the react-reconciler (2228.js in tui-ink-core) resolves at render time. Module 2313.js notably combines the JSX runtime with OpenDroid-specific Ink extensions for keyboard input handling.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 2206.js | 21.5 KB | React 19.2.3 production runtime (hooks, components, createElement) | Vendor |
| 2313.js | 10.5 KB | JSX DEV runtime + KeypressProvider (Ink stdin/focus context) | Vendor/App hybrid |
| 3888.js | ~4 KB | `react-is` type introspection (Symbol-based React type checking) | Vendor |
| 3893.js | ~12 KB | `prop-types` runtime (type validators for React component props) | Vendor |
| 3889.js | ~1 KB | `object-assign` polyfill (Object.assign fallback) | Vendor |
| 3890.js | ~0.2 KB | PropTypes secret key constant | Vendor |
| 3891.js | ~0.2 KB | `hasOwnProperty` bound helper | Vendor |
| 3892.js | ~1 KB | PropTypes `checkPropTypes` factory | Vendor |

## Architecture

### React Runtime Layering

The React runtime follows a standard three-tier architecture used in all React custom renderer setups:

```
┌───────────────────────────────────┐
│  Application Components           │  (app modules using hooks)
│  useState, useEffect, useContext  │
└──────────────┬────────────────────┘
               │ import from
               ▼
┌───────────────────────────────────┐
│  2206.js: React Production RT    │  Hook facades → __CLIENT_INTERNALS
│  createElement, createContext,    │  Component/PureComponent base classes
│  forwardRef, memo, lazy, start…  │  Version: "19.2.3"
└──────────────┬────────────────────┘
               │ resolved by reconciler at render time
               ▼
┌───────────────────────────────────┐
│  2228.js: react-reconciler       │  (see tui-ink-core report)
│  Host config → Ink DOM nodes      │  Hooks dispatcher wired here
└───────────────────────────────────┘
```

### Hook Dispatch Mechanism

All hooks in 2206.js are thin facades. They call `LH()` to obtain the current React dispatcher, then delegate:

```js
// 2206.js, line ~1016
CLf.useEffect = function (c, wH) {
    return LH().useEffect(c, wH);
};
CLf.useState = function (c) {
    return LH().useState(c);
};
CLf.useContext = function (c) {
    return wH.useContext(c);  // via dispatcher
};
```

The `LH()` function accesses `__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE.H`: the shared internals object that contains `H` (current hook dispatcher), `A` (act queue), `T` (transition), etc. The reconciler (2228.js) sets the dispatcher before entering the render phase.

### React Symbols Registry (2206.js)

The runtime defines all React element type symbols at line 508-520:

```js
var QH = Symbol.for("react.transitional.element"),
    EH = Symbol.for("react.portal"),
    KH = Symbol.for("react.fragment"),
    JH = Symbol.for("react.strict_mode"),
    MH = Symbol.for("react.profiler"),
    XH = Symbol.for("react.consumer"),
    FH = Symbol.for("react.context"),
    OH = Symbol.for("react.forward_ref"),
    ZH = Symbol.for("react.suspense"),
    bH = Symbol.for("react.suspense_list"),
    SH = Symbol.for("react.memo"),
    RH = Symbol.for("react.lazy"),
    F  = Symbol.for("react.activity");
```

### Context Creation Mechanism

`createContext` (line ~820) produces a context object with Provider/Consumer pattern:

```js
CLf.createContext = function (c) {
    c = {
        $$typeof: FH,            // react.context symbol
        _currentValue: c,
        _currentValue2: c,
        _threadCount: 0,
        Provider: null,
        Consumer: null,
    };
    c.Provider = c;               // Provider IS the context object
    c.Consumer = { $$typeof: XH, _context: c };  // Consumer wraps it
    c._currentRenderer = null;
    c._currentRenderer2 = null;
    return c;
};
```

### Component Base Classes

2206.js defines the classic React component hierarchy:

- **`$` (Component)**: Line ~26: `this.props`, `this.context`, `this.refs`, `this.updater`. Has `setState()` and `forceUpdate()`.
- **`D` (PureComponent)**: Line ~39: extends Component prototype, sets `isPureReactComponent = true`.
- **Updater stub** (`t`): Default no-op updater: logs warnings ("not yet mounted") since real updater is injected by reconciler.

### Client Internals Structure

The shared internal state object `k` (line ~560):

```js
var k = {
    H: null,          // Hook dispatcher (set by reconciler during render)
    A: null,          // Act queue
    T: null,          // Transition state
    S: null,          // Transition subscriber
    actQueue: null,
    asyncTransitions: 0,
    isBatchingLegacy: false,
    didScheduleLegacyUpdate: false,
    didUsePromise: false,
    thrownErrors: [],
    getCurrentStack: null,
    recentlyCreatedOwnerStacks: 0,
};
```

This is exported as `__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE` and consumed by both 2313.js (JSX runtime) and the reconciler.

### Utility: Debounce/Throttle (2206.js tail)

After the React runtime, 2206.js includes utility functions at lines 1090-1150:

- **`m7I(fn, ms, opts)`**: Debounce implementation with `leading`/`trailing` edge support and `AbortSignal` integration.
- **`l7I(fn, wait, opts)`**: Throttle wrapper with `maxWait` support.

These are app-level utilities bundled in the same module, not part of the React runtime.

## Key Components

### Module 2206.js: React 19.2.3 Core

**React Hook Exports** (lines 950-1080):

```js
// useState: line 1064
CLf.useState = function (c) {
    return LH().useState(c);
};

// useEffect: line 1016
CLf.useEffect = function (c, wH) {
    return LH().useEffect(c, wH);
};

// useCallback: line 998
CLf.useCallback = function (c, wH) {
    return LH().useCallback(c, wH);
};

// useContext: line 1004
CLf.useContext = function (c) {
    var wH = LH();
    return wH.useContext(c);
};

// useReducer: line 1054
CLf.useReducer = function (c, wH, zH) {
    return LH().useReducer(c, wH, zH);
};

// useMemo: line 1046
CLf.useMemo = function (c, wH) {
    return LH().useMemo(c, wH);
};

// useRef: line 1057
CLf.useRef = function (c) {
    return LH().useRef(c);
};
```

**createElement** (line ~870):

```js
CLf.createElement = function (c, wH, zH) {
    // ... key extraction, props merging, children flattening
    // Returns: w(c, yH, YH, nH, _A ? Error("react-stack-top-frame") : UL, ...)
};
```

**forwardRef** (line ~910):

```js
CLf.forwardRef = function (c) {
    var wH = { $$typeof: OH, render: c };
    // ... displayName setup
    return wH;
};
```

**Version**: `CLf.version = "19.2.3"` (line ~1083).

### Module 2313.js: JSX Runtime + KeypressProvider

**JSX DEV Runtime** (lines 1-280):

```js
// Fragment export
L0f.Fragment = Q;  // Symbol.for("react.fragment")

// jsxDEV: line ~279
L0f.jsxDEV = function (JH, MH, XH, FH) {
    return P(JH, MH, XH, FH, OH ? Error("react-stack-top-frame") : QH, ...);
};
```

This is the JSX transform runtime: `jsxDEV` is called by JSX-compiled components instead of `createElement`.

**KeypressProvider Context** (lines ~290-620):

The module contains a substantial `KeypressProvider` component that bridges Node.js `readline` to React's context system:

```js
// KeypressProvider component: line ~303
function qCI({ children: H, enabled: A = true }) {
    let [L, $] = nB.useState(A),        // enabled state
        { stdin: I, setRawMode: D } = $s(),
        E = nB.useRef(new Set()).current, // subscriber set
        f = nB.useRef(void 0),            // handler ref
        M = nB.useRef(null),              // double-@ detection
        [U, P] = nB.useState(true),       // terminal focus state
        // ... refs for exit spec mode, comments, mission proposal
```

Key features:
- **Bracketed paste support**: Detects paste-start/paste-end sequences, buffers pasted content
- **Focus reporting**: Uses `\x1B[I` / `\x1B[O` escape sequences to detect terminal focus
- **Kitty keyboard protocol**: Checks for `ESC_KITTY` escape sequence
- **Input queue**: Queues rapid keystrokes, processes via `setTimeout(LH, 0)` for sequential delivery
- **Subscriber pattern**: Uses `Set` for subscribers with `subscribe()`/`unsubscribe()` callbacks
- **Escape key debounce**: 5ms timeout for bare `\` vs `\↵` disambiguation

The context value exposed via `fm.Provider`:

```js
let C = nB.useMemo(() => ({
    subscribe: Q,
    unsubscribe: w,
    handlerRef: f,
    isTerminalFocused: U,
    setEnabled: $,
    isEnabled: L,
    setSelectedExitSpecModeOptionIndex: (Y) => { B.current = Y; },
    getSelectedExitSpecModeOptionIndex: () => B.current,
    setExitSpecModeComment: (Y) => { W.current = Y; },
    getExitSpecModeComment: () => W.current,
    setMissionProposalComment: (Y) => { V.current = Y; },
    getMissionProposalComment: () => V.current,
}), [Q, w, U, L]);
```

### Module 3888.js: react-is

Type introspection using Symbol-based `$$typeof` checks. Exports:

```js
RyM.AsyncMode = x;         RyM.isAsyncMode = KH;     // deprecated
RyM.ConcurrentMode = v;    RyM.isConcurrentMode = JH;
RyM.ContextConsumer = g;   RyM.isContextConsumer = MH;
RyM.ContextProvider = e;   RyM.isContextProvider = XH;
RyM.Element = l;           RyM.isElement = FH;
RyM.ForwardRef = LH;       RyM.isForwardRef = OH;
RyM.Fragment = DH;         RyM.isFragment = ZH;
RyM.Lazy = VH;             RyM.isLazy = bH;
RyM.Memo = IH;             RyM.isMemo = SH;
RyM.Portal = i;            RyM.isPortal = RH;
RyM.isValidElementType = q;  // checks if value is valid React element type
RyM.typeOf = N;             // returns $$typeof category
```

Supports both Symbol-based (modern) and numeric fallback (60103-60121) type IDs.

### Module 3893.js: prop-types

PropTypes runtime providing validators: `array`, `bigint`, `bool`, `func`, `number`, `object`, `string`, `symbol`, `any`, `arrayOf`, `element`, `elementType`, `instanceOf`, `node`, `objectOf`, `oneOf`, `oneOfType`, `shape`, `exact`.

Dependencies chain: 3893 → 3889 (object-assign) + 3890 (secret key) + 3891 (hasOwnProperty) + 3892 (checkPropTypes).

## Theme / Style Tokens

(Not applicable for React runtime: no theme tokens in these modules. See tui-theme-system report.)

## Keyboard / Input Handling

While the React runtime modules (2206.js, 3888.js) do not handle input directly, **2313.js contains a complete keyboard input system** through its `KeypressProvider` component:

### Input Processing Pipeline

1. **Raw stdin data** → `i(data)` handler (line ~475): detects focus reporting sequences, strips them, routes to readline
2. **Readline keypress events** → `IH(char, key)` handler (line ~355): processes paste, escape sequences, Unicode Kitty protocol
3. **Normalized key event** → `DH(event)` dispatcher (line ~340): queues via microtask or delivers directly
4. **Subscribers** → `g(event)` fan-out (line ~325): iterates subscriber Set, calls each with normalized event

### Key Normalization

The `VH(key)` function (line ~385) normalizes readline key objects:

```js
// Maps readline key names to Ink-friendly format
case "tab" → { tab: true }
case "return" → { return: true }
case "escape" → { escape: true }
case "backspace" → { backspace: true }
case "up" → { upArrow: true }
case "down" → { downArrow: true }
case "left" → { leftArrow: true }
case "right" → { rightArrow: true }
```

### Terminal Mode Management

- On mount: enables raw mode (`D(true)`), writes `ENABLE_FOCUS_REPORTING` and `ENABLE_BRACKETED_PASTE` to stdout
- On unmount: writes `DISABLE_FOCUS_REPORTING` and `DISABLE_BRACKETED_PASTE`, disables raw mode, drains paste buffer

## Integration Points

### Upstream: react-reconciler (tui-ink-core)

- **2206.js** exports `__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE` which the reconciler (2228.js) uses to set the hook dispatcher (`k.H`) before entering the render phase
- The reconciler calls `useState`, `useEffect`, etc. on the dispatcher stored in the shared internals, not on the facade functions directly
- `createElement` and `jsxDEV` produce element objects that the reconciler processes through its diffing algorithm

### Downstream: Ink Components (tui-ink-core)

- **2313.js**'s `KeypressProvider` is an Ink component that uses `nB.useContext(fm)` (its own context) and `$s()` (Ink's `useStdin` hook from 2292.js)
- The context `fm` is consumed by app-level components via `gX()` accessor function (line ~290)
- `useRef`, `useState`, `useCallback`, `useMemo`, `useEffect` from 2206.js are used extensively in the KeypressProvider

### Cross-system: JSX Transform

- **2313.js** is the JSX DEV runtime module: Babel/SWC-compiled JSX calls `jsxDEV()` instead of `createElement()`
- All Ink components (Box, Text, etc.) and app components ultimately route through this module for element creation

### External: PropTypes (3893.js chain)

- The PropTypes chain (3893 + deps) provides development-time prop validation
- Used by React component libraries in the bundle for type checking
- Production builds typically strip these checks

## Implementation Notes

### React Runtime Compatibility

The TUI system uses **React 19.2.3** (the latest stable at time of extraction). OpenDroid porting considerations:

1. **Hook dispatcher pattern**: The `LH()` → `__CLIENT_INTERNALS.H` pattern is React 19 specific. React 18 used `ReactCurrentDispatcher` from `ReactSharedInternals`. Any porting must match the React version.

2. **`react.transitional.element` symbol**: React 19 uses `Symbol.for("react.transitional.element")` instead of the legacy `Symbol.for("react.element")` (60103). The `react-is` module (3888.js) supports both for backward compatibility.

3. **KeypressProvider is OpenDroid-specific**: This is not a standard Ink component: it's a custom OpenDroid extension with:
   - Exit spec mode state management (ref-based)
   - Mission proposal comment tracking (ref-based)
   - Terminal focus detection via escape sequences
   These OpenDroid-specific features need to be replicated or adapted in OpenDroid.

4. **Debounce/throttle utilities**: The `m7I` and `l7I` functions at the end of 2206.js are bundled alongside React. If needed separately, extract them to a standalone utility module.

5. **Context system**: The `createContext` API is standard. Any custom contexts (like `fm` in 2313.js) can be replicated using the same pattern.

6. **No server components**: This is a pure client runtime: no server component support needed for TUI porting.

## Module Reference

| Module | Key Lines | Content |
|--------|-----------|---------|
| 2206.js | 1-80 | Component/PureComponent base classes, setState, forceUpdate |
| 2206.js | 508-520 | React Symbol registry (element types) |
| 2206.js | 560-600 | Client internals (`k` object), stack frames |
| 2206.js | 820-850 | `createContext`: context creation |
| 2206.js | 870-900 | `createElement`: element factory |
| 2206.js | 910-940 | `forwardRef`, `memo` wrappers |
| 2206.js | 950-1080 | All hook exports (useState, useEffect, useContext, etc.) |
| 2206.js | 1083 | Version: "19.2.3" |
| 2206.js | 1090-1150 | Debounce (`m7I`) and throttle (`l7I`) utilities |
| 2313.js | 1-80 | Type introspection, JSX runtime setup |
| 2313.js | 220-280 | Symbol registry, jsxDEV export |
| 2313.js | 290-310 | KeypressProvider: useState, useRef, useCallback setup |
| 2313.js | 320-470 | Input processing: bracketed paste, key normalization, escape handling |
| 2313.js | 470-550 | Raw stdin handler, focus reporting, readline integration |
| 2313.js | 550-621 | Context value memoization, Provider render, module globals |
| 3888.js | 1-80 | Symbol-based type definitions (modern + legacy numeric fallbacks) |
| 3888.js | 80-167 | Type checker exports (isElement, isFragment, etc.) |
| 3893.js | 1-80 | PropTypes factory, validator base types |
| 3893.js | 80-482 | All PropTypes validators (arrayOf, oneOf, shape, etc.) |
| 3889.js | 1-35 | Object.assign polyfill |
| 3890.js | 1-3 | Secret key constant |
| 3891.js | 1-3 | hasOwnProperty bind helper |
| 3892.js | 1-45 | checkPropTypes factory with warning cache |
