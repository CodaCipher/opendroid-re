# infra-cli-entry: Architecture Notes

## Overview

OpenDroid's CLI entry point is built on **Commander.js** (module 0209.js), a battle-tested Node.js CLI framework. The main entry point is module **4071.js** (48KB, `bA0` module), which orchestrates the entire CLI lifecycle: console redirection to log files, telemetry initialization, auto-update checks, settings initialization, and Commander program assembly. The CLI supports two primary modes: **interactive** (default `droid [prompt]`) and **non-interactive** (`droid exec [options] <prompt>`). Seven subcommands are registered: `exec`, `mcp`, `plugin`, `daemon`, `search`/`find`, `ssh`, `computer`, and `update`. Global options include `--resume`, `--version`, and `--help`. The exec subcommand (module 3710.js) is the most feature-rich, supporting autonomy levels, model selection, streaming modes, session continuation, and tool management.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 4071.js | 48KB | Main CLI entry point: Commander program assembly, startup sequence, interactive mode handler | App |
| 3710.js | 33KB | `droid exec` subcommand: non-interactive mode with full flag set (model, autonomy, tools, sessions) | App |
| 0209.js | 33KB | Commander.js library: `Command` class (BPA), option parsing, subcommand dispatch, help generation | Vendor (commander) |
| 3717.js | 3KB | `droid mcp` subcommand: MCP server management (add/remove) | App |
| 3724.js | 3KB | `droid plugin` subcommand: plugin lifecycle (install/uninstall/update/list) | App |
| 3783.js | 3KB | Daemon command shell: delegates to full daemon module (3783.js inner) | App |
| 3785.js | 3KB | `droid search`/`find` subcommand: session search with typo-tolerant matching | App |
| 3787.js | 7KB | `droid ssh` subcommand: SSH tunnel management with WebSocket relay | App |
| 3788.js | 5KB | `droid computer` subcommand: relay computer registration management | App |
| 3789.js | 6KB | `droid update` subcommand: version check and binary update | App |
| 1549.js | 3374KB | Mega-bundle (not directly CLI-related; contains bundled dependencies) | Mixed |
| 2772.js | 24KB | Google IAM protobuf definitions (seed module: NOT CLI-related) | Vendor |
| 3159.js | 16KB | OpenTelemetry semantic conventions (seed module: NOT CLI-related) | Vendor |
| 2774.js | 22KB | Google operations protobuf (seed module: NOT CLI-related) | Vendor |
| 0110.js | 12KB | ASP.NET identity auth constants (seed module: NOT CLI-related) | Vendor |
| 2773.js | 20KB | Google locations protobuf (seed module: NOT CLI-related) | Vendor |

> **Note:** Seed modules 2772.js, 3159.js, 2774.js, 0110.js, and 2773.js are protobuf/vendor modules unrelated to the CLI system. The actual CLI modules were discovered through grep-based discovery: 4071.js, 3710.js, 0209.js, 3717.js, 3724.js, 3783.js, 3785.js, 3787.js, 3788.js, 3789.js.

## Architecture

### CLI Startup Sequence

The CLI follows a carefully ordered initialization sequence (4071.js, lines 1440-1674):

1. **Console Redirection** (lines 16-95): All `console.*` calls are intercepted and logged to `~/.opendroid/logs/console.log` with timestamps
2. **PTY Library Setup** (lines 101-112): Platform-specific PTY library paths are resolved (Bun runtime)
3. **Telemetry Initialization** (lines 1400-1436): OpenTelemetry tracing client (`apA` class) is initialized if `OPENDROID_OTEL_ENABLED=true`
4. **Global Error Handlers** (lines 1438-1450): `uncaughtException` and `unhandledRejection` handlers installed
5. **Bootstrap Metrics** (line 1452): `cli_startup_bootstrap_latency` counter recorded
6. **Auth Validation** (lines 1473-1478): `EO()` checks authentication validity
7. **Certificate Loading** (line 1479): `EFI()` loads SSL certificates
8. **Auto-Update Check** (lines 1480-1510): Parallel with settings init; skipped for `--help`, `--version`, and subcommands
9. **Settings Initialization** (lines 1500-1510): `TvH.initialize()`, autonomy defaults, built-in droids
10. **Task Tool Manager** (lines 1517-1518): `ensureTaskToolManagerInitialized()`
11. **Commander Program Assembly** (lines 1520-1640): Main program + all subcommands registered
12. **Parse & Execute** (line 1641): `M.parseAsync(process.argv)`

### Command Dispatch Architecture

