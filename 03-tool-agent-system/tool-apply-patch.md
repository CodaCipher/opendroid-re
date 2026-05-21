# Apply Patch Tool: Tool System Architecture Notes

## Overview

The `apply_patch` tool is OpenDroid's file-editing mechanism that uses a custom, stripped-down diff format instead of standard unified diff. It supports both creating new files (`*** Add File:`) and updating existing files (`*** Update File:`). The tool parses patches via a dedicated `RX` class (Module 1191.js), applies changes with fuzzy context matching, and generates a readable diff output using a vendored diff library (modules 0896.js, 0903.js). The executor `RrA` (Module 2479.js) orchestrates file I/O, patch application, and metrics tracking.

## Module Map

| Module | Size | Role | Vendor/App |
|---|---|---|---|
| 1211.js | ~1 KB | Tool definition (`eI` factory) | App |
| 2479.js | ~3 KB | Executor class `RrA`: orchestrates read/apply/write | App |
| 1191.js | ~15 KB | Patch parser `RX`, application logic `kt`, `oD1`, `aD1` | App |
| 1192.js | ~6 KB | Tool description string `KZ$` with patch format grammar | App |
| 1189.js | ~3 KB | Patch format constants (`St`, `UC`, `NGH`, `PZA`, etc.) | App |
| 0905.js | ~1 KB | Diff generation `so()`: wraps vendor diff for output | App |
| 0896.js | ~small preload bridge | Vendor diff core (`OX` class: Myers algorithm) | Vendor-diff |
| 0903.js | ~small preload bridge | Vendor patch creation (`fwA`, `nxH`, `MwA`) | Vendor-diff |
| 0084.js | ~5.7 KB | Permission/confirmation schema for `apply_patch` | App |
| 0060.js | ~2 KB | Enum `ApplyPatch = "apply_patch"` | App |

## Architecture

### Tool Definition → Executor → Patch Engine Pipeline

```
LLM tool call
    ↓
Tool Registry (3264.js) dispatches to apply_patch
    ↓
Executor RrA (2479.js)
    ├── Parse input with yGH schema
    ├── Detect operation type: create | update (rR, nR helpers)
    ├── For update: read original file content (pZ)
    ├── Normalize line endings (bgH, cc, hGH)
    ├── Call kt() patch engine (1191.js)
    │   ├── tD1(): validate format + instantiate RX parser
    │   ├── RX.parsePatch(): parse into actions
    │   ├── aD1() / oD1(): apply chunks to file content
    │   └── so(): generate diff output (0905.js)
    ├── Write result back (gh)
    └── Yield result + metrics (MR)
```

### Patch Format Grammar (from 1192.js)

The tool uses a **custom diff format**, not standard unified diff:

```
Patch := Begin { FileOp } End
Begin := "*** Begin Patch" NEWLINE
End   := "*** End Patch" NEWLINE
FileOp := AddFile | UpdateFile
AddFile    := "*** Add File: " path NEWLINE { "+" line NEWLINE }
UpdateFile := "*** Update File: " path NEWLINE { Hunk }
Hunk       := "@@" [ header ] NEWLINE { HunkLine } [ "*** End of File" NEWLINE ]
HunkLine   := (" " | "-" | "+") text NEWLINE
```

Key design choices:
- One file per tool call
- `@@` headers are **optional**: context matching is fuzzy, not line-number based
- `*** End of File` terminates a hunk (optional)
- Three levels of context matching: exact, trim-end, trim-both, ordered subsequence

### Conflict Detection & Fuzzy Matching

The `RX` class implements a multi-tier context locator:

1. **Exact match** (`findExactContextMatch`): full line content including whitespace
2. **Trim-end match**: trailing whitespace ignored
3. **Trim-both match**: all whitespace ignored, score += 100
4. **Ordered subsequence match** (`findOrderedSubsequenceMatch`): finds lines in order within a 100-line window, score += 200+
5. **Trimmed ordered subsequence**: same but with trimmed comparison, score += 500+

If no match found at score threshold (default 75%), a `PatchApplicationError` is thrown with the failing context line.

## Key Findings

### 1. Tool Registration (Module 1211.js, lines 9–24)

