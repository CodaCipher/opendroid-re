# tui-panel-theme-selector: TUI Architecture Notes

## Overview

The ThemeSelector is not a standalone panel but an integrated setting within the Settings panel (4025.js). It provides a simple binary toggle between "Dark" and "Light" terminal color modes, displayed as a list item in the Settings panel under the "Preferences" section. When the user activates this setting item, the `terminalColorMode` value is toggled between `"dark"` and `"light"`, persisted via the SettingsService (0939.js), and a local React state update is triggered. The actual theme palette resolution occurs through a function chain: `MFI()` registers a settings getter, `lfH()` resolves the current mode, and `HHH()` returns the appropriate palette object (`BlA` for dark, `UCI` for light) which is exported as module-level variable `o` consumed by all UI components.

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 4025.js | ~10.5 KB | Settings panel: contains theme color mode toggle as a settings item | app |
| 0939.js | ~20.7 KB | SettingsService: `getTerminalColorMode()`/`setTerminalColorMode()` persistence | app |
| 2102.js | ~0.5 KB | Color mode resolver: `lfH()` reads settings getter, returns "dark"/"light" | app |
| 3151.js | ~0.3 KB | Theme resolution: `HHH()` returns palette based on color mode | app |
| 2312.js | ~5 KB | Theme token definitions: dark (BlA) and light (UCI) palette objects | app |
| 3873.js | ~3 KB | Panel wrapper component (`AI`): border/title container for all panels | app |
| 3875.js | ~2 KB | `wD` hook: selectable item list with keyboard navigation | app |
| 4071.js | ~large | TUI initialization: registers `MFI()` callback for theme resolution | app |

## Architecture

### Theme Selection Flow

The theme selection is **not a dedicated panel** but a single toggle item within the broader Settings panel. The architecture consists of:

1. **UI Layer**: Settings panel (4025.js) renders a scrollable list of settings items. The "Terminal color mode" item shows current value (Dark/Light) and toggles on activation.
2. **Selection Hook**: `wD` hook (3875.js) manages keyboard-driven navigation (↑↓ arrows, j/k) and selection (Enter) across all settings items.
3. **Panel Wrapper**: `AI` component (3873.js) provides the bordered container with title, help text, and consistent styling.
4. **Persistence Layer**: SettingsService (0939.js) handles `getTerminalColorMode()` and `setTerminalColorMode()` to read/write the setting.
5. **Resolution Chain**: On TUI startup (4071.js), `MFI(() => TvH.getTerminalColorMode())` registers the getter. The resolver chain `lfH() → HHH() → o` resolves the palette.

### Theme Application Flow

```
User presses Enter on "Terminal color mode" setting item
    ↓
4025.js: action() callback fires
    ↓
bH() reads Y.general?.terminalColorMode (local React state)
    ↓
Z = bH() === "light" ? "dark" : "light"  (toggle)
    ↓
vA().setTerminalColorMode(Z)  → 0939.js: this.updateSettings({general: {terminalColorMode: Z}})
    ↓
y({terminalColorMode: Z})  → React setState, re-renders settings list
    ↓
[On TUI restart] MFI() callback reads new mode → lfH() → HHH() → palette `o`
```

**Important**: Theme changes are persisted immediately but **require TUI restart** to take visual effect, because the palette object `o` is resolved at module initialization time, not reactively through React state.

### Settings Panel Layout

The Settings panel uses a flat list of items organized by section headers:

```
┌─ Settings ──────────────────────────────────────────────────────┐
│                                                                  │
│ Session Defaults                                                 │
│   > Default model: Claude Sonnet 4                               │
│     Default reasoning level: High                                │
│     Default interaction mode: Auto                               │
│     Default autonomy level: Off                                  │
│     Default spec mode model: Same as main                        │
│                                                                  │
│ Preferences                                                      │
│     Diff display mode: GitHub                                    │
│  >  Terminal color mode: Dark              ← Theme toggle here  │
│     Todo display mode: Pinned above input                        │
│     ... more settings ...                                        │
│                                                                  │
│ Use ↑↓ to navigate, Enter to select, Esc or Q to go back        │
└──────────────────────────────────────────────────────────────────┘
```

### Component Hierarchy

```
AI (3873.js): Panel wrapper
  └─ Box (borderStyle="round", borderColor=o.border)
       ├─ Header: "Settings" title
       └─ Children: hH.map() → list items
            ├─ Headers (type="header"): section labels
            └─ Items (type="item"):
                 ├─ "Terminal color mode" → toggle action
                 ├─ Other settings...
                 └─ Selected item prefixed with "> "
```

## Key Components

### 4025.js: Settings Panel (lines 11-170, theme-specific)

