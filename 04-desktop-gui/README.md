# Desktop GUI Architecture Notes

This section maps the desktop surface around autonomous coding agents: Electron-style main/preload/renderer boundaries, IPC topology, daemon lifecycle management, session navigation, onboarding, settings, and lazy-loaded document tooling.

## Start Here

1. `gui-html-entry.md`: renderer entry point, CSP shape, theme bootstrap, and asset loading.
2. `gui-preload-bridge.md`: preload API surface and IPC bridge design.
3. `gui-main-process-core.md`: BrowserWindow lifecycle, app shell, and daemon manager.
4. `gui-main-process-ipc.md`: main-process IPC catalog.
5. `gui-integration.md`: cross-layer synthesis.

## Reports

| Area | Report |
|------|--------|
| HTML entry | `gui-html-entry.md` |
| Preload bridge | `gui-preload-bridge.md` |
| Main process core | `gui-main-process-core.md` |
| Main process IPC | `gui-main-process-ipc.md` |
| Main process daemon | `gui-main-process-daemon.md` |
| Main process helpers | `gui-main-process-helpers.md` |
| Renderer core | `gui-renderer-core.md` |
| Renderer IPC client | `gui-renderer-ipc-client.md` |
| Renderer onboarding | `gui-renderer-onboarding.md` |
| Renderer mission panel | `gui-renderer-mission-panel.md` |
| Renderer diff settings | `gui-renderer-diff-settings.md` |
| Secondary chunks | `gui-renderer-secondary-chunks.md` |
| Theme and CSS | `gui-theme-css.md` |
| PDF/XLSX notes | `gui-pdfjs-xlsx-note.md` |
| Integration synthesis | `gui-integration.md` |

## Key Themes

- Renderer and daemon communicate directly over WebSocket while the main process manages lifecycle.
- Preload exposes a narrow `electronAPI` bridge for auth, navigation, sessions, notifications, window state, and diagnostics.
- The renderer combines routing, session surfaces, mission panels, diff tooling, onboarding, and lazy document viewers.
- Desktop-specific concerns are isolated around IPC, daemon startup, deep links, native dialogs, notifications, and app lifecycle.
