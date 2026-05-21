# HTML Entry Point (index.html): GUI Architecture Notes

## Overview

`index.html` is the Electron renderer entry point for the OpenDroid desktop application, built with Vite. It is a single-page application shell that loads one JavaScript module bundle (`renderer-main.js`, large renderer bundle) and one CSS stylesheet (`theme.css`, 29KB). The file includes a comprehensive Content Security Policy (CSP) header, an inline theme-detection script that runs synchronously before React mounts, and an SVG-based loading animation inside the React root element. The loading strategy uses native ES modules (`type="module"`) which are deferred by default: no explicit `defer` or `async` attributes. Secondary chunks (`renderer-chunk-a.js`, `renderer-chunk-b.js`, `pdf-viewer.chunk.js`, `spreadsheet.chunk.js`) are NOT referenced in the HTML: they are loaded via dynamic imports from the main bundle at runtime.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer/main_window/index.html` | ~10.5 KB | Electron renderer entry point, SPA shell |
| `renderer/main_window/assets/renderer-main.js` | ~large renderer bundle (~8.1M bytes) | Main renderer bundle (React app + all core code) |
| `renderer/main_window/assets/theme.css` | 29 KB (29,053 bytes) | Full CSS theme (Tailwind + custom properties) |
| `renderer/main_window/assets/renderer-chunk-a.js` | 471 KB (482,174 bytes) | Secondary chunk: lazy-loaded (NOT in HTML) |
| `renderer/main_window/assets/renderer-chunk-b.js` | 92 KB (94,535 bytes) | Secondary chunk: lazy-loaded (NOT in HTML) |
| `renderer/main_window/assets/pdf-viewer.chunk.js` | 1.3 MB | PDF.js viewer: lazy-loaded (NOT in HTML) |
| `renderer/main_window/assets/spreadsheet.chunk.js` | 419 KB (429,446 bytes) | XLSX library: lazy-loaded (NOT in HTML) |

## Architecture

### Document Structure

```
<!doctype html>
<html lang="en">
  <head>
    ├── meta charset, viewport, title, description
    ├── CSP meta tag (comprehensive: see below)
    ├── Inline <script>: theme detection (runs BEFORE React)
    ├── <script type="module" crossorigin src="./assets/renderer-main.js">
    └── <link rel="stylesheet" crossorigin href="./assets/theme.css">
  <body>
    └── <div id="root">
        ├── inline <style>: loader CSS (dark/light aware)
        └── .initial-loader: SVG animation (6-frame OpenDroid logo)
```

### Loading Sequence

1. **HTML parsed**: document structure loaded
2. **Inline theme script executes** (synchronous, blocking): reads `localStorage('opendroid-theme-preference')`, sets `data-theme="dark"` on `<html>` if dark mode
3. **CSS stylesheet loaded** (`theme.css`): render-blocking, applies theme
4. **Main bundle deferred** (`renderer-main.js`, `type="module"`): ES modules are deferred by default, executes after HTML parsing completes
5. **React mounts** into `<div id="root">`: replaces the initial loader SVG with the actual app
6. **Dynamic imports**: secondary chunks loaded on demand as routes/features are accessed

### Asset Loading Strategy

- **`<script type="module" crossorigin>`**: The `type="module"` attribute makes the script behave like `defer`: it downloads in parallel during HTML parsing and executes after the DOM is ready. The `crossorigin` attribute enables proper CORS handling for the script.
- **`<link rel="stylesheet" crossorigin>`**: Standard render-blocking stylesheet load. `crossorigin` for CORS.
- **No `preload`, `prefetch`, or `modulepreload` links**: All secondary chunks are loaded purely via dynamic `import()` calls at runtime: no HTML-level preload hints.
- **Vite convention**: Hash-based filenames (e.g., `renderer-main.js`) for cache busting. The `crossorigin` attribute on both script and link tags is a Vite default for production builds.

## Key Findings

### 1. React Mount Point: `<div id="root">`

The React application mounts into `<div id="root">` using `ReactDOM.createRoot(document.getElementById('root'))`. This is the standard Vite + React pattern.

### 2. Inline Initial Loader

The root div contains an inline SVG animation: a 6-frame OpenDroid logo that cycles through different geometric patterns at 1-second intervals. The loader is theme-aware:

- **Light mode**: background `#f2f0f0`, text color `#000000`
- **Dark mode** (`[data-theme='dark']`): background `#161413`, text color `#f2f0f0`

