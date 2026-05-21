# Main Process Helper Files: GUI Architecture Notes

## Overview

This report analyzes the two auxiliary files in the `build/` directory: `session-search.js` (small helper chunk) and `main.js` (~50 bytes). **`session-search.js` is a session search engine module**: a self-contained library implementing bloom filter-based full-text search over OpenDroid's JSONL session logs. It handles bloom filter construction/serialization, trigram-based fuzzy matching with Levenshtein distance fallback, multi-worker parallel search, and incremental index caching. **`main.js` is a stub entry point**: a single line that requires `main-process.js`, making it the Electron main process bootstrap file that delegates all logic to the primary main process bundle. Neither file contains `ipcMain` handlers, `BrowserWindow` calls, or daemon logic. `session-search.js` is a pure utility/data module that supplements the main process with search capabilities.

## File Map

| File | Size | Role |
|------|------|------|
| `build/session-search.js` | small helper chunk | Session search engine: bloom filter indexing, trigram fuzzy search, worker pool, JSONL parsing |
| `build/main.js` | ~50 bytes | Electron main process entry stub: requires main-process.js |
| `build/main-process.js` | main process bundle | Primary main process bundle: BrowserWindow, ipcMain handlers, daemon WebSocket |
| `build/preload.js` | small preload bridge | IPC bridge: contextBridge.exposeInMainWorld (not analyzed here) |

## Architecture

### Build/ Directory Dependency Graph

```
main.js  ──requires──►  main-process.js  (Electron entry chain)
                          ▲
session-search.js ──requires──┘  (imports shared symbols)
```

**`main.js`** is the Electron entry point referenced by the application manifest. Its sole purpose is to bootstrap `main-process.js` via `require("./main-process.js")`. This is a standard Vite/Electron build pattern where the entry file is a thin wrapper.

**`session-search.js`** is a **code-split chunk** from the same Vite build. It is NOT an independent entry point: it is a module imported by (or importable from) the main process bundle. It requires `main-process.js` to access shared symbols (constants, logging, utility functions). This is the Vite code-splitting pattern: `main-process.js` is the "shared chunk" and `session-search.js` is a "lazy chunk" containing the search subsystem.

### Module Internal Architecture (session-search.js)

```
┌─────────────────────────────────────────────────────────┐
│                session-search.js                         │
├─────────────────────────────────────────────────────────┤
│  Bloom Filter Core                                       │
│  ├── createBloom(): 2048-bit filter (64 × Uint32)      │
│  ├── bloomAddText(): add trigrams with double-hash     │
│  ├── bloomScoreForQuery(): trigram overlap scoring      │
│  ├── bloomToBase64() / bloomFromBase64(): serialize    │
│  └── SHA-1 hashing for bloom positions                   │
├─────────────────────────────────────────────────────────┤
│  Text Processing                                         │
│  ├── stripSystemReminders(): remove <system-reminder>   │
│  ├── extractStrings(): recursive string extraction      │
│  ├── buildSnippet(): context-aware match highlighting   │
│  └── extractTrigrams(): 3-char substring generation     │
├─────────────────────────────────────────────────────────┤
│  Document Extractors (4 kinds)                           │
│  ├── MessageTextExtractor: text blocks from messages    │
│  ├── DocumentExtractor: file/document content blocks    │
│  ├── ToolUseExtractor: tool invocation metadata+input   │
│  └── ToolResultExtractor: tool output content            │
├─────────────────────────────────────────────────────────┤
│  Search Engine                                           │
│  ├── Session discovery (scans ~/.opendroid/sessions/)      │
│  ├── Bloom index cache (manifest.json)                   │
│  ├── Incremental reindexing (tail-append only)           │
│  ├── Multi-worker parallel search (Worker Threads)       │
│  └── Fuzzy fallback (Levenshtein + trigram overlap)      │
├─────────────────────────────────────────────────────────┤
│  Worker Thread Pool                                      │
│  ├── Spawns up to min(cpus, 8) workers                   │
│  ├── Inline worker code (eval:true)                      │
│  ├── 30-second timeout per search batch                  │
│  └── Fallback to single-threaded on worker failure       │
└─────────────────────────────────────────────────────────┘
```

