# infra-wasm-runtime: Architecture Notes

## Overview

OpenDroid's WASM runtime integration centers on **Yoga** (Facebook's cross-platform layout engine) compiled to WebAssembly via Emscripten with embind bindings, and **protobuf.js** with inline WASM for 64-bit integer operations. Contrary to the initial assumption, the codebase does **not** embed tiktoken or tree-sitter WASM modules directly. Token counting for LLM context management uses a lightweight character-based heuristic (`Math.ceil(charCount / 4)`) and the Anthropic API's server-side `/v1/messages/count_tokens` endpoint. Syntax highlighting is provided by highlight.js (including a WASM text-format language definition), not tree-sitter. The WASM loading follows standard Emscripten patterns: base64-encoded wasm binary, WebAssembly.instantiate/instantiateStreaming, typed array memory views, and `_malloc`/`_free` memory management.

## Module Map

| Module | Size | Role | Category |
|--------|------|------|----------|
| 2218.js | 1524 lines (~60KB) | Yoga flexbox WASM runtime (Emscripten + embind) | App |
| 2659.js | 587 lines (~23KB) | protobuf.js Long type with inline WASM for int64 ops | Vendor |
| 3351.js | 201 lines (~8KB) | highlight.js language registration (includes "wasm" lang) | Vendor |
| 3345.js | 92 lines (~4KB) | highlight.js WASM text-format language grammar | Vendor |
| 3246.js | 467 lines (~12KB) | highlight.js JavaScript/HTML language def (mentions WebAssembly keyword) | Vendor |
| 1886.js | 320 lines (~11KB) | PostgreSQL (pg) OpenTelemetry instrumentation | Vendor |
| 1109.js | 314 lines (~10KB) | MCP client with SSE transport (no WASM) | App |
| 2194.js | 326 lines (~10KB) | FileSnapshotService (no WASM) | App |
| 2366.js | ~80 lines | Anthropic beta API client with countTokens() | App |
| 2373.js | ~40 lines | Anthropic stable API client with countTokens() | App |
| 2569.js | 384 lines (~15KB) | Compaction/token estimation (char-based heuristic) | App |
| 2477.js | ~100 lines | File type detection (".wasm" → BINARY) | App |

## Architecture

### WASM Loading Mechanism

The primary WASM module (2218.js: Yoga) uses the standard **Emscripten module pattern**:

1. **Binary embedding**: The WASM binary is embedded as a `data:application/octet-stream;base64,...` data URI directly in the JavaScript module (line 92). This eliminates the need for separate `.wasm` file loading at runtime.

2. **Instantiation chain**: The module tries `WebAssembly.instantiateStreaming()` first (fetching from the data URI), falling back to `WebAssembly.instantiate()` with an ArrayBuffer if streaming fails (line 1445-1455).

3. **Memory management**: A shared `WebAssembly.Memory` buffer (`B`) is managed through typed array views: `HEAP8` (Int8Array), `HEAP16`, `HEAP32`, `HEAPU8` (Uint8Array), `HEAPU16`, `HEAPU32`, `HEAPF32`, `HEAPF64`. Memory grows via `B.grow()` with 64KB-aligned chunks (line 1319-1330).

4. **embind bindings**: The module uses Emscripten's embind system to expose C++ classes (`YGNode`, layout config) to JavaScript. Functions like `_malloc`, `_free`, `___getTypeName`, `__embind_initialize_bindings` are exported (lines 1461-1476).

5. **FinalizationRegistry**: Uses `FinalizationRegistry` for garbage collection of WASM-backed objects (ErrTracker init section7-264).

### Token Counting Architecture

OpenDroid does **not** use tiktoken WASM for token counting. Instead:

- **Character-based estimation** (2569.js, line 11): `Math.ceil(charCount / 4)`: a simple heuristic that divides total character count by 4. This provides a rough upper bound for token counts.
- **Server-side counting** (2366.js, 2373.js): The Anthropic API exposes a `/v1/messages/count_tokens` endpoint that returns exact token counts. The beta version uses `?beta=true` with a `token-counting-2024-11-01` beta header.
- **Context budget calculation** (2569.js): Functions `E8f()` and `WRI()` sum character counts across system prompts, tool definitions, and message content, then apply the `/4` heuristic.

### protobuf.js WASM (2659.js)

An inline WASM module containing just 5 functions (`mul`, `div_s`, `div_u`, `rem_s`, `rem_u`, `get_high`) for efficient 64-bit integer arithmetic. Created via `new WebAssembly.Instance(new WebAssembly.Module(new Uint8Array([...]))`: no file loading needed.

### Syntax Highlighting (not tree-sitter)

The codebase uses **highlight.js** for syntax highlighting, not tree-sitter. Module 3351.js registers ~150 languages, and 3345.js provides a WASM text-format grammar for highlighting `.wasm` WAT/WAST files. The mention of "WebAssembly" in 3246.js and 3259.js is as a JavaScript built-in keyword identifier, not as actual WASM functionality.

## Key Findings

1. **No tiktoken WASM**: Token counting is done via `charCount / 4` heuristic and Anthropic's server-side API. No WASM-based tokenizer is bundled. This is a deliberate architectural choice: the heuristic is fast and requires no WASM overhead for token estimation.

2. **Yoga WASM is the primary WASM consumer**: The Emscripten-compiled Yoga flexbox engine (2218.js) is the largest WASM module, used for computing CSS-like flex layouts. Source path references were observed as upstream CI build paths and are not required for runtime behavior.

3. **Base64-embedded WASM binaries**: Both WASM modules (Yoga and protobuf.js) embed their binaries as base64 data URIs, avoiding external `.wasm` file loading. This simplifies deployment but increases JS bundle size.

