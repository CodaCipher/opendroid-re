# gui-renderer-secondary-chunks: GUI Architecture Notes

## Overview

OpenDroid's renderer bundle uses **Vite code-splitting** to lazy-load two heavy feature modules on demand:
1. **Mammoth.js docx parser** (`renderer-chunk-a.js`, 471 KB): extracts raw text from `.docx` attachments
2. **Custom Mermaid-to-SVG renderer** (`renderer-chunk-b.js`, 92 KB): renders diagram code blocks in markdown

Both chunks are loaded via `import()` from the main renderer bundle (`renderer-main.js`, ~large renderer bundle). The main bundle wraps each dynamic import in a `WUe` preloader utility (likely a React.lazy / preload pattern) and caches the result. Neither chunk contains React components directly; they are pure JS libraries re-exported as ESM namespaces.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer-main.js` | ~large renderer bundle | Main renderer bundle (React app): contains dynamic `import()` call sites |
| `renderer-chunk-a.js` | 471 KB | Secondary chunk: Mammoth.js + Underscore.js + Bluebird: docx text extraction |
| `renderer-chunk-b.js` | 92 KB | Secondary chunk: Custom Mermaid parser/renderer + ELK layout: diagram SVG generation |

## Architecture

```
Main Renderer Bundle (renderer-main.js)
├── fRi() ──dynamic-import──► renderer-chunk-a.js ──► extractRawText(docx)
│                              ├── Mammoth.js (docx XML parsing)
│                              ├── Underscore.js (utilities)
│                              └── Bluebird (promises)
│
└── Pmo (Mermaid React component)
    └── useRef callback ──dynamic-import──► renderer-chunk-b.js ──► renderMermaidSVG(code, theme)
                                   ├── HTML entity decoder
                                   ├── Custom parsers: flowchart, sequence, class, state, ER, XY chart
                                   ├── ELK (Eclipse Layout Kernel) Web Worker for graph layout
                                   └── SVG generators per diagram type
