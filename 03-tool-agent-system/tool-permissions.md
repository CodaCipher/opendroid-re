# Tool Permissions Engine: Tool System Architecture Notes

## Overview

OpenDroid's permission engine governs whether tool calls require user confirmation before execution. The system operates on a layered autonomy model combining **3 interaction modes** (`auto`, `spec`, `agi`) with **4 autonomy levels** (`off`, `low`, `medium`, `high`) to produce **5 composite autonomy modes** (`normal`, `spec`, `auto-low`, `auto-medium`, `auto-high`). Each tool call is classified into one of **7 confirmation types** (exec, create, edit, apply_patch, exit_spec_mode, start_mission_run, mcp_tool) and the user is presented with one of **7 outcome options** (proceed_once, proceed_always, proceed_edit, proceed_auto_run_low/medium/high, cancel): totaling 14 distinct permission decision points. The engine includes a command denylist/allowlist system, risk-level-based auto-approval, and per-session permission state.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 3139.js | 207 lines | `hWD()`: Confirmation type resolver; decides if a tool needs permission | App |
| 3141.js | ~230 lines | `UYH()`: Permission option generator; `ee()`: Outcome processor | App |
| 2324.js | ~140 lines | `lCI` class (VV): Command denylist/allowlist checker | App |
| 0885.js | ~105 lines | Autonomy mode/level conversion utilities (`QQ`, `ao`, `H4H`, `MO`) | App |
| 1185.js | 1823 lines | `OGH` session service: autonomy level management, permission queries | App |
| 0724.js | ~45 lines | `J_$` class (nf): Session permission state (skipAll, allowedToolIds) | App |
| 3417.js | ~220 lines | `gIA` AcpAdapter: bridges permission requests over IPC | App |
| 3418.js | ~40 lines | `gYH` PermissionRequestHandler: generic permission request handler | App |
| 3413.js | ~20 lines | `t6D()`: Outcome-to-kind classifier (allow_once/allow_always/reject_once) | App |
| 3416.js | ~40 lines | `vYH()`: Single-tool permission option builder for ACP protocol | App |
| 0212.js | ~43 lines | Default command denylist (rm -rf /, mkfs, dd, shutdown, etc.) | App |
| 3137.js | ~25 lines | `eEL()`: Impact level to numeric value converter | App |
| 1256.js | ~20 lines | `GFH()`: Risk level string normalizer | App |
| 3140.js | ~44 lines | `SWD()`: Autonomy mode to human-readable description | App |

## Architecture

### 1. Autonomy Model (3 modes × 4 levels → 5 composite modes)

```
InteractionMode (m_):  auto | spec | agi
AutonomyLevel (d_):    off  | low  | medium | high
AutonomyMode (I8):     normal | spec | auto-low | auto-medium | auto-high
```

Conversion logic (0885.js):
- `QQ(interactionMode, autonomyLevel)` → AutonomyMode composite string
- `ao(autonomyMode)` → { mode, level } decomposition
- `H4H(level)` → numeric (off=0, low=1, medium=2, high=3)
- `MO(level, maxLevel)` → boolean: can user set this level?

Level hierarchy: `off < low < medium < high`. Organization can set `maxAutonomyLevel` to cap user autonomy.

### 2. Confirmation Type Resolution (`hWD` in 3139.js)

When a tool call arrives, `hWD(toolName, input)` determines if confirmation is needed:

```
Tool Name → Confirmation Decision:
─────────────────────────────────────────────────
Read, LS        → false (always auto-approved)
AskUser         → false (always auto-approved)
Create          → "create" IF !shouldAutoApproveFileEdits()
Edit            → "edit"   IF !shouldAutoApproveFileEdits()
ApplyPatch      → "apply_patch" IF !shouldAutoApproveFileEdits()
Execute         → "exec"   based on riskLevel vs autonomyLevel
                  OR if command is denied → always "exec" (high impact)
ExitSpecMode    → "exit_spec_mode"
ProposeMission  → "propose_mission"
StartMissionRun → "start_mission_run" (with concurrent run warning)
{server}___{tool} → "mcp_tool" (based on readOnlyHint + autonomy)
```

