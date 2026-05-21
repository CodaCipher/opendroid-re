# Execute Tool (Shell Command Execution): Tool System Architecture Notes

## Overview

The Execute tool is OpenDroid's shell command execution engine, enabling the AI agent to run arbitrary shell commands with controlled risk assessment, timeout enforcement, and background process management. The system uses Node.js `child_process.spawn` under the hood, with a Windows/Unix bifurcation: on Windows, commands are routed through PowerShell (`-ExecutionPolicy Bypass -Command`), while on Unix/macOS they use bash. A three-tier risk level system (low/medium/high) acts as a gate for user approval before execution. The fireAndForget mode enables long-running background processes via `nohup` (Unix) or `Start-Process` (Windows). Dangerous command patterns are blocked pre-execution with regex-based detection.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 1201.js | 241 lines | Execute tool input schema (zod), risk level schema, tool factory `eI()` | App |
| 2335.js | 285 lines | ShellExecutor class: cross-platform spawn, Windows/Unix process creation, secret scrubbing | App |
| 2339.js | 461 lines | Execute tool handler: streaming output, timeout, fireAndForget, dangerous command detection, interactive command blocking | App |
| 1183.js | 33 lines | PowerShell discovery: cached `pwsh.exe`/`powershell.exe` resolution, `WindowsPowerShellNotFoundError` | App |
| 1123.js | 20 lines | cross-spawn vendor wrapper: `spawn()`, `spawnSync()`, ENOENT handling | Vendor (cross-spawn) |
| 1256.js | 17 lines | Risk level normalization: `GFH()` maps string to low/medium/high, defaults to "high" | App |
| 0966.js | 33 lines | Tool name constants: `C4H = "Execute"`, risk level enum `TO` declaration | App |
| 0944.js | 152 lines | System wake-lock manager: PowerShell/pwsh resolution, `commandExists()` | App |
| 3867.js | 64 lines | UI layer: tracks background Execute processes and Task subagents for progress display | App (GUI/02-orchestration integration) |
| 3428.js | 426 lines | JSON-RPC transport: `request_permission`, `ask_user` methods, process.stdout.write | App |

## Architecture

### Execution Pipeline

```
LLM tool_call("Execute", {command, cwd, riskLevel, timeout, fireAndForget})
  │
  ├── 1. Risk Level Validation (1256.js GFH)
  │     └── Normalize to low|medium|high, default "high"
  │
  ├── 2. Permission Gate (3428.js request_permission)
  │     └── User approval based on riskLevel + approved_tools
  │
  ├── 3. Pre-execution Safety Checks (2339.js)
  │     ├── Dangerous command regex blocklist
  │     ├── Interactive command detection (vim, nano, top, REPLs)
  │     └── fireAndForget safety validation
  │
  ├── 4a. [Normal] ShellExecutor.execute() → streaming
  │     ├── Unix: child_process.spawn(cmd, {shell: "bash"})
  │     ├── Windows: spawn(powershell, ["-ExecutionPolicy", "Bypass", "-Command", cmd])
  │     └── Fallback: spawn(cmd, {shell: true})
  │
  ├── 4b. [Background] ShellExecutor.execute({fireAndForget: true})
  │     ├── Unix: nohup bash -c 'cmd' > /tmp/droid-bg-*.out 2>&1
  │     └── Windows: Start-Process -FilePath powershell -WindowStyle Hidden -PassThru
  │
  ├── 5. Output Capture + Secret Scrubbing (2335.js scrubAndUpdateOutput)
  │     └── Rolling window (100 lines) for secret redaction
  │
  └── 6. Result Serialization
        ├── Normal: streaming updates + final result
        └── Background: PID + output file path
```

### PowerShell vs Bash Detection

The system detects the platform at schema generation time (1201.js line 47):
```js
yZ$.platform() === "win32"
  ? "The command is executed using Windows Powershell with `-ExecutionPolicy Bypass -Command`"
  : "The command is executed using the user's default MacOS shell (likely zsh)"
```

PowerShell executable resolution (1183.js):
- Tries `pwsh.exe` first, then `powershell.exe` (list `LZ$ = ["pwsh.exe", "powershell.exe"]`)
- Caches the found executable in module-level variable `fC`
- Falls back to `System32\WindowsPowerShell\v1.0\powershell.exe` (1138.js)
- Throws `WindowsPowerShellNotFoundError` if neither found

### Risk Level System

Three risk levels defined via enum `TO` (0966.js):

| Level | Examples | Approval Behavior |
|-------|----------|-------------------|
| `low` | `pwd`, `ls`, `git status`, `cat` | Usually auto-approved |
| `medium` | `npm install`, `git commit`, `mkdir`, `cp` | May require approval |
| `high` | `sudo rm -rf`, `curl \| bash`, `git push` | Always requires approval |