```
droid (root Command)
├── [prompt...]  → Interactive mode (default action)
├── --resume     → Resume last session or specific session
├── --version    → Version output
├── --help       → Help output
├── exec         → Non-interactive mode (3710.js)
│   ├── --model <id>
│   ├── --auto <level>     (low|medium|high)
│   ├── --output-format    (text|stream-json|acp|acp-daemon|debug)
│   ├── --input-format     (stream-json|stream-jsonrpc)
│   ├── --session-id <id>
│   ├── --reasoning-effort
│   ├── --enabled-tools / --disabled-tools
│   └── --skip-permissions-unsafe
├── mcp          → MCP server management (3717.js)
│   ├── add <name> <urlOrCommand>
│   └── remove <name>
├── plugin       → Plugin management (3724.js)
│   ├── install <plugin>
│   ├── uninstall <plugin>
│   ├── update [plugin]
│   └── list
├── daemon       → Daemon server (3783.js)
│   ├── --port <number>
│   ├── --host <address>
│   ├── --unix <path>
│   ├── --debug
│   └── --droid-path <path>
├── search/find  → Session search (3785.js)
│   ├── <query>
│   ├── --kind (message_text|document|tool_use|tool_result|all)
│   └── --limit-sessions / --limit-hits / --json / --reindex
├── ssh          → SSH tunnel (3787.js)
├── computer     → Relay computer management (3788.js)
│   ├── register [name]
│   └── remove
└── update       → Version update (3789.js)
    ├── --check
    └── --version <version>
```

### Global Flags & Precedence

1. **`-v, --version`**: Outputs version (`0.64.0`), parsed first by Commander
2. **`-h, --help`**: Displays help text, parsed by Commander
3. **`-r, --resume [sessionId]`**: Resume session (root-level option only)
4. **`[prompt...]`**: Positional arguments joined as initial prompt for interactive mode

Flag precedence (exec subcommand):
- CLI flags (`--model`, `--auto`, etc.) > session settings > global defaults
- `--skip-permissions-unsafe` overrides `--auto` level (mutually exclusive conceptually)
- `--enabled-tools` / `--disabled-tools` are additive to the model's default tool set

## Key Findings

### 1. Commander.js as CLI Framework (0209.js)

The CLI uses **Commander.js** (class `BPA` / exported as `Command`). Key characteristics:
- Located at 0209.js, line 16-999
- Full subcommand support via `.addCommand()` and `.command()` methods
- Option parsing with `.option()`, `.addOption()`, `.argument()`
- Help generation via `_helpFlags`, `_helpCommandName`, `outputHelp()`
- Error handling with `_showHelpAfterError`, `_showSuggestionAfterError`
- Exports: `NAE.Command = BPA` (line 999)

### 2. Main Entry Point is Module 4071.js

Module 4071.js (`bA0`) is the primary CLI entry point:
- **Console logging** intercepts ALL console output to `~/.opendroid/logs/console.log` (lines 16-95)
- **Auto-update** runs in parallel with settings init, but skips for help/version/subcommands (lines 1480-1510)
- **Program name**: `droid`: set at line 1562
- **Version**: `0.64.0`: hardcoded at line 1560
- **7 subcommands** registered dynamically via lazy imports
- **Interactive mode** is the default action handler (lines 1614-1640)

### 3. Exec Command is the Most Complex Subcommand (3710.js)

The `exec` command (3710.js, `ZuD` module) implements non-interactive execution with:
- **4 output formats**: `text`, `stream-json`, `acp` (agent control protocol), `acp-daemon`
- **3 input formats**: default (stdin), `stream-json`, `stream-jsonrpc`
- **Autonomy levels**: `low`, `medium`, `high`, plus `--skip-permissions-unsafe`
- **Session continuation**: `--session-id` loads existing session context
- **Model selection**: `--model <id>` with custom model support (`custom:model-name`)
- **Tool controls**: `--enabled-tools`, `--disabled-tools`, `--list-tools`
- **Internal flags**: `--depth`, `--calling-session-id`, `--calling-tool-use-id` (for subagent spawning)

### 4. Seed Modules are NOT CLI-Related

The 5 non-mega-bundle seed modules (2772, 3159, 2774, 0110, 2773) are all vendor/infrastructure modules:
- 2772.js = Google IAM protobuf (`PAL` module)
- 3159.js = OpenTelemetry semantic conventions / 1C:Enterprise (`MVD` module)
- 2774.js = Google operations protobuf (`aeH` module)
- 0110.js = ASP.NET identity auth constants (`aKL` module)
- 2773.js = Google locations protobuf (`BAL` module)

