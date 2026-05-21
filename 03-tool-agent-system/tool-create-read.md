# Create & Read Tools (File IO): Tool System Architecture Notes

## Overview

The Create and Read tools provide the core file I/O layer for OpenDroid. The **Create** tool writes new files to disk with automatic directory creation and trailing-newline normalization. The **Read** tool reads file contents with offset/limit pagination for text, base64-encoded output for images (JPEG/PNG up to 5 MB), and plain-text passthrough for all other types. Both tools enforce absolute-path validation using Node's `path.isAbsolute()` and implement write-protection for the artifacts directory and mission system files. Image handling leverages a JPEG compression pipeline (decode → resize → re-encode) to keep results within LLM context limits. The tools exist in two variants: a CLI-mode set (`create-cli` / `read-cli`) using simple `{file_path, content}` schemas, and a full-mode set (`Create` / `Read`) with richer output schemas that include diff tracking and image content blocks.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 1217.js | 266 lines | Zod schema definitions for all tools (Read, Edit, Execute, Grep, Glob, Create, Todo, AskUser) | App |
| 1218.js | 40 lines | Read tool registration (`read-cli`) with eI() factory, input/output schemas | App |
| 1224.js | 25 lines | Create CLI tool registration (`create-cli`) with eI() factory | App |
| 1213.js | 30 lines | Create tool registration (`Create`) for full mode, model-specific enablement | App |
| 2510.js | ~110 lines | Read tool handler class LKH: file reading, image detection, base64 encoding, text chunking | App |
| 2320.js | ~171 lines | Create tool handler class N7H: file writing, directory creation, artifacts protection, skill file refresh | App |
| 2509.js | ~100 lines | Text chunking utility RzI(): offset/limit slicing, character-limit truncation, line-count annotations | App |
| 1148.js | 74 lines | Image media-type enum `f2 = ["image/jpeg", "image/png"]` and MCP content serialization | App |
| 1174.js | ~193 lines | Image compression pipeline: decode (JPEG/PNG) → resize → re-encode to JPEG, quality control | Vendor (png-js + jpeg-js) |
| 2477.js | ~40 lines | MIME type resolver `rtH`: maps file extensions to media types via MIME database | App |
| 2197.js | ~30 lines | Artifacts directory write protection check `MMH()` | App |
| 2199.js | ~30 lines | Mission system files write protection check `_MH()` | App |
| 2189.js | ~50 lines | File tracking utilities: `$rH` (read tracking), `wmA` (write tracking) | App |

## Architecture

### Tool Registration Flow

```
Schema (1217.js)  →  eI() factory (1201.js)  →  Registry (1177.js)
       ↓                     ↓
  mGH (Read)          tR = read-cli (1218.js)
  sZ$ (Create CLI)    eR = create-cli (1224.js)
  bGH (Create full)   gGH = Create (1213.js)
```

The `eI()` function registers each tool with id, llmId, displayName, inputSchema, outputSchemas, toolkit, and enablement conditions. Tools are organized by `uiGroupId`:
- Read: `view_file`
- Create CLI: `edit_file`
- Create: `create_file`

### Read Tool Pipeline

```
Input: { file_path, offset=0, limit=2400 }
  ↓
Validate file_path (string, absolute)
  ↓
Access file (with AM/PM unicode normalization fallback)
  ↓
Detect MIME type via rtH.getType()
  ↓
┌─ image/* (jpeg/png, non-svg) ──→ Check size ≤ 5MB ──→ Read binary ──→ Compress (bc) ──→ base64 + content blocks
│
└─ text/* (default) ──→ Read UTF-8 ──→ Slice by offset/limit (RzI) ──→ Truncate if >60k chars ──→ string result
```

### Create Tool Pipeline

```
Input: { file_path, content }
  ↓
Validate file_path (string), content (not null/undefined)
  ↓
Check write protection:
  - MMH() → artifacts directory? REJECT
  - _MH() → mission system files? REJECT
  ↓
mkdir -p dirname (recursive)
  ↓
Normalize: append trailing newline if missing
  ↓
writeFile (UTF-8)
  ↓
Track: wmA() → file write tracking
       captureToolFileChange() → diff system
       Y0f() → skill file? refresh settings + open in IDE
  ↓
Read back (sp) → compute diff summary → return "File created successfully"
```

### Image Handling Subsystem

The Read tool uses a compression pipeline for images:
1. **Decode**: JPEG via `jpeg-js.decode()`, PNG via `png-js` sync read
2. **Resize**: Scale down to fit max dimension (1024px) while preserving aspect ratio
3. **Re-encode**: Convert to JPEG at quality=100 (initial), with iterative quality reduction if needed
4. **Return**: `{ buffer, contentType: "image/jpeg", originalSize, finalSize }`

The output schema for image reads is:
```typescript
[
  { type: "text", text: "Image file: filename.ext (N.N KB)" },
  { type: "image", source: { type: "base64", media_type: "image/jpeg"|"image/png", data: "..." } }
]
```

### Path Validation

