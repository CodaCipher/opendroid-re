# tui-panel-background-process: TUI Architecture Notes

## Overview

The Background Process Manager is a TUI panel that provides a terminal-based interface for monitoring and controlling background processes spawned by the OpenDroid droid system. It consists of a React Ink panel component (`kiD` in 3876.js) that renders a process list with PID, age, session, and command columns; a persistent JSON-backed tracker class (`yMH` / `BackgroundProcessTracker` in 2323.js) that manages process lifecycle registration and cleanup; an in-memory tool-level process tracker (`kMH` / `ProcessTracker` in 2334.js) that tracks processes per tool ID with graceful and forceful kill escalation; and a command handler (`bg-process` in 3792.js) that opens the panel from the slash-command system. The panel is rendered within a shared `AI` panel frame component (3873.js) with rounded borders and theme-aware styling.

## Module Map

| Module | Size | Role | Type |
|--------|------|------|------|
| 3876.js | ~5 KB | Background Process Manager panel UI component (`kiD`) | app |
| 2323.js | ~4 KB | `BackgroundProcessTracker`: JSON-persisted process registry with kill/cleanup | app |
| 2334.js | ~5 KB | `ProcessTracker`: in-memory tool-process tracking with SIGTERM→SIGKILL escalation | app |
| 3792.js | ~1 KB | `bg-process` slash command handler: opens panel via `showBgProcessManager()` | app |
| 0076.js | ~large | Config schema: `allowBackgroundProcesses` field (deprecated/removed via preprocess) | app |
| 3873.js | ~3 KB | Shared `AI` panel frame component: border, title, helpText, pagination | app |

## Architecture

### Component Hierarchy

```
4065.js (App coordinator)
  └─ showBgProcessManager: () => AL(true)
       └─ <Box width={VA}>
            └─ <kiD onClose={() => AL(false)} />   ← 3876.js
                 └─ <AI title="Background Process Manager" ...>   ← 3873.js
                      ├─ Loading state
                      ├─ Status message (success/error)
                      ├─ Empty state ("No background processes running.")
                      └─ Process list table
                           ├─ Header row: PID | Age | Session | Command
                           └─ Process rows (selected highlighted)
```

### Data Flow

1. **Registration**: Tool execution spawns subprocess → `BackgroundProcessTracker.registerProcess(pid, command, cwd, sessionId, outputFile)` persists to `background-processes.json`
2. **Tool-level tracking**: `ProcessTracker.registerProcess(toolId, pid, metadata)` tracks per-tool in memory via `Map<toolId, Set<pid>>`
3. **Panel load**: `kiD` calls `y2.getProcesses()` (via `BackgroundProcessTracker`), maps to display objects with computed `age` and `isCurrentSession`
4. **User action**: `wD` selection hook captures keyboard → triggers `killProcess`, `cleanupDeadProcesses`, `killSessionProcesses`, or `killAllProcesses`
5. **Kill escalation**: SIGTERM first → wait 5s → SIGKILL if still running (BackgroundProcessTracker); SIGTERM → wait 1s → SIGKILL (ProcessTracker)

### Dual Tracker Architecture

The system uses two complementary process tracking mechanisms:

- **BackgroundProcessTracker** (2323.js): File-backed singleton. Stores `{ pid, command, cwd, startTime, sessionId, outputFile }` in JSON. Survives app restarts. Used for user-facing process management.
- **ProcessTracker** (2334.js): In-memory singleton. Stores `{ toolId → Set<pid> }` and `{ pid → metadata }`. Tied to tool lifecycle, cleaned up on tool exit. Used for programmatic tool-process cleanup.

## Key Components

### 3876.js: Panel Component `kiD`

**State management:**

```js
// Module 3876.js, line 5
function kiD({ onClose: H }) {
  let [A, L] = ES.useState([]),       // process list
      [$, I] = ES.useState(true),      // loading state
      [D, E] = ES.useState(null);      // status message
```

**Process list fetching with age computation:**

```js
// Module 3876.js, line 11
f = ES.useCallback(() => {
  I(true);
  let Q = BA().getCurrentSessionId(),
    C = y2.getProcesses().map((Y) => {
      let q = Date.now() - Y.startTime,
        N = Math.floor(q / 1000);
      return {
        pid: Y.pid,
        command: Y.command,
        sessionId: Y.sessionId,
        age: `${N}s`,
        isCurrentSession: Y.sessionId === Q,
      };
    });
  (L(C), I(false));
}, []);
```

**Keyboard handling via `wD` selection hook:**

