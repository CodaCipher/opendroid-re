# infra-session-manager: Architecture Notes

## Overview

OpenDroid's session management system is built around a central **SessionService** class (module 1185.js, 1823 lines, class `OGH`). Sessions are persisted as **JSONL files** in `~/.opendroid/sessions/`, with each session comprising a `.jsonl` transcript file and a `.settings.json` sidecar. The system supports full lifecycle: create → append messages → save/load → compact (summarize) → fork → archive/destroy. Session compaction is a multi-module system involving a summarizer (2580.js), compaction algorithm (2569.js), compaction orchestrator (4038.js), and the compact slash command (3801.js). Cloud sync to OpenDroid's API provides cross-device session persistence.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 1185.js | 54KB | Core SessionService class (`OGH`): lifecycle, create/load/save/destroy, compaction state, settings | App |
| 0945.js | 3KB | JSONL session file parser (`$G$`, `YmE`): reads session_start + message events | App |
| 2569.js | 11KB | Compaction algorithm (`GRI`, `FRI`): token budget calculation, suffix selection, summary injection | App |
| 2580.js | 12KB | Summarizer (`naH`): LLM-based summary generation with prompt templates | App |
| 2579.js | 3KB | Summary system prompt builder (`yRI`): 10-section summary template | App |
| 2570.js | 3KB | Transcript formatter (`OKH`, `M8f`): messages to text for summarizer input | App |
| 3975.js | 4KB | Worker session transcript reader: mission worker session file access | App |
| 4038.js | 4KB | TUI compaction orchestrator (`MH0`): bridges SessionService + summarizer + UI | App |
| 3801.js | 2KB | `/compact` slash command definition (alias: handoff, compress) | App |
| 3754.js | 65KB | Dispatch/service layer: session creation via IPC/daemon, droid registry | App |
| 4065.js | 122KB | GUI session management: compact confirmation UI, session loading | App |

> **Note:** Seed modules 3774.js (xterm.js terminal emulator), 0466.js (AWS S3 SDK), 3154.js, 3244.js (GameMaker), 3228.js, 1194.js (CI checks) are **not** session-related. Actual session modules were discovered through grep-based discovery.

## Architecture

### Session Lifecycle Flow

```
create → appendMessage(s) → saveSessionSettings → [compact if needed] → fork/archive → destroy
```

1. **Create**: `createNewSession()` → generates UUID → creates `.jsonl` file with `session_start` event → writes `.settings.json` → syncs to cloud API (`/api/sessions/create`)
2. **Append**: `appendMessage()` → appends JSON line to `.jsonl` → optionally syncs to cloud via WebSocket
3. **Save**: `saveSessionSettings()` → writes `.settings.json` (model, autonomy, tokens, etc.) → optionally syncs to cloud (`/api/v0/sessions/{id}`)
4. **Load**: `loadSession()` → reads `.jsonl` via parser (`$G$`) → restores settings from `.settings.json` → executes session start hooks
5. **Compact**: `MH0()` → summarizes messages → creates new session with summary → saves compaction state
6. **Fork**: `forkSession()` → copies message history up to target message → creates new session with forked content
7. **Archive/Delete**: `archiveSession()` / `deleteSession()` → moves/removes session files → syncs to cloud

### Session File Format

**Transcript file** (`{sessionId}.jsonl`): One JSON object per line (JSONL format).

```jsonl
{"type":"session_start","id":"uuid","title":"Session Title","owner":"username","version":2,"cwd":"/path"}
{"type":"message","id":"msg-uuid","timestamp":"ISO","message":{"role":"user","content":[...]},"parentId":"prev-msg-id","tokens":{...}}
{"type":"message","id":"msg-uuid2","timestamp":"ISO","message":{"role":"assistant","content":[...]},"parentId":"msg-uuid"}
{"type":"compaction_state","id":"comp-uuid","timestamp":"ISO","summaryText":"...","summaryTokens":500,"summaryKind":"llm_summary","removedCount":42,"systemInfo":{...}}
{"type":"todo_state","id":"todo-uuid","timestamp":"ISO","todos":[...],"messageIndex":15}
```

**Settings file** (`{sessionId}.settings.json`):
```json
{
  "model": "claude-sonnet-4-20250514",
  "reasoningEffort": "high",
  "interactionMode": "auto",
  "autonomyLevel": "off",
  "autonomyMode": "normal",
  "assistantActiveTimeMs": 123456,
  "tokenUsage": {"inputTokens":0,"outputTokens":0,"cacheCreationTokens":0,"cacheReadTokens":0,"thinkingTokens":0}
}
```

**Storage path**: `~/.opendroid/sessions/{project-hash}/{sessionId}.jsonl` where `{project-hash}` is derived from the working directory via `Fk(cwd)`. This organizes sessions per project.