This loader is visible during the brief gap between HTML rendering and React hydration.

### 3. Theme Detection Script (Inline, Synchronous)

A critical inline `<script type="text/javascript">` runs **before** the CSS and React bundle load:

```javascript
(function () {
  try {
    var STORAGE_KEY = 'opendroid-theme-preference';
    var ATTRIBUTE = 'data-theme';
    var stored = localStorage.getItem(STORAGE_KEY);
    var theme = stored === 'light' || stored === 'dark'
      ? stored
      : window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    if (theme === 'dark') {
      document.documentElement.setAttribute(ATTRIBUTE, 'dark');
    }
  } catch (e) {
    console.warn(e);
  }
})();
```

**Key details:**
- Storage key: `opendroid-theme-preference`
- Attribute: `data-theme` on `<html>` element
- Fallback: `prefers-color-scheme` media query
- Only sets attribute for dark mode (light is default: no attribute needed)
- Wrapped in try/catch for safety (localStorage may be unavailable)

### 4. Content Security Policy (CSP)

The CSP is set via `<meta http-equiv="Content-Security-Policy">` and is **extremely comprehensive**. The comment states: *"Content Security Policy is managed programmatically in `src/main/window.ts` using the shared configuration from `@opendroid/common/security` via HTTP headers."* This suggests the meta tag is a fallback: the main process sets CSP via HTTP headers at runtime.

**CSP Directive Summary:**

| Directive | Key Sources |
|-----------|-------------|
| `default-src` | `'self'` |
| `script-src` | `'self' 'unsafe-inline' 'unsafe-eval' blob:` + AnalyticsSvc, CloudDB, Google Auth/Analytics, LinkedIn CDN, `localhost:8097` |
| `style-src` | `'self' 'unsafe-inline'` |
| `img-src` | `'self' blob: data:` + Google, IdProvider CDN, OpenDroid wiki S3 buckets |
| `font-src` | `'self'` |
| `connect-src` | `'self'` + extensive list (see WebSocket/API analysis below) |
| `frame-src` | `'self'` + `localhost:31415`, `localhost:41832`, Google/CloudDB |
| `object-src` | `'none'` |
| `base-uri` | `'self'` |
| `form-action` | `'self'` |

**WebSocket endpoints in `connect-src`:**
- `ws://localhost:31415`: likely the local opendroid.exe daemon (OpenDroid agent)
- `ws://localhost:41832`: possibly dev server or secondary daemon
- `ws://localhost:8080` / `http://localhost:8080`: another local service
- `ws://localhost:5173` / `http://localhost:5173`: Vite dev server
- `ws://localhost:8097` / `http://localhost:8097`: React DevTools standalone
- `wss://relay.opendroid.dev` / `wss://relay-dev.opendroid.dev`: OpenDroid cloud relay (WebSocket)
- `wss://*.sandboxsvc.app`: SandboxSvc sandbox WebSocket connections

**API endpoints in `connect-src`:**
- `https://dev.api.opendroid.dev`, `staging`, `preprod`, `https://api.opendroid.dev`: OpenDroid API (multi-environment)
- `https://dev.telemetry.opendroid.dev`, `https://telemetry.opendroid.dev`: Telemetry
- `https://auth.idprovider.com`: IdProvider authentication
- `https://api.sandboxsvc.dev/sandboxes`: SandboxSvc sandbox API
- `https://*.ingest.us.errtracker.io`, `https://*.ingest.errtracker.io`: ErrTracker error reporting
- `https://cdn.analyticssvc.com`, `https://api.analyticssvc.io`: AnalyticsSvc analytics
- CloudDB: `identitytoolkit.googleapis.com`, `securetoken.googleapis.com`, `firestore.googleapis.com`, `*.clouddbio.com`, `*.clouddbapp.com`
- Google Analytics: `googletagmanager.com`, `google-analytics.com`
- LinkedIn: `snap.licdn.com`, `px.ads.linkedin.com`
- `http://localhost:3000`, `http://localhost:3002`, `http://localhost:3301`: local dev services
- `http://localhost:4318`: likely OTLP telemetry endpoint (OpenTelemetry)