```js
// Module 3876.js, line 42
{ selectedIndex: X } = wD({
  items: A,
  initialIndex: 0,
  wrapAround: true,
  onSelect: (Q) => { U(Q.pid); },      // Enter → kill
  onCancel: H,                          // Esc → close
  additionalKeys: {
    K: () => { let Q = A[V.current]; if (Q) U(Q.pid); },  // K → kill
    r: () => f(),   R: () => f(),    // R → reload
    c: () => P(),   C: () => P(),    // C → cleanup dead
    a: () => { B(); },  A: () => { B(); },  // A → kill all
    s: () => { W(); },  S: () => { W(); },  // S → kill session
  },
});
```

**Process list rendering:**

```js
// Module 3876.js, line 93
rz.jsxDEV(a, { flexDirection: "column", children: [
  // Header row
  rz.jsxDEV(a, { marginBottom: 1, children:
    rz.jsxDEV(u, { color: o.text.secondary, children: [
      "PID".padEnd(8), " ", "Age".padEnd(8), " ",
      "Session".padEnd(12), " Command"
    ]})
  }),
  // Process rows
  A.map((Q, w) => {
    let C = w === X,                    // is selected
        Y = C ? o.primary : void 0,     // highlight color
        q = Q.isCurrentSession ? "(current)" : Q.sessionId?.slice(0, 8) || "unknown";
    return rz.jsxDEV(a, { children:
      rz.jsxDEV(u, { color: Y, children: [
        C ? "> " : "  ",
        Q.pid.toString().padEnd(8),
        Q.age.padEnd(8),
        q.padEnd(12),
        Q.command.length > 50 ? `${Q.command.slice(0, 47)}...` : Q.command,
      ]})
    }, Q.pid);
  })
]})
```

### 2323.js: BackgroundProcessTracker (yMH)

**Singleton with JSON persistence:**

```js
// Module 2323.js, line 14
class yMH {
  static instance;
  storePath;
  constructor() {
    this.storePath = dCI.join(C$(), sL, "background-processes.json");
    this.ensureStoreExists();
  }
  static getInstance() {
    if (!yMH.instance) yMH.instance = new yMH();
    return yMH.instance;
  }
```

**Process registration:**

```js
// Module 2323.js, line 44
registerProcess(H, A, L, $, I) {
  let E = this.loadStore().processes.filter((f) => f.pid !== H);
  E.push({ pid: H, command: A, cwd: L, startTime: Date.now(), sessionId: $, outputFile: I });
  this.saveStore({ processes: E });
}
```

**Kill with SIGTERM→SIGKILL escalation (5s timeout):**

```js
// Module 2323.js, line 81
async killProcess(H) {
  if (!this.isProcessRunning(H)) return (this.cleanupDeadProcesses(), false);
  return new Promise((A) => {
    woH.default(H, "SIGTERM", (L) => {
      if (L) { A(false); return; }
      setTimeout(() => {
        if (this.isProcessRunning(H))
          woH.default(H, "SIGKILL", ($) => { ... });
        else (this.removeProcessFromStore(H), A(true));
      }, 5000);    // 5-second grace period
    });
  });
}
```

**Session-scoped kill:**

```js
// Module 2323.js, line 125
async killSessionProcesses(H) {
  let A = this.getProcesses(H);
  await Promise.all(A.map(async ($) => this.killProcess($.pid)));
}
```

### 2334.js: ProcessTracker (kMH)

**Tool-scoped process tracking:**

```js
// Module 2334.js, line 42
class kMH {
  activeProcesses = new Map();     // toolId → Set<pid>
  processMetadata = new Map();     // pid → { command, toolId, ... }
```

**Kill with shorter 1s timeout:**

```js
// Module 2334.js, line 80
async killProcess(H, A) {
  let $ = 1000;  // 1-second wait
  await I(A);    // send signal
  if (!(await this.waitForProcessExit(H, $)) && this.isProcessRunning(H))
    await I("SIGKILL");  // force kill
}
```

**Kill all tool processes:**

```js
// Module 2334.js, line 64
async killAllProcesses(H = "SIGTERM") {
  let L = Array.from(this.activeProcesses.keys())
    .map(($) => this.killToolProcesses($, H));
  await Promise.allSettled(L);
}
```

### 3792.js: `bg-process` Command Handler

```js
// Module 3792.js, line 5
AmD = {
  name: "bg-process",
  description: "Manage background processes (list, kill, cleanup)",
  execute: async (H, A) => {
    let { addMessage: L, showBgProcessManager: $ } = A;
    if ($) return ($(), { handled: true });
    // fallback: show system message if panel not available
    L("system", "Background Process Manager not available in this context.", { ... });
  },
};
```

### 0076.js: Config Schema (allowBackgroundProcesses)

```js
// Module 0076.js (preprocess strip)
p.preprocess((H) => {
  if (H && typeof H === "object" && !Array.isArray(H)) {
    let A = { ...H };
    return (delete A.allowBackgroundProcesses, A);  // field removed from config
  }
  return H;
}, cUA)
```

Note: `allowBackgroundProcesses` is a deprecated config field that is stripped during schema preprocessing. This suggests the feature was once gated by a config flag but is now always available.

