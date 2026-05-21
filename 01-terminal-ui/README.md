# Terminal UI Architecture Notes

This section maps the terminal interface layer used by long-running autonomous coding agents: React Ink-style rendering, keyboard flow, panels, themes, status surfaces, and terminal animation patterns.

## Start Here

1. `tui-renderer-main-loop.md`: main render loop and app shell.
2. `tui-ink-core.md`: Ink primitives and runtime behavior.
3. `tui-composer-input.md`: prompt/composer interaction model.
4. `tui-panel-session-selector.md`: session navigation and selection surface.
5. `tui-theme-system.md`: terminal theme and token model.

## Reports

| Area | Report |
|------|--------|
| Terminal primitives | `tui-ink-core.md` |
| Main loop | `tui-renderer-main-loop.md` |
| Composer | `tui-composer-input.md` |
| Session panel | `tui-panel-session-selector.md` |
| Permission dialog | `tui-panel-permission-dialog.md` |
| Model selector | `tui-panel-model-selector.md` |
| MCP manager | `tui-panel-mcp-manager.md` |
| Background process panel | `tui-panel-background-process.md` |
| Status bar | `tui-status-bar.md` |
| Theme selector | `tui-panel-theme-selector.md` |
| Theme system | `tui-theme-system.md` |
| React runtime | `tui-react-runtime.md` |
| Yoga layout | `tui-yoga-layout.md` |
| ANSI utilities | `tui-ansi-terminal-utils.md` |
| Logo animation | `tui-logo-animation.md` |