### 5. Single Script Tag = Monolithic Bundle Strategy

Only **one** `<script>` tag references `renderer-main.js`: the large renderer bundle monolithic renderer bundle. All other JS chunks are loaded via dynamic `import()` from within this bundle. This is standard Vite production behavior: the entry chunk is linked in HTML, and code-split chunks are loaded on demand.

### 6. No `<link rel="preload">` or `<link rel="modulepreload">`

Vite does not emit preload hints by default. All non-entry assets are loaded at runtime via the module graph. This means:
- Fonts (Geist, Mono woff2 files) are loaded via CSS `@font-face` declarations in the stylesheet
- Secondary JS chunks are loaded via dynamic `import()` from the main bundle
- Media files (step1-4.mp4) are loaded on demand by the onboarding component

## Code Examples

### Theme Detection Script (inline, head)

```javascript
// index.html, <head>, inline <script>
(function () {
  try {
    var STORAGE_KEY = 'opendroid-theme-preference';
    var ATTRIBUTE = 'data-theme';
    var stored = localStorage.getItem(STORAGE_KEY);
    var theme = stored === 'light' || stored === 'dark'
      ? stored
      : window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    if (theme === 'dark') {
      document.documentElement.setAttribute(ATTRIBUTE, 'dark');
    }
  } catch (e) { console.warn(e); }
})();
```

### React Mount Point with Initial Loader

```html
<!-- index.html, <body> -->
<div id="root">
  <style>
    html, body { overflow: hidden; margin: 0; }
    .initial-loader {
      display: flex; align-items: center; justify-content: center;
      height: 100vh; width: 100%;
      background-color: #f2f0f0; color: #000000;
    }
    [data-theme='dark'] .initial-loader {
      background-color: #161413; color: #f2f0f0;
    }
  </style>
  <div class="initial-loader">
    <!-- 6-frame animated SVG OpenDroid logo -->
    <svg ...>...</svg>
  </div>
</div>
```

### Script and Stylesheet Tags

```html
<!-- Main renderer bundle: ES module, deferred by default -->
<script type="module" crossorigin src="./assets/renderer-main.js"></script>
<!-- Full CSS theme: render-blocking -->
<link rel="stylesheet" crossorigin href="./assets/theme.css">
```

## Theme / Style Tokens

### Initial Loader Colors

| Token | Light | Dark | Context |
|-------|-------|------|---------|
| Loader background | `#f2f0f0` | `#161413` | `.initial-loader` background |
| Loader foreground | `#000000` | `#f2f0f0` | `.initial-loader` SVG stroke color |
| Body overflow | `hidden` | hidden | Prevents scrollbars during load |

### Theme Mechanism
- **Storage key**: `opendroid-theme-preference` (localStorage)
- **HTML attribute**: `data-theme="dark"` on `<html>` element
- **Detection**: inline script reads localStorage → falls back to `prefers-color-scheme` media query
- **Default**: light (no attribute set): dark mode is opt-in via attribute

## IPC Channels

Not applicable for this feature. The HTML entry point does not contain IPC channel definitions. See `gui-preload-bridge.md` for the complete IPC surface.

## Integration Points

### Cross-System Dependencies