### Three-File Relationship Summary

| Aspect | main.js | main-process.js | session-search.js |
|--------|---------|--------------------|--------------------|
| **Role** | Entry stub | Main process logic | Search engine utility |
| **ipcMain** | No | Yes (all handlers) | No |
| **BrowserWindow** | No | Yes (creation) | No |
| **Daemon WebSocket** | No | Yes (client) | No |
| **Node APIs** | No | Yes (extensive) | Yes (fs, path, crypto, readline, os, worker_threads) |
| **Contains exports** | No | Yes (shared symbols) | Yes (7 named exports) |
| **Size** | ~50B | main process bundle | small helper chunk |

## Key Findings

### 1. main.js is a Pure Bootstrap Stub

The file contains exactly one line of functional code:
```js
"use strict";require("./main-process.js");
```
This is the Electron `main` entry referenced in `package.json` or electron-builder config. Its only job is to load the real main process bundle. This is a common Vite+Electron build artifact.

### 2. session-search.js is a Session Search Engine

This module implements a complete **bloom filter-based full-text search** over OpenDroid's session JSONL files. Key capabilities:

- **Bloom filter indexing**: 2048-bit filters with double-hashing (SHA-1 + XOR variant) for trigram-based probabilistic set membership
- **4 document extractors**: MessageText, Document, ToolUse, ToolResult: each extracts searchable text from session event blocks
- **Incremental indexing**: Only re-indexes appended bytes (tail updates), handles file shrinks by rebuilding
- **Persistent cache**: Stores bloom filters as base64 in a manifest.json, with schema versioning
- **Worker thread pool**: Parallelizes search across sessions using Node.js `worker_threads`
- **Fuzzy matching**: Trigram overlap scoring combined with Levenshtein edit distance for approximate string matching
- **Snippet generation**: Context-aware match highlighting with `<mark>` tags and code fence preservation

### 3. No ipcMain/BrowserWindow/Daemon Logic in session-search.js

This module contains ZERO Electron API calls. It does not:
- Register `ipcMain.handle` or `ipcMain.on` handlers
- Create `BrowserWindow` instances
- Connect to the daemon via WebSocket
- Use `app.` lifecycle hooks
- Interact with the renderer process in any way

It is a **pure Node.js data processing module** that happens to run in the main process context.

### 4. Dependency on main-process.js for Shared Symbols

`session-search.js` imports the following from `main-process.js` (via `l = require("./main-process.js")`):

| Symbol | Usage |
|--------|-------|
| `SYSTEM_REMINDER_START` | String constant for stripping `<system-reminder>` tags from text |
| `SYSTEM_REMINDER_END` | String constant for closing tag `</system-reminder>` |
| `SessionSearchDocKind` | Enum object: `{ MessageText, Document, ToolUse, ToolResult }` |
| `logWarn` | Logging function for warnings |
| `logInfo` | Logging function for info messages |
| `logException` | Logging function for exceptions with context |
| `getOpenDroidHome` | Returns the OpenDroid home directory path |
| `OPENDROID_DIR_NAME` | Constant: `.opendroid` directory name |

This confirms Vite's code-splitting strategy: shared constants/utilities live in the primary bundle, and the search module imports them.

### 5. Inline Worker Thread Code

The worker thread code is embedded as a template literal string (stored in variable `Ge`, ~200 lines) and spawned via `new ve.Worker(Ge, {eval: true})`. The worker code is a self-contained search function that:
- Receives `{type: "search", task}` messages
- Reads JSONL files line-by-line
- Extracts documents and performs text matching
- Returns `{type: "result", result}` messages
- Posts `{type: "ready"}` on initialization

This avoids needing a separate worker file in the build output.

