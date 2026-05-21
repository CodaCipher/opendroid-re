# Grep & Glob Tools (ripgrep-backed): Tool System Architecture Notes

## Overview

The Grep and Glob tools provide high-performance file search capabilities built on top of **ripgrep (rg)**. The Grep tool (`grep_tool_cli`) performs content search with regex support, file type filtering, context lines, and dual output modes. The Glob tool (`glob-search-cli`) performs file path matching using glob patterns with inclusion/exclusion support. Both tools delegate to a shared ripgrep binary that is **extracted at runtime** from a bundled WASM/embedded resource. The system includes a Node.js glob fallback when ripgrep is unavailable. Two generations of search tools exist: the **CLI-facing tools** (rich parameter schemas, LLM-facing) and the **internal search tools** (simpler, used by other subsystems like ACP).

## Module Map

| Module | Size (lines) | Role | Category |
|--------|-------------|------|----------|
| 1217.js | 266 | Tool parameter schemas (zod): Grep (`tZ$`), Glob, all tools | App |
| 1222.js | ~40 | Grep tool definition via `eI()`: `grep_tool_cli` | App |
| 1223.js | ~50 | Glob tool definition via `eI()`: `glob-search-cli` | App |
| 0966.js | ~80+ | Tool name constants (`KKA="Grep"`, `wKA="Glob"`) and status messages | App |
| 2501.js | ~60 | Glob vendor library (Node.js `glob` package) + ripgrep binary extraction (`wzI()`) | App |
| 2502.js | ~60 | Ripgrep execution wrapper (`N9H()`): shared by Grep/Glob/LS | App |
| 2503.js | ~120 | Internal Glob tool handler (`GoA`) with ripgrep+Node.js fallback | App |
| 2504.js | ~110 | CLI Glob tool handler (`twH`): glob patterns with exclusion | App |
| 2505.js | ~70 | Internal Grep tool handler (`QoA`): simple file search | App |
| 2506.js | ~130 | CLI Grep tool handler (`awH`): full-featured content search | App |
| 3374.js | 214 | Glob tool TUI rendering (Ink components) | App |
| 3375.js | 208 | Grep tool TUI rendering (Ink components) | App |
| 3744.js | ~60 | LS tool helper: `listFiles()` using ripgrep `--files` | App |
| 1251.js | ~30 | Ripgrep argument builder (`vZA()`) for internal grep | App |

## Architecture

### Ripgrep Binary Management

```
[2501.js: wzI()]
  Embedded base64 blob (ToA = "B:/~BUN/root/rg-56avgsjb.")
    → SHA-256 verified extraction to ~/.opendroid/bin/rg.exe
    → Cached with .rg-sha256 checksum file
    → Re-extracted on SHA mismatch
```

The ripgrep binary is embedded as a base64-encoded blob within module 2501.js. On first use, it is extracted to `~/.opendroid/bin/rg.exe` (or platform equivalent). A SHA-256 hash is stored alongside and checked on subsequent runs to detect corruption or updates.

### Shared Ripgrep Execution Layer

```
[2502.js: N9H(args, cwd, options)]
  Accepts: args[], cwd, {timeout, maxBuffer, platform, homeDir}
  Resolves binary path via TaH() → wzI()
  execFile(rg_path, args, {cwd, encoding, timeout, maxBuffer})
  Exit codes: 0=success, 1=no matches (returns empty), 2+=error (throws)
  macOS home dir: auto-excludes .DS_Store, .Trash, Library, Pictures, Music, Movies
```

### Grep Tool (CLI-facing): `grep_tool_cli`

**Registration** (1222.js):
- `eI()` with id `"grep_tool_cli"`, llmId `"Grep"`, toolkit `"Code Search"`
- `requiresConfirmation: false`, `isTopLevelTool: true`, `isVisibleToUser: true`
- `executionLocation: "client"`

