# tui-renderer-main-loop: TUI Architecture Notes

## Overview

This report documents the OpenDroid CLI application entry point and main render loop. The system supports multiple execution modes: terminal UI (Ink-based TUI), interactive CLI (ACP daemon/child, JSON-RPC), non-interactive CLI, and streaming exec: each with distinct signal handling, stdin/stdout plumbing, and mode-switching via a centralized `droidMode` state. Module 4070.js serves as the primary TUI bootstrapper, invoking Ink's `render()` with a nested provider/component tree, while companion modules handle alternative I/O paths.

## Module Map
| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 4070.js | ~2.5 KB | Main TUI entry point: Ink render, logo animation, terminal disconnect handler | App |
| 3710.js | ~33 KB | Non-interactive CLI exec command: mode detection, output format routing, tool listing | App |
| 3708.js | ~32 KB | JSON-RPC streaming mode: message queue, SIGINT handling, session controller | App |
| 3427.js | ~5 KB | Streaming exec: JSON-over-stdin readline, SIGINT, 30s drain timeout | App |
| 3425.js | ~1.5 KB | ACP daemon runner: interactive CLI "acp-daemon" submode | App |
| 3426.js | ~1.6 KB | ACP child runner: interactive CLI "acp-child" submode | App |
| 2101.js | ~1.6 KB | Kitty keyboard protocol detector: ANSI probe, raw mode toggle | App |

## Architecture

The OpenDroid CLI is a multi-mode application whose entry path is determined at runtime by command-line flags and TTY availability. The architecture can be visualized as a mode router with four primary branches:

```
[process.argv / env parsing]
         |
    +----+----+
    |         |
[isTTY?]   [non-TTY]
    |         |
 4070.js   3710.js
(TUI mode) (non-interactive-cli)
    |
  Sb(jsx, {exitOnCtrlC:false})
    |
[l9H provider -> qCI provider -> wA0 App]
    |
[OA0() terminal disconnect monitor]
```

### Mode Switching (`droidMode`)

All entry points call `Qf().setDroidMode(mode, submode)` followed by `cB("droidMode", mode)` to broadcast the current operational mode to telemetry and internal state:

- **`terminal-ui`** (4070.js): Full Ink TUI with keyboard-driven panels.
- **`interactive-cli`** (3425.js, 3426.js, 3708.js): Text-based or JSON-RPC protocols over stdin/stdout. Submodes: `acp-daemon`, `acp-child`, `json-rpc`.
- **`non-interactive-cli`** (3710.js): One-shot command execution, piped input/output.

### Main Render Loop (4070.js)

The TUI branch in 4070.js performs the following initialization sequence:

1. **Logging & Telemetry**: `BH("[Main] Starting OpenDroid CLI TUI application")`, `RgH("Droid")`, `cB("droidMode", "terminal-ui")`.
2. **Resource Monitoring**: `startResourceMonitoring()` / `stopResourceMonitoring()`.
3. **Security**: `ensureAllSecurePermissions()` on startup (non-blocking catch).
4. **Client Mode**: `B_.getInstance().setClientMode("tui")`.
5. **Logo Animation**: Conditional `U.shouldShowLogoAnimation()` → `CA0()` full-screen spinner (see tui-logo-animation), then `markLogoAnimationShown()`.
6. **MCP Service**: `JE().start()` initializes the MCP tool subsystem.
7. **Ink Render**: `Sb(...)` (Ink `render()`) mounts the React component tree with `exitOnCtrlC: false` and `patchConsole: false`.
8. **Disconnect Handler**: `OA0({ enabled: isTTY, onDisconnect: Q })` listens for terminal detachment.
9. **Wait & Cleanup**: `await B.waitUntilExit()`, then unmount, clear, and shutdown.

### Signal Handling

Every mode registers its own `SIGINT` and/or `SIGTERM` handlers:

- **4070.js**: Uses Ink's `exitOnCtrlC: false`, relying on `OA0()` disconnect + graceful unmount.
- **3425.js / 3426.js**: `process.on("SIGINT", D)` and `SIGTERM` call async dispose + `process.exit(0)`.
- **3708.js**: Custom `sigintHandler` emits JSON-RPC error, closes readline, and resolves the run promise.
- **3427.js**: SIGINT writes error JSON to stdout, removes listener, closes readline, and resolves.
- **3710.js**: Implicit through the command framework; non-interactive mode typically exits on completion.