Both tools enforce absolute path requirements:
- **Read**: `path.isAbsolute(file_path)`: rejects relative paths with `llmError: "file_path must be an absolute path, not a relative path"`
- **Create**: No explicit `isAbsolute` check, but relies on `path.resolve()` internally via `wmA()` tracking

### Write Protection

Two guard functions prevent writing to protected directories:
- `MMH(filePath)`: Checks if resolved path falls within `~/.opendroid/artifacts/` directory
- `_MH(filePath)`: Checks if resolved path is a mission system file (`state.json`, `progress_log.jsonl`)

### Text Chunking (RzI function)

The `RzI` function in 2509.js handles pagination for text files:
- Splits content by newlines
- Applies 0-based offset and limit (default: 0 to 2400 lines)
- Supports negative offsets (count from end)
- Enforces 60,000 character hard limit per result
- Appends `[Showing lines X-Y of N total lines, truncated to Nk characters]` annotation when truncated

### Create Tool Variants

Two Create tool variants exist:
1. **`create-cli`** (1224.js): Enabled when `modelProvider !== "openai"`, uses simple `sZ$` schema `{file_path, content}`, returns plain string result
2. **`Create`** (1213.js): Full mode with `bGH` schema `{repoLocation, filePath, content}`, enabled based on `modelProvider` and `useTopLevelFileEditing` flags. Returns `{updates, result}` with diff tracking support

### Model-Specific Enablement

```javascript
// create-cli (1224.js)
isToolEnabled: ({ modelProvider }) => modelProvider !== "openai"

// Create (1213.js) 
isToolEnabled: ({ modelProvider, useTopLevelFileEditing }) => {
  if (useTopLevelFileEditing === undefined) return true;
  return useTopLevelFileEditing 
    ? modelProvider === "anthropic" || modelProvider === "google" 
    : false;
}
```

## Key Findings

### 1. Read Tool Schema (1217.js, lines 3-13)

```javascript
mGH = p.object({
  file_path: p.string().describe("The absolute path to the file to read (must be absolute, not relative)"),
  offset: p.number().optional().default(0).describe("The line number to start reading from (0-based, defaults to 0)"),
  limit: p.number().optional().default(2400).describe("The maximum number of lines to read (defaults to 2400)"),
})
```

### 2. Read Tool Handler: Path Validation (2510.js, lines 11-31)

```javascript
let { file_path: L, offset: $ = 0, limit: I = 2400 } = A;
if (!L || typeof L !== "string") {
  yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
    llmError: "file_path is required and must be a string" };
  return;
}
if (!SzI.isAbsolute(L)) {
  yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
    llmError: "file_path must be an absolute path, not a relative path" };
  return;
}
```

### 3. Image Type Detection & Encoding (2510.js, lines 57-95)

```javascript
let E = await D(L),
  f = rtH.getType(E);
if (f?.startsWith("image/") && f !== "image/svg+xml" && f) {
  if (!f2.includes(f)) { /* reject unsupported */ return; }
  let B = await AKH.readFile(E),
    W = await AKH.stat(E);
  if (W.size > JgH) { /* reject > 5MB */ return; }
  $rH(E, H.toolCallId);
  let V = await bc(B, f),  // compress image
    X = V.contentType || f,
    Q = [
      { type: "text", text: `Image file: ${SzI.basename(E)} (${(W.size / 1024).toFixed(1)} KB)` },
      { type: "image", source: { type: "base64", mediaType: X, data: V.buffer.toString("base64") } },
    ];
  yield { type: "result", isError: false, value: Q };
}
```

### 4. Supported Image Media Types (1148.js, line 3)

```javascript
f2 = ["image/jpeg", "image/png"];
```

Max image size: `JgH = 5242880` (5 MB, 1147.js line 8).

### 5. Create Tool Handler: Write Protection (2320.js, lines 77-112)

```javascript
if (MMH(L)) {
  yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
    llmError: "Cannot write to artifacts directory. This directory is reserved for system-generated outputs." };
  return;
}
if (_MH(L)) {
  yield { type: "result", isError: true, errorType: "invalidParameterLLMError",
    llmError: "Cannot write to mission system files (state.json, progress_log.jsonl)." };
  return;
}
```

### 6. File Creation with Auto-Mkdir (2320.js, lines 115-123)

```javascript
let I = Mm.dirname(L);
await JoH.mkdir(I, { recursive: true });
let D = $.endsWith("\n") ? $ : `${$}\n`;  // normalize trailing newline
await JoH.writeFile(L, D, "utf8");
wmA(L);  // track write
```

### 7. Text Chunking with Character Limit (2509.js, lines 5-55)

```javascript
function RzI(H, A = 0, L = 2400, $ = false, I = 60000) {
  let D = H.split("\n"), E = D.length, f;
  if (A < 0) f = Math.max(0, E + A);  // negative offset = from end
  else f = Math.max(0, A);
  let M = Math.min(E, f + L),
    U = D.slice(f, M);
  // ... character limit enforcement at 60000 chars ...
  // ... annotation: "[Showing lines X-Y of N total lines, truncated to Nk characters]"
  return B;
}
```

