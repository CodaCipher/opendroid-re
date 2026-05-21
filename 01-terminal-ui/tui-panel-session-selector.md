# tui-panel-session-selector: TUI Architecture Notes

## Overview

The SessionSelector panel is a full-screen overlay UI component that allows users to browse, search, and select from previous conversation sessions. It is implemented as a React Ink component (`UeD` in 4017.js) wrapped in a shared `PanelDialog` container (3873.js). The panel features three filterable views (Current Folder, All, Favorites), keyboard-driven navigation with arrow keys, inline session renaming via Ctrl+R, and real-time search filtering. Session data is fetched via a session service (1185.js) and managed through React state in the main TUI orchestrator (4065.js). The `/sessions` slash command (3835.js) serves as the primary entry point to trigger the panel.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 4017.js | ~12 KB | **SessionSelector panel component** (`UeD`): main UI rendering, keyboard handling, filter/search/rename logic | App |
| 4065.js | 47.2 KB | **TUI orchestrator**: session state management, loadSession, renameSession callbacks, panel visibility toggle | App |
| 3835.js | ~2 KB | **`/sessions` slash command**: triggers showSessionSelector with session data | App |
| 1185.js | ~20 KB | **Session service**: `getAllSessions`, `getAllNonEmptySessions`, session metadata (title, modifiedTime, isFavorite, isCurrentProject) | App |
| 3873.js | ~5 KB | **PanelDialog wrapper** (`AI`): shared bordered panel container with title, help text, pagination | App |
| 3862.js | ~1 KB | **`useInput` hook** (`jL`): stdin key event subscription for Ink components | App |
| 3861.js | ~10 KB | **TextInput component** (`C8`): single-line text input with cursor, selection, paste support | App |
| 3381.js | ~1 KB | **`useWindowSize` hook** (`w8`): terminal resize detection, provides width/height | App |

## Architecture

### Component Hierarchy

```
App (4065.js)
  ├── state: aA (sessions list): useState(null)
  ├── state: P$ (mode: "browse" | "rename"): useState("browse")
  │
  ├── SlashCommand: "sessions" (3835.js)
  │     └── calls showSessionSelector(WL) with session array
  │
  └── Conditional Render:
        if (aA !== null) →
          PanelDialog (AI, 3873.js) wrapper
            └── SessionSelector (UeD, 4017.js)
                  ├── Filter tabs: CurrentFolder | All | Favorites
                  ├── Search: TextInput (C8, 3861.js) for filtering
                  ├── Session list: mapped rows with columns
                  └── Rename mode: inline TextInput
```

### Data Flow

1. **Trigger**: User types `/sessions` → 3835.js executes → calls `showSessionSelector(WL)` with session data
2. **State Update**: `WL(sessionArray)` sets `aA` state in 4065.js → triggers re-render showing SessionSelector
3. **User Interaction**: User navigates with arrows, types in search, presses Enter/Ctrl+R
4. **Selection**: `GS(selectedSessionId)` → calls `qD({sessionId})` → `loadSession()` on session service → closes panel via `WL(null)`
5. **Rename**: `vF(sessionId, newTitle)` → updates session title → refreshes list

### Panel Container (AI: 3873.js)

The `PanelDialog` component provides a shared bordered panel wrapper used by all selector panels:
- **Border**: `borderStyle: "round"`, `borderColor: o.border` (from theme)
- **Layout**: `flexDirection: "column"`, configurable `width`, `minWidth: 78`
- **Title**: Bold header with optional `headerRight` slot (used for tab indicators)
- **Help text**: Footer with keyboard shortcut hints, colored `o.text.muted`
- **Pagination**: Optional page indicator ("Showing X-Y of Z")

## Key Components

### Module 4017.js: SessionSelector Component (UeD)