```js
W1H = eI({
  id: "apply_patch",
  llmId: "apply_patch",
  uiGroupId: "edit_file",
  displayName: "Apply Patch",
  description: KZ$,
  executionLocation: "client",
  inputSchema: yGH,
  isVisibleToUser: false,
  isTopLevelTool: true,
  requiresConfirmation: false,
  outputSchemas: { updates: oR, result: _C },
  toolkit: "Base",
  isToolEnabled: ({ modelProvider: H, useTopLevelFileEditing: A }) => {
    if (A === void 0) return true;
    return A ? H === "openai" : false;
  },
});
```

Notable: `isToolEnabled` gates the tool behind OpenAI when `useTopLevelFileEditing` is explicitly enabled.

### 2. Executor Orchestration (Module 2479.js, lines 16–90)

```js
class RrA {
  async *execute(H, A) {
    let $ = yGH.parse(A),
        I = rR($.input);        // detect "create" | "update"
    let D = nR($.input);        // extract file path
    // ... path resolution ...
    if (I === "update") {
      let w = await pZ({ repoPath: ..., filePath: f });
      U[M] = cc(w.content);     // normalize to LF
    }
    if (I === "create") {
      // fail if file already exists
    }
    let B = kt({ operationType: I, filePath: M, patchText: P, fileContentRecord: U });
    // ... write back with preserved line endings ...
    yield { type: "result", isError: false, value: Q };
  }
}
```

### 3. Patch Parser RX Class (Module 1191.js, lines 28–220)

```js
class RX {
  originalFileContents;
  patchLines;
  currentLineIndex = 0;
  parsedPatch = { actions: {} };
  fuzzyMatchScore = 0;

  parsePatch() {
    while (!this.isParsingComplete([UC])) {
      let H = this.readLineWithPrefix(NGH);   // "*** Update File: "
      if (H) {
        let L = this.parseFileUpdateAction(A ?? "");
        this.parsedPatch.actions[H] = L;
        continue;
      }
      H = this.readLineWithPrefix(PZA);        // "*** Add File: "
      if (H) {
        let L = this.parseFileAddAction();
        this.parsedPatch.actions[H] = L;
        continue;
      }
      throw new PatchApplicationError_1190("Unexpected line encountered", ...);
    }
  }
}
```

### 4. Hunk Parsing: parseNextChangeSection (Module 1191.js, lines 155–195)

```js
static parseNextChangeSection(H, A) {
  let L = A, $ = [], I = [], D = [], E = [], f = "context";
  while (L < H.length) {
    let U = H[L];
    if (["@@", UC, NGH, kgH].some((X) => U.startsWith(X.trim()))) break;
    L += 1;
    let B = f, W = U;
    if (W[0] === BZA) f = "addition";      // '+'
    else if (W[0] === QZ$) f = "deletion"; // '-'
    else if (W[0] === WZA) f = "context";  // ' '
    else ((f = "context"), (W = WZA + W));
    // ... collect linesToDelete / linesToInsert ...
  }
  // ... build chunk objects ...
  return [$, E, L, M];
}
```

### 5. Chunk Application: oD1 (Module 1191.js, lines 225–245)

```js
function oD1(H, A, L) {
  if (A.type !== "update") throw new vH("Unsupported action type", ...);
  let $ = H.split(`\n`), I = [], D = 0;
  for (let E of A.chunks) {
    if (E.originalLineIndex > $.length)
      throw new PatchApplicationError_1190("chunk position exceeds file length", ...);
    if (D > E.originalLineIndex)
      throw new PatchApplicationError_1190("chunks out of order", ...);
    I.push(...$.slice(D, E.originalLineIndex));
    D = E.originalLineIndex;
    if (E.linesToInsert.length > 0) I.push(...E.linesToInsert);
    D += E.linesToDelete.length;
  }
  return I.push(...$.slice(D)), I.join(`\n`);
}
```

### 6. Diff Generation via Vendor Library (Module 0905.js, lines 13–26)

```js
function so({ originalContent: H, editedContent: A, contextLines: L = 3 }) {
  if (H === A) return "<file contents unchanged>";
  try {
    let $ = MwA("previous", "current", H, A, "", "",
                  { context: L, ignoreWhitespace: true });
    // ... strip leading "=====" separator ...
    return $.trim();
  } catch ($) {
    return "(diff unavailable)";
  }
}
```

`MwA` (module 0903.js) wraps the vendor diff engine (`dxH` → `sX$.diff`) and formats the result as a unified-diff-like patch with hunks.

