# tui-status-bar: TUI Architecture Notes

## Overview

The OpenDroid TUI status bar is a footer-level information display rendered at the bottom of the terminal interface. It consists of two subsystems: (1) a **custom status line** that executes user-configured shell commands and renders their stdout output as dimmed text, and (2) a **built-in status indicator** that displays elapsed assistant active time and context window usage percentage. The status bar integrates with the global settings store (0939.js) for configuration, uses a singleton service pattern (4032.js) for command execution with caching, and renders via Ink primitives (`<Box>` and `<Text>`) in the main app component (4065.js).

## Module Map

| Module | Size | Rol | Type |
|--------|------|-----|------|
| 4032.js | ~7 KB | StatusLineService: singleton command executor with caching (300ms debounce, 5s timeout) | app |
| 4033.js | ~1.5 KB | StatusLine React Component: renders custom status line text via Ink `<Box>/<Text>` | app |
| 3865.js | ~2 KB | StatusIndicator Component (TiD): built-in timer + context usage indicator | app |
| 3849.js | ~7 KB | `/status` command + statusline setup agent prompt | app |
| 3850.js | ~3 KB | `/statusline` slash command handler: configures custom status line | app |
| 0939.js | ~25 KB | Settings Store: `getStatusLine()`, `getShowTokenUsageIndicator()` accessors | app |
| 1185.js | ~varies | Session Manager: `getTokenUsage()`, token tracking for context percentage | app |
| 4065.js | ~245 KB | Main App Component: renders both TiD and oeD in footer area | app |

## Architecture

### Component Hierarchy

```
Main App (4065.js)
└── Footer Container (Box with gap: 0)
    ├── TiD (3865.js): StatusIndicator
    │   ├── Timer display: "⏱ {elapsed}"
    │   └── Context usage: "context: {percent}%"
    └── oeD (4033.js): StatusLine (custom)
        └── Text { dimColor: true }
            └── Command output string (padded)
```

### Data Flow

```
Settings Store (0939.js)
  ├─ getStatusLine() → { command: string, padding?: number }
  ├─ getShowTokenUsageIndicator() → boolean
  └─ getModel() → model ID

Session Manager (1185.js)
  ├─ getTokenUsage() → { inputTokens, outputTokens, ... }
  ├─ getAssistantActiveTime() → milliseconds
  └─ emit "session_token_usage_changed" via event bus

StatusLineService (4032.js)
  ├─ Singleton via neD()
  ├─ execute() → cached (300ms) → executeCommand()
  │   ├─ spawn("cmd.exe", ["/c", command])
  │   ├─ stdin: JSON.stringify(buildContext())
  │   ├─ stdout: first line trimmed
  │   └─ timeout: 5000ms (SIGTERM)
  └─ buildContext() → { session_id, transcript_path, cwd, model, version }
```

### Rendering Pipeline

1. **Main App** (4065.js) renders footer container with `gap: 0` in a flex row
2. **TiD** (3865.js) checks `statusState` against active states set `WyM`, tracks elapsed time via `setInterval(1000)`, computes context percentage
3. **oeD** (4033.js) polls `vA().getStatusLine()` every 1000ms, calls `neD().execute()` with 300ms client-side debounce, renders result or returns `null`
4. Both components return `null` when inactive: the footer row auto-hides

## Key Components

### StatusLineService (4032.js, lines 85-175)

```js
class ieD {
  lastExecutionTime = 0;
  pendingExecution = null;
  cachedResult = null;
  async execute() {
    let H = vA().getStatusLine();
    if (!H) return null;
    let A = Date.now();
    if (A - this.lastExecutionTime < rvM) return this.cachedResult;  // 300ms cache
    if (this.pendingExecution) return this.pendingExecution;
    // ... spawn child process
  }
  buildContext() {
    return {
      session_id: H.getCurrentSessionId() || "",
      transcript_path: H.getSessionTranscriptPath() || "",
      cwd: process.cwd(),
      model: { id: L, display_name: $.shortDisplayName || $.displayName },
      version: "0.64.0",
    };
  }
}
var rvM = 300, leD = 5000, sVL = null;
function neD() {
  if (!sVL) ((sVL = new ieD()), BH("[StatusLineService] Initialized"));
  return sVL;
}
```

### StatusLine Component (4033.js, lines 7-52)

```js
function oeD({ sessionId: H, messageCount: A, width: L }) {
  let [$, I] = _N.useState(null),    // status text
    [D, E] = _N.useState(false),      // has statusLine config
    f = _N.useRef(0),                  // last execution timestamp
    M = _N.useRef(null),               // interval ref
    U = _N.useCallback(async () => {
      let W = vA().getStatusLine();
      if ((E(!!W), !W)) { I(null); return; }
      let V = Date.now();
      if (V - f.current < 300) return;  // client debounce
      f.current = V;
      let X = await neD().execute();
      I(X);
    }, []);
  // useEffect: re-execute on sessionId/messageCount change
  // useEffect: poll every 1000ms for statusLine config changes
  // Render: Box(width=L, marginLeft=1) > Text(dimColor=true, padded text)
  let B = vA().getStatusLine()?.padding ?? 0;
  return eVL.jsxDEV(a, { width: L, marginLeft: 1,
    children: eVL.jsxDEV(u, { dimColor: true,
      children: [B > 0 && " ".repeat(B), $, B > 0 && " ".repeat(B)]
    })
  });
}
```

