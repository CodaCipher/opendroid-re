# Edit Tool (Text Replacement): Tool System Architecture Notes

## Overview

The Edit tool provides in-place text replacement for files using exact string matching with `old_str`/`new_str` semantics. It exists in two variants: a single-change **Edit** tool (class `R7H` in 2322.js) and a **MultiEdit** tool (class `xrA` in 2484.js) that applies an array of changes atomically. The core matching and replacement engine lives in module 1250.js (functions `oGH` for validation and `tGH` for execution), with CRLF-aware normalization (`uz$`), trailing-newline preservation (`gz$`), and a fuzzy-match fallback using a bit-parallel approximate string matching algorithm (`brA` in 2484.js). Changes are tracked via diff generation (`_4` in 2188.js) and surfaced to the UI through TUI renderers (3370.js). The Edit tool is registered in the tool registry at module 2528.js (ACP path) and 2548.js (core path), using schema `aZ$` (single edit) and `Js9` (multi-edit) defined in 1217.js.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 1217.js | 266 lines | **Schema definitions**: `H11` (single change), `Js9` (multi-edit), `aZ$` (Edit tool input), Read/Grep/Create schemas | App |
| 1250.js | 120 lines | **Core replacement engine**: `oGH` (validate match), `tGH` (apply replacement), `pz$` (multi-change batch), `cz$` (CRLF normalize change), `gz$` (newline preserve), `uz$` (line ending normalize) | App |
| 2321.js | ~100 lines | **Edit I/O handler**: `pCI` function orchestrating read→validate→replace→write pipeline | App |
| 2322.js | 118 lines | **Edit tool executor**: class `R7H` with parameter validation, artifacts protection, diff output, file change tracking | App |
| 2484.js | 252 lines | **MultiEdit tool executor**: class `xrA` with fuzzy-match fallback (`brA` via bit-parallel algorithm `BZI`), repo save support, language detection | App |
| 2188.js | 190 lines | **Diff generator**: `_4(oldContent, newContent, contextLines)` using `HrH` (Myers diff), produces `{type, content, lineNumber}` arrays | App |
| 3139.js | ~100 lines | **Edit confirmation**: converts Edit/MultiEdit parameters into change arrays, validates via `pz$`, returns confirmation details | App |
| 3370.js | 276 lines | **Edit TUI renderer**: renders diff output, handles `replace_all` flag display, truncation for large diffs, success messages | App |
| 2528.js | ~50 lines | **ACP tool registration**: registers both Edit (`xrA`/`pGH`) and MultiEdit (`R7H`/`sR`) via `X8().register()` | App |
| 2548.js | ~35 lines | **Core tool registration**: registers Edit via `k0().register()` with `sR` schema | App |
| 1220.js | 201 lines | **Tool descriptions**: Edit/MultiEdit usage guidelines for LLM prompt | App |
| 0943.js | ~310 lines | **MultiEdit → Edit normalization**: `name: H.name === "MultiEdit" ? "Edit" : H.name` maps MultiEdit to Edit for LLM function calling | App |

## Architecture

### Input Schema

**Single Edit** (schema `aZ$` in 1217.js):
```js
p.object({
  file_path: p.string().describe("The path to the file to edit"),
  old_str: p.string().describe("The exact text to find and replace in the file"),
  new_str: p.string().describe("The text to replace the old_str with"),
  change_all: p.boolean().optional().default(false)
    .describe("Whether to replace all occurrences (true) or just the first one (false)")
})
```

**Multi-Edit** (schema `Js9` in 1217.js):
```js
p.object({
  file_path: p.string().describe("The path to the file to edit"),
  changes: p.array(H11).min(1).describe("Array of changes to apply to the file")
})
```

Each change within MultiEdit uses schema `H11`:
```js
p.object({
  old_str: p.string().describe("The exact text to find and replace"),
  new_str: p.string().describe("The text to replace the old_str with"),
  change_all: p.boolean().optional().default(false)
    .describe("Whether to replace all occurrences (true) or just the first one (false)")
})
```