**Auto-approval rules:**
- `nf().getSkipAllConfirmations()` → skip all
- `shouldAutoApproveFileEdits()` → skip create/edit/apply_patch (true when AGI mode or autoRun mode)
- For Execute: if `impactLevel_numeric <= autonomyLevel_numeric` → auto-approve
- For MCP tools: same impact vs autonomy check, with `readOnlyHint=true` → low impact

### 3. Permission Option Generation (`UYH` in 3141.js)

Based on confirmation context, user-facing options are generated:

**Standard mode (single tool):**
| Option | Value | Kind |
|--------|-------|------|
| Yes, allow | `proceed_once` | allow_once |
| Yes, and always allow {level} | `proceed_always` | allow_always |
| No, cancel | `cancel` | reject_once |

**Standard mode (batch tools):**
| Option | Value | Kind |
|--------|-------|------|
| Yes, allow all | `proceed_once` | allow_once |
| Yes, and always allow {level} | `proceed_always` | allow_always |
| No, cancel all | `cancel` | reject_once |

**Spec mode exit:**
| Option | Value | Kind |
|--------|-------|------|
| Proceed with implementation | `proceed_once` | allow_once |
| Proceed, and allow file edits... (Low) | `proceed_auto_run_low` | allow_always |
| Proceed, and allow reversible commands (Medium) | `proceed_auto_run_medium` | allow_always |
| Proceed, and allow all commands (High) | `proceed_auto_run_high` | allow_always |
| No, keep iterating on spec | `cancel` | reject_once |

**Denied command / mission run context:**
Simplified to just `proceed_once` and `cancel`.

### 4. Outcome Processing (`ee` in 3141.js)

```javascript
function ee({ outcome, tools, approvedToolIds, updateAction, selectedOptionName }) {
  if (outcome === "cancel") return [];                        // reject all
  if (outcome === "proceed_always") {
    o8().setAutonomyLevel(a2A(sEL(tools)));                  // upgrade autonomy
    return tools.map(f => f.toolUseId);                       // approve all
  }
  if (outcome === "proceed_edit") {
    o8().setAutonomyMode("normal");                           // exit spec → normal
    return tools.map(E => E.toolUseId);
  }
  if (["proceed_auto_run_low","proceed_auto_run_medium","proceed_auto_run_high"].includes(outcome)) {
    o8().setAutonomyMode(outcome === "proceed_auto_run_low" ? "auto-low" : ...);
    return tools.map(f => f.toolUseId);
  }
  // Default: return approvedToolIds or all tool IDs
  let D = approvedToolIds || tools.map(E => E.toolUseId);
  return D;
}
```

### 5. Command Denylist / Allowlist (2324.js, `lCI` class)

Singleton `VV` manages command-level security:

**Denylist patterns** (0212.js defaults):
```javascript
r$H = [
  "rm -rf /", "rm -rf /*", "rm -rf .", "rm -rf ~", "rm -rf ~/*",
  "rm -rf $HOME", "rm -r /", "rm -r /*", "rm -r ~", "rm -r ~/*",
  "mkfs", "mkfs.ext4", "mkfs.ext3", "mkfs.vfat", "mkfs.ntfs",
  "dd if=/dev/zero of=/dev", "dd of=/dev",
  "shutdown", "reboot", "halt", "poweroff", "init 0", "init 6",
  ":(){ :|: & };:", ":() { :|:& };:",
  "chmod -R 777 /", "chmod -R 000 /", "chown -R",
  "Format-Volume", "format.com",
  "powershell Remove-Item -Recurse -Force"
];
```

**Evaluation logic:**
- `isCommandDenied(cmd)` → regex-based pattern matching against denylist
- `isCommandAllowed(cmd)` → exact match in allowlist Set
- Denied commands always require confirmation with `impactLevel: "high"`
- Allowed commands skip confirmation

### 6. Session Permission State (0724.js, `J_$` class)

Per-session state accessible via `nf()`:
```javascript
class J_$ {
  allowedToolIds = null;          // Set of approved tool call IDs
  skipAllConfirmations = false;   // bypass all permission checks
  askUserToolEnabled = false;     // whether AskUser tool is active
  enabledToolIds = [];            // which tools are enabled
  depth = 0;                      // nesting depth (subagents)
}
```

