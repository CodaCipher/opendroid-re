# PDF.js and XLSX Library Integration: GUI Architecture Notes

## Overview

This report documents the architecture-level integration patterns of two vendor libraries shipped as renderer assets: **PDF.js** (`pdf-viewer.chunk.js`, 1.3MB) and **SheetJS XLSX** (`spreadsheet.chunk.js`, 419KB). Both are loaded on-demand via ESM dynamic imports wrapped in Vite's preload helper. They serve a single purpose in the renderer: extracting text/data from user-uploaded file attachments (PDFs, spreadsheets, CSVs) so the content can be fed into the chat context as parsed markdown.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer/main_window/assets/pdf-viewer.chunk.js` | 1.3 MB | PDF.js viewer + parser bundle (ESM, exports `pdfjsLib`) |
| `renderer/main_window/assets/spreadsheet.chunk.js` | 419 KB | SheetJS v0.18.5 (UMD-style, read/write XLSX/CSV/ODS) |
| `renderer/main_window/assets/renderer-main.js` | ~large renderer bundle | Main renderer bundle: contains lazy-load orchestration code |

## Architecture

```
User drops / selects file
       │
       ▼
  l5t(t) in main bundle        ← file type router
       │
   ┌──┴──┬────────┬──────────┐
   ▼      ▼        ▼          ▼
  PDF   DOCX     XLSX/CSV   plain text
   │      │        │          │
   ▼      ▼        ▼          ▼
qUe()   fRi()   pRi()      cRi()
   │      │        │
   ▼      ▼        ▼
import() import()  import()
   │      │        │
   ▼      ▼        ▼
pdfjs-  index-   xlsx-
CHlOTkL_.js C4qR2_ml.js CWc3kuOC.js
```

Both libraries are **not** part of the initial renderer boot chunk. They are fetched only when a user attaches a file whose MIME type or extension matches the supported attachment types.

## Key Findings

### PDF.js Integration

| Aspect | Finding |
|--------|---------|
| **Library version** | PDF.js modern build (ESM, exports `pdfjsLib` globally + named ESM exports) |
| **Load trigger** | On-demand: when user uploads a `.pdf` file (MIME `application/pdf`) |
| **Integration pattern** | ESM dynamic import via Vite `WUe(() => import("./pdf-viewer.chunk.js"), [], import.meta.url)` |
| **Cache strategy** | Singleton module-level variable `XLe`: loaded once, reused for all subsequent PDF parses |
| **Usage context** | Text extraction only (`getDocument` → `sRi` with `mergePages:!0` → `.text`). No UI rendering of PDF pages in the app. |
| **Worker usage** | `PDFWorker` class is present in the bundle; likely used internally by `getDocument` for off-main-thread parsing |
| **WASM** | WebAssembly module (`initSync`, `WebAssembly.Module`) bundled for image decoding |

**Entry-point code in main bundle** (offset ~3175):
```js
let XLe; // singleton cache
async function qUe(t, e={}) {
  const { getDocument } = await K4i();
  return await n({ data: t, isEvalSupported: !1, useSystemFonts: !0, ...e }).promise;
}
async function K4i() { return XLe || await o5t(), XLe; }
async function o5t(t, { reload: e=!1 }={}) {
  if (!(XLe && !e))
    try { XLe = await WUe(() => import("./pdf-viewer.chunk.js"), [], import.meta.url); }
    catch (n) { throw new Error(`Serverless PDF.js bundle could not be resolved: ${n}`); }
}
function PXn(t) { return typeof t == "object" && t !== null && "_pdfInfo" in t; }
```

### XLSX Integration

| Aspect | Finding |
|--------|---------|
| **Library version** | SheetJS `xlsx.js` v0.18.5 (`Ia.version = "0.18.5"`) |
| **Load trigger** | On-demand: when user uploads `.xlsx`, `.xlsm`, `.xlsb`, `.ods`, or `.csv` |
| **Integration pattern** | ESM dynamic import via Vite `WUe(() => import("./spreadsheet.chunk.js"), [], import.meta.url)` |
| **Cache strategy** | No singleton cache visible; imported fresh per call (lightweight enough at 419KB) |
| **Usage context** | Spreadsheet → markdown table conversion (`e.read()` → `e.utils.sheet_to_json()` → pipe-table markdown) |
| **Export formats** | The library supports writing many formats (xlsx, xlsb, csv, html, etc.) but the renderer only uses **read** paths |

**Entry-point code in main bundle** (offset ~3180):
```js
async function pRi(t) {
  const e = await WUe(() => import("./spreadsheet.chunk.js"), [], import.meta.url);
  const n = await t.arrayBuffer();
  const i = e.read(n, { type: "array" });
  return i.SheetNames.map(l => {
    const c = i.Sheets[l];
    const u = e.utils.sheet_to_json(c, { header: 1 });
    // ... converts to pipe-table markdown
  });
}
```

