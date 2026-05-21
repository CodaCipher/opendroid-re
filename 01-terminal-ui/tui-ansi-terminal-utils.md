# tui-ansi-terminal-utils: TUI Architecture Notes

## Overview

The ANSI/terminal utility layer provides three core capabilities for the OpenDroid TUI system: (1) a full **chalk-compatible color system** supporting ANSI basic, 256-color, and 16m (truecolor) modes, with automatic terminal capability detection; (2) a comprehensive **ANSI escape sequence library** (ansi-escapes) for cursor movement, screen clearing, scrolling, and terminal control; and (3) an **ANSI-aware word wrapping engine** that preserves ANSI codes while wrapping text to terminal width. Together these modules form the low-level terminal output foundation that Ink and the theme system build upon.

## Module Map

| Module   | Size    | Role                                      | Vendor/App |
|----------|---------|-------------------------------------------|------------|
| 2549.js  | 3.8 KB  | ANSI styles definition + color support detection (vendor chalk/ansi-styles + supports-color) | Vendor |
| 2249.js  | 3.8 KB  | Duplicate copy of ANSI styles + color support detection | Vendor |
| 2548.js  | 4.6 KB  | ANSI styles builder: color conversion (rgb/hex↔ansi), escape code assembly | Vendor (app bundle) |
| 2551.js  | 3.0 KB  | Chalk runtime: prototype-based style chaining, level-based ANSI generation | Vendor (app bundle) |
| 2210.js  | 4.9 KB  | ANSI escape sequences library (cursor, erase, scroll, screen, links, images) | Vendor |
| 2225.js  | 4.6 KB  | ANSI-aware word wrap engine with style preservation | Vendor |
| 2539.js  | 8.5 KB  | Mission tool infrastructure (not ANSI-related, misclassified) | App |
| 2541.js  | 3.7 KB  | Mission/worker orchestration prompts (not ANSI-related, misclassified) | App |
| 2540.js  | 4.2 KB  | Mission proposal tool (not ANSI-related, misclassified) | App |
| 2547.js  | 4.0 KB  | Task tool manager / file watcher (not ANSI-related, misclassified) | App |

## Architecture

```
┌─────────────────────────────────────────────────┐
│           Application Layer (Ink, Theme)         │
├─────────────────────────────────────────────────┤
│  2551.js: Chalk Prototype (style chaining)      │
│  ├─ level detection → selects ANSI mode          │
│  ├─ .rgb() / .hex() / .ansi256() color methods   │
│  └─ .visible, .bold, .dim, modifiers             │
├─────────────────────────────────────────────────┤
│  2548.js: ANSI Styles Builder                   │
│  ├─ yUf(): builds TP styles object               │
│  ├─ rgbToAnsi256 / hexToRgb / ansi256ToAnsi      │
│  └─ RNI / hNI / jNI: ANSI code generators        │
├─────────────────────────────────────────────────┤
│  2549.js / 2249.js: Style Definitions            │
│  ├─ TP/MP: modifier/color/bgColor code maps       │
│  └─ uUf/WwI: terminal color level detection       │
├─────────────────────────────────────────────────┤
│  2210.js: ANSI Escape Sequences (cursor/screen) │
│  2225.js: ANSI-aware Word Wrap Engine            │
└─────────────────────────────────────────────────┘
```

### Color Level Detection Flow
1. Check `FORCE_COLOR` env var → override level (0-3)
2. Check CLI flags: `--color=16m`, `--color=256`, `--color=truecolor`
3. Platform checks: Win32 version check (10.0+ → level 2/3)
4. CI detection: GitHub Actions → 3, Travis/AppVeyor → 1
5. Terminal env: `COLORTERM=truecolor` → 3, `TERM=xterm-kitty` → 3
6. `TERM_PROGRAM`: iTerm (v3+ → 3), Apple_Terminal → 2
7. Fallback regex: `*-256*` → 2, `screen|xterm|vt100` → 1
8. Final: `COLORTERM` exists → 1, else default 0

### Chalk Style Chaining Architecture
- `FKH()` creates a chalk instance bound to detected stdout/stderr color level
- Prototype chain: `Object.setPrototypeOf(FKH.prototype, Function.prototype)`
- Lazy getters: each style property (bold, red, bgBlue, etc.) creates a closure that applies open/close ANSI codes
- Nested chaining via `NaH()` wraps new styler onto existing, building up ANSI sequences
- Color mode selection: `pNI = ["ansi", "ansi", "ansi256", "ansi16m"]` indexed by level

## Key Components

### Module 2549.js: ANSI Style Definitions + Color Support Detection