### StatusIndicator Component (3865.js, lines 12-70)

```js
function TiD({ statusState: H, sessionId: A, lastTokenUsage: L }) {
  // Tracks assistant active time with setInterval(1000) when state is in WyM
  let U = Math.max(0, $ + (E.current ? Date.now() - E.current : 0));
  let P = t$A(U);  // format elapsed time
  let B = P !== "0s";
  let V = vA().getShowTokenUsageIndicator() && L !== null && L > 0
    ? VyM(L ?? null) : null;  // context window percentage
  if (!B && !V) return null;
  let X = [];
  if (B) X.push(`⏱ ${P}`);
  if (V) X.push(`context: ${V}`);
  return ViD.jsxDEV(u, { color: "gray", children: ["[", X.join(", "), "]"] });
}
```

### Context Percentage Calculation (3865.js, lines 7-11)

```js
function VyM(H) {
  let L = vA().getModel(),
    D = F$(L).modelProvider === "anthropic" ? TyM : daH,  // 200000 or other
    E = Math.max(0, H - WiD),  // WiD = 11000 (base overhead)
    f = D - WiD,
    M = Math.min(100, Math.round((E / f) * 100));
  if (M === 0) return "<1%";
  return `${M}%`;
}
var WiD = 11000, TyM = 200000;
```

## Theme / Style Tokens

### Status Bar Styling

- **Custom StatusLine text**: `dimColor: true`: renders with reduced brightness
- **StatusIndicator text**: `color: "gray"`: uses gray foreground color
- **Container**: Box with `width: L` (terminal width), `marginLeft: 1`, `gap: 0`
- **Padding**: configurable via `statusLine.padding` setting: adds `" ".repeat(padding)` before and after text
- **Context display format**: `[⏱ 5m 30s, context: 45%]`: gray text with brackets

### Related Theme Tokens (from tui-theme-system)

- `o.text.muted`: used in other footer/secondary text contexts
- `o.text.secondary`: used for secondary information display
- `o.primary`: used for interactive element highlighting

## Keyboard / Input Handling

The status bar components do **not** handle keyboard input directly. They are passive display components that:
- Subscribe to global state via `vA()` (settings store) and `BA()` (session manager)
- React to `sessionId` and `messageCount` prop changes
- Use `setInterval` polling (1000ms) for periodic state checks
- The `/statusline` slash command (3850.js) is triggered via the command input handler, not the status bar itself

## Integration Points

- **tui-theme-system** (0939.js): Settings store provides `getStatusLine()`, `getShowTokenUsageIndicator()`, `getModel()`: the status bar reads theme tokens indirectly through the settings store
- **tui-renderer-main-loop** (4065.js): Main app renders `TiD` and `oeD` in the footer area, passes `sessionId`, `messageCount`, `width`, `statusState`, `lastTokenUsage` props
- **tui-ink-core** (2289.js): Status bar uses Ink `<Box>` (aliased as `a`) and `<Text>` (aliased as `u`) primitives for rendering
- **Session management** (1185.js): `getTokenUsage()`, `getAssistantActiveTime()`, event bus for `session_token_usage_changed`
- **Command system** (3849.js, 3850.js): `/status` displays full status report, `/statusline` configures custom status line

## Implementation Notes

To implement a similar status bar in OpenDroid:

1. **Create a StatusLineService singleton**: Cache command execution results with a 300ms debounce. Execute user-configured commands via `child_process.spawn`, passing context JSON via stdin. Return first line of stdout, trimmed.

2. **Create a StatusIndicator component**: Track assistant active time with `Date.now()` and `setInterval`. Compute context window usage as `(tokens - 11000) / (maxTokens - 11000) * 100`.

3. **Create a StatusLine React component**: Poll settings for `statusLine` config every 1s. Call service to execute command. Render with Ink `<Box width={termWidth}>` and `<Text dimColor>`. Support configurable padding.

4. **Integrate in main app layout**: Render both components in a footer `<Box flexDirection="row" gap={0}>`. Both return `null` when inactive.

5. **Settings schema**: Add `statusLine: { type: "command", command: string, padding?: number }` and `showTokenUsageIndicator: boolean` to user settings.

6. **Slash commands**: Implement `/status` for status dump and `/statusline` for interactive configuration via an agent prompt.

## Module Reference

| Module | Lines | Key Content |
|--------|-------|-------------|
| 4032.js | 85-175 | StatusLineService class (ieD), neD singleton factory, buildContext, command execution |
| 4033.js | 1-52 | StatusLine component (oeD), polling, rendering with Box/Text |
| 3865.js | 1-70 | StatusIndicator (TiD), context calculation (VyM), timer tracking, constants |
| 3849.js | 1-210 | `/status` command, statusline setup agent prompt (vlD) |
| 3850.js | 1-40 | `/statusline` slash command handler (glD) |
| 0939.js | varies | `getStatusLine()`, `getShowTokenUsageIndicator()` in settings class |
| 1185.js | varies | `getTokenUsage()`, `addTokenUsage()`, token usage emitter |
| 4065.js | varies | Footer rendering with TiD and oeD components |