4. **Standard Emscripten memory pattern**: The Yoga module follows the standard Emscripten memory model with typed array HEAP views, `_malloc`/`_free` for manual memory management, and `FinalizationRegistry` for GC integration.

5. **WebAssembly.Exception handling**: ErrTracker's error handling (1358.js) recognizes `WebAssembly.Exception` as an error type, indicating the runtime handles WASM-originated errors.

## Code Examples

### Token estimation heuristic (2569.js, line 11-12)
```js
function oZ(H) {
  return Math.ceil(H / 4);
}
```

### WASM instantiation with streaming fallback (2218.js, lines 1445-1455)
```js
typeof WebAssembly.instantiateStreaming != "function" || QH(EH) || typeof fetch != "function"
  ? JA(HA)
  : fetch(EH, { credentials: "same-origin" }).then(function (rA) {
      return WebAssembly.instantiateStreaming(rA, SA).then(HA, function (VL) {
        return (M("wasm streaming compile failed: " + VL),
                M("falling back to ArrayBuffer instantiation"), JA(HA));
      });
    });
```

### HEAP memory view setup (2218.js, lines 55-65)
```js
function g() {
  var tH = B.buffer;
  ((X = tH),
    (L.HEAP8 = Q = new Int8Array(tH)),
    (L.HEAP16 = C = new Int16Array(tH)),
    (L.HEAP32 = q = new Int32Array(tH)),
    (L.HEAPU8 = w = new Uint8Array(tH)),
    // ... HEAPU16, HEAPU32, HEAPF32, HEAPF64
  );
}
```

### Anthropic countTokens API (2373.js, lines 31-32)
```js
countTokens(H, A) {
  return this._client.post("/v1/messages/count_tokens", { body: H, ...A });
}
```

### Inline protobuf.js WASM (2659.js, line 42-80)
```js
A = new WebAssembly.Instance(
  new WebAssembly.Module(new Uint8Array([
    0, 97, 115, 109, 1, 0, 0, 0, 1, 13, 2, 96, 0, 1, 127, 96, 4, 127, ...
  ])), {}
).exports;
```

## Integration Points

- **Yoga WASM → GUI (04-desktop-gui)**: The flexbox layout engine computes node positions for the GUI layer. The `YGNode` class manages a tree of layout nodes with flex properties.
- **Token estimation → Session Manager**: The `oZ()` heuristic feeds into compaction logic (2569.js) that determines when conversation history should be summarized.
- **Token estimation → Config Loader**: Context window sizes (`contextMaxTokens`) are computed using the same heuristic to enforce model token limits.
- **Anthropic API → Token counting**: The `countTokens()` method on the Anthropic client provides exact server-side token counts for precise context budgeting.
- **highlight.js → TUI (01-terminal-ui)**: Syntax highlighting for code rendering in terminal sessions, including WASM text format support.
- **protobuf.js WASM → IPC/Session**: 64-bit integer support enables proper protobuf serialization for session data and IPC messages.
- **File type detection (2477.js)**: `.wasm` files classified as BINARY type for file handling operations.

## Implementation Notes

1. **Yoga WASM replacement**: If porting to a native environment, replace the WASM Yoga with a native Yoga C++ binding or a JS-native flexbox implementation (e.g., `yoga-layout-prebuilt`). The embind API surface is: `Node.create()`, `node.set*()`, `node.calculateLayout()`, `node.getComputed*()`.

2. **Token counting**: Replace the `/4` heuristic with a proper tiktoken WASM module if exact token counting is needed offline. The current heuristic is adequate for estimation but not precise.

3. **protobuf.js inline WASM**: The inline WASM module for int64 arithmetic can be replaced with `BigInt` in modern JS environments, or kept as-is since it's self-contained.

4. **highlight.js**: Can be replaced with any syntax highlighting library (Prism, Shiki, tree-sitter-wasm) depending on requirements. The current implementation includes WASM text format highlighting.

5. **WASM binary embedding**: Consider extracting base64-embedded WASM to separate `.wasm` files for better caching and lazy loading in production.

## Module Reference

| Module | Lines | Key Functions/Classes |
|--------|-------|----------------------|
| 2218.js:55-65 | HEAP memory setup | `g()`: typed array view initialization |
| 2218.js:92 | WASM binary | Base64 data URI containing Yoga flexbox engine |
| 2218.js:257-264 | GC | `FinalizationRegistry` for WASM object cleanup |
| 2218.js:1426 | Instantiation | `WebAssembly.instantiate()` with fallback |
| 2218.js:1445-1455 | Streaming | `WebAssembly.instantiateStreaming()` fallback chain |
| 2218.js:1461-1476 | Exports | `_malloc`, `_free`, `___getTypeName`, `__embind_initialize_bindings` |
| 2659.js:42-80 | Inline WASM | protobuf.js 64-bit integer ops (mul/div/rem) |
| 2569.js:11-12 | Token heuristic | `oZ(H)` = `Math.ceil(H / 4)` |
| 2569.js:54-63 | Context budget | `E8f()`: system prompt + tool token estimation |
| 2366.js:65-74 | Token counting | Anthropic beta `countTokens()` API |
| 2373.js:31-32 | Token counting | Anthropic stable `countTokens()` API |
| 3351.js:170-196 | Language reg | highlight.js registers "wasm" language |
| 3345.js:1-92 | WASM grammar | highlight.js WASM text format syntax definition |
| 3246.js:80-84 | JS keywords | "WebAssembly" in JavaScript built-in keyword list |
| 2477.js:25-30 | File type | `.wasm` extension → BINARY classification |
| 1358.js:7-9 | Error handling | `WebAssembly.Exception` recognition in ErrTracker |
