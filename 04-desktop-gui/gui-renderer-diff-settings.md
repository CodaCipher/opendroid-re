# Diff Viewer, Settings & Debug Flags: GUI Architecture Notes

## Overview

This report documents three related GUI subsystems discovered in the OpenDroid renderer bundle:

1. **Diff Viewer**: A feature-flagged code review UI with git diff fetching, semantic diff generation (AI-powered), PR creation, and inline commenting. Gated behind the `diff_panel` feature flag.
2. **Settings & Debug Flags**: A localStorage-backed debug flag system (`opendroid:debug-flags`) with a zustand-like persist layer, a "DEBUG MODE" banner UI, and a "Settings Resolution" modal for inspecting effective configuration.
3. **Feature Flags System**: feature flag-driven feature flags controlling model access, UI features, and experimental capabilities.

All findings come from `renderer-main.js` (~large renderer bundle renderer bundle), with CSS theme tokens cross-referenced from `theme.css`.

---

## File Map

| File | Size | Role |
|------|------|------|
| `renderer-main.js` | ~large renderer bundle | Main renderer bundle: contains diff viewer components, feature flags, debug system |
| `theme.css` | 29 KB | Theme CSS: `data-theme='dark'` selectors, HLJS diff colors |

---

## Architecture

### Diff Viewer

The diff viewer is a **React component** (minified name suggests it lives in the mission/chat panel area) with the following architecture:

```
Git Diff Fetch → Semantic Diff Parser → PR Creation UI
     ↓                    ↓                    ↓
  Metrics              bNn()               QWs component
  (18 counters)    extracts AI diff      title/body/draft
```

**Key behaviors:**
- Fetches git diff via IPC/API (metrics: `GET_GIT_DIFF_*`)
- Parses AI-generated semantic diffs from message bodies using `bNn()`
- Supports PR creation with title, body, draft toggle
- Has "Add to PR description" and "Regenerate semantic diff" actions
- Shows file tree, changed files, and per-file diff content
- Inline comment threads (`Tqs` component with `comments`, `onCommentSubmit`, `onDeleteComment`)

**Semantic Diff Parsing (`bNn`, line ~10228):**
- Searches for `<!-- OPENDROID_SEMANTIC_DIFF_START -->` ... `<!-- OPENDROID_SEMANTIC_DIFF_END -->`
- Falls back to `## Semantic Diff` markdown heading
- Strips `<details>` wrapper and heading noise
- Returns `{body, semanticDiff}`: body is the human message, semanticDiff is the structured AI output

### Debug Flags System

A lightweight feature-flag-like system for local developer overrides:

```
localStorage["opendroid:debug-flags"] → A0o() parser → C0o persist wrapper → x0o() hook → G0o() UI
```

**`x0o()` hook API (line ~10539):**
- `flags: Record<string, boolean>`: all current flags
- `getFlag(key)`: read single flag
- `setFlag(key, value)`: set flag
- `toggleFlag(key)`: flip boolean
- `toggleExclusive(key, groupKeys[])`: radio-button behavior within a group
- `clearFlags()`: wipe all flags

**`G0o()` UI Component:**
- Renders a "DEBUG MODE" warning banner (gold/amber color)
- "Settings Resolution" button: opens modal showing effective values, hierarchy, available models
- "Flags" dropdown menu: toggle background colors and any unknown flags

### Feature Flags (Feature Flag Service)

All major features are gated through the feature flag service. The renderer bundle contains a large feature flag registry.

**Selected flags (from line ~3659):**

| Flag | Flag Name | Default |
|------|--------------|---------|
| `DiffPanel` | `diff_panel` | `false` |
| `EnableStructuredDiffs` | `enable_structured_diffs` | `true` |
| `EmbeddedEditor` | `embedded_editor` | `false` |
| `Automations` | `automations` | `false` |
| `Wiki` | `wiki` | `false` |
| `GitAssistant` | `git_assistant` | `false` |
| `TeamMode` | `team_mode` | `false` |
| `DroidComputers` | `droid_computers` | `false` |
| `ManagedComputers` | `managed_computers` | `false` |
| `ForceEnterpriseControls` | `force_enterprise_controls` | `false` |
| `AppDebugMode` | `app_debug_mode` | `false` |
| `AnalyticsDashboards` | `analytics_dashboards` | `false` |
| `PluginsCommand` | `plugins_command` | `false` |
| `ApplyPatchFreeform` | `apply_patch_freeform` | `false` |
| `UltraPlan` | `ultra_plan` | `false` |
| `EnableAskUserTool` | `enable_ask_user_tool` | `true` |
| `ChatInputV1Point1` | `chat_input_v1.1` | `false` |