```

Both chunks are **ESM entry points** that:
- Import shared utilities from the main bundle (`renderer-main.js`): e.g. `renderer-chunk-b.js` imports `Ne` (likely a shared helper/class)
- Re-export their public API as named exports at the bottom of the file

## Key Findings

### Chunk 1: renderer-chunk-a.js (Docx Parser, 471 KB)

**Purpose:** Lazy-loaded when a `.docx` file attachment is processed in chat. Extracts plain text from Word documents.

**Libraries bundled:**
- **Mammoth.js**: Full docx-to-text pipeline: XML relationship readers, footnotes/endnotes/comments parsers, style mappings, paragraph/run converters, table handlers
- **Underscore.js** (~1.13.8): Utility functions: `isEqual`, `debounce`, `throttle`, `map`, `reduce`, `filter`, `template`, `clone`, `groupBy`, etc.
- **Bluebird** (3.x): Promise library with cancellation, long-stack traces, `Promise.map`, `Promise.reduce`, `coroutine`

**Public API:**
| Export | Symbol | Description |
|--------|--------|-------------|
| `i` (default namespace) | `gg` | Module object containing `extractRawText` |

Actual usage in main bundle:
```js
const { extractRawText } = await import("./renderer-chunk-a.js").then(l => l.i);
```

### Chunk 2: renderer-chunk-b.js (Mermaid Renderer, 92 KB)

**Purpose:** Lazy-loaded when a markdown code block has `language-mermaid`. Renders Mermaid diagram syntax into self-contained SVG strings.

**Not the official `mermaid` npm package**: this is a **custom, lightweight re-implementation** optimized for the app's needs.

**Diagram types supported:**
| Type | Parser function | SVG renderer |
|------|----------------|--------------|
| Flowchart / Graph | `an()` (parseMermaid) | `Xn()` |
| Sequence Diagram | `mn()` | `us()` |
| Class Diagram | `bn()` | `Ss()` |
| State Diagram | `cn()` | `Xn()` (shared with flowchart) |
| ER Diagram | `vn()` | `Rs()` |
| XY Chart | `An()` | `qs()` |

**Layout engine:** ELK (Eclipse Layout Kernel) is loaded via an inline Web Worker (`Ne` class imported from main bundle). The `In()` function initializes the worker synchronously, and `Ft()` dispatches layout jobs to it.

**Theme system:** Dark/light themes via CSS custom properties injected into the SVG `<style>` block. Reads `document.documentElement.getAttribute("data-theme")` at render time.

**Public API:**
| Export | Symbol | Description |
|--------|--------|-------------|
| `DEFAULTS` | `Qt` | Default color tokens (`bg`, `fg`) |
| `parseMermaid` | `an` | Parses Mermaid text → AST object |
| `renderMermaidSVG` | `oi` | Full pipeline: parse → ELK layout → SVG string |

### Dynamic Import Pattern

Both chunks use the same preload wrapper `WUe` in the main bundle:

```js
// Docx chunk (~line 3180 in main bundle)
const { extractRawText } = await WUe(async () => {
  const { extractRawText: o } = await import("./renderer-chunk-a.js").then(l => l.i);
  return { extractRawText: o };
}, [], import.meta.url);
```

```js
// Mermaid chunk (~line 3169 in main bundle, inside Pmo component)
const { renderMermaidSVG } = await WUe(async () => {
  const { renderMermaidSVG: x } = await import("./renderer-chunk-b.js");
  return { renderMermaidSVG: x };
}, [], import.meta.url);
```

`WUe` appears to be a Vite-generated preload helper (`__vitePreload`) that:
1. Adds `<link rel="modulepreload">` tags for the chunk
2. Executes the import factory
3. Caches the result so subsequent calls are instant

## Code Examples

### Docx text extraction call site
**File:** `renderer-main.js`, offset ~3175–3185
```js
async function fRi(t) {
  const { extractRawText: e } = await WUe(async () => {
    const { extractRawText: o } = await import("./renderer-chunk-a.js").then(l => l.i);
    return { extractRawText: o };
  }, [], import.meta.url);
  const n = await t.arrayBuffer();
  return (await e({ arrayBuffer: n })).value;
}
```

### Mermaid SVG rendering call site
**File:** `renderer-main.js`, offset ~3160–3175 (inside `Pmo` React component)
```js
h = B.useRef(async y => {
  try {
    const { renderMermaidSVG: S } = await WUe(async () => {
      const { renderMermaidSVG: x } = await import("./renderer-chunk-b.js");
      return { renderMermaidSVG: x };
    }, [], import.meta.url);
    const C = Nmo(); // theme picker
    const A = S(y, C);
    o(A);
    n(null);
  } catch (S) {
    // error handling
  }
});
```

### Mermaid chunk entry
**File:** `renderer-chunk-b.js`, line 1
```js
import { E as Ne } from "./renderer-main.js";
```

### Mammoth chunk exit
**File:** `renderer-chunk-a.js`, line 220
```js
const gg = Object.freeze(Object.defineProperty({
  __proto__: null,
  extractRawText: fg
}, Symbol.toStringTag, { value: "Module" }));
export { gg as i };
```

### Mermaid chunk exit
**File:** `renderer-chunk-b.js`, line 112
```js
export { Qt as DEFAULTS, an as parseMermaid, oi as renderMermaidSVG };
```

## Theme / Style Tokens

The Mermaid chunk generates SVGs with inline CSS custom properties derived from the app's theme:

```css
--bg: #FFFFFF / #transparent
--fg: #27272A / #e4e4e7
--line: color-mix(in srgb, var(--fg) 50%, var(--bg))
--accent: #3b82f6 / #a78bfa
--muted: color-mix(in srgb, var(--fg) 40%, var(--bg))
--surface: color-mix(in srgb, var(--fg) 3%, var(--bg))
--border: color-mix(in srgb, var(--fg) 20%, var(--bg))
```

These are injected as a `<style>` block inside every generated SVG, making diagrams theme-aware without external CSS.

## IPC Channels

None directly. Both chunks are pure renderer-side libraries. The docx extractor receives `ArrayBuffer` data from file input / attachment handling (already in renderer memory). The Mermaid renderer receives code strings from React markdown rendering.

## Integration Points

- **Main bundle → Mammoth chunk:** Triggered by file attachment processing (`.docx` MIME type `application/vnd.openxmlformats-officedocument.wordprocessingml.document`). Cross-references `fRi` function in `gui-renderer-core` / `gui-renderer-diff-settings` reports.
- **Main bundle → Mermaid chunk:** Triggered by `react-markdown` code block with `language-mermaid`. The `Pmo` component is referenced in markdown rendering pipeline.
- **Mermaid chunk → Main bundle:** Imports `E as Ne` from main bundle (ELK Web Worker class or shared layout utility). This is a **reverse dependency**: the chunk needs a class defined in the main bundle.
- **Cross-mission:** No direct IPC or daemon protocol usage. Both are isolated renderer features.

## Implementation Notes

### Mammoth / Docx Extraction
- **Option A:** Keep Vite's automatic code splitting. Import `mammoth` normally in source; Vite will produce a similar lazy chunk.
- **Option B:** If bundle size is critical, replace Mammoth + Underscore + Bluebird with a smaller modern alternative (e.g. `@xmldom/xmldom` + native `fetch`/`Promise`). The actual surface area used is only `extractRawText({ arrayBuffer })`.
- **Key API to preserve:** `async (file: File | ArrayBuffer) => string`: extract plain text from docx.

### Mermaid Renderer
- **Option A:** Keep the custom implementation. It is lightweight (92 KB) and supports exactly the diagram types OpenDroid uses. Port the parser + SVG generator files as-is.
- **Option B:** Replace with official `mermaid` npm package. Trade-off: larger bundle (~500 KB+) but full spec compliance and community maintenance.
- **ELK dependency:** The chunk relies on an inline Web Worker for ELK layout. In OpenDroid, ensure `ELK` is either:
  - Bundled as a separate worker (Vite handles this with `?worker` imports)
  - Or replaced with a simpler DAG layout algorithm if graph complexity is low
- **Key APIs to preserve:**
  - `parseMermaid(code: string) => AST`
  - `renderMermaidSVG(code: string, theme: ThemeConfig) => string` (SVG HTML)
- **Theme integration:** Ensure `data-theme` attribute on `<html>` is read/watched so diagrams update on dark/light toggle.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `renderer-main.js` | ~3175–3190 | `fRi` | Docx text extraction entry; calls `import("./renderer-chunk-a.js")` |
| `renderer-main.js` | ~3155–3175 | `Pmo` | React component for Mermaid rendering; calls `import("./renderer-chunk-b.js")` |
| `renderer-main.js` | ~3150–3160 | `WUe` | Vite preload utility wrapper (shared by both chunks) |
| `renderer-chunk-a.js` | 1–50 | Underscore.js entry | `ye.VERSION = "1.13.8"` |
| `renderer-chunk-a.js` | ~100–210 | Mammoth.js parsers | Docx XML tokenizers, relationship readers, paragraph converters |
| `renderer-chunk-a.js` | 211–220 | `gg` / `i` export | ESM namespace export of `extractRawText` |
| `renderer-chunk-b.js` | 1 | `import { E as Ne }` | Reverse import from main bundle (ELK worker class) |
| `renderer-chunk-b.js` | ~10–90 | Parser functions | `an` (flowchart), `mn` (sequence), `bn` (class), `cn` (state), `vn` (ER), `An` (XY chart) |
| `renderer-chunk-b.js` | ~40–70 | `In()`, `Ft()` | ELK Web Worker init + synchronous layout dispatch |
| `renderer-chunk-b.js` | 112 | Exports | `DEFAULTS`, `parseMermaid`, `renderMermaidSVG` |