### Keyboard / Input Routing

- **TUI mode**: Ink captures raw stdin via its internal `useStdin` / `useInput` hooks (see tui-ink-core, tui-composer-input).
- **Interactive CLI modes**: `readline.createInterface({ input: process.stdin, output: process.stdout, terminal: false })` for line-oriented JSON or text protocols.
- **Kitty protocol** (2101.js): Probes the terminal with `\x1B[?u` and `\x1B[c` ANSI sequences; if the terminal reports `u` capability, enables enhanced key reporting via `\x1B[>1u` and registers an exit hook to reset with `\x1B[<u`. Uses 150 ms timeout for detection.

## Key Components

### Module 4070.js: Main TUI Bootstrap

```js
// Module 4070.js, lines 4-9
async function uuM(H, A, L, $, I) {
  (BH("[Main] Starting OpenDroid CLI TUI application"), RgH("Droid"), cB("droidMode", "terminal-ui"));
  let { startResourceMonitoring: D, stopResourceMonitoring: E } = await Promise.resolve().then(
    () => (fXL(), yA0),
  );
  (D(), await $tA());
```

Ink render invocation with nested providers:

```js
// Module 4070.js, lines 37-55
let B = Sb(
  gMA.jsxDEV(
    l9H,
    {
      orgId: M?.orgId,
      children: gMA.jsxDEV(
        qCI,
        {
          children: gMA.jsxDEV(
            wA0,
            { initialPrompt: H, resumeSessionId: A, originalCwd: $ },
            void 0, false, void 0, this,
          ),
        },
        void 0, false, void 0, this,
      ),
    },
    void 0, false, void 0, this,
  ),
  { exitOnCtrlC: false, patchConsole: false },
);
```

Graceful shutdown on terminal disconnect:

```js
// Module 4070.js, lines 81-93
let Q = (w) => {
  if (X) return;
  ((X = true),
    BH("[Main] Terminal disconnected, exiting", { reason: w }),
    MBH().catch(() => {}));
  try { B.unmount(); } catch {}
  if (((V = setTimeout(() => { yD(0); }, 2000)),
      typeof V.unref === "function")
    V.unref();
};
```

### Module 3710.js: Non-Interactive CLI Detection

```js
// Module 3710.js, lines 42-44
function fNM() {
  return !process.stdin.isTTY;
}
```

Mode switch to non-interactive:

```js
// Module 3710.js, lines 591-592
(Qf().setDroidMode("non-interactive-cli"), cB("droidMode", "non-interactive-cli"));
```

### Module 3708.js: JSON-RPC Signal & Input Handlers

```js
// Module 3708.js, lines 851-861
setupSignalHandlers(H) {
  ((this.sigintHandler = async () => {
    (BH("[JsonRpc] SIGINT received"),
      this.emitError(null, { code: -32603, message: "Session interrupted by user" }),
      this.removeSigintHandler(),
      this.rl?.close(),
      H());
  }),
    process.on("SIGINT", this.sigintHandler));
}
```

### Module 3425.js: ACP Daemon Signal Handling

```js
// Module 3425.js, lines 38-44
(process.on("SIGINT", () => {
  D().catch(() => {});
}),
  process.on("SIGTERM", () => {
    D().catch(() => {});
  }),
  process.stdin.resume());
```

### Module 2101.js: Kitty Protocol Probe

```js
// Module 2101.js, lines 9-14
async function XnH() {
  if (TnH) return YpA;
  return new Promise((H) => {
    if (!process.stdin.isTTY || !process.stdout.isTTY) {
      ((TnH = true), H(false));
      return;
    }
    let A = process.stdin.isRaw;
    if (!A) process.stdin.setRawMode?.(true);
```

## Keyboard / Input Handling

- **Raw mode toggle** (2101.js): Temporarily enables `setRawMode(true)` to probe terminal capabilities, then restores the previous state. This is the only place in the analyzed modules that directly manipulates raw mode.
- **Ink stdin capture** (4070.js): Delegated entirely to Ink's reconciler/runtime (see tui-ink-core). The `exitOnCtrlC: false` option prevents Ink from intercepting Ctrl+C, leaving it to the OS signal path or the `OA0()` disconnect handler.
- **Readline interfaces** (3708.js, 3427.js): Line-oriented input for JSON-RPC and streaming exec. `terminal: false` avoids ANSI processing, keeping the channel clean for machine-readable JSON.
- **ACP adapters** (3425.js, 3426.js): Use `hIA(mIA(process.stdout), pIA(process.stdin))` to wrap stdio into a structured transport, with `dIA(I)` producing an adapter layer.