### 8. AM/PM Unicode Normalization Fallback (2510.js, lines 39-55)

```javascript
let D = async (E) => {
  try { return (await AKH.access(E), E); }
  catch (f) {
    if (typeof f === "object" && f !== null && "code" in f && f.code === "ENOENT") {
      let M = E.replace(" PM", "\u202FPM").replace(" AM", "\u202FAM");  // narrow no-break space
      if (M !== E) try { return (await AKH.access(M), M); } catch {}
    }
    throw f;
  }
};
```

This handles a Windows path edge case where file paths contain " AM"/" PM" but the actual filesystem uses Unicode narrow no-break space before AM/PM.

## Integration Points

### Cross-Mission Dependencies

- **Terminal UI section (TUI)**: Read tool results are displayed in the terminal UI's file viewer component. Image results render as inline base64 previews. Create tool results trigger diff display in the TUI.
- **Orchestration section (Mission)**: The `captureToolFileChange()` call in Create handler integrates with the mission validation system to track file changes for feature completion verification.
- **Desktop GUI section (GUI)**: Skill file creation (`Y0f()` check) triggers `ideClient.openFile()` to open the created skill in the IDE.
- **Infrastructure section (Infra)**: File tracking (`$rH`, `wmA`) feeds into the telemetry/metrics system (`O1.addToCounter`). The `MMH()` and `_MH()` protection guards reference `.opendroid/` infrastructure paths.

### Tool System Internal Dependencies

- **eI() factory** (1201.js): Used by both Read and Create tool registrations
- **k0() registry** (1177.js): Stores registered tools for lookup and LLM manifest generation
- **Permission engine**: Both tools marked `requiresConfirmation: false`, `executionLocation: "client"`
- **Sandbox**: Write protection (`MMH`, `_MH`) serves as a complementary layer to the sandbox system

## Implementation Notes

### Read Tool API

```typescript
interface ReadToolInput {
  file_path: string;   // absolute path required
  offset?: number;     // 0-based line number, default 0
  limit?: number;      // max lines, default 2400
}

type ReadToolOutput = 
  | string                              // text files
  | Array<{                             // image files
      type: "text" | "image";
      text?: string;
      source?: {
        type: "base64";
        media_type: "image/jpeg" | "image/png";
        data: string;
      };
    }>;
```

**Key implementation notes:**
- Use Node.js `path.isAbsolute()` for path validation
- Use `fs/promises` for async file operations
- Image max size: 5MB, supported types: JPEG, PNG
- Text chunking: 2400 lines default, 60K char hard limit
- Negative offset supported (counts from end of file)

### Create Tool API

```typescript
interface CreateToolInput {
  file_path: string;   // file path
  content: string;     // file content
}

// CLI variant adds repoLocation for workspace-aware paths
interface CreateFullInput extends CreateToolInput {
  repoLocation: { type: string; path: string; };
}
```

**Key implementation notes:**
- Auto-creates parent directories (`mkdir -p`)
- Appends trailing newline if content doesn't end with one
- Must implement write protection for artifacts/ and mission system files
- File write tracking for telemetry
- Skill file detection for IDE integration (optional)

### Protection Model

Port the two guard functions:
1. `MMH(path)` → resolve path, check if under `~/.opendroid/artifacts/`
2. `_MH(path)` → resolve path, check if matches `state.json` or `progress_log.jsonl` patterns

Both should return `{ isError: true, errorType: "invalidParameterLLMError" }` on violation.

### Image Compression Pipeline

For the Read tool's image handling, port:
1. MIME type detection (extension-based → `mime-db` or similar)
2. JPEG/PNG decode (use `sharp` or `jimp` for portability)
3. Resize if needed (max 1024px dimension)
4. Re-encode to JPEG with quality control
5. Base64 encode the result

## Open Questions / Left Undone

- **PDF handling in Read tool**: The Read handler (2510.js) does NOT have explicit PDF reading logic: it falls through to text mode for PDFs. PDF handling exists in the attachment/LLM layer (0942.js, 1185.js) where `application/pdf` documents get base64-encoded and sent as `document` blocks with `parsedData`. The Read tool itself reads PDFs as text (likely garbled). This is a potential gap or design choice.
- **`image_quality` parameter**: The public tool description mentions `image_quality` parameter (`"default"` vs `"high"`) but the deobfuscated code in the Read handler (2510.js) does NOT expose or use this parameter: the schema `mGH` only has `{file_path, offset, limit}`. The `image_quality` option may be handled at a higher layer (LLM manifest or UI layer) not yet traced.
- **Create tool absolute path enforcement**: Unlike Read which explicitly checks `path.isAbsolute()`, the Create handler (2320.js) does not validate for absolute paths: it accepts relative paths. The path gets resolved via `path.resolve()` in tracking but the raw path is used for `writeFile`. This may be intentional (CLI context) or a gap.
- **GKA constant**: The Read tool's `llmId` is set to `GKA` (a runtime constant). Its value was not traced in this analysis.
- **`sp()` function**: The Create handler calls `sp(ideClient, path, 1, 500)` after writing to read back the file for diff computation. This function was not traced.