### Text Matching Algorithm

The core matching is implemented in `oGH()` (1250.js):

1. **CRLF Normalization**: `uz$(fileContent, old_str)` normalizes `old_str` to match the file's line ending style (CRLF if file uses `\r\n`, LF otherwise).
2. **Empty File Edge Case**: If `old_str === ""` and file content is also empty, return success immediately (changesApplied: 1).
3. **Exact Match Check**: `fileContent.includes(normalized_old_str)`: uses JavaScript's native `String.includes()` for exact substring match.
4. **Occurrence Count**: `fileContent.split(normalized_old_str).length - 1` counts occurrences.
5. **Ambiguous Match Error**: If count > 1 and `change_all` is false, return error with helpful message:
   ```
   Error: Found N occurrences of the specified text, but change_all is false. Either:
   1. Provide more context in old_str to make it unique
   2. Set change_all to true if you want to replace all N occurrences
   ```
6. **No Match Error**: If `includes()` returns false, return:
   ```
   Error: The text to replace was not found in the file. Please ensure the old_str
   parameter matches the exact text in the file, including whitespace and line breaks.
   ```

### Replacement Execution

`tGH()` (1250.js) applies the actual replacement:

- **`change_all: true`**: Uses `String.split(old_str).join(new_str)`: replaces ALL occurrences.
- **`change_all: false`**: Uses `String.replace(old_str, () => new_str)`: replaces only the FIRST occurrence. The callback form ensures special replacement patterns (`$1`, `$&`, etc.) are treated literally.

### Multi-Change Batch Processing

`pz$()` (1250.js) applies an array of changes sequentially:

```js
function pz$(fileContent, changes) {
  let content = fileContent, appliedCount = 0;
  for (let i = 0; i < changes.length; i++) {
    let validation = oGH(content, changes[i], i);  // validate with change index
    if (!validation.success) 
      return { success: false, message: validation.message, oldContent: fileContent };
    content = tGH(content, changes[i]);  // apply to updated content
    appliedCount += validation.changesApplied;
  }
  return { success: true, oldContent: fileContent, newContent: content, changesApplied: appliedCount };
}
```

Key behavior: changes are applied **sequentially**: each subsequent change operates on the result of the previous one. If any change fails validation, the entire batch is rejected and no modifications are written.

### CRLF / Line Ending Handling

`uz$(fileContent, text)` (1250.js):
- Detects if file uses `\r\n` (CRLF)
- Normalizes `old_str`/`new_str` line endings to match the file's style
- Ensures exact matching regardless of line ending differences between platforms

`gz$(originalContent, newContent)` (1250.js):
- Preserves trailing newline behavior from the original file
- If original ended with `\n` but replacement doesn't, appends one
- If original didn't end with `\n` but replacement does, strips it

### Fuzzy Match Fallback (MultiEdit / Anthropic path)

In the `xrA` executor (2484.js), when exact match fails, a fuzzy match algorithm is invoked:

```js
let C = brA(fileContent, old_str, JZ$);  // JZ$ = 50 (max edit distance)
if (C.length > 0) {
  // Show the similar text found in the file
  llmError: `The exact text was not found.
Found similar text:
${fileContent.slice(C[0].start, C[0].end)}
You provided: ${old_str}
Please use the exact text from the file (including whitespace and formatting) in the old_str parameter.`
}
```

The fuzzy matcher (`brA` → `BZI`) implements a **bit-parallel Myers diff algorithm** (compiled to WebAssembly) that finds approximate matches within a configurable edit distance (default 50 characters). This provides helpful error messages when the LLM's `old_str` is slightly off from the actual file content.

### Diff Generation

`_4(oldContent, newContent, contextLines)` in 2188.js:

- Uses `HrH()` (Myers diff algorithm, likely from the `diff` npm package)
- Produces an array of `{type: "added"|"removed"|"unchanged", content: string, lineNumber: {old?, new?}}` objects
- `contextLines` parameter controls how many unchanged lines to show around changes (default: 3)
- Optimized: skips all unchanged lines before first change and after last change, with context window