The risk level is provided by the LLM as `riskLevel: { value: "low"|"medium"|"high", reason: "..." }` and validated via `GFH()` in 1256.js which normalizes and defaults to `"high"` for invalid values.

### Timeout Mechanism

- Default timeout: **2000ms** (schema in 1201.js line 131) for internal tool calls, **60 seconds** for the Execute handler (2339.js line 175)
- `ignoreTimeout` flag available for long operations (git fetch, npm ci, etc.)
- Timeout kills the process via terminal manager's `kill(terminalId)`
- Partial output is captured and returned before timeout message

### Fire-and-Forget (Background Processes)

When `fireAndForget: true`:
1. **Safety blocklist checked first**: rm -rf, sudo, pipe to bash, eval, curl|bash, etc.
2. **Unix path**: Uses `nohup bash -c 'cmd' > /tmp/droid-bg-{timestamp}.out 2>&1` with `detached: true`
3. **Windows path**: Uses `Start-Process -FilePath powershell -WindowStyle Hidden -PassThru` to get PID
4. Returns `{pid, isComplete, outputFile}`: the LLM is told to read the output file later
5. Process registry (`y2.registerProcess()`) tracks background PIDs for later cleanup

### Secret Scrubbing

Output goes through `scrubAndUpdateOutput()` (2335.js) which:
- Maintains a rolling window of last 100 lines
- Applies secret detection via `Pf()` (external module)
- Redacts secrets in the combined output before sending to LLM

## Key Findings

### 1. Execute Tool Input Schema (1201.js, lines 44-165)

```js
bZ$ = p.object({
  command: p.string().describe(`Command to execute.
  - Commands are spawned using the Node.js child_process.spawn function with shell: true...
  - ${yZ$.platform() === "win32" ? "Windows Powershell..." : "default MacOS shell (likely zsh)"}...`),
  cwd: p.string().describe("REQUIRED: Directory where the command will be executed..."),
  riskLevel: p.object({
    value: p.nativeEnum(TO),
    reason: p.string().describe("A concise one-sentence explanation justifying the chosen risk level."),
  }).describe(`REQUIRED: the risk level...`),
  timeout: p.number().optional().default(2000),
  ignoreTimeout: p.boolean().optional().default(false),
})
```

### 2. Dangerous Command Blocklist (2339.js, lines 176-191)

```js
let D = [
  /^rm\s+-rf\s+\/\s*$/,           // rm -rf /
  /^rm\s+-rf\s+\/\*/,             // rm -rf /*
  /^rm\s+-rf\s+~\s*$/,            // rm -rf ~
  /^rm\s+-rf\s+~\//,              // rm -rf ~/
  /^rm\s+-rf\s+\$HOME/,           // rm -rf $HOME
  /:\(\)\{\s*:\|\s*:\s*&\s*\};/,  // fork bomb
  />\s*\/dev\/sda/,                // overwrite device
  /dd\s+if=\/dev\/zero\s+of=\/dev/, // dd to device
]
```

### 3. Windows Process Creation (2335.js, lines 62-82)

```js
static createWindowsProcess({ command, cwd, caller, childEnv, fireAndForget, outputFile }) {
  let E = M1H(),  // Get cached PowerShell path
      f = ["-ExecutionPolicy", "Bypass"];
  if (caller === "FILE_API") f.push("-NoProfile");
  if (fireAndForget) {
    let M = command.replace(/"/g, '`"'),
        P = `Start-Process -FilePath '${E}' -ArgumentList '-ExecutionPolicy Bypass -Command "& { ${M} }"' -WindowStyle Hidden -PassThru | Select-Object -ExpandProperty Id`;
    f.push("-Command", P);
    return k7H(E, f, { cwd, windowsHide: true, env: childEnv, stdio: ["ignore", "pipe", "pipe"], detached: false, shell: false });
  }
  f.push("-Command", command);
  return k7H(E, f, { cwd, windowsHide: true, env: childEnv, stdio: ["ignore", "pipe", "pipe"], detached: false, shell: false });
}
```

### 4. Unix Process Creation (2335.js, lines 38-56)

```js
static createUnixProcess({ command, cwd, childEnv, fireAndForget, outputFile }) {
  let D = command;
  if (fireAndForget) {
    D = `nohup bash -c '${command.replace(/'/g, "'\\''")}' > '${outputFile || "/dev/null"}' 2>&1`;
    return k7H(D, [], {
      cwd, windowsHide: true,
      env: { ...childEnv, TERM: "dumb", PS1: "", DEBIAN_FRONTEND: "noninteractive" },
      stdio: ["ignore", "ignore", "pipe"], shell: "bash", detached: true,
    });
  }
  return k7H(command, [], {
    cwd, windowsHide: true,
    env: { ...childEnv, TERM: "dumb", PS1: "", DEBIAN_FRONTEND: "noninteractive" },
    stdio: ["ignore", "pipe", "pipe"], shell: "bash", detached: false,
  });
}
```

### 5. PowerShell Discovery with Caching (1183.js)

```js
LZ$ = ["pwsh.exe", "powershell.exe"];  // Priority order
var fC;  // Cached PowerShell path