**Input Schema** (1217.js, `tZ$`):
```
pattern:        z.string()        : regex/literal pattern (ripgrep regex syntax)
path:           z.string().optional: absolute file/dir path
glob_pattern:   z.string().optional: ripgrep --glob filter
output_mode:    z.enum(["file_paths", "content"]).default("file_paths")
case_insensitive: z.boolean().default(false) : ripgrep -i
type:           z.string().optional: ripgrep --type (js, py, rust, cpp, etc.)
context_before: z.number().optional: ripgrep -B (content mode only)
context_after:  z.number().optional: ripgrep -A (content mode only)
context:        z.number().optional: ripgrep -C (content mode only)
line_numbers:   z.boolean().default(false): ripgrep -n (content mode only)
head_limit:     z.number().optional: truncate output to N lines
multiline:      z.boolean().default(false): ripgrep -U --multiline-dotall
fixed_string:   z.boolean().default(false): ripgrep -F
```

**Execution Flow** (2506.js, class `awH`):

```
1. Validate: pattern required, path existence check (file vs dir)
2. Build args:
   if output_mode === "content":
     --line-number (if line_numbers)
     --context N | --before-context N + --after-context N
   else:
     --files-with-matches
3. Common args:
   --no-heading --color=never --hidden --glob !.git/**
   --ignore-case (if case_insensitive)
   --type <mapped_type> (tsx→ts, jsx→js normalization)
   --glob <glob_pattern>
   --multiline --multiline-dotall (if multiline)
   --fixed-strings (if fixed_string)
4. Execute: N9H(args, cwd, {timeout: 30000, maxBuffer: 10485760})
5. Post-process: split by newline, filter empty, apply head_limit
6. Return: joined string or "No matches found" / "No matching files found"
```

### Glob Tool (CLI-facing): `glob-search-cli`

**Registration** (1223.js):
- `eI()` with id `"glob-search-cli"`, llmId `"Glob"`, toolkit `"Code Search"`
- Same visibility/confirmation settings as Grep

**Input Schema** (1223.js, `NZA`):
```
patterns:        z.union([z.string(), z.array(z.string())]): inclusion globs
excludePatterns: z.union([z.string(), z.array(z.array(...))]).optional: exclusion globs
folder:          z.string().optional: search directory
```

**Execution Flow** (2504.js, class `twH`):

```
1. Normalize: patterns → string[]; excludePatterns → string[] | undefined
2. Validate: folder exists and is directory, patterns non-empty
3. Build args via V9f():
   --files --hidden --glob !.git/**
   for each pattern: --glob <pattern>
   for each exclude: --glob !<exclude>
   .
4. Execute: N9H(args, cwd, {timeout: 30000, maxBuffer: 10485760})
5. Parse: split stdout by newline, filter, truncate at 1000
6. Return: joined filenames or "No matching files found"
```

### Internal Search Tools (Non-CLI)

Two simpler tool handlers exist for internal use:

**Internal Grep** (2505.js, class `QoA`):
- Uses `vZA()` arg builder: `--files-with-matches --no-heading --no-line-number --color=never --max-count 1`
- Returns `{files: [{filePath, numberOfLines}], isTruncated}`
- Truncates at `jO` (configurable limit)

**Internal Glob** (2503.js, class `GoA`):
- **Dual engine**: First checks if ripgrep works via `P9f(cwd)` (version check)
- If ripgrep available: `B9f()`: runs `rg --files --glob "<pattern>" --max-count 1000`
- If ripgrep unavailable: Falls back to Node.js `glob` library (`OzI()` via 2501.js `PaH`)
- Returns `{files: [{filePath, numberOfLines}], isTruncated}`

### TUI Rendering

Both tools have Ink-based TUI renderers (3374.js for Glob, 3375.js for Grep) that:
- Display header labels with pattern, folder, glob type, context info
- Show "Pending...", match count, or error states
- Handle ripgrep-not-found with reinstall instructions: `irm https://app.opendroid.dev/cli/windows | iex`

## Key Findings

### 1. Ripgrep Binary Extraction (2501.js, lines 25-55)