The `ZeD` component is the main Settings panel. Key theme-related code:

```js
// Module 4025.js, line 40: Read current color mode from settings
var bH = () => Y.general?.terminalColorMode ?? "dark";

// Module 4025.js, lines 151-158: Theme toggle settings item
{
  id: "terminal-color-mode-setting",
  label: "Terminal color mode",
  value: bH() === "light" ? "Light" : "Dark",
  action: () => {
    let Z = bH() === "light" ? "dark" : "light";
    (vA().setTerminalColorMode(Z), y({ terminalColorMode: Z }));
  },
  type: "item",
}
```

The settings list is constructed as a `useMemo` array (lines 56-310) with items of type `"header"` and `"item"`. The theme item is under the "Preferences" header section.

```js
// Module 4025.js, lines 342-353: Selection state via wD hook
let { selectedIndex: qH } = wD({
  items: hH,
  initialIndex: NH(),
  isSelectable: (Z) => Z.type === "item" && !Z.disabled,
  onSelect: (Z) => {
    if (Z.type === "item" && !Z.disabled) Z.action();
  },
  onCancel: A,
  isActive: !L && !I && !M && !P && !W && !X && !w,
});
```

The rendering maps each item to either a header label or a selectable line:

```js
// Module 4025.js, lines 500-530: Item rendering with selection highlight
let UH = b === qH && Z.type === "item";  // is selected?
let eH = Z.disabled;
let _L = eH ? o.text.muted : UH ? o.primary : void 0;  // color logic
// Renders: "> Terminal color mode: Dark" (selected)
//       or "  Terminal color mode: Dark" (not selected)
```

### 3873.js: Panel Wrapper Component (`AI`)

```js
// Module 3873.js, lines 14-18: Panel wrapper props
function AI({
  title: H,
  children: A,
  helpText: L,
  showDefaultHelp: $ = true,
  width: I,
  minWidth: D = 78,
  marginTop: E = 1,
  paddingX: f = 1,
  paddingY: M = 0,
  pagination: U,
  headerRight: P,
})
```

Renders a bordered Box container with:
- `borderStyle: "round"` + `borderColor: o.border`
- Title bar with bold text
- Help text footer: `"Use ↑↓ to navigate, Enter to select, Esc or Q to go back"`
- Optional pagination info

```js
// Module 3873.js: Default help text constant
var CyM = "Use ↑↓ to navigate, Enter to select, Esc or Q to go back";
```

### 3875.js: Selection Hook (`wD`)

```js
// Module 3875.js, lines 8-14: Hook interface
function wD({
  items: H,
  onSelect: A,
  onCancel: L,
  initialIndex: $ = 0,
  isSelectable: I,
  wrapAround: D = false,
  additionalKeys: E = {},
  isActive: f = true,
  onIndexChange: M,
})
```

Returns `{ selectedIndex, selectedItem, setSelectedIndex }`. Internally manages keyboard input via `jL()` (useInput wrapper).

### 2102.js: Color Mode Resolver

```js
// Module 2102.js: Simple callback-based resolver
var ZpA = null;

function MFI(H) {
  ZpA = H;  // Register settings getter callback
}

function lfH() {
  if (ZpA) {
    if (ZpA() === "light") return "light";
  }
  return "dark";  // Default to dark
}
```

### 3151.js: Theme Resolution Function

```js
// Module 3151.js: Returns palette based on color mode
function HHH() {
  return lfH() === "light" ? UCI : BlA;
}
var o;  // Module-level theme palette object
```

### 0939.js: SettingsService (theme persistence, lines 865-870)

```js
// Module 0939.js, lines 865-870
getTerminalColorMode() {
  return this.settings.general?.terminalColorMode ?? "dark";
}
setTerminalColorMode(H) {
  this.updateSettings({ general: { terminalColorMode: H } });
}
```

### 4071.js: TUI Initialization (line 1464)

```js
// Module 4071.js, line 1464: Registers theme getter at startup
MFI(() => TvH.getTerminalColorMode());
```

## Theme / Style Tokens

The theme selector itself uses the following theme tokens for rendering:

| Token | Value (Dark) | Usage in Settings |
|-------|-------------|-------------------|
| `o.border` | `#888888` | Panel border (`borderColor`) |
| `o.primary` | `#d56a26` | Selected item text color |
| `o.text.muted` | `#80756f` | Disabled items + help text |
| `o.text.secondary` | `#b3a9a4` | Section header labels (bold) |

The two theme palettes being toggled between are:

| Token | Dark (`BlA`) | Light (`UCI`) |
|-------|-------------|---------------|
| `primary` | `#d56a26` | `#F27B2F` |
| `border` | `#888888` | `#9B8E87` |
| `text.primary` | `#f2f0f0` | `#000000` |
| `text.secondary` | `#b3a9a4` | `#59514D` |
| `text.muted` | `#80756f` | `#665C58` |
| `text.user` | `#EBB28C` | `#BC4B00` |
| `success` | `green` | `#5B8E63` |
| `error` | `#ef4444` | `#E54048` |

## Keyboard / Input Handling

Theme selection inherits keyboard handling from the Settings panel's `wD` selection hook (3875.js):

| Key | Action |
|-----|--------|
| `↑` / `k` | Move selection up one item |
| `↓` / `j` | Move selection down one item |
| `Enter` | Activate selected item (toggle theme if on color mode item) |
| `Escape` / `q` | Close Settings panel (return to previous view) |

The hook skips non-selectable items (headers, disabled items) when navigating. The `isActive` flag is `false` when a sub-panel (model selector, reasoning selector, etc.) is open.

## Integration Points

- **tui-theme-system**: Direct dependency: the theme toggle in Settings persists `terminalColorMode`, which feeds into the theme resolution chain (`MFI` → `lfH` → `HHH` → `o`). The Settings panel itself uses theme tokens (`o.border`, `o.primary`, `o.text.muted`) for rendering.
- **tui-renderer-main-loop**: The TUI initialization in 4071.js registers `MFI(() => TvH.getTerminalColorMode())` to establish the theme resolution callback at startup.
- **tui-ink-core**: Panel rendering uses Ink's `Box` component with `borderStyle: "round"` and `borderColor: o.border` for the Settings container.
- **All panel features**: Every TUI panel imports the theme palette object `o` (3151.js) and uses its tokens for coloring. A theme mode change affects all panels on next TUI restart.
- **tui-logo-animation**: Uses theme colors for FullScreenAnimation art rendering; will reflect new palette on restart.
- **tui-status-bar**: Consumes `o.text.muted` and `o.primary` for status indicators.

## Implementation Notes

To implement a ThemeSelector in a custom TUI:

1. **Simple toggle approach**: Add a settings item with binary toggle (dark/light):
   ```js
   {
     id: "terminal-color-mode-setting",
     label: "Terminal color mode",
     value: currentMode === "light" ? "Light" : "Dark",
     action: () => {
       const newMode = currentMode === "light" ? "dark" : "light";
       settingsService.setTerminalColorMode(newMode);
       setLocalState({ terminalColorMode: newMode });
     },
     type: "item",
   }
   ```

2. **Panel wrapper**: Use a reusable bordered container with consistent styling:
   ```jsx
   <Box borderStyle="round" borderColor={theme.border} paddingX={1}>
     <Text bold>{title}</Text>
     {children}
   </Box>
   ```

3. **Selection hook**: Implement a `useSelectableItems` hook that handles keyboard navigation (↑↓/jk), Enter to select, Escape to close. Returns `{ selectedIndex }` and renders items with selection indicator (`"> "` prefix) and color highlighting.

4. **Persistence**: Store color mode preference in a settings file. Read on startup to resolve the correct palette before rendering.

5. **Theme resolution**: Use a simple function chain (no React context needed):
   ```js
   const palette = getColorMode() === "light" ? lightPalette : darkPalette;
   ```

6. **Restart requirement**: If using module-level palette resolution (not reactive), communicate to users that theme changes take effect on restart.

## Module Reference

| Module | Lines | Key Content |
|--------|-------|-------------|
| 4025.js | 1-100 | Settings panel component, settings getters, state management |
| 4025.js | 40 | `bH()`: reads terminalColorMode from local state |
| 4025.js | 56-310 | `hH` useMemo: full settings item list including theme toggle |
| 4025.js | 151-158 | Theme toggle item definition (id, label, value, action) |
| 4025.js | 342-353 | `wD` hook usage: selection state for all settings items |
| 4025.js | 500-530 | Settings list rendering: selection highlight, colors |
| 0939.js | 865-870 | `getTerminalColorMode()` / `setTerminalColorMode()` persistence |
| 2102.js | 1-16 | Color mode resolver: `MFI()`, `lfH()`, `ZpA` callback |
| 3151.js | 1-9 | Theme resolution: `HHH()` → palette `o` |
| 2312.js | 1-38 | Dark (BlA) and Light (UCI) palette definitions |
| 3873.js | 14-138 | Panel wrapper `AI`: bordered container with title/help |
| 3873.js | 138 | Help text constant: `CyM` |
| 3875.js | 8-101 | `wD` selection hook: keyboard nav, selection state |
| 4071.js | 1464 | TUI init: `MFI(() => TvH.getTerminalColorMode())` |