function M1H() {
  if (fC === null) throw new WindowsPowerShellNotFoundError();
  if (fC) return fC;
  return EZA((H) => {
    return (ND1('Write-Output "ok"', { shell: H, stdio: "ignore", windowsHide: true }), H);
  });
}
```

### 6. Background Process Safety Check (2339.js, lines 220-248)

```js
// fireAndForget safety patterns
let U = [
  { pattern: /\brm\b.*-[rRfF]/, desc: "recursive/force rm" },
  { pattern: /\bsudo\b/, desc: "sudo elevation" },
  { pattern: /\|.*\bbash\b/, desc: "piping to bash" },
  { pattern: /\beval\b/, desc: "eval command" },
  { pattern: /\bcurl\b.*\|\s*bash/, desc: "curl piped to bash" },
  // ... more patterns
];
```

## Integration Points

- **Tool Registry (tool-core-registry)**: Execute tool registered via `eI()` factory in 1201.js as `C4H = "Execute"` (0966.js)
- **Permissions Engine (tool-permissions)**: Risk level values (low/medium/high) gate user approval flow via `request_permission` RPC in 3428.js
- **Sandbox (tool-sandbox)**: Execute tool calls go through sandbox validation before reaching ShellExecutor
- **Agent State (tool-agent-state)**: Background process registry tracks PIDs per tool call ID for cleanup on state transitions
- **Streaming (tool-streaming)**: Normal execution yields `type: "update"` chunks during execution and `type: "result"` on completion
- **TUI/GUI (01-terminal-ui/04-desktop-gui)**: 3867.js renders background process list in UI with PID, command, output file
- **JSON-RPC Transport (3428.js)**: Permission prompts and ask_user flows go through `process.stdout.write` JSON-RPC protocol

## Implementation Notes

### Execute Tool API

```
Tool Name: "Execute"
Input Schema:
  command: string (required): shell command to run
  cwd: string (required): absolute working directory
  riskLevel: { value: "low"|"medium"|"high", reason: string } (required)
  timeout: number (default: 60 seconds for handler, 2000ms for schema)
  ignoreTimeout: boolean (default: false)
  fireAndForget: boolean (default: false)

Output:
  Normal: { type: "result", value: string, isError: boolean }
  Background: { pid: number, isComplete: boolean, outputFile: string }
```

### Key Interfaces to Implement

1. **ShellExecutor** (2335.js): Platform-aware spawn with `createWindowsProcess()`, `createUnixProcess()`, `createFallbackProcess()`
2. **PowerShell Discovery**: Cache-first `pwsh.exe`/`powershell.exe` resolution
3. **Dangerous Command Detection**: Regex blocklist + interactive command detection
4. **Background Process Manager**: `nohup`/`Start-Process` with output file tracking
5. **Secret Scrubbing**: Rolling window output redaction

### Risk Level Implementation Notes

- Default to "high" for any unrecognized risk level (1256.js `GFH()`)
- The LLM provides both `value` and `reason`: the reason is surfaced to the user
- Three-tier system (low/medium/high) with explicit examples in the tool's system prompt description

## Open Questions / Left Undone

- **TO enum definition**: `TO` is declared in 0966.js but initialized elsewhere (likely in a dependency module not yet traced): the values are confirmed as "none", "low", "medium", "high" from usage context
- **`BvH` environment variable**: Referenced in ShellExecutor.execute() but definition not traced (likely contains additional env vars injected into child processes)
- **`Pf()` secret scrubbing implementation**: Called in scrubAndUpdateOutput but the actual secret detection logic is in an external module not analyzed
- **Terminal Manager (`I = Pm()`)**: The streaming execution uses a terminal abstraction (`create()`, `getOutput()`, `kill()`, `release()`) whose implementation was not traced
- **`y2.registerProcess()` background registry**: Process tracking for cleanup on session end: implementation not analyzed
- **`RoH()` command pre-validation**: Called before streaming execution, likely a shell-quote or command validation step: not traced
- **`VV.getExtractedCommands()`**: Command extraction/parsing utility: not traced
- **Permission approval persistence**: How risk-level approvals are stored across sessions: overlaps with tool-permissions feature