### 6. Search Configuration Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `q` (BLOOM_SIZE) | 2048 | Bloom filter size in bits |
| `Ce` (LIMIT_SESSIONS) | 100 | Max sessions to search |
| `Ae` (LIMIT_HITS) | 3 | Max hits per session per kind |
| `Oe` (CONTEXT_CHARS) | 80 | Context chars around match |
| `Ne` (CANDIDATE_MULTIPLIER) | 50 | Bloom candidate over-sampling |
| `Pe` (MAX_CANDIDATES) | 500 | Max bloom-filtered candidates |
| `Ke` (EMPTY_QUERY_SCORE) | 0.1 | Fallback score for empty queries |
| `se` (BATCH_SIZE) | 16 | Session search batch size |
| `Le` (MIN_FUZZY_LEN) | 5 | Minimum query length for fuzzy matching |
| `Fe` (MIN_TRI_OVERLAP) | 0.8 | Minimum trigram overlap ratio |
| `X` (SCHEMA_VERSION) | 2 | Index cache schema version |
| `Ve` (SESSION_STAT_CONCURRENCY) | 50 | Concurrent file stat calls |

## Code Examples

### main.js: Complete Content
```js
// File: build/main.js (~50 bytes)
"use strict";require("./main-process.js");
```
*Reference: Full file, line 1*

### session-search.js: Module Header and Imports
```js
"use strict";
Object.defineProperty(exports, Symbol.toStringTag, { value: "Module" });
const ce = require("crypto"),
      _e = require("fs"),
      Ee = require("path"),
      Me = require("readline"),
      l = require("./main-process.js"),  // Shared symbols from main bundle
      Re = require("os"),
      ve = require("worker_threads");
```
*Reference: File start, line 1*

### Bloom Filter Double-Hash Insertion
```js
function qe(e, n) {
    const s = de(n),            // SHA-1 hash of trigram
          t = (s ^ GOLDEN_RATIO_32) >>> 0;  // XOR with golden ratio constant
    oe(e, s % q),               // Set bit at position hash1 % 2048
    oe(e, t % q);               // Set bit at position hash2 % 2048
}
```
*Reference: offset ~0, function qe*

### System Reminder Stripping
```js
function j(e) {
    if (!e) return e;
    let n = e;
    for (;;) {
        const s = n.indexOf(l.SYSTEM_REMINDER_START);  // "<system-reminder>"
        if (s < 0) break;
        const t = n.indexOf(l.SYSTEM_REMINDER_END, s + l.SYSTEM_REMINDER_START.length);
        if (t < 0) { n = n.slice(0, s); break; }
        n = `${n.slice(0, s)}${n.slice(t + l.SYSTEM_REMINDER_END.length)}`;
    }
    return n;
}
```
*Reference: offset ~0, function j: removes system-reminder XML tags from searchable text*

### Exports (End of File)
```js
exports.SessionSearchDocKind = l.SessionSearchDocKind;
exports.bloomAddText = fe;
exports.bloomFromBase64 = te;
exports.bloomScoreForQuery = he;
exports.bloomToBase64 = le;
exports.buildSnippet = G;
exports.createBloom = ee;
exports.runDroidSearch = yt;
```
*Reference: File end: 8 named exports*

### Worker Pool Launch
```js
const et = Math.min(je.cpus().length, 8);  // Max 8 workers

async function tt(e, n, s, t = et) {
    if (e.length === 0) return [];
    // ... spawn workers with inline code
    for (let f = 0; f < i; f++) {
        const u = new ve.Worker(Ge, { eval: true });  // Ge = inline worker code string
        v(u);  // attach message/error/exit handlers
        o.push(u);
    }
    // 30-second timeout
    k = setTimeout(() => { /* return partial results */ }, 3e4);
}
```
*Reference: offset ~mid-file, function tt*

### Search Index Cache Path
```js
function ge() {
    return O.join(l.getOpenDroidHome(), l.OPENDROID_DIR_NAME, "cache", "search");
}
function ye() {
    return O.join(ge(), "manifest.json");
}
```
*Reference: offset ~mid-file: cache stored at `~/.opendroid/cache/search/manifest.json`*

### Main Search Entry Point
```js
async function yt(e, n) {
    // e = query string, n = options
    const t = {
        kind: n?.kind ?? "all",
        limitSessions: n?.limitSessions ?? 100,
        limitHitsPerSession: n?.limitHitsPerSession ?? 3,
        contextChars: n?.contextChars ?? 80,
        reindex: n?.reindex ?? false
    };
    // ... build bloom index cache, filter candidates, search
    return { query: e, sessions: v };
}
```
*Reference: File end, function yt: the primary export `runDroidSearch`*