## Theme / Style Tokens

The Background Process Manager uses standard theme tokens from the shared `o` theme object:

| Token | Usage | Context |
|-------|-------|---------|
| `o.text.muted` | Loading text, empty state, help text | "Loading...", "No background processes running." |
| `o.text.secondary` | Process list header row | PID / Age / Session / Command labels |
| `o.primary` | Selected process row highlight | Current selection indicator (`> ` prefix) |
| `o.success` | Success status message | "Successfully killed process XXX" |
| `o.error` | Error status message | "Failed to kill process XXX" |
| `o.border` | Panel border color | Applied via `AI` panel frame (3873.js) |

Panel frame styling (from 3873.js):
- `borderStyle: "round"`: rounded border around entire panel
- `paddingX: 1`, `paddingY: 0`: minimal padding
- `marginTop: 1`: spacing from content above
- `minWidth: 78`: minimum panel width

## Keyboard / Input Handling

The panel uses the `wD` hook (from 3875.js, shared selection hook used across all panels) with custom `additionalKeys`:

| Key | Action | Handler |
|-----|--------|---------|
| `↑` / `↓` / `j` / `k` | Navigate process list | Built-in `wD` navigation |
| `Enter` | Kill selected process | `onSelect` → `U(pid)` |
| `K` | Kill selected process (vim-style) | `additionalKeys.K` |
| `R` / `r` | Reload process list | `f()`: re-fetches from tracker |
| `C` / `c` | Cleanup dead processes | `P()` → `y2.cleanupDeadProcesses()` |
| `S` / `s` | Kill session processes | `W()` → `y2.killSessionProcesses(sessionId)` |
| `A` / `a` | Kill all processes | `B()` → `y2.killAllProcesses()` |
| `Esc` | Close panel | `onCancel: H` (onClose prop) |
| `Q` | Close panel (via `gq` hook) | `gq(H)`: close-on-Q handler |

The `gq` hook (from 3874.js) provides a close-on-Escape/Q behavior using `useInput` with Ink's input context.

## Integration Points

- **tui-panel-session-selector**: Session context via `BA().getCurrentSessionId()`: processes are scoped to sessions; "kill session" kills only current session's processes
- **tui-theme-system**: Theme tokens (`o.text.muted`, `o.primary`, `o.border`, etc.) from the shared theme object: panel uses standard panel theme
- **tui-ink-core**: Renders via React Ink primitives (`<Box>`, `<Text>`) through `rz.jsxDEV` (Ink's createElement)
- **4065.js (App coordinator)**: Panel is conditionally rendered based on `AL` state flag; triggered via `showBgProcessManager()` callback passed to command executor
- **tui-panel-mcp-manager / tui-panel-model-selector / other panels**: Shares the `AI` panel frame (3873.js) and `wD` selection hook (3875.js): consistent UI patterns across all panels
- **Tool system boundary**: `ProcessTracker` (2334.js) is the tool-level counterpart: tools register their subprocesses for cleanup, but this is a separate concern from the user-facing BackgroundProcessTracker
- **Config system**: `allowBackgroundProcesses` was once a config flag (0076.js) but is now stripped during preprocessing: feature is always enabled

## Implementation Notes

To port the Background Process Manager panel:

1. **Panel component** (`kiD` in 3876.js): Straightforward React Ink component. Port with standard Ink `<Box>`/`<Text>`. Replace `wD` with your own selection hook or port the existing one.

2. **Process tracker**: The `BackgroundProcessTracker` class uses Node.js `process.kill(pid, 0)` for liveness checks and `tree-kill` (via `woH`) for signal dispatch. On alternative runtimes, replace with platform-appropriate process signaling.

3. **Persistence**: Process state is persisted to `background-processes.json` in the app data directory. Port the file I/O layer.

4. **Kill escalation**: The SIGTERM→wait→SIGKILL pattern with 5-second timeout is a standard Unix pattern. Adjust for Windows (where signals behave differently).

5. **Panel frame**: The `AI` component (3873.js) is shared across all panels. Either port it once or implement per-panel.

6. **Command registration**: The `bg-process` slash command (3792.js) opens the panel. Integrate with your command dispatcher.

## Module Reference

| Module | Lines Read | Key Sections |
|--------|-----------|--------------|
| 3876.js | 1-218 (full) | Panel component `kiD`: state, fetch, keyboard, rendering |
| 2323.js | 1-155 (full) | `BackgroundProcessTracker` class: register, get, kill, cleanup, persistence |
| 2334.js | 1-169 (full) | `ProcessTracker` class: tool-scoped tracking, kill with 1s escalation |
| 3792.js | 1-38 (full) | `bg-process` command handler |
| 0076.js | grep only | Config schema: `allowBackgroundProcesses` field (stripped) |
| 3873.js | 1-138 (full) | Shared `AI` panel frame component |