**ANSI Code Map (lines 1-60):**
```js
TP = {
  modifier: {
    reset: [0, 0], bold: [1, 22], dim: [2, 22],
    italic: [3, 23], underline: [4, 24], overline: [53, 55],
    inverse: [7, 27], hidden: [8, 28], strikethrough: [9, 29],
  },
  color: {
    black: [30, 39], red: [31, 39], green: [32, 39], yellow: [33, 39],
    blue: [34, 39], magenta: [35, 39], cyan: [36, 39], white: [37, 39],
    blackBright: [90, 39], gray: [90, 39], redBright: [91, 39],
    greenBright: [92, 39], yellowBright: [93, 39], blueBright: [94, 39],
    magentaBright: [95, 39], cyanBright: [96, 39], whiteBright: [97, 39],
  },
  bgColor: {
    bgBlack: [40, 49], bgRed: [41, 49], bgGreen: [42, 49],
    bgYellow: [43, 49], bgBlue: [44, 49], bgMagenta: [45, 49],
    bgCyan: [46, 49], bgWhite: [47, 49],
    bgBlackBright: [100, 49], bgGray: [100, 49], bgRedBright: [101, 49],
    bgGreenBright: [102, 49], bgYellowBright: [103, 49], bgBlueBright: [104, 49],
    bgMagentaBright: [105, 49], bgCyanBright: [106, 49], bgWhiteBright: [107, 49],
  },
}
```
Each entry `[open, close]` maps to `\x1B[{open}m` / `\x1B[{close}m`.

**Color Support Detection (lines 80-130):**
```js
function uUf(H, { streamIsTTY: A, sniffFlags: L = true } = {}) {
  // Level 0 = no color, 1 = basic ANSI, 2 = 256-color, 3 = 16m truecolor
  if (VP.TERM === "dumb") return D;
  if (zdA.platform === "win32") {
    let E = YIf.release().split(".");
    if (Number(E[0]) >= 10 && Number(E[2]) >= 10586)
      return Number(E[2]) >= 14931 ? 3 : 2;
    return 1;
  }
  if (UP.COLORTERM === "truecolor") return 3;
  if (VP.TERM === "xterm-kitty") return 3;
  // ... more terminal detection
}
```

### Module 2548.js: ANSI Styles Builder (Color Conversion)

