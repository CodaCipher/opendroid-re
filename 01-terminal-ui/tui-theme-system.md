# tui-theme-system: TUI Architecture Notes

## Overview

The OpenDroid TUI theme system is a lightweight, settings-driven color palette that supports **dark** and **light** terminal color modes. Unlike traditional React context-based theme providers, this system uses a simple function-based resolution: a global `HHH()` function reads the `terminalColorMode` setting (via `lfH()`) and returns one of two hardcoded palette objects: `BlA` (dark) or `UCI` (light). The resolved palette is exported as a module-level variable `o`, which every UI component imports directly to access color tokens like `o.border`, `o.primary`, `o.text.muted`, etc. Border rendering is handled by Ink's built-in border pipeline (2252.js), which reads `borderStyle` and `borderColor` from Yoga-computed layout nodes and renders box-drawing characters with ANSI color codes.

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 2312.js | ~5 KB | Core theme token definitions: dark (BlA) and light (UCI) palettes + key constants | app |
| 2102.js | ~0.5 KB | Color mode resolver: `lfH()` reads settings getter, returns "dark"/"light" | app |
| 3151.js | ~0.3 KB | Theme resolution function: `HHH()` returns palette based on color mode | app |
| 2252.js | ~3 KB | Ink border rendering: reads `borderStyle`/`borderColor`, draws box characters | app |
| 2253.js | ~1 KB | Ink background color rendering: fills backgroundColor behind content | app |
| 4025.js | ~10.5 KB | Settings panel: includes terminalColorMode toggle UI | app |
| 0939.js | ~20.7 KB | SettingsService: `getTerminalColorMode()`/`setTerminalColorMode()` | app |

## Architecture

### Theme Resolution Pipeline

```
Settings (terminalColorMode: "dark"|"light")
    ↓
MFI(callback) registers settings getter in 2102.js
    ↓
lfH() calls callback → returns "light" or "dark" (2102.js)
    ↓
HHH() = lfH() === "light" ? UCI : BlA  (3151.js)
    ↓
o = HHH(): module-level theme palette object
    ↓
Every component imports o → reads o.border, o.primary, o.text.muted, etc.
```

### No Context Provider for Theme

Unlike typical React theme systems, there is **no React context or useTheme hook**. The theme is resolved at module initialization time via a settings callback. This means:

1. **Theme changes require TUI restart**: the palette is not reactive through React state.
2. **All components share the same `o` variable**: imported as a side-effect of the bundled ESM modules.
3. **The callback `MFI()` registers `ZpA`**: a function that reads `terminalColorMode` from `SettingsService`.

### Border Rendering Pipeline (Ink)

```
Component JSX: <Box borderStyle="round" borderColor={o.border}>
    ↓
Yoga layout computes width/height
    ↓
2252.js gIf() reads style.borderStyle + style.borderColor
    ↓
Looks up box-drawing chars from wwI.default[style] (e.g., "round")
    ↓
Applies ANSI color codes via Zh() (chalk wrapper)
    ↓
Writes border chars to terminal output buffer
```

### Background Color Pipeline

```
Component JSX: <Box backgroundColor="#hex">
    ↓
2253.js reads style.backgroundColor + accounts for border insets
    ↓
Generates colored spaces (Zh + "background" mode)
    ↓
Writes to output buffer row by row
```

## Key Components

### 2312.js: Theme Token Definitions (lines 1-38)

Two complete color palettes defined as object literals:

```js
// Module 2312.js, lines 3-19: Dark palette
BlA = {
  primary: "#d56a26",
  border: "#888888",
  success: "green",
  error: "#ef4444",
  warning: "#fbbf24",
  spec: "#b5b1fc",
  agi: "#22c55e",
  text: { primary: "#f2f0f0", secondary: "#b3a9a4", muted: "#80756f", user: "#EBB28C" },
  diff: {
    added: { text: "greenBright" },
    removed: { text: "red" },
    border: "#555555",
    lineNumber: "#80756f",
    header: "#569CD6",
    unchanged: { text: "#80756f", dimText: "#665C58" },
  },
}

// Module 2312.js, lines 21-38: Light palette
UCI = {
  primary: "#F27B2F",
  border: "#9B8E87",
  success: "#5B8E63",
  error: "#E54048",
  warning: "#F0A330",
  spec: "#157AC6",
  agi: "#16a34a",
  text: { primary: "#000000", secondary: "#59514D", muted: "#665C58", user: "#BC4B00" },
  diff: { /* same structure as dark */ },
}

// Module 2312.js, line 39: Default export
n8 = BlA;  // Default palette reference (dark)
```

