# Sandbox / Isolation System: Tool System Architecture Notes

## Overview

OpenDroid's sandbox system is a multi-layer defense-in-depth architecture combining command allowlist/denylist policies, filesystem path sandboxing, git argument exploit filtering, Content Security Policy network gating, environment variable scrubbing, and LLM-enforced risk-level classification. The system does not use OS-level containers; instead it relies on policy gates at the tool-handler layer, UI confirmation prompts, and path traversal checks.

## Module Map

| Module | Size | Role | Vendor/App |
|---|---|---|---|
| 0212.js | 2.0 KB | Default denylist (`r$H`) and org policy template (`ANL`) | app-tools |
| 0213.js | 1.8 KB | Default allowlist (`xAE`) and user settings defaults (`pJ`) | app-tools |
| 2324.js | 4.5 KB | Command policy engine (`lCI`): allowlist/denylist resolution | app-tools |
| 0939.js | 18.3 KB | Settings service: `getCommandAllowlist`, `getCommandDenylist`, `getEnableOpenDroidShield` | app-tools |
| 3748.js | 1.8 KB | Automation FS sandbox (`HWL`, `AWL`, `XcD`): path traversal prevention | app-tools |
| 3749.js | small preload bridge | File write gate (`$WL`): calls `AWL` before writing | app-tools |
| 0278.js | 28.2 KB | Git security gate: blocks `--upload-pack`, `--receive-pack`, `protocol.allow` | app-tools |
| 0894.js | small preload bridge | CSP policy: `connectSrc` network allowlist including SandboxSvc sandbox | app-tools |
| 1217.js | 5.2 KB | Tool input schemas: `Execute` risk level / `riskLevelReason` definition | app-tools |
| 3137.js | 1.1 KB | Risk-level enum parser (`eEL`): low/medium/high → 1/2/3 | app-tools |
| 3139.js | 4.0 KB | Confirmation gate (`hWD`): decides if Execute needs user prompt | app-tools |
| 3144.js | 2.5 KB | Checkpoint scrubber (`dWD`): strips `transcript_path`, `settings_path` | app-tools |
| 3371.js | 2.2 KB | UI render: shows "denylisted" / "allowlisted" / `impact` labels | app-tools |
| 3872.js | 4.5 KB | Confirmation prompt text generator (`nl`): "Run (denylisted): ..." | app-tools |
| 0076.js | 4.4 KB | Zod schema: `commandAllowlist`, `commandDenylist`, `enableOpenDroidShield` | app-tools |
| 0074.js | 2.0 KB | Zod schema: `toolPolicy` with `commandDenylist` + `droidShieldEnabled` | app-tools |
| 2553.js | 1.8 KB | CLI flag validation: `--skip-permissions-unsafe` vs `--auto` mutual exclusion | app-tools |
| 2568.js | 4.2 KB | Tool resolution: `skipPermissionsUnsafe` expands allowed tool set | app-tools |

## Architecture

### Layer 1: Command Policy (Allowlist / Denylist)

The `lCI` class (Module 2324.js) is the central command policy engine. It maintains:
- `commandAllowList`: a `Set` of explicitly allowed commands
- `denyListPatterns`: an array of regex-like strings for command denylist

On initialization it reads from `SettingsService` (0939.js) which falls back to defaults:
- **Allowlist** (`xAE`, 0213.js): `ls`, `pwd`, `dir`, `git status`, `git diff`, `git log`, `git show`, `git blame`, `git ls-files`
- **Denylist** (`r$H`, 0212.js): `rm -rf /`, `rm -rf ~`, `mkfs`, `dd if=/dev/zero of=/dev`, `shutdown`, `reboot`, `poweroff`, `chmod -R 777 /`, `Format-Volume`, `powershell Remove-Item -Recurse -Force`, etc.