The actual CLI modules were discovered via grep: 4071.js, 3710.js, 0209.js, 3717.js, 3724.js, 3783.js, 3785.js, 3787.js, 3788.js, 3789.js.

### 5. Lazy Loading Pattern for Subcommands

All subcommands are loaded via dynamic imports:
```javascript
let { createExecCommand: U } = await SuD().then(() => juD);
let { createMcpCommand: P } = await Promise.resolve().then(() => (JBL(), ouD));
```
This ensures fast startup for `--help` and `--version` without loading heavy subcommand handlers.

### 6. Daemon Command is Double-Wrapped

The `daemon` subcommand has a unique pattern (4071.js, lines 1584-1596):
- A lightweight shell command is registered in the main program
- When invoked, it dynamically loads the full `createDaemonCommand` and re-parses `process.argv`
- This allows daemon-specific options to be parsed correctly

## Code Examples

### Main Program Assembly (4071.js, lines 1555-1641)
```javascript
let M = new mP();  // mP = Commander's Command class
M.exitOverride();
M.allowUnknownOption(true);
M.name("droid")
  .description("Droid - OpenDroid's AI coding agent in your terminal")
  .version("0.64.0", "-v, --version", "output the version number")
  .helpOption("-h, --help", "display help for command");

// Register subcommands
M.addCommand(U(H));     // exec
M.addCommand(P());      // mcp
M.addCommand(B());      // plugin
M.addCommand(W);         // daemon
M.addCommand(V());      // search
M.addCommand(X());      // ssh
M.addCommand(Q());      // computer
M.addCommand(w());      // update

// Root-level options
M.addOption(new jB("-r, --resume [sessionId]", "Resume a session"));
M.arguments("[prompt...]").action(async (C, Y) => { /* interactive mode */ });

await M.parseAsync(process.argv);
```

### Exec Command Options (3710.js, lines 484-540)
```javascript
return new mP("exec")
  .description("Execute a single command (non-interactive mode)")
  .option("-o, --output-format <format>", "Output format", "text")
  .option("--auto <level>", "Autonomy level: low|medium|high")
  .option("-m, --model <id>", "Model ID to use")
  .option("-r, --reasoning-effort <level>", "Reasoning effort")
  .option("--enabled-tools <ids>", "Enable specific tools", zuD, [])
  .option("--disabled-tools <ids>", "Disable specific tools", zuD, [])
  .option("--skip-permissions-unsafe", "Skip ALL permission checks")
  .option("-s, --session-id <id>", "Continue existing session")
  .action(async ($, I) => { /* execution logic */ });
```

### Console Redirection (4071.js, lines 16-95)
```javascript
let H = P9A.join(eA0.homedir(), sL),
    A = P9A.join(H, "logs");
if (!NWH.existsSync(A)) NWH.mkdirSync(A, { recursive: true });
// Redirects console.log/info/debug/warn/error to file
console.log = E("info");
console.error = E("error");
```

## Integration Points

### Cross-System Dependencies

- **Config Loader** → Settings initialization (`TvH.initialize()`, `BA()`) at 4071.js, line 1505: CLI flags feed into the config system
- **Session Manager** → `--resume` loads sessions via `BA().getAllNonEmptySessions()` and `BA().loadSession()` at 4071.js, lines 1616-1640
- **Plugin System** → `droid plugin` subcommand (3724.js) manages plugin lifecycle; auto-install at startup (4071.js, line 1466)
- **Auto-Updater** → `droid update` subcommand (3789.js); auto-update check at 4071.js, line 1490
- **Daemon/IPC** → `droid daemon` subcommand (3783.js) starts WebSocket server; exec `--output-format acp-daemon` connects to daemon
- **Telemetry (OTel)** → `CliOtelTracingClient` (4071.js, lines 1400-1436) initialized before CLI parsing
- **Tool System** → `--enabled-tools`/`--disabled-tools` in exec (3710.js, lines 502-506) control tool availability
- **Auth System** → `EO()` auth check at startup (4071.js, line 1473); API key via `OPENDROID_API_KEY` env var
- **MCP (Model Context Protocol)** → `droid mcp` subcommand (3717.js) manages MCP server connections
- **Search** → `droid search`/`find` (3785.js) uses session indexing; cache warmed at startup (4071.js, line 1464)
- **SSH/Computer** → `droid ssh` (3787.js) and `droid computer` (3788.js) for remote relay operations
- **TUI** → Default interactive mode launches TUI via `{ main }` at 4071.js, line 1634