### Compact Algorithm (Module 2569.js)

The compaction system has two entry points:
1. **`GRI()`** (Auto-compaction): Token-budget-aware compaction that preserves a suffix of recent messages
2. **`FRI()`** (Manual `/compact`): Full summarization that creates a new session with only the summary

**Auto-compaction flow (`GRI`)**:
1. Calculate token estimates per message using `WRI()` (~4 chars per token)
2. Sum system prompt + tools tokens via `E8f()`
3. Find cutoff index: `suffixStartIndex` = first message where total tokens ≤ `postAbsolute - summaryReserve - contextTokens`
4. If previous LLM summary exists, summarize only messages since last summary anchor (delta compaction)
5. Otherwise, summarize all prefix messages
6. Result: `[summaryMessage, ...suffixMessages]`

**Key thresholds**:
- `postAbsolute`: 40,000 tokens (max after compaction)
- `summarySoftCap`: 2,000 tokens (target summary length)
- `summaryReserve`: 4,000 tokens (reserved for summary overhead)
- `daH`: 250,000 tokens (conversation limit flag)

**Manual `/compact` flow (`FRI`)**:
1. Execute `PreCompact` hooks (plugins can block compaction with exit code 2 or 3)
2. If previous LLM summary, summarize only delta messages
3. Otherwise, summarize entire conversation
4. Creates entirely new session with only the summary as context
5. Old session preserved; new session linked via `parentSessionId`

### Summary Generation (Module 2580.js)

**Model selection**: Uses current session model, falls back to default Anthropic model (`pTH`). Supports custom models via config.

**Provider routing**:
- OpenAI → `responses.create()` API
- Generic chat → `chat.completions.create()` API
- Default → Anthropic `messages.stream()` API with streaming

**Summary prompt** (module 2579.js, `yRI`): 10-section structured template:
1. Chronological Play-by-Play
2. Primary Request and Intent
3. Approach
4. Key Technical Work
5. Questions and Clarifications
6. Files and Code Sections
7. Error Resolution
8. Pending Tasks
9. Current Work
10. Next Steps

**Delta summarization**: When a previous summary exists, the prompt asks to incorporate new messages into existing summary, preserving important details from both.

### Summary Injection (Module 2569.js, `CtA`)

When a session is restored with a compaction summary, the summary is injected as a synthetic user message:

```
"A previous instance of Droid has summarized the conversation thus far as follows:
<summary>
{summaryText}
</summary>

IMPORTANT: This summary was created by a previous instance of Droid. Files referenced 
in the summary may not be available until you explicitly view them again."
```

For `provider_switch_serialization` kind, the text is wrapped in code fences instead.

## Key Findings

1. **JSONL append-only format**: Sessions use JSONL (one JSON object per line) for efficient append-only writes. Messages, compaction states, and TODO states are all event types in the same stream. This avoids rewriting the entire file for each message.

2. **Dual persistence**: Both local filesystem (`~/.opendroid/sessions/`) and cloud API sync. Cloud sync is configurable via `cloudSessionSync` flag and uses OpenDroid's REST API endpoints.

3. **Compaction creates new sessions**: Manual `/compact` doesn't modify the existing session. It creates a brand new session with the summary as the first message, linking via `parentSessionId`. The old session file is preserved.

4. **Delta compaction**: If a previous LLM summary exists, only messages since the last summary anchor are summarized (not the entire history). This makes repeated compactions efficient.

5. **Per-project session isolation**: Sessions are stored in subdirectories named by a hash of the working directory (`Fk(cwd)`), ensuring project-specific session lists.

6. **Settings sidecar pattern**: Session settings (model, autonomy, token usage) are stored separately in `.settings.json` to avoid parsing the potentially large `.jsonl` transcript for settings changes.

7. **Worker session transcripts** (module 3975.js): Mission worker sessions use a separate transcript system at `~/.opendroid/sessions/{work-dir-hash}/{workerSessionId}.jsonl` with tail-reading for efficiency (max 2MB read, 10K lines).

8. **Compaction hooks**: Plugins can intercept compaction via `PreCompact` hook events. Exit codes 2/3 block compaction entirely.

## Code Examples

### Session creation (1185.js, lines 351-420)
```js
async createSessionWithId(H) {
    let { sessionId: A, firstUserMessage: L, parentSessionId: $, ... } = H;
    let v = q ? swA(q) : OGH.generateTitle(L);
    let e = { type: "session_start", id: A, title: v, owner: g, version: 2, cwd: M, ... };
    a$.writeFileSync(x, `${JSON.stringify(e)}\n`);
    this.sessionSettings = { model: PH, reasoningEffort: r3(PH, QH), interactionMode: IH, ... };
    this.saveSessionSettings({ async: false, shouldSyncToCloud: true });
}
```