## Theme / Style Tokens

Not applicable for this feature: these are Node.js main process modules with no CSS/UI rendering.

## IPC Channels

**Not applicable for this feature.** Neither `main.js` nor `session-search.js` contains any IPC channel registrations (`ipcMain.handle`, `ipcMain.on`) or BrowserWindow interactions. The search module is a pure data-processing utility invoked by the main process logic in `main-process.js`, likely through an `ipcMain.handle` handler defined there (e.g., a "search-sessions" channel).

The search module's `runDroidSearch` function is presumably called by an IPC handler in `main-process.js` when the renderer requests session search via `window.api.*`. The specific channel name would be documented in the `gui-main-process-ipc.md` report.

## Integration Points

### Cross-System Dependencies

1. **main-process.js → session-search.js**: The main process bundle imports and calls `runDroidSearch` when the renderer requests session search. The search module depends on shared symbols (logging, constants, home directory path) from the main bundle.

2. **File System**: The search engine reads JSONL session files from `~/.opendroid/sessions/` (discovered via `getOpenDroidHome()` + `OPENDROID_DIR_NAME`). Session files follow the naming convention `{sessionId}.jsonl` with optional project-scoped subdirectories (prefixed with `-`).

3. **Search Cache**: Bloom filter index is cached at `~/.opendroid/cache/search/manifest.json` with incremental updates (tail-append only, shrink-detection rebuild).

4. **Worker Threads**: Uses Node.js `worker_threads` module for parallel session search. Workers communicate via `parentPort` message passing with `{type: "search"|"result"|"ready"}` message protocol.

5. **System Reminder Stripping**: Both the main module and the inline worker code strip `<system-reminder>` XML tags from text before indexing/searching: this is the same processing applied by the AI model runtime.

### Cross-Mission References