Also contains key constants (line 40+):
```js
// Module 2312.js, lines 40-80: Terminal key constants
SI = {
  ESC: "\x1B", BACKSPACE: "\x7F", DELETE: "\x7F",
  FORWARD_DELETE: "\x1B[3~", CTRL_W: "\x17",
  NEWLINE: "\n", CARRIAGE_RETURN: "\r", TAB: "\t",
  // ... full terminal escape sequence map for arrow keys, delete, paste, etc.
}

// Module 2312.js, lines ~80-90: UI constants
S2 = {
  INPUT_WIDTH: 60, MAX_SUGGESTIONS: 6,
  VIEWPORT_WIDTH: 80, VIEWPORT_HEIGHT: 24,
  ESC_TIMEOUT_MS: 50, FILE_SUGGESTIONS_DEBOUNCE_MS: 120,
}
```

### 2102.js: Color Mode Resolver (lines 1-16)

```js
// Module 2102.js, lines 7-16
var ZpA = null;  // Settings getter callback

function MFI(H) {
  ZpA = H;  // Register settings getter
}

function lfH() {
  if (ZpA) {
    if (ZpA() === "light") return "light";
  }
  return "dark";  // Default to dark
}
```

### 3151.js: Theme Resolution (lines 6-9)

```js
// Module 3151.js, lines 6-9
function HHH() {
  return lfH() === "light" ? UCI : BlA;
}
var o;  // Module-level theme object: assigned at render time
```

### 2252.js: Border Rendering (lines 5-45)

```js
// Module 2252.js, lines 7-45: Border rendering function
var gIf = (H, A, L, $) => {
  if (L.style.borderStyle) {
    // Get box-drawing characters for border style
    let E = typeof L.style.borderStyle === "string"
      ? wwI.default[L.style.borderStyle]  // e.g., "round", "bold", "classic"
      : L.style.borderStyle;

    // Per-side color resolution (fallback to borderColor)
    let f = L.style.borderTopColor ?? L.style.borderColor;
    let M = L.style.borderBottomColor ?? L.style.borderColor;
    let U = L.style.borderLeftColor ?? L.style.borderColor;
    let P = L.style.borderRightColor ?? L.style.borderColor;

    // Per-side dim support
    let B = L.style.borderTopDimColor ?? L.style.borderDimColor;
    // ... similar for other sides

    // Compose border characters with color
    let N = Zh(borderChars, f, "foreground");  // chalk wrapper
    // Write to output buffer
    $.write(H, A, N, { transformers: [] });
  }
};
```

### 2253.js: Background Color Rendering (lines 1-16)

```js
// Module 2253.js, lines 1-16
var cIf = (H, A, L, $) => {
  if (!L.style.backgroundColor) return;
  // Account for border insets
  let E = L.style.borderStyle && L.style.borderLeft !== false ? 1 : 0;
  // ... similar for other sides
  let W = Zh(" ".repeat(P), L.style.backgroundColor, "background");
  for (let V = 0; V < B; V++)
    $.write(H + E, A + M + V, W, { transformers: [] });
};
```

## Theme / Style Tokens

### Core Semantic Tokens (Dark Mode)

| Token | Value | Usage |
|-------|-------|-------|
| `primary` | `#d56a26` | Brand color, highlights, active items |
| `border` | `#888888` | Panel borders (default `borderColor`) |
| `success` | `green` | Success messages, completed states |
| `error` | `#ef4444` | Error messages, failed states |
| `warning` | `#fbbf24` | Warning messages, pending approvals |
| `spec` | `#b5b1fc` | Spec mode indicators |
| `agi` | `#22c55e` | AGI mode indicators, running processes |
| `text.primary` | `#f2f0f0` | Primary text color |
| `text.secondary` | `#b3a9a4` | Secondary/description text |
| `text.muted` | `#80756f` | Muted/meta text, file paths, hints |
| `text.user` | `#EBB28C` | User message text |

### Core Semantic Tokens (Light Mode)

| Token | Value | Usage |
|-------|-------|-------|
| `primary` | `#F27B2F` | Brand color (slightly brighter for light bg) |
| `border` | `#9B8E87` | Panel borders |
| `success` | `#5B8E63` | Success (darker green for contrast) |
| `error` | `#E54048` | Error messages |
| `warning` | `#F0A330` | Warning messages |
| `spec` | `#157AC6` | Spec mode (blue instead of purple) |
| `agi` | `#16a34a` | AGI mode |
| `text.primary` | `#000000` | Primary text (black on light bg) |
| `text.secondary` | `#59514D` | Secondary text |
| `text.muted` | `#665C58` | Muted text |
| `text.user` | `#BC4B00` | User message text |

### Diff-Specific Tokens (Shared)