### Compaction state storage (1185.js, lines 816-848)
```js
saveCompactionSummary(H) {
    let L = { type: "compaction_state", id: A, timestamp: H.timestamp,
        summaryText: H.summaryText, summaryTokens: H.summaryTokens,
        summaryKind: H.summaryKind, removedCount: H.removedCount, systemInfo: H.systemInfo };
    a$.appendFileSync($, `${JSON.stringify(L)}\n`);
}
```

### Summary prompt for delta compaction (2580.js, lines 143-166)
```js
if (I) {
    return `Previous summary:\n\`\`\`\n${I}\n\`\`\`\n
These messages were sent since the previous summary:\n\`\`\`\n${$}\n\`\`\`\n
Please incorporate the new messages into the existing summary, preserving 
important details from both. Remember to wrap your final summary in <summary> tags.`;
}
```

## Integration Points (cross-system)

- **Config Loader (infra-config-loader)**: SessionService reads model, autonomy, and interaction settings from global config via `vA()` getter. Settings cascade: session settings → global defaults.
- **Chat Session (infra-chat-session)**: The ChatSession/conversation model is the consumer of SessionService, calling `appendMessage()` for each conversation turn and `loadSession()` on resume.
- **CLI Entry (infra-cli-entry)**: The `--resume` flag in CLI triggers `loadSession()` to restore a previous session. The `exec` subcommand creates sessions.
- **Plugin Manager (infra-plugin-manager)**: `PreCompact` hook event allows plugins to intercept/modify/block compaction. `session-end` hooks execute cleanup on exit.
- **IPC/Daemon (infra-ipc-daemon)**: Module 3754.js dispatches automation runs via daemon IPC, creating sessions through `initializeSession()` on the transport layer.
- **Telemetry (infra-telemetry-otel)**: Compaction events are logged with `eventType: "compaction"` including duration, token counts, success/failure for metrics collection.
- **GUI (04-desktop-gui)**: Module 4065.js provides the compact confirmation dialog UI with abort support, session list display, and session loading.
- **TUI (01-terminal-ui)**: Module 3801.js registers `/compact` slash command for terminal UI.

## Implementation Notes

1. **JSONL session format is portable**: The append-only JSONL format is efficient and self-contained. Port directly: no schema migration needed.
2. **SessionService is the single entry point**: All session operations flow through `OGH` class methods. Replace the class with your implementation, keeping the same public API.
3. **Decouple cloud sync**: The `cloudSessionSync` flag gates all API calls. Set to `false` for a standalone port that uses only local filesystem.
4. **Summarizer is model-agnostic**: The summarizer (2580.js) routes to OpenAI, Anthropic, or generic APIs based on provider. Replace the provider routing with your LLM client.
5. **Session path uses opendroid home**: `~/.opendroid/sessions/` is the root. Override `C$()` to change the base directory for porting.

## Module Reference

| Module | Lines | Key Functions/Classes |
|--------|-------|----------------------|
| 1185.js | 1-1823 | `OGH` class (SessionService): `createNewSession()` (L350), `createSessionWithId()` (L351), `appendMessage()` (L757), `saveCompactionSummary()` (L816), `loadLatestCompactionSummary()` (L838), `loadLatestLlmCompactionSummary()` (L879), `loadSession()` (L981), `forkSession()` (L507), `saveSessionSettings()` (L1528), `getSessionMessagesPath()` (L1504), `getSessionSettingsPath()` (L1508) |
| 0945.js | 1-80 | `$G$()` session parser (L64), `YmE()` JSONL reader (L27), `qmE()` message normalizer (L17) |
| 2569.js | 1-384 | `GRI()` auto-compaction algorithm (L136), `FRI()` manual compaction (L261), `CtA()` summary injection (L55), `WRI()` token estimator (L25), `E8f()` context token calculator (L65) |
| 2580.js | 1-337 | `naH()` summarizer factory (L125), prompt builder (L130), `C()` LLM caller (L198), `A()` summary extractor (L177) |
| 2579.js | 1-30 | `yRI()` system prompt builder with 10-section template |
| 2570.js | 1-80 | `OKH()` transcript formatter (L53), `M8f()` message formatter (L39), `os()` compaction state mapper (L57) |
| 3975.js | 1-105 | `ZzH()` worker session path (L30), `axM()` JSONL parser (L61), `NaD()` tail reader (L27) |
| 4038.js | 1-142 | `MH0()` TUI compaction orchestrator (L15) |
| 3801.js | 1-25 | `JmD` compress/compact slash command definition (L4) |
| 3754.js | 1-2197 | `tD` class (dispatch service): `dispatchAutomationRun()` (L25) |
| 4065.js | 2500-2660 | GUI compact confirmation handler with abort, session loading |