**Model access flags (preview/experimental):**
- `ModelTitan`, `ModelGrokFast`, `ModelCodexFast`, `ModelFast`, `ModelMini`
- `ModelOpal`, `ModelAcorn`, `ModelOpus`, `ModelOnyx`, `ModelOrbit`
- `ModelAlpha` ("built-in model"), `ModelMax`, `ModelOlm`
- Azure variants: `UseAzureGPT5`, `UseAzureGPT5Codex`, `UseAzureGPT51`, `UseAzureGPT51Codex`

---

## Key Findings

### Diff Viewer Metrics (18 counters)

All metrics prefixed `diff_panel_` and tracked via telemetry:

| Metric | Type |
|--------|------|
| `get_git_diff_count` | Counter |
| `get_git_diff_success_count` | Counter |
| `get_git_diff_failure_count` | Counter |
| `get_git_diff_latency` | Latency |
| `semantic_diff_generate_count` | Counter |
| `semantic_diff_generate_success_count` | Counter |
| `semantic_diff_generate_failure_count` | Counter |
| `semantic_diff_generate_latency` | Latency |
| `semantic_diff_cache_hit_count` | Counter |
| `semantic_diff_cache_miss_count` | Counter |
| `semantic_diff_truncated_count` | Counter |
| `create_pr_count` | Counter |
| `create_pr_success_count` | Counter |
| `create_pr_failure_count` | Counter |
| `create_pr_latency` | Latency |
| `pr_status_fetch_count` | Counter |
| `pr_status_fetch_failure_count` | Counter |
| `pr_status_pr_found_count` | Counter |

### HLJS Diff Syntax Highlighting

- Highlight.js is bundled with **50+ languages** including `diff`
- CSS tokens for diff highlighting:
  - `.hljs-addition`: green in light mode (`#098658`), lighter green in dark mode (`#b5cea8`)
  - `.hljs-deletion`: red in light mode, lighter red in dark mode
- Dark mode variant wrapped in `[data-theme='dark'] & { ... }`

### Debug Flags: Background Color Toggles

The `Kyt` array defines 4 background color debug options (line ~10564):

| Key | Label | Group |
|-----|-------|-------|
| `bgPureWhite` | BG: Pure white | `bg` |
| `bgWarmWhite` | BG: Warm white | `bg` |
| `bgMutedGray` | BG: Muted gray | `bg` |
| `bgWarmGray` | BG: Warm gray | `bg` |

These use `toggleExclusive()`: only one background can be active at a time.

### Theme / Style Tokens

**Theme-aware color objects** (line ~6563):
```javascript
const Jkn = {
  darkMode: "var(--tungsten-3)",
  lightMode: "var(--tungsten-20)"
};
const NMe = {
  darkMode: "var(--tungsten-2)",
  lightMode: "var(--tungsten-20)"
};
```

**CSS theme selectors** (`theme.css` + inline styles):
- `[data-theme='dark'] & { ... }`: widespread pattern for dark overrides
- `[data-theme='dark'] &.hljs-addition { color: #b5cea8; }`
- Styled-components with `$darkModeColor` prop

---

## Code Examples

### Semantic Diff Parser

```javascript
// Line ~10228
const mNn = "<!-- OPENDROID_SEMANTIC_DIFF_START -->";
const VWs = "<!-- OPENDROID_SEMANTIC_DIFF_END -->";
const pyt = "## Semantic Diff";

function bNn(t) {
  const e = t.indexOf(mNn);
  if (e >= 0) {
    const i = t.indexOf(VWs),
          o = e + mNn.length,
          l = i >= 0 ? i : t.length,
          u = t.slice(o, l).trim()
             .replace(/^<details>[\s\S]*?<\/summary>\s*/, "")
             .replace(/\s*<\/details>\s*$/, "")
             .replace(new RegExp(`^${pyt...}`), "")
             .trim();
    return { body: t.slice(0, e).replace(/\n*---\n*$/, "").trimEnd(),
             semanticDiff: u || null };
  }
  const n = t.indexOf(pyt);
  if (n >= 0) {
    const i = t.slice(n + pyt.length).trim();
    return { body: t.slice(0, n).replace(/\n*---\n*$/, "").trimEnd(),
             semanticDiff: i || null };
  }
  return { body: t, semanticDiff: null };
}
```

### Debug Flags Hook

```javascript
// Line ~10539
const fvr = "opendroid:debug-flags";
const qme = {};

function A0o() {
  if (typeof window > "u") return qme;
  try {
    const t = localStorage.getItem(fvr);
    if (t) {
      const e = { ...qme, ...JSON.parse(t) };
      return $ge(e), e;
    }
  } catch (t) { ... }
  return qme;
}

const C0o = Y4(fvr, A0o(), {
  getItem: t => { ...localStorage.getItem(t)... },
  setItem: (t, e) => { localStorage.setItem(t, JSON.stringify(e)) },
  removeItem: t => { localStorage.removeItem(t) }
});
```

### Diff Viewer Component: Body + Semantic Diff Render