| Token | Value | Usage |
|-------|-------|-------|
| `diff.added.text` | `greenBright` | Added lines in diff view |
| `diff.removed.text` | `red` | Removed lines in diff view |
| `diff.border` | `#555555` | Diff panel borders |
| `diff.lineNumber` | `#80756f` | Line numbers in diff |
| `diff.header` | `#569CD6` | File headers in diff |
| `diff.unchanged.text` | `#80756f` | Unchanged context lines |
| `diff.unchanged.dimText` | `#665C58` | Dimmed unchanged lines |

### Border Style Tokens

| Style | Value | Usage |
|-------|-------|-------|
| `borderStyle: "round"` | Most common | All panels (settings, branches, sessions, etc.) |
| `borderStyle: "classic"` | Used in terminal emulator | Terminal frame borders |
| `borderStyle: "bold"` | Error/info boxes | Error states |
| `borderStyle: "single"` | Used occasionally | Specific dialogs |

### UI Layout Constants

| Constant | Value | Usage |
|----------|-------|-------|
| `INPUT_WIDTH` | `60` | Default text input width |
| `MAX_SUGGESTIONS` | `6` | Max autocomplete suggestions |
| `VIEWPORT_WIDTH` | `80` | Default viewport width |
| `VIEWPORT_HEIGHT` | `24` | Default viewport height |
| `ESC_TIMEOUT_MS` | `50` | Escape key timeout |
| `FILE_SUGGESTIONS_DEBOUNCE_MS` | `120` | File suggestion debounce |

## Keyboard / Input Handling

No direct keyboard handling in the theme system itself. Theme switching is triggered via:
- **Settings panel** (4025.js): Toggle `terminalColorMode` between "Dark" and "Light"
- **SettingsService** (0939.js): `setTerminalColorMode("dark"|"light")` persists to settings

## Integration Points

- **tui-ink-core**: Border rendering (2252.js) and background fill (2253.js) are part of Ink's render pipeline, called during the commit phase of the reconciler.
- **tui-logo-animation**: FullScreenAnimation (4065.js) references theme colors for art rendering.
- **All panel features** (session-selector, permission-dialog, model-selector, theme-selector, mcp-manager, background-process): Every panel uses `borderStyle: "round"` + `borderColor: o.border` from the theme system.
- **tui-ansi-terminal-utils**: The chalk color system (`Zh()` function in 2252.js/2253.js) wraps ANSI escape sequences for applying theme colors to terminal output.
- **tui-react-runtime**: React's `useContext(fm)` in 2313.js provides terminal focus state (used alongside theme for conditional rendering).
- **tui-status-bar**: Uses `o.text.muted` for status indicators and `o.primary` for active state.
- **tui-composer-input**: Uses theme tokens for composer UI coloring.
- **tui-renderer-main-loop**: Registers the settings getter (`MFI()`) during TUI initialization, establishing the theme resolution callback chain.

## Implementation Notes

To replicate this theme system in a custom TUI:

1. **Define two palette objects** (dark/light) with the same token structure:
   - `primary`, `border`, `success`, `error`, `warning`, `spec`, `agi`
   - `text.primary`, `text.secondary`, `text.muted`, `text.user`
   - `diff.*` sub-tokens for diff view

2. **Implement a simple resolver**: no React context needed:
   ```js
   function getTheme() {
     return settings.terminalColorMode === "light" ? lightPalette : darkPalette;
   }
   ```

3. **Border rendering**: Use Ink's `<Box borderStyle="round" borderColor={theme.border}>`: the built-in pipeline handles box-drawing character composition and ANSI coloring.

4. **Theme switching**: Persist `terminalColorMode` in settings; re-read on TUI startup. Theme changes require TUI restart (not hot-swappable).

5. **Color format support**: Ink accepts CSS hex (`#rrggbb`), named colors (`"green"`, `"red"`), `rgb()`, and `ansi256()` formats.

## Module Reference

| Module | Lines | Key Content |
|--------|-------|-------------|
| 2312.js | 1-38 | Dark (BlA) and Light (UCI) palette definitions |
| 2312.js | 39 | Default export: `n8 = BlA` |
| 2312.js | 40-80 | Terminal key constants (SI) |
| 2312.js | ~80-90 | UI constants (S2) |
| 2102.js | 1-16 | Color mode resolver: `lfH()`, `MFI()` |
| 3151.js | 1-9 | Theme resolution: `HHH()` → `o` |
| 2252.js | 5-45 | Border rendering with borderColor/borderStyle |
| 2253.js | 1-16 | Background color fill rendering |
| 4025.js | 35-40 | terminalColorMode setting getter |
| 4025.js | 151-154 | terminalColorMode toggle UI |
| 0939.js | 865-870 | getTerminalColorMode/setTerminalColorMode |
| 3773.js | 1-1109 | xterm.js ThemeService (terminal emulator, separate from Ink theme) |