```js
// 4017.js, ErrTracker init section-30
function UeD({ sessions: H, onSelect: A, onCancel: L, onRename: $, onModeChange: I }) {
  let { width: D } = w8(),
    [E, f] = tW.useState("currentFolder"),   // active filter tab
    [M, U] = tW.useState(0),                  // scroll offset (pagination)
    [P, B] = tW.useState(""),                 // search filter text
    [W, V] = tW.useState(null),               // selected session ID
    [X, Q] = tW.useState("browse"),           // mode: "browse" | "rename"
    [w, C] = tW.useState(""),                 // rename input value
    [Y, q] = tW.useState(""),                 // original session title (placeholder)
    [N, x] = tW.useState(null),               // rename target session ID
    [v, g] = tW.useState(false);              // rename saving in progress
```

**Session filtering logic** (line ~45-60):
```js
// 4017.js, lines ~45-60: memoized filtered session list
let i = tW.useMemo(() => {
  let bH;
  switch (E) {
    case "currentFolder":
      bH = H.filter((RH) => RH.isCurrentProject);
      break;
    case "favorites":
      bH = H.filter((RH) => RH.isFavorite);
      break;
    case "all":
    default:
      bH = H;
  }
  let SH = P.trim().toLowerCase();
  if (SH)
    bH = bH.filter((RH) => {
      let F = RH.title.toLowerCase(),
        z = (RH.sessionTitle ?? "").toLowerCase();
      return F.includes(SH) || z.includes(SH);
    });
  return bH;
}, [H, E, P]);
```