- **Session JSONL format**: The JSONL structure (session_start, message events with content blocks) is shared with 01-terminal-ui (TUI) and 03-tool-agent-system (Tool & Agent): these missions may document the schema in more detail.
- **getOpenDroidHome() path**: This function and the `.opendroid` directory structure are documented in 05-infrastructure (Infrastructure).
- **Session search IPC channel**: The channel that triggers `runDroidSearch` is documented in `gui-main-process-ipc.md` (the ipcMain handler that calls this module's `runDroidSearch` export).

## Implementation Notes

### main.js Porting

The `main.js` stub pattern is straightforward to replicate:
1. Create a minimal entry file that requires the primary bundle
2. This is a standard Electron + bundler pattern: no OpenDroid-specific logic here
3. In a custom Electron setup, this could be replaced with a direct entry point reference in `package.json` or electron-builder config

### session-search.js Porting

This is a **high-value module for OpenDroid**: it implements session search functionality:

1. **Bloom filter search can be reused as-is**: The bloom filter implementation (`createBloom`, `bloomAddText`, `bloomScoreForQuery`) is generic and framework-independent. It could be extracted into a standalone npm package.

2. **Document extractors define the session data model**: The 4 extractor types (MessageText, Document, ToolUse, ToolResult) and their field mappings reveal the exact JSONL schema. This is critical for any system that needs to parse OpenDroid session files.

3. **Worker thread pool pattern**: The inline worker code + pool management pattern is portable to any Node.js application. The `Promise.race`-based concurrency control with timeout is a clean pattern to replicate.

4. **Incremental indexing**: The tail-append reindexing strategy (only process new bytes since last index) is efficient and should be preserved in a port.

5. **Dependency injection needed**: The module imports `getOpenDroidHome`, `OPENDROID_DIR_NAME`, `logWarn`, `logInfo`, `logException`, `SessionSearchDocKind` from the main bundle. For OpenDroid, these need to be replaced with local implementations:
   - `getOpenDroidHome()` → OpenDroid's config directory resolver
   - `OPENDROID_DIR_NAME` → OpenDroid's data directory name
   - `log*` → OpenDroid's logging system
   - `SessionSearchDocKind` → OpenDroid's session document type enum
   - `SYSTEM_REMINDER_START/END` → May not be needed if OpenDroid doesn't use system reminders

6. **Search API surface**: The exported `runDroidSearch(query, options)` function accepts `{kind, limitSessions, limitHitsPerSession, contextChars, reindex}` and returns `{query, sessions}`: this is the public API to preserve.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| main.js | Line 1 | `require("./main-process.js")` | Complete file: sole purpose is loading primary bundle |
| session-search.js | Start | `require("crypto")` | SHA-1 hashing for bloom filter positions |
| session-search.js | Start | `require("fs")` | File system operations for JSONL reading and cache |
| session-search.js | Start | `require("path")` | Path joining for session directories |
| session-search.js | Start | `require("readline")` | Line-by-line JSONL file reading |
| session-search.js | Start | `require("./main-process.js")` as `l` | Shared symbols import (8 symbols used) |
| session-search.js | Start | `require("os")` | CPU count for worker pool sizing |
| session-search.js | Start | `require("worker_threads")` | Worker thread pool for parallel search |
| session-search.js | Early | `createBloom()` → `ee()` | Creates 2048-bit bloom filter (64 Uint32 entries) |
| session-search.js | Early | `bloomFromBase64()` → `te()` | Deserializes bloom from base64 string |
| session-search.js | Early | `bloomToBase64()` → `le()` | Serializes bloom to base64 string |
| session-search.js | Early | `bloomAddText()` → `fe()` | Adds trigrams to bloom filter |
| session-search.js | Early | `bloomScoreForQuery()` → `he()` | Scores bloom filter against query |
| session-search.js | Early | `stripSystemReminders()` → `j()` | Removes `<system-reminder>` tags |
| session-search.js | Early | `extractStrings()` → `Y()` | Recursive string extraction from objects |
| session-search.js | Early | `buildDocId()` → `U()` | Generates `sessionId:kind:eventId:blockKey` IDs |
| session-search.js | Mid | `DocumentExtractor` → `Ue()` | Extracts document blocks (file name, path, content) |
| session-search.js | Mid | `MessageTextExtractor` → `$e()` | Extracts text blocks with role (user/assistant) |
| session-search.js | Mid | `ToolResultExtractor` → `He()` | Extracts tool result content |
| session-search.js | Mid | `ToolUseExtractor` → `Ye()` | Extracts tool name + input |
| session-search.js | Mid | `getExtractors()` → `pe()` | Returns all 4 extractors with config |
| session-search.js | Mid | `promisePool()` → `Ze()` | Generic concurrent promise executor |
| session-search.js | Mid | `getSessionsDir()` → `Qe()` | Returns `~/.opendroid/sessions/` path |
| session-search.js | Mid | `discoverSessions()` → `Xe()` | Scans session dir for JSONL files |
| session-search.js | Mid | `Ge` (template literal) | Inline worker thread code (~200 lines) |
| session-search.js | Mid | `workerPoolSearch()` → `tt()` | Multi-worker parallel search with timeout |
| session-search.js | Mid | `extractFromMessage()` → `me()` | Extracts search docs from a message event |
| session-search.js | Mid | `buildSnippet()` → `G()` | Context-aware match highlighting with `<mark>` |
| session-search.js | Mid | `getSearchCacheDir()` → `ge()` | Returns `~/.opendroid/cache/search/` |
| session-search.js | Mid | `getManifestPath()` → `ye()` | Returns `~/.opendroid/cache/search/manifest.json` |
| session-search.js | Mid | `indexSession()` → `Q()` | Bloom-indexes a JSONL file from offset |
| session-search.js | Mid | `loadManifest()` → `at()` | Loads/validates search cache manifest |
| session-search.js | Mid | `saveManifest()` → `lt()` | Persists search cache manifest |
| session-search.js | Mid | `updateIndex()` → `dt()` | Incremental reindexing of all sessions |
| session-search.js | Mid | `computeConfigHash()` → `ut()` | Hashes search config for cache validation |
| session-search.js | Mid | `levenshtein()` → `ht()` | Edit distance computation |
| session-search.js | Mid | `fuzzyScoreToken()` → `pt()` | Trigram + edit distance combined scoring |
| session-search.js | Late | `searchSession()` → `gt()` | Searches a single session JSONL file |
| session-search.js | End | `runDroidSearch()` → `yt()` | **Main export**: full search pipeline |