### Mission Cross-References
- **01-terminal-ui (TUI)**: The `[prompt...]` default action launches the TUI via `main(x, v, H, g, D)` (4071.js, line 1634)
- **02-orchestration (Mission/Orchestration)**: `--depth` and `--calling-session-id` flags support subagent spawning (3710.js, lines 528-534)
- **03-tool-agent-system (Tool System)**: `--enabled-tools`/`--disabled-tools`/`--list-tools` (3710.js, lines 502-506)
- **04-desktop-gui (GUI)**: Daemon mode serves the GUI via WebSocket; `droid computer` registers relay endpoints

## Implementation Notes

1. **Replace Commander.js** with any CLI framework (yargs, oclif, etc.): the interface is straightforward. The key abstraction is `Command` class with `.option()`, `.command()`, `.action()` methods
2. **Entry point** (4071.js) should be split into separate modules: console-logger, telemetry-init, startup-sequence, program-assembly
3. **Subcommands** are already modular (each in its own file): port each independently
4. **Console redirection** pattern (intercepting console.* to file) is Bun-specific; for Node.js use proper winston/pino transports
5. **Auto-update** mechanism uses Bun-specific APIs (`Bun.spawnSync`): replace with `child_process.spawnSync` or platform-native update mechanisms
6. **Exec modes** (`stream-json`, `stream-jsonrpc`, `acp`, `acp-daemon`) are OpenDroid-specific protocols: either implement compatible protocols or design replacements

## Module Reference

| File | Lines | Function/Class | Description |
|------|-------|----------------|-------------|
| 4071.js | 1-100 | Console redirect setup | Intercepts all console output to `~/.opendroid/logs/console.log` |
| 4071.js | 101-112 | PTY library setup | Resolves platform-specific PTY binary paths |
| 4071.js | 1400-1436 | `class apA extends TRH` | CLI OpenTelemetry tracing client |
| 4071.js | 1438-1450 | Error handlers | `uncaughtException` / `unhandledRejection` |
| 4071.js | 1452-1520 | Startup sequence | Auth check, cert load, auto-update, settings init |
| 4071.js | 1555-1641 | Commander program assembly | `new mP()`, subcommand registration, `parseAsync()` |
| 4071.js | 1555 | `new mP()` | Root Commander program creation |
| 4071.js | 1560 | `.version("0.64.0")` | Version flag registration |
| 4071.js | 1562 | `.name("droid")` | Program name |
| 4071.js | 1584-1596 | Daemon command shell | Double-wrapped daemon registration |
| 4071.js | 1614-1640 | Default action handler | Interactive mode / session resume |
| 4071.js | 1641 | `M.parseAsync(process.argv)` | CLI entry: parses arguments |
| 3710.js | 1-80 | Module setup | Helper functions (`H7`, `Mw`, `zuD`, `DNM`) |
| 3710.js | 81-100 | Token usage helper | `huD()` returns token counts |
| 3710.js | 300-470 | `QNM()` | Help text generation for exec command |
| 3710.js | 484-540 | `JNM()` / `createExecCommand` | Exec command factory: all option registrations |
| 3710.js | 540-700 | Exec action handler | Main exec logic: mode selection, session management |
| 3710.js | 560-570 | Streaming mode dispatch | `stream-json`, `stream-jsonrpc`, `acp`, `acp-daemon` |
| 3710.js | 600-700 | Default exec mode | Stdin/file prompt handling, session creation |
| 0209.js | 16-99 | `class BPA` constructor | Commander.js Command class initialization |
| 0209.js | 88-130 | `copyInheritedSettings()` | Subcommand settings inheritance |
| 0209.js | 109-128 | `.createCommand()`, `.addCommand()` | Subcommand registration |
| 0209.js | 130-132 | `.addCommand()` validation | Ensures command has a name |
| 0209.js | 999 | `NAE.Command = BPA` | Commander export |
| 3717.js | 19-72 | `kNM()` / `createMcpCommand` | MCP server management (add/remove) |
| 3724.js | 10-83 | `gNM()` / `createPluginCommand` | Plugin lifecycle management |
| 3783.js | 15-81 | `ahM()` / `createDaemonCommand` | Daemon server startup with WebSocket |
| 3785.js | 8-83 | `shM()` / `createSearchCommand` | Session search with filtering |
| 3787.js | 20-178 | `IjM()` / `createSshCommand` | SSH tunnel via WebSocket relay |
| 3788.js | 29-116 | `MjM()` / `createComputerCommand` | Relay computer registration |
| 3789.js | 37-154 | `WjM()` / `createUpdateCommand` | Version check and binary update |