**Rename detection helper** (MeD function, line ~8-24):
```js
// 4017.js, lines 8-24: Ctrl+R detection (cross-platform: kitty, xterm, ctrl)
function MeD(H) {
  let A = H.key?.sequence;
  if (A) {
    let $ = String.fromCharCode(27),
      I = new RegExp(`${$}\\[(\\d+);(\\d+)u`).exec(A);
    if (I) {
      let D = Number(I[1]);
      if ((Number(I[2]) & 4) === 4 && (D === 82 || D === 114)) return true;
    }
  }
  if (H.input === "\x12") return true;  // Ctrl+R raw
  let L = H.key;
  if (L?.ctrl) {
    let $ = H.input?.toLowerCase();
    if (L.name?.toLowerCase() === "r" || $ === "r") return true;
  }
  return false;
}
```

### Module 4065.js: TUI Orchestrator (Session State Management)

**Session state initialization** (line 144):
```js
// 4065.js, line 144
[aA, WL] = _$.useState(null),  // aA = sessions array, WL = setter
```

**onSelect handler: GS** (lines 1399-1410):
```js
// 4065.js, lines 1399-1410
let GS = _$.useCallback(
  async (NA) => {
    try {
      await qD({ sessionId: NA });  // load session
    } catch (sA) {
      uH(sA, "Failed to load session");
      X("system", "Error loading session", { messageType: "text" });
    } finally {
      WL(null);  // close panel
    }
  },
  [qD, X],
);
```

**onRename handler: vF** (lines 1412-1427):
```js
// 4065.js, lines 1412-1427
vF = _$.useCallback(
  async (NA, sA) => {
    try {
      let hL = BA();  // session service
      await hL.updateSessionTitle(NA, sA, { manual: true });
      let K$ = await hL.getAllNonEmptySessions({
        currentCwd: process.cwd(),
        fetchOutsideCWD: true,
        maxOtherSessions: 100,
      });
      WL(K$);  // refresh session list
    } catch (hL) {
      uH(hL, "Failed to rename session");
      X("system", "Error renaming session", { messageType: "text" });
    }
  }, ...
);
```

### Module 3835.js: `/sessions` Slash Command

```js
// 3835.js, lines 1-40 (full module)
xdD = {
  name: "sessions",
  description: "List and select previous sessions to resume",
  execute: async (H, A) => {
    let { addMessage: L, showSessionSelector: $ } = A;
    try {
      let I = process.cwd(),
        E = (await BA().getAllNonEmptySessions({
          currentCwd: I,
          fetchOutsideCWD: true,
          maxOtherSessions: WSM,
        })).sort((U, P) => P.modifiedTime.getTime() - U.modifiedTime.getTime());
      if (E.length === 0)
        return (L("system", "No sessions found..."), { handled: true });
      if ($) return ($(E), { handled: true });  // shows selector
      return (L("system", "Session selector not available..."), { handled: true });
    } catch (I) { /* error handling */ }
  },
};
```

### Module 1185.js: Session Service (Data Layer)

```js
// 1185.js, lines 1144-1157: session metadata structure
{
  id: f,
  title: W.title,
  sessionTitle: W.sessionTitle,
  owner: W.owner,
  messageCount: V,
  modifiedTime: P.mtime,
  createdTime: P.birthtime || P.ctime,
  isFavorite: I.has(f),
  cwd: W.cwd,
  isCurrentProject: A,
  decompSessionType: W.decompSessionType,
}
```

## Theme / Style Tokens

The SessionSelector uses theme tokens from the global theme object `o`:

| Token | Usage | Context |
|-------|-------|---------|
| `o.border` | Panel border color | `borderColor: o.border` on PanelDialog |
| `o.primary` | Selected row indicator, active tab highlight | `color: z` where `z = F ? o.primary : void 0` |
| `o.text.muted` | Column headers, inactive tabs, help text, search label, CWD path | `color: o.text.muted` throughout |

**Tab indicators**: Active tab shows `◉` (circled bullet, `\u25C9`), inactive shows `○` (`\u25CB`).

**Selected row**: Highlighted with `o.primary` color across all columns, preceded by `>` indicator.

**Column layout widths**:
- Indicator: 3 chars (`> ` or `  `)
- Modified: 11 chars
- Created: 11 chars
- Size: 6 chars (message count)
- Title: dynamic (`LH`, ~25% of remaining width)
- Summary: dynamic (`DH`, ~75% of remaining width)

## Keyboard / Input Handling

The SessionSelector uses the `jL` (useInput, 3862.js) hook for all keyboard events. Input handling differs by mode:

### Browse Mode

| Key | Action | Code Reference |
|-----|--------|---------------|
| `↑` / `upArrow` | Move selection up | `V(prev => i[max(0, idx-1)]?.id)` |
| `↓` / `downArrow` | Move selection down | `V(prev => i[min(len-1, idx+1)]?.id)` |
| `Enter` / `return` | Select session and load | `A(i[HH].id)` → calls `GS(sessionId)` |
| `Escape` | Close panel | `L()` → calls `onCancel` |
| `q` | Close panel (alternative) | `L()` → calls `onCancel` |
| `Tab` | Cycle filter tab | `f(RH => JH(RH))`: cycles: currentFolder → all → favorites → currentFolder |
| `Ctrl+R` | Enter rename mode | `PH()`: sets mode to "rename" for selected session |
| Printable chars | Appended to search filter | Handled by `C8` TextInput component |

### Rename Mode

| Key | Action |
|-----|--------|
| `Escape` | Cancel rename, return to browse |
| `Enter` | Submit rename (via `EH()` callback) |
| Printable chars | Typed into rename input field |

### Text Input (C8: 3861.js)

The `TextInput` component provides:
- **Cursor management**: position tracking, Home/End support
- **Word navigation**: Ctrl+Left/Right, Meta+Left/Right
- **Word deletion**: Ctrl+W (delete word back), Ctrl+U (delete to start), Ctrl+V (delete to end)
- **Paste support**: Detects `isPaste` flag, sanitizes with `fiD(UiD(input))`
- **Mask mode**: Optional character masking for sensitive input
- **Ctrl+C**: Explicitly ignored (no-op)
- **Focus control**: Subscribes/unsubscribes from key events via context (`fm`)

### useInput Hook (jL: 3862.js)

```js
// 3862.js, lines 7-20
function jL(H, A = {}) {
  let { isActive: L = true } = A,
    $ = dl.useContext(fm),  // KeyInput context
    I = dl.useRef(H);
  I.current = H;
  let D = dl.useCallback((E) => {
    if (!E.isPaste) I.current(E.input, E.key);
  }, []);
  dl.useLayoutEffect(() => {
    if (!$ || !L) return;
    return ($.subscribe(D), () => { $.unsubscribe(D); });
  }, [$, L, D]);
}
```

### Scroll Pagination

The list uses a fixed window of `iVL = 10` visible items:
- `M` = scroll offset (first visible index)
- When `HH < M`: scroll up → `U(HH)`
- When `HH >= M + 10`: scroll down → `U(HH - 10 + 1)`

## Integration Points

- **tui-theme-system**: Consumes theme tokens (`o.border`, `o.primary`, `o.text.muted`) for panel styling. The `o` object is imported from the theme module.
- **tui-ink-core**: Uses Ink primitives (`<Box>` as `a`, `<Text>` as `u`) for layout and text rendering.
- **tui-renderer-main-loop** (4065.js): The main TUI orchestrator manages session state, renders the SessionSelector conditionally, and routes the `/sessions` command.
- **Session service** (1185.js): Backend data layer: `getAllNonEmptySessions()`, `loadSession()`, `updateSessionTitle()`. Outside TUI scope but critical for data flow.
- **tui-panel-model-selector**: Shares the same `PanelDialog` (AI, 3873.js) wrapper component.
- **tui-panel-theme-selector**: Also uses the `PanelDialog` container.
- **Slash command system**: `/sessions` command (3835.js) is the primary entry point; also triggered from keyboard shortcuts in 4065.js.
- **Mission system boundary**: Session `decompSessionType` field references mission decomposition types: outside 01-terminal-ui scope.

## Implementation Notes

To implement a SessionSelector panel in OpenDroid:

1. **Panel Container**: Create a reusable `PanelDialog` component (round border, title, help text) that all selector panels share. See 3873.js architecture.

2. **State Management**: Use React `useState` for:
   - Session list data (fetched async)
   - Selected item ID
   - Active filter/group tab
   - Search query text
   - Current mode (browse/rename)

3. **Keyboard Navigation**: Implement via Ink's `useInput` hook:
   - Arrow keys for selection (track by ID, not index)
   - Tab for cycling between filter groups
   - Enter for selection
   - Escape for cancel
   - Ctrl+R for rename mode (with cross-platform key detection)

4. **Filtering**: Three filter groups (current project, favorites, all) combined with text search across title fields.

5. **Scroll Window**: Fixed-size visible window (10 items) with automatic scroll adjustment when selection approaches edges.

6. **Rename Flow**: Two-phase (browse → rename mode), uses TextInput component for inline editing, saves async via service call, refreshes list on completion.

7. **Theme Tokens**: Use consistent theme tokens for border color, primary highlight, and muted text. Active tab uses `◉` vs `○` for inactive.

## Module Reference

| Module | Lines Read | Key Sections |
|--------|-----------|--------------|
| 4017.js | Full (177 lines) | SessionSelector component (UeD), filter/search/navigation/rename logic |
| 4065.js | Lines 141-145, 414-430, 476-482, 643, 751-764, 1049-1053, 1399-1427, 2898-2920 | Session state, onSelect/rename callbacks, rendering condition |
| 3835.js | Full (42 lines) | `/sessions` command, data fetch and showSessionSelector call |
| 1185.js | Lines 1144-1194 | Session metadata structure, getAllSessions/getAllNonEmptySessions |
| 3873.js | Lines 1-138 (full) | PanelDialog wrapper (AI), border/title/help rendering |
| 3862.js | Full (21 lines) | useInput hook (jL), key event subscription |
| 3861.js | Lines 79-243 | TextInput component (C8), cursor/selection/paste handling |
| 3381.js | Full (24 lines) | useWindowSize hook (w8), terminal resize detection |