The denylist matcher builds regexes dynamically. For single-token patterns it uses:
```js
new RegExp(`(^|[\\s;&|(\`]+)${$}([\\s;&|)]+|$)`, "i")
```
For multi-token patterns it builds a chained regex with `[^;&|]{0,100}` separators.

### Layer 2: FS Path Sandbox

Two complementary mechanisms:

**Automation Directory Sandbox** (3748.js, `HWL`):
```js
if (!M.startsWith(U + D3.sep))
  throw new vH("Security violation: Attempted write outside automation directory");
```
Validates that automation IDs do not contain traversal (`/`, `\`, `.`, `..`) and that resolved paths stay under the `.opendroid/automations` directory.

**General File Write Gate** (3749.js, `$WL`):
```js
if (!AWL(H, L))
  throw new vH("Security violation: Attempted write outside automation directory", {
    filePath: H, path: L,
  });
```
`AWL` (3748.js) compares realpath-resolved directories:
```js
function AWL(H, A) {
  let L = XcD(H), $ = XcD(A);
  return L === $ || L.startsWith($ + D3.sep);
}
```

### Layer 3: Git Security Gate

Module 0278.js blocks known git exploit vectors unless `allowUnsafePack` / `allowUnsafeProtocolOverride` flags are enabled:
```js
function mfE(H, A) {
  if (/^\s*--(upload|receive)-pack/.test(H))
    throw new uy(void 0, "unsafe", "Use of --upload-pack or --receive-pack is not permitted...");
  if (A === "clone" && /^\s*-u\b/.test(H))
    throw new uy(void 0, "unsafe", "Use of clone with option -u is not permitted...");
  if (A === "push" && /^\s*--exec\b/.test(H))
    throw new uy(void 0, "unsafe", "Use of push with option --exec is not permitted...");
}
```

### Layer 4: Network Gate (CSP)

Module 0894.js defines a strict Content-Security-Policy. `connectSrc` acts as a network allowlist:
```js
connectSrc: [
  "'self'",
  "ws://localhost:31415",
  ...dgE,  // SandboxSvc sandbox API: https://api.sandboxsvc.dev/sandboxes, wss://*.sandboxsvc.app
  ...lgE,  // Sprites sandbox
  ...ngE,  // OpenDroid API endpoints
  ...ogE,  // Relay WebSockets
]
```

### Layer 5: Environment Scrubbing

**Checkpoint scrubber** (3144.js, `dWD.sanitizeCheckpoint`):
```js
if (typeof $.transcript_path === "string") $.transcript_path = "[scrubbed]";
if (typeof $.settings_path === "string") $.settings_path = "[scrubbed]";
```

**PowerShell PATH scrubbing** (0703.js): When spawning PowerShell on Windows, the script constructs a sanitized `PATH` variable and returns it as JSON.

### Layer 6: Risk-Level & Confirmation Gate

The Execute tool (1217.js) schema **requires** `riskLevel` and `riskLevelReason`. Values are `low | medium | high`.

The confirmation resolver (3139.js, `hWD`) for `Execute`:
1. Extracts commands via `VV.getExtractedCommands($)`
2. If `VV.isCommandDenied($)` → returns high-impact confirmation
3. If `VV.isCommandAllowed($)` → auto-approve (no prompt)
4. Otherwise compares parsed risk level against user's `autonomyLevel` (via `eEL` and `H4H`)
5. If risk ≤ autonomy → auto-approve; else → prompt

UI labels (3371.js, 3872.js):
- Denylisted commands show `"denylisted"` + `"Run (denylisted): ..."`
- Allowlisted commands show `"allowlisted"` + `"Run (allowlisted): ..."`
- Others show `"impact: ${E}"`

Auto-approve levels (3421.js):
- `normal`: read-only only
- `auto-low`: file edits + low risk
- `auto-medium`: medium risk
- `auto-high`: all actions

### Layer 7: Droid Shield

A boolean toggle (`enableOpenDroidShield`, default `true`) in `toolPolicy` (0074.js, 0212.js). Controlled via SettingsService (0939.js). Its exact runtime behavior is gated behind this flag but the implementation details are spread across tool handlers.

## Key Findings

### 1. Default Denylist (0212.js, lines 3–37)
```js
(r$H = [
  "rm -rf /", "rm -rf /*", "rm -rf .", "rm -rf ~", "rm -rf ~/*",
  "rm -rf $HOME", "rm -r /", "rm -r /*", "rm -r ~", "rm -r ~/*",
  "mkfs", "mkfs.ext4", "mkfs.ext3", "mkfs.vfat", "mkfs.ntfs",
  "dd if=/dev/zero of=/dev", "dd of=/dev",
  "shutdown", "reboot", "halt", "poweroff", "init 0", "init 6",
  ":(){ :|: & };:", ":() { :|:& };:",
  "chmod -R 777 /", "chmod -R 000 /", "chown -R",
  "Format-Volume", "format.com",
  "powershell Remove-Item -Recurse -Force",
]);
```

### 2. Command Policy Engine: Denylist Regex Builder (2324.js, lines 67–96)
```js
checkAgainstDenylistPatterns(H) {
  for (let A of this.denyListPatterns) {
    if (!A || !A.trim()) continue;
    let L = A.split(/\s+/).filter(($) => $.length > 0);
    if (L.length === 0) continue;
    if (L.length === 1) {
      let $ = L[0].replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
      try {
        if (new RegExp(`(^|[\\s;&|(\`]+)${$}([\\s;&|)]+|$)`, "i").test(H)) return true;
      } catch { continue; }
    } else {
      let $ = L.map((f) => f.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")),
        I = [`(^|[\\s;&|(\`]+)${$[0]}`];
      for (let f = 1; f < $.length - 1; f++) {
        let M = L[f + 1];
        if (M === "/" || M?.startsWith("/")) I.push(`[^;&|.]{0,100}${$[f]}`);
        else I.push(`[^;&|]{0,100}${$[f]}`);
      }
      // ... tail matching
      try {
        if (new RegExp(I.join(""), "i").test(H)) return true;
      } catch { continue; }
    }
  }
  return false;
}
```

### 3. FS Path Sandbox: AWL Traversal Check (3748.js, lines 83–88)
```js
function AWL(H, A) {
  let L = XcD(H), $ = XcD(A);
  return L === $ || L.startsWith($ + D3.sep);
}
```
`XcD` resolves paths via `realpathSync` with graceful fallback for non-existent path prefixes.

### 4. Git Exploit Argument Blocking (0278.js, lines 1019–1045)
```js
function LfE(H) { return /^--upload-pack(=|$)/.test(H); }
function UkL(H, A, L) {
  let $ = ["clone", ...L];
  if ((WX(H) && $.push(H), WX(A) && $.push(A), $.find(LfE)))
    return nJ("git.fetch: potential exploit argument blocked.");
  return IQ($);
}
function ffE(H) { return /^--upload-pack(=|$)/.test(H); }
function MfE(H, A, L) {
  let $ = ["fetch", ...L];
  if (H && A) $.push(H, A);
  if ($.find(ffE)) return nJ("git.fetch: potential exploit argument blocked.");
  return { commands: $, format: "utf-8", parser: DfE };
}
```

### 5. Execute Confirmation Gate (3139.js, lines 56–90)
```js
case "Execute": {
  let L = A.command, $ = _4H(L), I = VV.getExtractedCommands($), D = VV.isCommandDenied($);
  if (!D && VV.isCommandAllowed($)) return false;
  if (D)
    return {
      type: "exec", fullCommand: $, command: I.join(", "),
      extractedCommands: I, impactLevel: "high", onConfirm: async () => {},
    };
  let E = GFH(A.riskLevel);
  try {
    let f = BA(), M = f.isAGIMode() ? "high" : f.getAutonomyLevel(),
      U = eEL(E), P = H4H(M);
    if (U <= P) return false;
  } catch (f) { BH("Impact parsing error", { error: f, command: $ }); }
  return {
    type: "exec", fullCommand: $, command: I.join(", "),
    extractedCommands: I, impactLevel: E, onConfirm: async () => {},
  };
}
```

### 6. Checkpoint Environment Scrubbing (3144.js, lines 92–93)
```js
sanitizeCheckpoint(H) {
  let { transcript: A, ...L } = H;
  if (L.agent_metadata && typeof L.agent_metadata === "object" && L.agent_metadata !== null) {
    let $ = L.agent_metadata;
    if (typeof $.transcript_path === "string") $.transcript_path = "[scrubbed]";
    if (typeof $.settings_path === "string") $.settings_path = "[scrubbed]";
  }
  return L;
}
```

### 7. `--skip-permissions-unsafe` Bypass (2553.js, lines 77–79)
```js
if (H.auto && H.skipPermissionsUnsafe)
  throw new vH(
    "Invalid flags: --auto and --skip-permissions-unsafe cannot be used together. Use only one of: --auto <low|medium|high> OR --skip-permissions-unsafe."
  );
```

## Integration Points

- **Permissions engine** (tool-permissions feature): The confirmation gate in 3139.js is the runtime bridge between sandbox policy and the permission resolver. Denylisted commands always trigger a high-impact confirmation regardless of auto-approve level.
- **Agent state machine** (tool-agent-state): When a command is denylisted or exceeds the autonomy threshold, the agent transitions to `waiting_for_tool_confirmation` (0060.js).
- **Execute tool** (tool-execute): Spawns processes via `cross-spawn` but the command string is pre-filtered by `lCI` before reaching the spawn layer.
- **TUI/Ink** (01-terminal-ui scope): Confirmation UI rendered by 3371.js and 3872.js: these are Ink React components showing denylisted/allowlisted badges.
- **MCP client** (tool-mcp-client): MCP tools prefixed with `___` use a fallback risk-level gate (3139.js, default `high`) when no explicit sandbox policy exists.
- **SandboxSvc integration** (0894.js): SandboxSvc sandbox API URLs (`https://api.sandboxsvc.dev/sandboxes`, `wss://*.sandboxsvc.app`) are explicitly allowlisted in the CSP `connectSrc` policy.

## Implementation Notes

1. **Command Policy Engine**: Port `lCI` (2324.js) as a standalone policy class. Maintain `Set`-based allowlist and regex-based denylist. Provide `isCommandAllowed()`, `isCommandDenied()`, `addAllowedCommand()`.
2. **FS Sandbox**: Port `AWL` / `XcD` (3748.js) as a path-validation utility. Use `path.resolve` + `fs.realpathSync` with fallback for non-existent prefixes. Reject any write where `!resolvedPath.startsWith(allowedBase + path.sep)`.
3. **Git Security Gate**: Port the `mfE` / `pfE` validators (0278.js) as a pre-spawn filter for git commands. Block `--upload-pack`, `--receive-pack`, `-u` in clone, `--exec` in push, and `protocol.allow` config unless unsafe flags are explicitly enabled.
4. **Risk Level Schema**: Enforce `riskLevel` + `riskLevelReason` as required fields in the Execute tool schema (1217.js pattern). Map `low=1`, `medium=2`, `high=3` for numeric comparison against autonomy level.
5. **Environment Scrubber**: Implement checkpoint serialization with field redaction (3144.js pattern): strip `transcript_path`, `settings_path`, and any other PII before remote transmission.
6. **Bypass Awareness**: If implementing a `--skip-permissions-unsafe` equivalent, ensure it is mutually exclusive with auto-approve levels and logged as a security event.

## Open Questions / Left Undone

1. **Droid Shield runtime behavior**: While `enableOpenDroidShield` is a settings flag (0939.js, 0074.js), the actual runtime enforcement code was not fully traced. It likely gates additional heuristics (e.g., network request inspection, file access auditing) but the specific hooks were not found in the analyzed modules.
2. **Subagent sandbox inheritance**: Whether a spawned subagent (`Task` tool) inherits the parent's `commandAllowlist` / `commandDenylist` or starts with fresh defaults was not traced. The daemon spawn code (3754.js) passes `--skip-permissions-unsafe` but not custom allowlists.
3. **SandboxSvc sandbox lifecycle**: The CSP references SandboxSvc sandbox URLs, but the actual SandboxSvc sandbox creation, code execution, and result serialization flow is outside the Tool & Agent system scope and was not analyzed.
4. **Windows-specific path normalization**: `AWL` uses `D3.sep` (platform-specific). On Windows, `path.resolve` behavior with mixed `/` and `\` separators could create bypass edge cases not tested in the current regex-based filters.
5. **Network gate beyond CSP**: The CSP `connectSrc` list is the primary network gate for the Electron renderer. Whether the Node main process has additional outbound URL filtering (e.g., for `FetchUrl` tool) was not found in the analyzed modules.