**ANSI Code Generators (lines ~80-90):**
```js
var RNI = (H = 0) => (A) => `\x1B[${A + H}m`;           // basic ANSI
var hNI = (H = 0) => (A) => `\x1B[${38 + H};5;${A}m`;   // 256-color
var jNI = (H = 0) => (A, L, $) => `\x1B[${38 + H};2;${A};${L};${$}m`; // truecolor
```
- `RNI()`: generates `\x1B[30m`-style codes (fg), `RNI(10)` for bg (`\x1B[40m`)
- `hNI()`: generates `\x1B[38;5;Nm` (fg) / `\x1B[48;5;Nm` (bg)
- `jNI()`: generates `\x1B[38;2;R;G;Bm` (fg) / `\x1B[48;2;R;G;Bm` (bg)

**RGB to ANSI-256 Conversion:**
```js
rgbToAnsi256(A, L, $) {
  if (A === L && L === $) {  // Grayscale
    if (A < 8) return 16;
    if (A > 248) return 231;
    return Math.round(((A - 8) / 247) * 24) + 232;
  }
  return 16 + 36 * Math.round((A / 255) * 5)
            + 6 * Math.round((L / 255) * 5)
            + Math.round(($ / 255) * 5);
}
```

**ANSI-256 to Basic ANSI Conversion:**
```js
ansi256ToAnsi(A) {
  if (A < 8) return 30 + A;
  if (A < 16) return 90 + (A - 8);
  // ... convert 256-color to nearest basic ANSI color
}
```

**Hex to RGB Conversion:**
```js
hexToRgb(A) {
  let L = /[a-f\d]{6}|[a-f\d]{3}/i.exec(A.toString(16));
  let [$] = L;
  if ($.length === 3) $ = [...$].map((D) => D + D).join("");
  let I = Number.parseInt($, 16);
  return [(I >> 16) & 255, (I >> 8) & 255, I & 255];
}
```

### Module 2551.js: Chalk Prototype (Style Chaining)

**Chalk Instance Creation (lines 1-65):**
```js
pNI = ["ansi", "ansi", "ansi256", "ansi16m"];  // Level → ANSI mode mapping
mUf = ["rgb", "hex", "ansi256"];                 // Supported color input modes

// Color methods are lazy getters on prototype:
v9H[H] = {
  get() {
    let { level: L } = this;
    return function (...$) {
      let I = ftA(EtA(H, pNI[L], "color", ...$), rZ.color.close, this[x9H]);
      return NaH(this, I, this[GKH]);
    };
  },
};
// Background: "bg" + capitalize first letter
let A = "bg" + H[0].toUpperCase() + H.slice(1);
```

**Level Property:**
```js
level: {
  enumerable: true,
  get() { return this[DtA].level; },
  set(H) { this[DtA].level = H; },
}
```

**Global Instances:**
- `iUf = FKH()`: default chalk (stdout level)
- `g48 = FKH({ level: cNI ? cNI.level : 0 })`: stderr chalk
- `q$ = iUf`: exported default

### Module 2210.js: ANSI Escape Sequences Library

**Platform Detection (lines 1-40):**
```js
xC = globalThis.window?.document !== void 0   // browser
XdU = globalThis.process?.versions?.node       // Node.js
YdU = ... // macOS detection
OdU = ... // Windows detection
```

**Cursor Control Functions (lines ~90-110):**
```js
ZLf = (H, A) => {  // cursorTo(x, y)
  if (typeof H !== "number") throw TypeError("The `x` argument is required");
  if (typeof A !== "number") return VM + (H + 1) + "G";  // \x1B[{x+1}G
  return VM + (A + 1) + $7H + (H + 1) + "H";             // \x1B[{y+1};{x+1}H
};
zLf = (H, A) => {  // cursorMove(dx, dy)
  let L = "";
  if (H < 0) L += VM + -H + "D";  // \x1B[N D (left)
  else if (H > 0) L += VM + H + "C";  // \x1B[N C (right)
  if (A < 0) L += VM + -A + "A";  // \x1B[N A (up)
  else if (A > 0) L += VM + A + "B";  // \x1B[N B (down)
  return L;
};
t7I = (H = 1) => VM + H + "A";  // cursorUp
NLf = (H = 1) => VM + H + "B";  // cursorDown
RLf = (H = 1) => VM + H + "C";  // cursorForward
hLf = (H = 1) => VM + H + "D";  // cursorBackward
```

**Screen Operations:**
- `FrH`: eraseScreen
- `s7I`: eraseLine (`\x1B[2K`)
- `iLf = "\x1Bc"`: clearTerminal
- `uLf(H)`: eraseLines: loops erasing N lines moving up

**Hyperlink Support:**
```js
H$f = (H, A) => {  // link(text, url)
  let L = GMH(`\x1B]8;;${A}\x07`);
  let $ = GMH(`\x1B]8;;\x07`);
  return L + H + $;
}
```

**Image Support (iTerm protocol):**
```js
A$f = (H, A = {}) => {  // image(data, options)
  let L = `\x1B]1337;File=inline=1`;
  if (A.width) L += `;width=${A.width}`;
  if (A.height) L += `;height=${A.height}`;
  let $ = Buffer.from(H);
  return GMH(L + `;size=${$.byteLength}:` + $.toString("base64") + \x07);
}
```

### Module 2225.js: ANSI-Aware Word Wrap Engine

**ANSI-Preserving Wrap Function (lines ~40-80):**
```js
C$f = (H, A, L = {}) => {  // wrapAnsiString(text, columns, options)
  // Splits text into words, tracks ANSI state across wrapping
  // Handles hard wrap vs word wrap modes
  // Preserves open ANSI codes across line breaks
  // Re-applies active styles after newline
};
```

The wrap engine:
1. Splits text into words, measures each word's visual width (stripping ANSI)
2. Fills lines up to the column limit
3. When a word exceeds remaining space, starts a new line
4. Tracks active ANSI codes (foreground color, hyperlinks) and re-emits them after each line break
5. Handles hard-wrap mode where long words are broken mid-character
6. Trims trailing whitespace per line while respecting `trim: false` option

### Module 2249.js: Duplicate ANSI Styles + Color Detection

This is an identical copy of module 2549.js with different minified variable names (`MP` vs `TP`, `vC` vs `aC`, `UP` vs `VP`, `zIf` vs `uUf`, `WwI` vs `kNI`). Contains the same ANSI style code map and color support detection logic. Likely bundled twice due to different import chains.

## Theme / Style Tokens

### ANSI Modifier Codes (from 2549.js/2249.js)
| Modifier      | Open Code | Close Code |
|---------------|-----------|------------|
| reset         | 0         | 0          |
| bold          | 1         | 22         |
| dim           | 2         | 22         |
| italic        | 3         | 23         |
| underline     | 4         | 24         |
| overline      | 53        | 55         |
| inverse       | 7         | 27         |
| hidden        | 8         | 28         |
| strikethrough | 9         | 29         |