### File Protection

The Edit executor (2322.js) enforces several protection rules:

1. **Artifacts directory**: `MMH(filePath)`: blocks editing files in `~/.opendroid/artifacts/`
2. **Mission system files**: `_MH(filePath)`: blocks editing `state.json`, `progress_log.jsonl`
3. **Parameter validation**: Checks `file_path`, `old_str`, `new_str` are present and correct types

### Confirmation Flow

Module 3139.js handles edit confirmations:

- For single Edit: wraps into `[{old_str, new_str, change_all}]` array
- For MultiEdit: maps each change from `A.changes`
- Pre-validates all changes via `pz$()` before showing confirmation dialog
- Returns `{type: "edit", filePath, fileName, oldContent, newContent}` for UI display

### Registration & Dispatch

Two registration paths exist:

1. **ACP (Agent Communication Protocol)**: 2528.js:
   ```js
   X8().register({ tool: pGH, executorFactory: () => new xrA() });  // MultiEdit with fuzzy match
   X8().register({ tool: sR, executorFactory: () => new R7H() });   // Standard Edit
   ```

2. **Core tool registry**: 2548.js:
   ```js
   k0().register({ tool: sR, executorFactory: () => new R7H() });  // Standard Edit
   ```

The MultiEdit tool is normalized to "Edit" in the LLM function calling manifest (0943.js line 307):
```js
name: H.name === "MultiEdit" ? "Edit" : H.name
```

## Key Findings

### 1. Core Replacement Engine (1250.js, lines 56-88)

```js
function oGH(H, A, L) {
  let $ = L !== void 0 ? `Change ${L + 1}: ` : "",
    I = cz$(H, A);
  if (I.old_str === "" && H === "") return { success: true, message: "", changesApplied: 1 };
  if (!H.includes(I.old_str))
    return {
      success: false,
      message: `${$}Error: The text to replace was not found in the file. Please ensure the old_str parameter matches the exact text in the file, including whitespace and line breaks.`,
    };
  let D = H.split(I.old_str).length - 1;
  if (D > 1 && !I.change_all)
    return {
      success: false,
      message: `${$}Error: Found ${D} occurrences of the specified text, but change_all is false. Either:
1. Provide more context in old_str to make it unique
2. Set change_all to true if you want to replace all ${D} occurrences`,
    };
  return { success: true, message: "", changesApplied: I.change_all ? D : 1 };
}

function tGH(H, A) {
  let L = cz$(H, A);
  if (L.change_all) return H.split(L.old_str).join(L.new_str);
  return H.replace(L.old_str, () => L.new_str);
}
```

### 2. Sequential Multi-Change Application (1250.js, lines 90-108)

```js
function pz$(H, A) {
  let L = H, $ = [], I = 0;
  for (let D = 0; D < A.length; D++) {
    let E = A[D], f = oGH(L, E, D);
    if (!f.success)
      return { success: false, message: f.message, oldContent: H, newContent: void 0 };
    L = tGH(L, E);
    I += f.changesApplied;
    $.push(`Change ${D + 1}: Replaced ${f.changesApplied} occurrence${f.changesApplied > 1 ? "s" : ""}`);
  }
  return { success: true, message: $.join("\n"), oldContent: H, newContent: L, changesApplied: I };
}
```

### 3. Edit Executor with Protection (2322.js, lines 6-30)

```js
class R7H {
  async *execute(H, A) {
    if (H.abortSignal?.aborted) throw new II();
    let { file_path: L, old_str: $, new_str: I, change_all: D } = A;
    if (!L || typeof L !== "string") {
      yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
        llmError: "file_path is required and must be a string" };
      return;
    }
    if ($ === void 0 || $ === null) {
      yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
        llmError: "old_str is required" };
      return;
    }
    if (MMH(L)) { /* artifacts protection */ return; }
    if (_MH(L)) { /* mission system files protection */ return; }
    // ... read, validate, apply, diff, return
  }
}
```