### Supported Attachment Types

Both libraries are referenced from the unified file-type allow-list in the main bundle (offset ~3170):

```js
const lve = [
  "application/pdf",
  "text/plain", "text/markdown", "application/json",
  "text/yaml", "application/x-yaml", "text/xml", "application/xml",
  "text/csv",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
];
const QUe = [".pdf",".txt",".md",".json",".yaml",".yml",".xml",".csv",".docx",".xlsx"];
```

## Code Examples

**PDF text extraction pipeline** (`dRi` in main bundle):
```js
async function dRi(t) {
  const e = await t.arrayBuffer();
  const n = await qUe(new Uint8Array(e));
  return (await sRi(n, { mergePages: !0 })).text;
}
```

**Spreadsheet-to-markdown pipeline** (`pRi` + `hRi` in main bundle):
```js
// hRi converts 2D array to pipe-table markdown
function hRi(t) {
  const e = t[0].map(l => String(l ?? ""));
  const n = e.map(() => "---");
  const i = t.slice(1).map(l => l.map(c => String(c ?? "")));
  return [`| ${e.join(" | ")} |`, `| ${n.join(" | ")} |`,
          ...i.map(l => `| ${l.join(" | ")} |`)].join("\n");
}
```

## IPC Channels

Not applicable for this feature: both libraries run entirely inside the renderer process. No `window.api` or `ipcRenderer` calls are involved in PDF/XLSX parsing.

## Integration Points

- **File upload UI** → `l5t()` router in main bundle decides which parser to invoke based on MIME type / extension.
- **Chat message input** → parsed text from PDF/XLSX is inserted as `type: "text"` attachment content into the message payload.
- **DOCX parser** (`renderer-chunk-a.js`, 471KB) uses the same `WUe` dynamic-import pattern and is triggered for `.docx` files. It is a sibling chunk to pdfjs/xlsx but out of scope for this report.
- **Cross-mission**: The parsed attachment text is sent to the backend via the chat API (`/api/sessions/.../messages` or similar): this boundary is documented in 03-tool-agent-system (Tool & Agent) and 02-orchestration (Mission System).

## Implementation Notes

1. **Preserve lazy-loading**: Both libraries are large (1.3MB + 419KB). Keep them out of the main vendor bundle and load via dynamic `import()` triggered by file drop/selection. The Vite `WUe` helper can be replaced with a simple `import()` in a Vite-based OpenDroid build.
2. **PDF.js singleton cache**: Replicate the `XLe` pattern (or use `import()` caching natively) to avoid re-parsing the 1.3MB bundle on every PDF upload.
3. **SheetJS read-only**: If OpenDroid only needs to ingest spreadsheets, tree-shake the write path (xlsx export, CSV generation, etc.) from SheetJS to reduce bundle size. The current renderer only uses `read()` + `utils.sheet_to_json()`.
4. **Unified file-type router**: The `l5t()` / `$Ue()` router pattern (MIME → parser dispatch) is clean and should be preserved. Extend the `lve` / `QUe` arrays for new attachment types.
5. **PDF.js worker**: PDF.js internally spawns a Web Worker. Ensure the OpenDroid build copies the worker script (`pdf.worker.js`) to the output directory or configure `GlobalWorkerOptions.workerSrc` if the bundled build does not inline it.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `renderer-main.js` | ~3170 | `lve`, `QUe`, `H4i` | MIME type / extension allow-lists for attachments |
| `renderer-main.js` | ~3175 | `o5t()`, `K4i()`, `qUe()` | PDF.js lazy loader + singleton cache |
| `renderer-main.js` | ~3180 | `pRi()`, `sRi()`, `rRi()` | XLSX lazy loader + spreadsheet parser |
| `renderer-main.js` | ~3185 | `l5t()`, `dRi()`, `fRi()`, `hRi()` | Unified file attachment router and format converters |
| `renderer-main.js` | ~3175 | `WUe()` | Vite preload helper (wraps `import()`) |
| `pdf-viewer.chunk.js` | start | polyfills, `globalThis.pdfjsLib` | DOMMatrix, FinalizationRegistry, navigator shims |
| `pdf-viewer.chunk.js` | end | ESM exports | `getDocument`, `PDFWorker`, `AnnotationLayer`, `TextLayer`, etc. |
| `spreadsheet.chunk.js` | start | `Ia.version = "0.18.5"` | SheetJS library header |
| `spreadsheet.chunk.js` | throughout | `ln`, `Zn`, `kc` | Read/write workbook functions (read path used only) |