```js
var ToA = "B:/~BUN/root/rg-56avgsjb.";

function wzI() {
  let H = VoA.join(C$(), sL, "bin"),
    A = "rg.exe",
    L = VoA.join(H, "rg.exe"),
    $ = VoA.join(H, D9f),
    I = E9f(ToA),
    D = !QzI(L);
  if (!D)
    if (QzI($)) {
      if (((D = XoA($, "utf8").trim() !== I), D)) BH("Ripgrep binary SHA mismatch, updating");
    } else D = true;
  if (D)
    try {
      (I9f(H, { recursive: true }),
        JzI(L, XoA(ToA), { mode: 493 }),
        JzI($, I),
        BH("Extracted ripgrep binary", { path: L }));
    } catch (E) {
      throw (gH("Failed to extract ripgrep binary", { cause: E, path: L }), E);
    }
  return L;
}
```

### 2. Grep Tool Schema with All Ripgrep Flags (1217.js, lines 50-130)

```js
(tZ$ = p.object({
  pattern: p.string().describe("...Supports ripgrep regex syntax."),
  path: p.string().optional().describe("..."),
  glob_pattern: p.string().optional().describe('...Maps to ripgrep --glob parameter.'),
  output_mode: p.enum(["file_paths", "content"]).optional().default("file_paths"),
  case_insensitive: p.boolean().optional().default(false)
    .describe("Perform case-insensitive matching (ripgrep -i flag)."),
  type: p.string().optional()
    .describe('Ripgrep file type filter...Examples: "js", "py", "rust", "cpp".'),
  context_before: p.number().optional()
    .describe('Number of lines to show before each match (ripgrep -B flag).'),
  context_after: p.number().optional()
    .describe('Number of lines to show after each match (ripgrep -A flag).'),
  context: p.number().optional()
    .describe('Number of lines to show before and after each match (ripgrep -C flag).'),
  line_numbers: p.boolean().optional().default(false)
    .describe('Show line numbers in output (ripgrep -n flag).'),
  head_limit: p.number().optional().describe("Limit output to first N lines/entries."),
  multiline: p.boolean().optional().default(false)
    .describe("Enable multiline mode... (ripgrep -U --multiline-dotall)."),
  fixed_string: p.boolean().optional().default(false)
    .describe("Treat the pattern as a literal string... (ripgrep -F flag)."),
})),
```

### 3. Grep Execution with Output Modes (2506.js, lines 18-65)

```js
let f = [];
if (D.output_mode === "content") {
  if (D.line_numbers) f.push("--line-number");
  if (D.context) f.push("--context", String(D.context));
  else {
    if (D.context_before) f.push("--before-context", String(D.context_before));
    if (D.context_after) f.push("--after-context", String(D.context_after));
  }
} else f.push("--files-with-matches");
if (
  (f.push("--no-heading"),
  f.push("--color=never"),
  f.push("--hidden"),
  f.push("--glob", "!.git/**"),
  D.case_insensitive)
)
  f.push("--ignore-case");
if (D.type) {
  let w = { tsx: "ts", jsx: "js" }[D.type] || D.type;
  f.push("--type", w);
}
if (D.glob_pattern) f.push("--glob", D.glob_pattern);
if (D.multiline) (f.push("--multiline"), f.push("--multiline-dotall"));
if (D.fixed_string) f.push("--fixed-strings");
(f.push("--"), f.push($));
```

### 4. Glob Dual-Engine Fallback (2503.js, lines 60-95)

```js
if (await P9f(L)) {
  BH("Using ripgrep for glob search", { pattern: I, cwd: L });
  try {
    D = await B9f(I, L, 1000);
  } catch (U) {
    (gH("Ripgrep glob failed, falling back to Node.js glob", {
      error: U instanceof Error ? U.message : String(U),
    }),
      (D = await OzI(I, L, 1000)));
  }
} else
  (BH("Using Node.js glob for search", { pattern: I, cwd: L }),
    (D = await OzI(I, L, 1000)));
```

### 5. Shared Ripgrep Exec Wrapper (2502.js, lines 15-45)