### 7. Vendor Diff Core: OX Class (Module 0896.js, lines 4–70)

```js
class OX {
  diff(H, A, L = {}) {
    let I = this.castInput(H, L),
        D = this.castInput(A, L),
        E = this.removeEmpty(this.tokenize(I, L)),
        f = this.removeEmpty(this.tokenize(D, L));
    return this.diffWithOptionsObj(E, f, L, $);
  }
  diffWithOptionsObj(H, A, L, $) {
    // Myers diff algorithm with edit-length limit and timeout
    let E = A.length, f = H.length, M = 1, U = E + f;
    if (L.maxEditLength != null) U = Math.min(U, L.maxEditLength);
    let P = L.timeout ?? Infinity, B = Date.now() + P, W = [{ oldPos: -1, lastComponent: void 0 }];
    // ... bidirectional search over edit graph ...
  }
}
```

The vendor library is **diff** (npm package): the classic Myers diff implementation with tokenization, custom comparator support, and timeout guards.

## Integration Points

- **Tool Registry (03-tool-agent-system / tool-core-registry):** `apply_patch` registered via `eI()` in 1211.js and attached to registry in 2528.js with executor factory `() => new RrA()`.
- **Permission Engine (03-tool-agent-system / tool-permissions):** `apply_patch` appears in confirmation schema (0084.js, line 210) as `type: AH.literal("apply_patch")` with `filePath`, `fileName`, `patchContent`, `oldContent`, `newContent`.
- **Edit Tool (03-tool-agent-system / tool-edit):** Both tools share `uiGroupId: "edit_file"`: they are grouped together in the UI.
- **File I/O (03-tool-agent-system / tool-create-read):** Executor uses `pZ()` to read files and `gh()` to write results: these are the same file-system abstractions used by Create/Read.
- **Metrics (03-tool-agent-system / tool-agent-state):** `MR` metric recorder tracks `droid_mode_apply_patch_success_count` / `error_count`.

## Implementation Notes

### Core Components to Port

1. **Patch format parser (`RX` class)**: ~220 lines in 1191.js. This is the most critical piece; it replaces any standard unified-diff parser.
2. **Chunk applier (`oD1`)**: ~25 lines. Straightforward line-array splice.
3. **Context matcher (`findExactContextMatch`, `findOrderedSubsequenceMatch`)**: ~60 lines. The fuzzy-matching logic is what makes the tool resilient to LLM-generated patches.
4. **Diff generator (`so`)**: Can either reuse the vendor `diff` npm package or swap for a lighter alternative (e.g., `fast-diff`).
5. **Executor (`RrA`)**: Orchestration layer; can be simplified if OpenDroid does not need metric tracking or complex repo-location resolution.

### Suggested Simplifications

- The `diff` vendor library (0896.js + 0903.js) is ~5.6 KB combined; for OpenDroid, a single dependency on `diff` npm package suffices.
- Line-ending preservation (`bgH`, `cc`, `hGH`) can be collapsed into one utility.
- The `isToolEnabled` OpenAI gate in 1211.js is a product decision, not a technical requirement.

## Open Questions / Left Undone

1. **Suggested module list mismatch:** The module map's `suggested_features[12].modules` lists 3376.js, 3957.js, 3405.js, 2570.js, 0253.js, 3373.js: none of which were found to contain apply_patch logic. The actual implementation is spread across 1211.js, 2479.js, 1191.js, 1192.js, 1189.js, 0905.js, 0896.js, 0903.js. The seed list may be stale.
2. **`gh` write-back implementation:** The `gh()` function used in `RrA.execute` to persist changes was not traced to its defining module: it likely lives in the file-IO utility layer (shared with Create/Read tools).
3. **Multi-file patch support:** The grammar description in 1192.js says "one file per call," but the parser (`RX.parsePatch`) loops over multiple file operations. The `kt()` function, however, only processes a single file path. True multi-file patch behavior is unclear.
4. **Fuzzy match threshold tuning:** The `findOrderedSubsequenceMatch` threshold `D = 0.75` (75% of context lines must match) is hardcoded. No user-facing override was found.
5. **Max patch size limits:** Constants `yt = 40000` and `mc = 2000` appear in 1192.js near the patch constants but their exact usage in size-gating was not traced.