### 7. Escalation Flow (End-to-End)

```
Tool Call Arrives
    │
    ▼
nf().getSkipAllConfirmations()? ──yes──► Auto-approve all
    │ no
    ▼
hWD(toolName, input) ──false──► Auto-approve (no confirmation needed)
    │ returns confirmation details
    ▼
Build options (UYH/vYH)
    │
    ├─ ACP mode (IDE connected): connection.requestPermission()
    │   └── IPC to GUI → user picks option → response
    │
    ├─ CLI mode: TUI renders confirmation dialog
    │   └── User selects option via keyboard
    │
    └─ Non-interactive CLI: auto-deny exec, auto-approve others
    │
    ▼
Process outcome (ee function)
    │
    ├─ cancel → return [] (reject all)
    ├─ proceed_once → return all toolUseIds (one-time approval)
    ├─ proceed_always → upgrade autonomy level, approve all
    ├─ proceed_edit → set normal mode, approve all
    └─ proceed_auto_run_* → set corresponding autonomy mode, approve all
```

## Key Findings

### Finding 1: The 14 Permission Options

The system has exactly **7 confirmation types** × **7 outcome options** = **14 distinct permission decision points**:

**7 Confirmation Types** (from `hWD` in 3139.js):
1. `exec`: Shell command execution
2. `create`: File creation
3. `edit`: File content editing
4. `apply_patch`: Patch application
5. `exit_spec_mode`: Exiting spec mode to implementation
6. `start_mission_run`: Starting a mission run with concurrent runs
7. `mcp_tool`: MCP server tool invocation

**7 Outcome Options** (from `UYH` in 3141.js):
1. `proceed_once`: Allow this one time only
2. `proceed_always`: Allow and persist as new autonomy level
3. `proceed_edit`: Allow with spec → normal mode transition
4. `proceed_auto_run_low`: Allow and set auto-low mode
5. `proceed_auto_run_medium`: Allow and set auto-medium mode
6. `proceed_auto_run_high`: Allow and set auto-high mode
7. `cancel`: Reject the operation

### Finding 2: Impact-Level vs Autonomy-Level Gate (3139.js, lines 68-76)

```javascript
// Execute tool auto-approval logic (3139.js)
let E = GFH(A.riskLevel);                    // normalize risk level
try {
  let f = BA(),
      M = f.isAGIMode() ? "high" : f.getAutonomyLevel(),
      U = eEL(E),                             // impact → numeric (low=1, med=2, high=3)
      P = H4H(M);                             // autonomy → numeric (off=0, low=1, med=2, high=3)
  if (U <= P) return false;                   // auto-approve if impact ≤ autonomy
}
```

This is the central gate: **if the tool's impact level is ≤ the current autonomy level, auto-approve**. Otherwise, escalate to user.

### Finding 3: Command Denylist Bypass Prevention (2324.js)