```javascript
// Line ~10461
const { body: Vt, semanticDiff: hn } = T?.body ? bNn(T.body) : { body: "", semanticDiff: null };
const pn = Vt || hn || kt.length > 0;

return (
  <pNn>
    <PZ direction="column" gap="sm">{mt()}</PZ>
    {pn ? (
      Vt ? (
        <>
          <PZ direction="column" gap="xs">
            <gNn color="heading" variant="label-2">Description</gNn>
            <Xy content={Vt} fontSize={13} />
          </PZ>
          {hn ? <PZ direction="column" gap="xs"><m$s content={hn} /></PZ> : null}
        </>
      ) : kt.map(un => ...)
    ) : null}
  </pNn>
);
```

---

## Theme / Style Tokens

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `.hljs-addition` | `#098658` | `#b5cea8` | Added lines in diff |
| `.hljs-deletion` | (red) | (lighter red) | Removed lines in diff |
| `--tungsten-3` |: | dark bg | `Jkn.darkMode` |
| `--tungsten-20` | light bg |: | `Jkn.lightMode` |
| `--text-error` | red | red | Error text (both modes) |

Theme switching uses `data-theme='dark'` attribute selectors, applied at component level via styled-components.

---

## IPC Channels

No new IPC channels were found directly in the diff viewer code (it appears to use the existing API/IPC client layer documented in `gui-renderer-ipc-client.md`). The diff viewer likely invokes git diff and PR operations through the standard `api.` or `window.electronAPI.` surfaces.

**Notable events from related systems:**
- `McpStatusChanged`: MCP server status updates
- `McpAuthRequired` / `McpAuthCompleted`: MCP OAuth flow

---

## Integration Points

| System | Integration | Note |
|--------|-------------|------|
| **feature flag service** | Feature flag evaluation | All model/feature gates go through the feature flag service |
| **Telemetry** | `QLe` telemetry client | 18 diff-viewer metrics + general usage analytics |
| **Git / GitHub** | PR creation, diff fetching | Likely via main-process IPC to daemon |
| **MCP** | `McpStatusChanged` events | MCP servers/tools integrated in same panel |
| **LLM Models** | Semantic diff generation | AI-generated diffs embedded in assistant messages |
| **Settings Resolution** | Debug flags modal | Cross-ref with 02-orchestration settings system |

---

## Implementation Notes

**Diff Viewer:**
- The diff viewer is a **premium feature** (default disabled). For OpenDroid, enable `diff_panel` flag or set default to `true`.
- The semantic diff parser (`bNn`) can be reused sanitized: it’s pure string manipulation.
- HLJS diff registration and CSS tokens are portable; just ensure `data-theme='dark'` matches your theme system.
- PR creation component (`QWs`) depends on branch/baseBranch/title/body/draft state: decouple from OpenDroid’s backend by swapping the submit handler.

**Debug Flags:**
- The entire `opendroid:debug-flags` system is self-contained and can be ported as-is.
- `A0o()` + `C0o` (zustand persist) + `x0o()` hook + `G0o()` UI forms a complete debug toolbar.
- The background color toggles (`bgPureWhite`, `bgWarmWhite`, etc.) are specific to OpenDroid’s aesthetic: replace with your own theme overrides.
- "Settings Resolution" modal is valuable for debugging config resolution: port the modal structure but adapt the data source.

**Feature Flags:**
- Replace the feature flag provider with your own feature flag provider (e.g., Unleash, hosted flag provider, or env-based flags).
- The model flags (`ModelAlpha`, `ModelFast`, etc.) are specific to OpenDroid’s LLM routing: replace with your model catalog.
- `EnableStructuredDiffs` defaults to `true`: this is the non-UI diff formatting flag, separate from the `DiffPanel` UI flag.

---

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `renderer-main.js` | Line 3168 | Metrics enum (DIFF_VIEWER_*) | 18 telemetry counters |
| `renderer-main.js` | Line 3659 | Feature flag registry | feature flag definitions |
| `renderer-main.js` | Line 4469 | HLJS language registration | 50+ languages including `diff` |
| `renderer-main.js` | Line 4568-4677 | HLJS CSS tokens | `.hljs-addition`/`.hljs-deletion` light+dark |
| `renderer-main.js` | Line 10228 | `bNn()` semantic diff parser | Extracts AI diff from message body |
| `renderer-main.js` | Line 10445 | `Jqs()` PR status badge | open/closed/merged/draft styling |
| `renderer-main.js` | Line 10461 | Diff viewer render component | Body + semanticDiff + PR creation |
| `renderer-main.js` | Line 10539 | Debug flags system | `A0o()`, `C0o`, `x0o()`, `opendroid:debug-flags` |
| `renderer-main.js` | Line 10564 | `G0o()` debug UI | DEBUG MODE banner + Settings Resolution |
| `renderer-main.js` | Line 6563 | Theme color tokens | `Jkn`, `NMe`, `e7n` dark/light maps |
| `theme.css` | Multiple | `data-theme='dark'` selectors | CSS variable overrides |
