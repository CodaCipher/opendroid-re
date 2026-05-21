# tui-logo-animation: TUI Architecture Notes

## Overview

The OpenDroid TUI logo animation is a frame-based ASCII art splash screen that plays on startup. It is implemented as a React Ink component called `FullScreenAnimation` (exported from module 4064.js as `OuM`). The animation cycles through a set of pre-rendered ASCII art frames depicting a stylized robot/droid logo using Unicode block characters (░▒▓█). Each frame is displayed at a configurable FPS (default 20), with a post-animation dissolve phase that progressively replaces dense characters with lighter ones across 4 steps at 120ms intervals. The animation is invoked from a boot orchestrator in module 4065.js via `CA0()`, which dynamically imports the animation module and renders it as a standalone Ink app with a 10-second hard timeout.

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 4064.js | 118.2 KB | Art asset frames + FullScreenAnimation component (OuM) + dissolve engine | app |
| 4065.js | 47.2 KB | Boot coordinator: `CA0()` orchestrator that invokes FullScreenAnimation, main TUI app shell | app |
| 2243.js | 227.8 KB | Main TUI coordinator: `GA0` module initialization, session management, full app render tree | app |

## Architecture

### Animation Pipeline

```
Boot (4065.js) → CA0()
    ↓
Dynamic import: GA0().then(() => XA0)
    ↓
Extract: { FullScreenAnimation: OuM } from XA0
    ↓
Standalone Ink render: Sb(RL.jsxDEV(OuM, { onComplete, fps=20 }))
    ↓
OuM component lifecycle:
  1. Hide terminal cursor (Tv.cursorHide)
  2. Read terminal dimensions (stdout.columns, stdout.rows)
  3. Start frame loop: setInterval(w, intervalMs)
     - intervalMs = Math.max(20, Math.floor(1000 / fps))
     - Default: Math.max(20, 50) = 50ms per frame
  4. Each tick: increment frameIndex ($)
  5. When frameIndex reaches last frame:
     a. clearInterval (stop frame loop)
     b. Begin dissolve phase: increment dissolveStep (D) every 120ms
     c. dissolveStep ranges 0..quM (4 steps)
  6. After dissolve completes:
     - Wait 100ms
     - Call onComplete()
    ↓
CA0() Promise resolves → iW() clears screen → main app starts
```

### Frame Data Structure

- `VA0`: Array of string arrays, each sub-array is one ASCII art frame (line-per-element)
- `_i`: Processed frames: `VA0.map(H => H.join("\n"))`: flattened to newline-joined strings
- `wuM`: Maximum line count across all frames (for vertical alignment padding)
- Frame count: ~10+ frames (the logo progressively fills in from sparse to dense)

### Dissolve Engine (Character Degradation)

```js
// Character replacement map: dense → lighter characters
KuM = {
  "█": ["▓", "▒", "░", " "],   // Full block → degrades through shades
  "░": [" ", " ", " ", " "],   // Light shade → immediate clear
  "▒": ["░", " ", " ", " "],   // Medium shade
  "▓": ["▒", "░", " ", " "],   // Dark shade
  "#": ["*", ".", " ", " "],   // Hash → star → dot → space
  "*": [".", " ", " ", " "],   // Star → dot → space
  ".": [" ", " ", " ", " "],   // Dot → space
}
CuM = [".", " ", " ", " "]     // Default fallback for unknown chars
```

Function `YuM(frameText, dissolveStep)`:
- Iterates each character in the frame text
- Looks up replacement array from `KuM` (or `CuM` fallback)
- Returns `replacementArray[Math.min(dissolveStep - 1, array.length - 1)]`
- Step 0 = original frame, steps 1-4 = progressively degraded

## Key Components

### FullScreenAnimation Component (4064.js, line 1168)

```text
FullScreenAnimation:
  read terminal dimensions
  maintain current frame and dissolve step state
  advance animation frames on a timer derived from fps
  after final frame, run dissolve sequence over a few steps
  clear timers, guard against duplicate completion, then call onComplete
  render centered bold frame text in a full-screen Ink layout
```

### Boot Orchestrator CA0() (4065.js, line 4040)

```text
BootAnimation:
  lazy-load FullScreenAnimation
  render it as a standalone Ink app
  on completion, unmount and clear terminal
  also enforce a hard timeout fallback to avoid blocking startup
```

### Art Frame Data (4064.js, lines 6-1148)

```js
// Module 4064.js, line 6
var TA0 = O(() => {
  VA0 = [
    [
      "                        *##*                                ",
      "                      *#░░░░*        *                      ",
      "                     *░░##░░#*##░░░░░░░░#                   ",
      // ... 23 lines per frame, depicting robot/droid head
      "                                **                          ",
    ],
    // ... multiple frames with progressive fill
    // Each frame uses characters: ░ ▒ ▓ █ # * . and space
    // Frames transition from sparse outline → fully filled logo
  ];
});
```

### Character Dissolve Map (4064.js, lines 21-36)