```js
async function N9H(H, A, L = {}) {
  let { timeout: $ = qzI.TIMEOUT, maxBuffer: I = qzI.MAX_BUFFER,
         platform: D = "win32", homeDir: E = CzI.homedir() } = L,
    f = TaH();
  if (!f)
    throw new RipgrepError(`Ripgrep binary not found...`, null, "");
  let M = [...H];
  if (D === "darwin" && A === E)
    M = [
      ...[".DS_Store", ".Trash/**", "Library/**", "Pictures/**",
           "Music/**", "Movies/**"].flatMap((B) => ["--glob", `!${B}`]),
      ...M,
    ];
  try {
    let U = await _9f(f, M, { cwd: A, encoding: "utf8", timeout: $, maxBuffer: I });
    return { stdout: U.stdout, stderr: U.stderr || "" };
  } catch (U) {
    if (U && typeof U === "object" && "code" in U) {
      if (U.code === YzI.NO_MATCHES) return { stdout: "", stderr: "" };
      if (typeof U.code === "number" && U.code >= YzI.ERROR)
        throw new RipgrepError("Ripgrep execution failed", P.code, ...);
    }
    throw U;
  }
}
```

### 6. Tool Constant Registry (0966.js, lines 18-30)

```js
var w4H = "No matches found",
  K4H = "No matching files found",
  KKA = "Grep",      // llmId for Grep tool
  wKA = "Glob",      // llmId for Glob tool
  XKA = "LS",        // shared by LS tool
```

## Integration Points

- **TUI (01-terminal-ui)**: Ink-based renderers in 3374.js (Glob) and 3375.js (Grep) provide terminal UI for search results: JSX/React-Ink components rendering match counts, error states, detailed views.
- **ACP Protocol**: Internal search tools (2503.js, 2505.js) expose `{files, isTruncated}` results consumed by ACP adapter (3417.js) for remote sessions.
- **Permission System**: Both Grep and Glob tools have `requiresConfirmation: false`, meaning they bypass the permission gate and execute immediately.
- **Tool Registry**: Both tools registered via `eI()` in the central registry, listed under `"Code Search"` toolkit, with `isTopLevelTool: true`.
- **LS Tool**: Shares the ripgrep binary (`N9H()`) via 3744.js `listFiles()` function which uses `rg --files`.
- **Execute Tool**: The Grep tool description mentions preferring `rg` (ripgrep) over shell `grep` for performance, suggesting Execute tool documentation references these tools.

## Implementation Notes

### Grep Tool API
```typescript
interface GrepInput {
  pattern: string;           // Required: regex or literal
  path?: string;             // Absolute path
  glob_pattern?: string;     // File filter (e.g., "*.js")
  output_mode: "file_paths" | "content";  // default: "file_paths"
  case_insensitive?: boolean;
  type?: string;             // "js" | "py" | "rust" | "cpp" | ...
  context_before?: number;   // -B flag (content mode)
  context_after?: number;    // -A flag (content mode)
  context?: number;          // -C flag (content mode)
  line_numbers?: boolean;    // -n flag (content mode)
  head_limit?: number;       // Truncate output
  multiline?: boolean;       // -U --multiline-dotall
  fixed_string?: boolean;    // -F flag
}
```

### Glob Tool API
```typescript
interface GlobInput {
  patterns: string | string[];         // Inclusion globs (OR logic)
  excludePatterns?: string | string[]; // Exclusion globs
  folder?: string;                     // Search directory
}
```

### Ripgrep Integration
- Binary embedded as base64 blob, extracted to `~/.opendroid/bin/rg.exe`
- Shared execution wrapper `N9H(args, cwd, options)`: timeout 10s (default), maxBuffer 10MB
- Exit code semantics: 0=found, 1=not found, 2+=error
- macOS home directory auto-excludes common system directories

## Open Questions / Left Undone

- **`vZA()` function in 1251.js**: The internal grep arg builder has an `engine` option ("default" | "pcre2") with PCRE2 support (`-P` flag): not exposed in CLI tool schema. Purpose unclear.
- **`jO` constant**: Truncation limit for internal search tools: value not traced (likely 100-500 range based on context).
- **`GZA` constant** in 2505.js: Default search path for internal grep: likely `process.cwd()`.
- **`Pf()` function**: Post-processing helper used in 2506.js to normalize ripgrep output: implementation not traced.
- **1203.js old Glob tool**: An older `glob_tool` registration exists with simpler schema: relationship to current `glob-search-cli` unclear (possibly deprecated/internal).
- **2501.js Node.js glob fallback**: Full `glob` package bundled: version and configuration options not fully traced.