### 4. Fuzzy Match Error Message (2484.js, lines 189-207)

```js
let C = brA(P, E, JZ$);  // fuzzy search with max distance 50
if (C.length > 0) {
  yield {
    type: "result", isError: true, errorType: "invalidParameterLLMError",
    llmError: `The exact text was not found.
Found similar text:
${P.slice(C[0].start, C[0].end)}
You provided:
${B.old_str}
Please use the exact text from the file (including whitespace and formatting) in the old_str parameter.`,
    userError: "The old_str was not found in the file",
  };
}
```

### 5. Diff Generation (2188.js, lines 13-50)

```js
function _4(H, A, L = 3) {
  let $ = [];
  if (!H && !A) return $;
  if (!H && A) { /* all added */ return $; }
  if (H && !A) { /* all removed */ return $; }
  let I = HrH(H, A),  // Myers diff
    D = 1, E = 1, f = false;
  // ... iterate diff hunks, emit {type, content, lineNumber} objects
  // with context window of L lines around changes
}
```

## Integration Points

- **TUI/01-terminal-ui**: Diff rendering in 3370.js uses React/Ink components (`<Box>`, `<Text>`): owned by 01-terminal-ui (TUI)
- **Permissions/02-orchestration**: Edit confirmation flow (3139.js) ties into the permission system: `shouldAutoApproveFileEdits()` check
- **Agent State/02-orchestration**: Tool execution outcome tracking via `MR` class (metrics recording)
- **File IO**: Edit reads via `sp(ideClient, filePath)` (IDE client) and `e6H(filePath)` (direct FS); writes via `DrH()`
- **ACP Protocol (3411.js)**: Edit results serialized as `{type: "content", content: {type: "text", text: filePath}}` for ACP
- **Tool Registry**: Registration through `X8().register()` (ACP) and `k0().register()` (core): covered by tool-core-registry feature
- **Diff vendor**: `HrH` in 2188.js imports from `ArH` module: likely the `diff` npm package
- **WebAssembly**: Fuzzy matcher in 2484.js uses WASM-compiled bit-parallel algorithm (1549.js contains the wasm binary)

## Implementation Notes

### Tool API
- Input: `{file_path: string, old_str: string, new_str: string, change_all?: boolean}`
- Multi-edit input: `{file_path: string, changes: Array<{old_str, new_str, change_all?}>}`
- Output: `{file_path, diffLines: Array<{type, content, lineNumber}>, content: string}`

### Key Implementation Points
1. **Exact match only**: no regex, no wildcards. The `old_str` must be an exact substring.
2. **Sequential application**: multi-edits apply changes one at a time on the evolving content.
3. **CRLF-aware**: normalizes line endings to match the file's style before matching.
4. **Atomic batch**: if any change in a multi-edit fails validation, none are applied.
5. **Fuzzy fallback**: the Anthropic/ACP path provides helpful "did you mean?" error messages.

### Error Codes
- `invalidParameterLLMError`: missing/invalid parameters, no match, ambiguous match
- `toolInternalError`: file I/O failures, write failures
- `externalAPIError`: LLM response failures during edit

## Open Questions / Left Undone

1. **File write mechanism** (`DrH`, `IrH`, `e6H`): the actual file system write primitives were not traced in detail; they likely involve sandbox checks and IDE client coordination.
2. **`sp()` IDE client read**: the dual-path reading (IDE client vs direct FS) needs further investigation to understand when each path is used.
3. **Tool schema `sR` vs `pGH`**: the relationship between the two tool schema variables and their LLM function-calling manifests needs cross-referencing with tool-core-registry findings.
4. **`BMH(sessionId)` call** in 2322.js after successful edit: purpose unclear, possibly session state update or analytics.
5. **`fMH(diffResult, filePath)`**: generates `systemReminder` content, purpose not fully traced.
6. **Sandbox integration**: the edit tool's file path validation likely goes through the sandbox layer; cross-ref with tool-sandbox feature needed.