### ANSI Foreground Colors
| Color          | Open | Close |
|----------------|------|-------|
| black          | 30   | 39    |
| red            | 31   | 39    |
| green          | 32   | 39    |
| yellow         | 33   | 39    |
| blue           | 34   | 39    |
| magenta        | 35   | 39    |
| cyan           | 36   | 39    |
| white          | 37   | 39    |
| blackBright/gray| 90  | 39    |
| redBright      | 91   | 39    |
| greenBright    | 92   | 39    |
| yellowBright   | 93   | 39    |
| blueBright     | 94   | 39    |
| magentaBright  | 95   | 39    |
| cyanBright     | 96   | 39    |
| whiteBright    | 97   | 39    |

### ANSI Background Colors
Same pattern offset by 10: bgBlack=40, bgRed=41, ..., bgWhite=47, bgBlackBright=100, ..., bgWhiteBright=107. Close code: 49.

### Color Levels
| Level | Mode     | Capability        |
|-------|----------|-------------------|
| 0     | none     | No color support  |
| 1     | ansi     | Basic 16 colors   |
| 2     | ansi256  | 256-color palette |
| 3     | ansi16m  | Truecolor (RGB)   |

## Keyboard / Input Handling

N/A: This feature covers low-level ANSI/terminal utilities. No keyboard/input handling at this layer. Input handling is managed by the Ink core (useInput) documented in tui-ink-core.

## Integration Points

- **tui-theme-system**: Theme tokens and color values from the palette module will consume the chalk color system (2551.js / 2548.js) for applying colors to terminal output. The theme system's color values are rendered via these ANSI escape sequences.
- **tui-ink-core**: Ink's `<Text>` component uses the chalk/ANSI system to apply foreground/background colors and text styles. The render pipeline outputs ANSI escape sequences to the terminal.
- **tui-renderer-main-loop**: The main render loop uses ANSI escape sequences from 2210.js for cursor positioning, screen clearing, and viewport management during re-renders.
- **tui-status-bar / tui-panel-***: All panel components that render colored/styled text transitively depend on this ANSI layer through Ink's Text component.
- **Module duplication**: 2249.js is a duplicate of 2549.js: both contain identical ANSI style definitions and color support detection. This suggests the bundler included the same vendor module twice via different import paths.

## Implementation Notes

When porting the TUI to OpenDroid:

1. **Color System**: Use the `chalk` npm package directly (v5+). The modules 2548-2551 implement a chalk-compatible API. No need to reverse-engineer the color conversion: use chalk's built-in `.rgb()`, `.hex()`, `.ansi256()` methods.

2. **Color Level Detection**: Use the `supports-color` npm package. Modules 2549.js/2249.js implement its logic. Key: check `FORCE_COLOR`, `COLORTERM`, `TERM` env vars, plus platform-specific version checks (Windows 10+).

3. **ANSI Escapes**: Use the `ansi-escapes` npm package. Module 2210.js provides cursor movement (`cursorTo`, `cursorMove`), erase operations, scrolling, hyperlink (`\x1B]8;;url\x07`), and iTerm image support.

4. **Word Wrap**: Use the `wrap-ansi` npm package. Module 2225.js implements ANSI-aware word wrapping that preserves color codes across line breaks.

5. **Style Definitions**: The full ANSI style code map (modifiers, fg/bg colors) is a standard reference. No vendor lock-in: it maps directly to ECMA-48 / ANSI X3.64 escape codes.

6. **Deduplication**: The duplicate modules (2549.js ≡ 2249.js, 2548.js ≡ part of 2549.js builder) can be consolidated to a single import.

## Module Reference

| Module   | Lines Analyzed | Key Content                                          |
|----------|----------------|------------------------------------------------------|
| 2549.js  | 1-130 (full)   | ANSI style map `TP`, color level detection `uUf()`, env checks |
| 2249.js  | 1-130 (full)   | Duplicate of 2549.js with different var names        |
| 2548.js  | 1-130 (full)   | `yUf()` builder, `RNI/hNI/jNI` generators, color conversion functions |
| 2551.js  | 1-65 (partial) | Chalk prototype `FKH()`, level-based style chaining, global instances |
| 2210.js  | 1-170 (full)   | ANSI escape sequences, cursor/erase/scroll/link/image functions |
| 2225.js  | 1-100 (full)   | `C$f()` ANSI-aware word wrap, style preservation across line breaks |
| 2539.js  | 1-20 (header)  | EndFeatureRun tool: not ANSI-related                |
| 2541.js  | 1-60 (header)  | Worker prompt templates: not ANSI-related           |
| 2540.js  | 1-10 (header)  | ProposeMission tool: not ANSI-related               |
| 2547.js  | 1-10 (header)  | TaskToolManager: not ANSI-related                   |