1. **Main process → CSP**: The comment references `src/main/window.ts` and `@opendroid/common/security` as the programmatic CSP source. The meta tag is likely a build-time fallback.
2. **Daemon WebSocket**: CSP `connect-src` reveals `ws://localhost:31415` as the expected opendroid.exe daemon port. Cross-reference with `gui-main-process-daemon.md`.
3. **Cloud relay**: `wss://relay.opendroid.dev` and `wss://relay-dev.opendroid.dev`: WebSocket relay to OpenDroid cloud services. Cross-reference with `gui-main-process-daemon.md`.
4. **SandboxSvc sandboxes**: `https://api.sandboxsvc.dev/sandboxes` and `wss://*.sandboxsvc.app`: sandbox execution environment. Cross-reference with 05-infrastructure (Infrastructure).
5. **IdProvider authentication**: `https://auth.idprovider.com`: auth provider. Cross-reference with 05-infrastructure.
6. **CloudDB**: Full CloudDB integration (Auth + Firestore + hosting). Cross-reference with 05-infrastructure.
7. **AnalyticsSvc analytics**: `cdn.analyticssvc.com` and `api.analyticssvc.io`: analytics tracking. Cross-reference with 05-infrastructure.
8. **ErrTracker**: Error reporting via `*.ingest.errtracker.io`. Cross-reference with 05-infrastructure.
9. **OpenTelemetry**: `http://localhost:4318` in CSP: OTLP telemetry collector endpoint. Cross-reference with 05-infrastructure.
10. **React DevTools**: CSP allows `localhost:8097` for React DevTools standalone connection.

### Local Service Ports (from CSP)

| Port | Protocol | Likely Purpose |
|------|----------|----------------|
| 31415 | WebSocket | opendroid.exe daemon main channel |
| 41832 | WebSocket/HTTP | Secondary daemon or dev service |
| 8080 | WebSocket/HTTP | Local service (TUI? relay?) |
| 8097 | WebSocket/HTTP | React DevTools standalone |
| 5173 | WebSocket/HTTP | Vite dev server (dev only) |
| 3000 | HTTP | Local dev API |
| 3002 | HTTP | Local dev service |
| 3301 | HTTP | Local dev service |
| 4318 | HTTP | OpenTelemetry OTLP collector |

## Implementation Notes

### Entry Point Replication

1. **HTML structure**: The OpenDroid port should use the same `<div id="root">` mount point pattern. This is standard React + Vite.
2. **Theme system**: The `data-theme` attribute + `localStorage('opendroid-theme-preference')` pattern is clean and portable. Replicate this directly: it provides FOUC-free theme detection.
3. **Initial loader**: The inline SVG loader pattern (theme-aware) should be replicated for a professional loading experience. The 6-frame animation approach is lightweight and effective.
4. **CSP strategy**: The OpenDroid CSP is very permissive (`'unsafe-inline' 'unsafe-eval' blob:`). For OpenDroid, consider a stricter CSP that doesn't require `unsafe-eval`. This may require changes to the build configuration.
5. **Analytics/auth integration**: Strip OpenDroid-specific domains (AnalyticsSvc, CloudDB, Google, LinkedIn, IdProvider) from the CSP and replace with OpenDroid's own auth/analytics providers.
6. **Daemon port**: `ws://localhost:31415` is the OpenDroid daemon port. OpenDroid should choose its own port and update the CSP accordingly.
7. **Bundle strategy**: Vite's default code-splitting (1 entry chunk in HTML, rest via dynamic import) is the right approach. Keep this pattern.
8. **`crossorigin` attribute**: Keep on script/link tags for proper CORS handling in Electron.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|-------------|-------------------|-------|
| `index.html` | Full file | `<div id="root">` | React mount point |
| `index.html` | Head, inline script | Theme detection IIFE | Reads `opendroid-theme-preference` from localStorage |
| `index.html` | Head, `<script type="module">` | `renderer-main.js` | Main renderer bundle (large renderer bundle), deferred ES module |
| `index.html` | Head, `<link rel="stylesheet">` | `theme.css` | Full CSS theme (29KB), render-blocking |
| `index.html` | Head, `<meta http-equiv="Content-Security-Policy">` | CSP directive | Comprehensive CSP with all integration domains |
| `index.html` | Body, `.initial-loader` | SVG 6-frame animation | OpenDroid logo animation, theme-aware colors |