## Integration Points

- **tui-ink-core**: `Sb()` in 4070.js is Ink's `render()` function; the reconciler and scheduler pipeline (module 2228.js et al.) processes the JSX tree `{l9H -> qCI -> wA0}`.
- **tui-react-runtime**: `gMA.jsxDEV(...)` is the JSX factory from the React runtime bundle (module 2206.js et al.).
- **tui-logo-animation**: 4070.js conditionally triggers `CA0()` (FullScreenAnimation) before mounting the main UI; see tui-logo-animation report for frame loop details.
- **tui-theme-system**: The rendered component tree `wA0` consumes theme tokens via the `l9H` / `qCI` provider stack; see tui-theme-system report.
- **tui-panel-session-selector**: `wA0` accepts `resumeSessionId: A`, linking to session selection panel logic.
- **tui-composer-input**: The main App `wA0` embeds the composer/text input components that handle multi-line editing via Ink `useInput`.
- **tui-status-bar**: The status bar is rendered as a child of the main App tree mounted by `Sb()`.
- **tui-panel-mcp-manager**: `JE().start()` in 4070.js initializes MCP services whose UI panel is documented in tui-panel-mcp-manager.
- **tui-panel-background-process**: Tracked subprocesses are managed through the same runtime initialized in the main loop.
- **tui-ansi-terminal-utils**: 2101.js writes raw ANSI escape sequences (`\x1B[?u`, `\x1B[c`, `\x1B[>1u`) directly; the ANSI utility modules (2539.js et al.) provide higher-level helpers used elsewhere.
- **02-orchestration-5 boundaries**: `JE()` (MCP service), `B_` (client state), `Qf()` (mode state), and `o8()` (session controller) are cross-system dependencies belonging to non-TUI subsystems; their internals are not analyzed here.

## Implementation Notes

1. **Multi-mode router**: Preserve the `droidMode` state machine (`terminal-ui` / `interactive-cli` / `non-interactive-cli`). OpenDroid needs equivalent mode detection (TTY check + argv parsing).
2. **Ink render bootstrap**: `render(<ProviderTree />, { exitOnCtrlC: false, patchConsole: false })` is the canonical pattern. OpenDroid should adopt the same Ink options to allow custom signal handling.
3. **Terminal disconnect monitor**: `OA0({ enabled: isTTY, onDisconnect: cleanup })` is a useful pattern for CI/SSH sessions where the terminal may vanish. Port as a small wrapper around `process.stdin` `end`/`error` events.
4. **Kitty protocol probe**: 2101.js shows a minimal, timeout-gated ANSI probe. OpenDroid can reuse this for enhanced key reporting without pulling in heavy dependencies.
5. **Graceful shutdown sequence**: 4070.js demonstrates the ordering: `unmount()` → `clear()` → `stopResourceMonitoring()` → `yD(0)`. Maintain this ordering to avoid Ink reconciliation errors during exit.
6. **Signal hygiene**: Each mode registers and later removes its own `SIGINT` handler. In a port, use `process.once("SIGINT", ...)` or explicit `removeListener` to prevent handler leaks when switching modes.

## Module Reference

- **4070.js** (lines 1–102): Full module read: main bootstrap, Ink render, disconnect handler, cleanup.
- **3710.js** (lines 1–80, 531–610): Entry CLI framework, `fNM()` TTY detection, non-interactive mode switch, output format routing (`acp-daemon`, `acp`, `stream-jsonrpc`).
- **3708.js** (lines 1–80, 801–880, 901–929): JSON-RPC class constructor, `setupInputHandlers()`, `setupSignalHandlers()`, `shutdown()` with 30 s timeout.
- **3427.js** (lines 1–80, 81–153): Streaming exec `runStreamingExec()`, readline setup, SIGINT handler, 30 s drain timeout (`L0M`).
- **3425.js** (lines 1–55): ACP daemon `runAcpDaemon()`, signal handling, adapter initialization.
- **3426.js** (lines 1–60): ACP child `runAcpChild()`, signal handling, capability parsing.
- **2101.js** (lines 1–40): Kitty protocol `XnH()`, ANSI probe, raw mode toggle, 150 ms timeout.