```javascript
// lCI.checkAgainstDenylistPatterns - multi-token regex matching
// Handles command chaining with ; & | ` delimiters
// Single token: (^|[\s;&|(`]+)token([\s;&|)]+|$)
// Multi-token: allows path separators (/) and wildcards (~.)
```

The denylist is regex-based and accounts for shell command chaining, making it difficult to bypass via `; rm -rf /` style injection.

### Finding 4: Autonomy Level Clamping (1185.js)

```javascript
// Organization-level maximum autonomy enforcement
function jgH(H) {
  let A = TZ$();              // getMaxAutonomyLevel from settings
  let L = RX$(H, A);         // clamp to max: RX$ picks lower of requested vs max
  if (L === H) return H;
  BH("[SessionService] Clamped autonomy level to organization maximum", 
     { before: H, after: L });
  return L;
}
```

This ensures organizations can cap the maximum autonomy level users can set.

### Finding 5: AGI Mode Permission Bypass (1185.js, lines 178-189)

```javascript
isAGIMode() {
  return this.getInteractionMode() === "agi";
}
shouldAutoApproveFileEdits() {
  if (this.isAGIMode()) return true;      // AGI → always auto-approve file edits
  return this.isAutoRunMode();             // auto mode with level > off → auto-approve
}
getEffectiveAutonomyModeForPermissions() {
  if (this.isAGIMode()) return "auto-high"; // AGI treated as highest autonomy for permissions
  return this.getCurrentAutonomyMode();
}
```

AGI mode is equivalent to `auto-high` for permission purposes, auto-approving all file modifications.

## Integration Points

- **TUI / Ink (01-terminal-ui):** Permission confirmation UI rendered as Ink components in 3872.js (`SiD` component) and 3146.js (TUI confirmation flow)
- **Mission System (02-orchestration):** `start_mission_run` confirmation type and `propose_mission` type interact with mission orchestration
- **MCP (03-tool-agent-system):** `mcp_tool` confirmation type handles MCP tools via `___` namespace convention; connects to tool registry for `readOnlyHint` annotations
- **GUI / Electron (04-desktop-gui):** `AcpAdapter` (3417.js) bridges permission requests to IDE via `connection.requestPermission()` IPC
- **Session / Infra (05-infrastructure):** Session settings store (`cUA` schema in 0076.js) persists `commandAllowlist`, `commandDenylist`, `autonomyLevel`, `maxAutonomyLevel`; settings file path managed by infra

## Implementation Notes

### Permission Engine API

```typescript
// Core interfaces to implement:
interface PermissionEngine {
  getConfirmationType(toolName: string, input: ToolInput): ConfirmationType | false;
  generateOptions(context: ConfirmationContext): PermissionOption[];
  processOutcome(outcome: PermissionOutcome): string[];  // approved tool IDs
}

interface AutonomyManager {
  getAutonomyLevel(): "off" | "low" | "medium" | "high";
  getInteractionMode(): "auto" | "spec" | "agi";
  shouldAutoApproveFileEdits(): boolean;
  getMaxAutonomyLevel(): AutonomyLevel | undefined;
}

interface CommandPolicy {
  isCommandDenied(command: string): boolean;
  isCommandAllowed(command: string): boolean;
  getExtractedCommands(command: string): string[];
}
```

### Permission Configuration

```yaml
# User/project settings (cUA schema)
sessionDefaultSettings:
  autonomyMode: "normal" | "spec" | "auto-low" | "auto-medium" | "auto-high"
  interactionMode: "auto" | "spec" | "agi"
  autonomyLevel: "off" | "low" | "medium" | "high"
maxAutonomyLevel: "off" | "low" | "medium" | "high"  # org cap
commandAllowlist: ["git", "npm test"]    # auto-approved commands
commandDenylist: ["rm -rf /"]            # always-denied commands
enableOpenDroidShield: true                   # enable denylist enforcement
```

### Key Files to Port
1. `0885.js` → Autonomy mode/level conversion utilities
2. `3139.js` → Confirmation type resolver
3. `3141.js` → Permission option generator + outcome processor
4. `2324.js` → Command policy checker (denylist/allowlist)
5. `0724.js` → Session permission state
6. `0212.js` → Default denylist patterns

## Open Questions / Left Undone

- **Approval persistence path:** The `proceed_always` outcome calls `o8().setAutonomyLevel()` but the exact file path for persisting this change (likely under `~/.opendroid/` or project `.opendroid/`) was not traced to disk I/O.
- **`approvedToolIds` fine-grained selection:** In batch mode, the GUI can selectively approve individual tools within a batch. The selection UI mechanism is in the Electron layer (04-desktop-gui) and was not deeply analyzed.
- **`pendingPermissions` schema in session state:** Referenced in 0087.js as `AH.array(sWH.extend({ requestId: AH.string() })).optional()` but the full lifecycle of pending permission requests was not fully traced.
- **`skipPermissionsUnsafe` flag:** This flag (from session creation params 0087.js) bypasses all permission checks but the security implications and when it's set (e.g., daemon mode, non-interactive CLI) were not fully explored.
- **`OpenDroidShield` feature:** Referenced as `enableOpenDroidShield` in settings: likely related to the denylist enforcement but the exact implementation was not chased beyond the denylist checker.