```js
// Module 4064.js, line 21-36 (inside GA0 async initializer)
KuM = {
  "\u2588": ["\u2593", "\u2592", "\u2591", " "],  // █ → ▓▒░ space
  "\u2590": ["#", "*", ".", " "],                    // ░ → #*. space
  "\u2593": ["\u2592", "\u2591", " ", " "],          // ▓ → ▒░ space
  "\u2592": ["\u2591", " ", " ", " "],               // ▒ → ░ space
  "\u2591": [" ", " ", " ", " "],                     // ░ → space
  "#": ["*", ".", " ", " "],                          // # → *. space
  "*": [".", " ", " ", " "],                          // * → . space
  ".": [" ", " ", " ", " "],                          // . → space
};
CuM = [".", " ", " ", " "];  // Default fallback
```

### Module Initialization (4064.js, line 1-7)

```js
// Module 4064.js, line 1-7
// resplit: esm module GA0 | exports: (none) | deps: e1A, TA0, xl, DL, XBH, XTL
var GA0 = O(async () => {
  e1A();   // React module init
  TA0();   // Art frame data init (VA0 array)
  xl();    // Additional module init
  await DL();  // Async dependency loading
  // ... then sets up animation variables
});
```

## Theme / Style Tokens

The FullScreenAnimation component renders its content with **bold text** (`<Text bold={true}>`) but does **not** apply theme colors from the palette system (module 3773.js / 0939.js). The logo uses the terminal's default foreground color with bold emphasis. No `o.border`, `o.primary`, or other theme tokens are consumed in the animation rendering.

The dissolve engine uses **Unicode block characters** for visual shading:
- `█` (U+2588 Full Block): densest
- `▓` (U+2593 Dark Shade)
- `▒` (U+2592 Medium Shade)
- `░` (U+2591 Light Shade): lightest before space
- `#`, `*`, `.`: ASCII fill characters

## Keyboard / Input Handling

The FullScreenAnimation component has **no keyboard input handling**. It is a passive animation:

- `exitOnCtrlC: false`: Ctrl+C does not exit during animation
- `patchConsole: false`: console.log is not suppressed
- The animation cannot be skipped by user input
- It terminates only via: (a) natural completion, or (b) the 10-second hard timeout

## Integration Points

- **tui-theme-system (3773.js, 0939.js, 3910.js)**: The animation does NOT consume theme tokens. It renders in terminal default colors with bold. However, the main app shell (4065.js) that invokes `CA0()` extensively uses theme tokens (`o.text.muted`, `o.border`, etc.) for the post-animation TUI.
- **tui-ink-core (2228.js)**: The animation uses Ink's `useStdout` hook (`Is()`), JSX runtime (`LXL.jsxDEV`), and Ink primitives (`a` = Box, `u` = Text). It renders as a standalone Ink app via `Sb()` (Ink's `render()` function).
- **tui-react-runtime (2206.js, 2313.js)**: Uses React hooks: `useState`, `useEffect`, `useRef`, `useCallback`, `useMemo`.
- **tui-ansi-terminal-utils**: Uses ANSI escape sequences `Tv.cursorHide` and `Tv.cursorShow` to toggle cursor visibility during animation. `iW()` clears the terminal after animation completes.
- **tui-renderer-main-loop**: The animation is the first thing rendered during TUI startup. After `CA0()` resolves, the main app shell `zuM` (line 4020-4025) is rendered, which is the full TUI application.

## Implementation Notes

### Animation Component
The `FullScreenAnimation` component is self-contained and portable:
1. **Dependencies**: React + Ink runtime, terminal stdout
2. **Props**: `onComplete` callback, `fps` (default 20)
3. **Art assets**: Replace `VA0` array with custom ASCII art frames
4. **Dissolve**: The `KuM` character map can be customized for different dissolve effects

### Key Timing Constants
- Frame interval: `Math.max(20, Math.floor(1000 / fps))` ms: default 50ms (20 FPS)
- Dissolve step delay: 120ms per step
- Dissolve steps: 4 (constant `quM`)
- Post-dissolve delay: 100ms before `onComplete`
- Hard timeout: 10 seconds (in `CA0()`)

### Integration Pattern
```js
// Recommended porting pattern:
async function showSplashScreen() {
  const { FullScreenAnimation } = await import('./animation-module');
  return new Promise((resolve) => {
    const instance = ink.render(
      React.createElement(FullScreenAnimation, {
        onComplete: resolve,
        fps: 20
      }),
      { exitOnCtrlC: false }
    );
    setTimeout(resolve, 10000); // safety timeout
  });
}
```

## Module Reference

| Module | Lines | Content |
|--------|-------|---------|
| 4064.js | 1-7 | Module GA0 initialization, dependency imports |
| 4064.js | 6-1148 | `VA0` array: 10+ ASCII art frames of robot/droid logo |
| 4064.js | 21-36 | `KuM` character dissolve map, `CuM` fallback |
| 4064.js | 1150-1151 | `XA0` exports: `FullScreenAnimation → OuM` |
| 4064.js | 1152-1166 | `YuM()`: dissolve character replacement function |
| 4064.js | 1168-1255 | `OuM()`: FullScreenAnimation React component |
| 4064.js | 1256-1262 | Module-level variables: `jF, LXL, _i, wuM, KuM, CuM, quM=4` |
| 4065.js | 4030-4057 | `CA0()`: boot animation orchestrator |
| 4065.js | 4040-4057 | Animation invocation: dynamic import → Ink render → 10s timeout |
| 4065.js | 1 | Module header (coordinator) |
