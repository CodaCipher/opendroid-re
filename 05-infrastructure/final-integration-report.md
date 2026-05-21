# xref-final-synthesis: OpenDroid Complete Integrated Architecture

## Overview

This report presents the complete integrated architecture of OpenDroid, synthesized from findings across all five reverse-engineering missions: 01-terminal-ui (TUI: 15 reports), 02-orchestration (Mission/Orchestration: 16 reports), 03-tool-agent-system (Tool System: 14 reports), 04-desktop-gui (GUI/Electron: 16 reports), and 05-infrastructure (Infrastructure: 20 reports + cross-module audit). OpenDroid is a multi-modal AI coding agent built on a **daemon-centric, layered architecture** where a long-running background process (`opendroidd` / `opendroid.exe`) manages sessions, tools, missions, and IPC, while two rendering surfaces: a terminal UI (Ink/React) and an Electron desktop app: connect via WebSocket JSON-RPC. The system processes ~3,200 JavaScript modules spanning CLI entry, configuration, plugin management, session persistence, tool execution, LLM integration, telemetry, cloud SDKs, and WASM runtimes. This report documents the complete startup sequence, observed protocol samples flow, shutdown sequence, and cross-system integration topology.

## Research Area Synthesis

### 01-terminal-ui: TUI (Terminal User Interface)
OpenDroid's terminal UI is built on **React/Ink**, a custom React renderer targeting terminal output. The architecture follows: React reconciler → host config → ink DOM nodes → Yoga WASM flexbox layout → ANSI output renderer. Key components include: Ink core render pipeline (2228.js, 2294.js), Yoga layout engine (2218.js WASM), theme system, composer input (single-line TextInput via 3861.js), status bar, logo animation, and 7 panel components (background process, MCP manager, model selector, permission dialog, session selector, theme selector). The TUI boots from module 4070.js which invokes Ink's `render()` with a nested provider/component tree, connecting to the daemon via LocalDaemonClient (0965.js) for headless/worker mode.

### 02-orchestration: Mission/Orchestration System
The mission system implements multi-agent orchestration through a **MissionRunner** (2542.js) that drives sequential feature execution by spawning worker sessions via daemon RPC. Workers receive bootstrap prompts with feature assignments and skill instructions, run autonomously, and call `EndFeatureRun` to submit results. The dual-layer state machine tracks mission-level state (`state.json`: initializing → running → orchestrator_turn → completed/failed) and feature-level state (`features.json`: pending → in_progress → completed/failed). The system auto-injects scrutiny and user-testing validators at milestone boundaries. Crash recovery resets feature status to `pending` and returns control to the orchestrator.

### 03-tool-agent-system: Tool System
OpenDroid's tool system is built on a centralized registry pattern (`qgH` class in 1176.js) with 19 built-in tools registered at startup via module 2548.js. Tools include: Read, Create, Edit, Grep, Glob, Execute (shell), WebSearch, FetchUrl, Task (subagent), TodoWrite, AskUser, Skill, ApplyPatch, MCP tools, and more. The Execute tool uses cross-platform spawning (PowerShell on Windows, bash on Unix) with a three-tier risk assessment (low/medium/high). The system integrates dual LLM API adapters: OpenAI/Azure (Responses API + Chat Completions) and Anthropic: with streaming SSE support. MCP tools are dynamically registered/unregistered through the plugin manager.

### 04-desktop-gui: GUI (Electron Desktop App)
The GUI is an Electron application with a three-layer IPC architecture: (1) `preload.js` exposes 27 channels via `contextBridge.exposeInMainWorld`, (2) the main process (main process bundle bundle) handles daemon lifecycle management, OAuth deep links, auto-updates, and 15 `ipcMain.handle` handlers, and (3) the renderer (large renderer bundle React 19 + TanStack Router SPA) communicates **directly** with the daemon via WebSocket bypassing Electron's main process for data. The main process spawns `opendroid.exe` as a child process, health-polls via HTTP `/health`, and manages restart with exponential backoff. The renderer uses Allotment for resizable split-pane layout.

### 05-infrastructure: Infrastructure (This Mission)
20 infrastructure subsystems analyzed: CLI entry (Commander.js), session manager (JSONL persistence), chat session (ConversationStateManager), config loader (hierarchical merge), plugin manager (MCP Service), cache (LRU + TTL), IPC daemon (WebSocket JSON-RPC), logging (OTel DiagAPI + gRPC + ErrTracker/winston), telemetry (OTel SDK), auth (OAuth2/JWT/AWS STS/MCP), HTTP client (gaxios), AWS SDK (S3 + SSO OIDC), GCP client (google-auth-library + gRPC), update checker (S3 binary distribution), crypto (jsonwebtoken + jose + AES-256-GCM), FS helpers (atomic write), semver/glob (node-semver + picomatch), WASM runtime (Yoga layout + protobuf), errors/runtime (MetaError hierarchy), and misc long-tail (~3,200 modules categorized).

## Architecture

### Architectural Diagram: Component Relationships

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          OPENDROID SYSTEM ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────────────┐                              ┌──────────────────────┐    │
│  │   CLI Entry      │                              │  Electron Main Proc  │    │
│  │   (Commander.js) │                              │  (main-process.js) │    │
│  │   4071.js        │                              │  - DaemonManager     │    │
│  │   3710.js (exec) │                              │  - Auto-updater      │    │
│  └────────┬─────────┘                              │  - OAuth deep links  │    │
│           │                                        │  - Tray/Menu         │    │
│           │ if isTTY                               └──────────┬───────────┘    │
│           ▼                                                   │ spawn+health     │
│  ┌──────────────────┐                                        │                  │
│  │   TUI (M1)       │                              ┌─────────▼───────────┐    │
│  │   React/Ink      │                              │   DAEMON            │    │
│  │   4070.js boot   │     WebSocket JSON-RPC       │   (opendroidd/       │    │
│  │   Yoga WASM      │◄────────────────────────────►│    opendroid.exe)       │    │
│  │   2228, 2294     │                              │                     │    │
│  └──────────────────┘                              │  ┌───────────────┐  │    │
│                                                     │  │ DroidRegistry │  │    │
│  ┌──────────────────┐     WebSocket JSON-RPC        │  │ (sessions)    │  │    │
│  │   GUI Renderer   │◄────────────────────────────►│  ├───────────────┤  │    │
│  │   (M4)           │                              │  │ RpcRouter     │  │    │
│  │   React 19 +     │                              │  │ (3755.js)     │  │    │
│  │   TanStack Router│                              │  ├───────────────┤  │    │
│  │   large renderer bundle bundle   │                              │  │ Broadcaster   │  │    │
│  │   preload bridge │                              │  │ (3756.js)     │  │    │
│  └──────────────────┘                              │  ├───────────────┤  │    │
│                                                     │  │ TerminalMgr   │  │    │
│                                                     │  │ CodeServerMgr │  │    │
│                                                     │  └───────────────┘  │    │
│                                                     └─────────┬───────────┘    │
│                                                               │                │
│                          ┌────────────────────────────────────┼──────────┐     │
│                          │            APPLICATION LAYER       │          │     │
│                          ├─────────────┬──────────────┬───────┼──────────┤     │
│                          │ Session Mgr │ Chat Session │ Plugin│ Mission  │     │
│                          │ (1185.js)   │ (2572.js)   │ Mgr   │ Runner   │     │
│                          │ JSONL files │ ConvStateMgr │(1178) │(2542.js) │     │
│                          └──────┬──────┴──────┬───────┴───┬──┴──────┬───┘     │
│                                 │             │           │         │          │
│                          ┌──────▼─────────────▼───────────▼─────────▼───┐     │
│                          │           TOOL SYSTEM (M3)                    │     │
│                          │  Registry (1176.js) · 19 built-in tools      │     │
│                          │  Execute · Read · Edit · Grep · Glob · Task  │     │
│                          │  MCP tools (dynamic) · Permissions engine     │     │
│                          └──────────────────┬───────────────────────────┘     │
│                                             │                                  │
│                          ┌──────────────────▼───────────────────────────┐     │
│                          │           LLM ADAPTERS                       │     │
│                          │  OpenAI Responses/Chat (3134.js)             │     │
│                          │  Anthropic Messages                          │     │
│                          │  Google Vertex AI                            │     │
│                          │  xAI · Azure OpenAI                          │     │
│                          └──────────────────┬───────────────────────────┘     │
│                                             │                                  │
│  ┌──────────────────────────────────────────▼──────────────────────────────┐  │
│  │                         SERVICE LAYER                                    │  │
│  │  Config Loader (0267.js) · Auth (OAuth2/JWT/STS) · HTTP Client (gaxios) │  │
│  │  Cache (LRU+TTL) · Logging (OTel DiagAPI) · Telemetry (OTel SDK)       │  │
│  │  FS Helpers (atomic write) · Crypto (jsonwebtoken+jose)                 │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                      CLOUD / EXTERNAL LAYER                              │  │
│  │  AWS SDK (S3, STS, SSO OIDC) · GCP (Auth, Firestore, KMS, gRPC)       │  │
│  │  IdProvider (identity) · ErrTracker (errors) · OTLP Exporter (telemetry)       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                      FOUNDATION LAYER                                    │  │
│  │  Errors/Runtme (MetaError) · Semver/Glob (node-semver+picomatch)        │  │
│  │  WASM Runtime (Yoga layout + protobuf Long) · Update Checker (S3)       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Startup Sequence (End-to-End)

The OpenDroid startup sequence follows a carefully ordered pipeline from CLI invocation to full operational readiness. The sequence differs slightly between TUI and GUI modes but shares the same daemon-centric initialization.

### Phase 1: CLI Entry & Bootstrap

**Source:** infra-cli-entry (4071.js), infra-config-loader (0267.js, 0940.js)

```
1. User invokes: droid [prompt] | droid exec | droid daemon
   └── Commander.js (0209.js) parses process.argv
       ├── Console Redirection → all console.* → ~/.opendroid/logs/console.log (4071.js:16-95)
       ├── PTY Library Setup → platform-specific paths (4071.js:101-112)
       ├── Telemetry Init → OpenTelemetry if OPENDROID_OTEL_ENABLED=true (4071.js:1400-1436)
       ├── Global Error Handlers → uncaughtException/unhandledRejection (4071.js:1438-1450)
       ├── Bootstrap Metrics → cli_startup_bootstrap_latency counter (4071.js:1452)
       ├── Auth Validation → EO() checks (4071.js:1473-1478)
       ├── Certificate Loading → EFI() SSL certs (4071.js:1479)
       ├── Auto-Update Check → parallel background check (4071.js:1480-1510)
       ├── Settings Init → TvH.initialize() singleton (4071.js:1500-1510)
       │   └── Config discovery walks CWD → home for .opendroid/ dirs (0260.js)
       │       Precedence: env > CLI flags > dynamic > user > project > folder > org > builtin
       ├── Task Tool Manager → ensureTaskToolManagerInitialized() (4071.js:1517)
       ├── Plugin Manager Init → MCP service discovery from config (1178.js)
       │   └── McpSettingsManager reads mcp.json → McpHub.reloadServers()
       └── Commander Program Assembly → all subcommands registered (4071.js:1520-1640)
           └── M.parseAsync(process.argv) dispatches to handler
```

### Phase 2: Daemon Startup (if daemon mode)

**Source:** infra-ipc-daemon (3758.js, 3779.js, 3757.js, 3754.js)

```
1. DaemonOrchestrator (NWL, 3779.js) created with config
2. Core subsystems initialized:
   ├── TerminalManager (i1A): PTY terminal lifecycle
   ├── DroidRegistry (hZH): tracks active droid sessions
   ├── CodeServerManager (I3): VS Code server lifecycle
   └── DueRunPoller (c1A): scheduled automation polling
3. DaemonServer (FWL, 3758.js) created:
   └── Bun.serve() binds HTTP + WebSocket to {host, port} or Unix socket
4. ConnectionManager (bl, 3757.js) handles WS lifecycle:
   ├── Auth handshake → user identity validation
   ├── RpcRouter (kl, 3755.js) → JSON-RPC method dispatch
   │   ├── DroidRequestHandler (tD, 3754.js): 50+ RPC methods
   │   └── TerminalHandler (VWL): terminal CRUD
   └── Broadcaster (XWL, 3756.js): notification fan-out
5. HeartbeatService starts server-side ping/pong
6. DueRunPoller starts scheduled automation checks
7. Daemon ready: listening for WebSocket connections
```

### Phase 3: GUI Startup (Electron path)

**Source:** 04-desktop-gui: gui-main-process-core, gui-main-process-daemon, gui-renderer-core

```
1. Electron app.whenReady() → BEt() bootstrap:
   ├── Auto-updater init (electron-updater, static storage, 10min interval)
   ├── Single instance lock (requestSingleInstanceLock)
   ├── Deep link handler (opendroid:// protocol)
   ├── Shell environment loading (macOS)
   └── IPC handler registration (15+ handlers)
2. DaemonManager (eEt class):
   ├── child_process.spawn("opendroid.exe", [...args])
   │   └── FD3 liveness pipe passed to daemon
   ├── Health polling → HTTP GET /health every N seconds
   └── State: Stopped → Starting → Running
3. BrowserWindow created:
   ├── contextIsolation: true, nodeIntegration: false
   ├── preload.js loaded → contextBridge.exposeInMainWorld
   │   └── 27 channels: auth, dialog, session, notification, window, bugReport, ErrTracker
   └── renderer/index.html loaded
4. Renderer (React 19 SPA) boots:
   ├── hqr.createRoot(#root).render(ivo())
   ├── AuthProvider → AuthGate → OrgGate
   ├── Layout (Allotment split-pane) → sidebar + main content
   └── WebSocket connection to daemon (DIRECT, bypassing main process)
       └── ws://localhost:{port} → JSON-RPC session control
```

### Phase 4: TUI Startup (Terminal path)

**Source:** 01-terminal-ui: tui-renderer-main-loop (4070.js), tui-ink-core

```
1. If isTTY → default interactive mode (4071.js action handler)
2. Module 4070.js invoked as TUI bootstrapper:
   ├── Ink render() called with component tree
   │   └── Provider tree: stdin/stdout/focus/exit contexts (2292.js)
   ├── React reconciler (2228.js) → custom host config (2245.js)
   │   └── Ink DOM nodes (2239.js) backed by Yoga WASM nodes (2218.js)
   ├── Logo animation displayed during loading
   ├── Main component renders:
   │   ├── Session list / session selector panel
   │   ├── Composer input (TextInput, 3861.js)
   │   ├── Message display area
   │   ├── Status bar
   │   └── Permission dialog (when needed)
   └── WebSocket to daemon (or direct in-process mode)
```

### Phase 5: Session Initialization

**Source:** infra-session-manager (1185.js), infra-chat-session (2572.js)

```
1. Session creation (new or resume):
   ├── If --resume → loadSession() reads .jsonl + .settings.json
   │   └── Parser ($G$, 0945.js) reads session_start + message events
   └── If new → createNewSession() → UUID → .jsonl + .settings.json
2. ChatSession/ConversationStateManager (2572.js, class `as`) created:
   ├── conversationHistory[] initialized
   ├── Streaming wrappers ready (2362.js OpenAI, 2398.js ChatCompletion)
   └── Tool tracking Map initialized
3. Plugin Manager finalizes MCP tool registration:
   └── registerAllMcpTools() → k0() global tool registry
4. Model resolved via config chain:
   └── F$() resolves model ID → provider/config (0934.js)
5. System prompt assembled with:
   ├── Tool definitions (JSON Schema from Zod)
   ├── Config-derived settings (autonomy, permissions)
   └── Session context (working directory, mission state if applicable)
```

## Observed Protocol Samples Flow

### Primary Conversation Loop

**Source:** infra-chat-session, 03-tool-agent-system: tool-core-registry, tool-llm-openai

```
┌──────────────────────────────────────────────────────────────────────┐
│                    RUNTIME DATA FLOW                                  │
│                                                                      │
│  User Input (TUI TextInput / GUI composer)                           │
│       │                                                              │
│       ▼                                                              │
│  [1] ConversationStateManager.appendMessage(role: "user")            │
│       │  2572.js → conversationHistory[] + .jsonl append             │
│       ▼                                                              │
│  [2] LLM Adapter invocation                                         │
│       │  3134.js routes by provider: openai | anthropic | google     │
│       │  ├── OpenAI: responses.create or chat.completions.create     │
│       │  ├── Anthropic: /v1/messages                                 │
│       │  └── Google: Vertex AI endpoint                              │
│       │  Message format conversion:                                  │
│       │  └── LWD() (Anthropic→OpenAI) or direct Anthropic format     │
│       ▼                                                              │
│  [3] Streaming SSE response                                          │
│       │  2362.js (P2H) consumes SSE events                           │
│       │  Content blocks: text | thinking | tool_use                  │
│       │  Each chunk → ConversationStateManager update                │
│       │  UI callback: uiMessageCallback → scheduleReactUpdate()      │
│       ▼                                                              │
│  [4] Tool execution (if tool_use in response)                        │
│       │  Tool registry lookup: k0().get(toolId)                      │
│       │  Permission check → risk level → user approval if needed     │
│       │  Tool handler executes:                                       │
│       │  ├── Read: fs.readFile with chunking                         │
│       │  ├── Edit: find-and-replace in file                          │
│       │  ├── Execute: child_process.spawn (bash/PowerShell)          │
│       │  ├── Grep/Glob: ripgrep subprocess                           │
│       │  ├── Task: spawn subagent via daemon                         │
│       │  └── MCP: callTool() through plugin manager                  │
│       │  Tool result → ConversationStateManager.appendMessage(        │
│       │      role: "tool", content: result)                          │
│       ▼                                                              │
│  [5] Return to [2] with tool results appended                       │
│       │  LLM sees full conversation + tool results                   │
│       │  May produce more tool calls or final text response           │
│       ▼                                                              │
│  [6] Final response rendered                                        │
│       TUI: Ink renderer → Yoga layout → ANSI output                  │
│       GUI: React state update → DOM re-render                        │
│       Session saved: .jsonl append + cloud sync                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Mission/Orchestration Data Flow

**Source:** 02-orchestration: mission-orchestrator-core, mission-worker-lifecycle

```
┌──────────────────────────────────────────────────────────────────────┐
│              Msample itemION ORCHESTRATION FLOW                               │
│                                                                      │
│  Orchestrator (AGI mode session)                                     │
│       │ calls start_mission_run tool (2544.js)                       │
│       ▼                                                              │
│  MissionRunner (2542.js, class toA)                                  │
│       │ reads features.json → finds first pending feature            │
│       ▼                                                              │
│  spawnWorker() → daemon RPC: spawnWorkerSession                      │
│       │ decompSessionType: "worker"                                  │
│       │ Bootstrap prompt with feature assignment + skill name        │
│       ▼                                                              │
│  Worker Session (autonomous daemon session)                          │
│       │ Executes feature implementation                              │
│       │ Has full tool access (Read, Edit, Execute, Grep, etc.)       │
│       │ Calls TodoWrite for progress tracking                        │
│       │ Writes analysis report to missionDir                         │
│       ▼                                                              │
│  EndFeatureRun() → handoff written to progress_log.jsonl             │
│       │ handoff: { salientSummary, whatWasImplemented,               │
│       │          whatWasLeftUndone, verification, tests, issues }    │
│       ▼                                                              │
│  MissionRunner polls progress_log.jsonl (1s interval)                │
│       │ Detects completion → reads handoff → marks feature done      │
│       │ Next pending feature → spawn next worker                     │
│       │ At milestone boundary → inject scrutiny/user-testing         │
│       ▼                                                              │
│  All features done → mission state → "completed"                     │
│       │ Validation contract checked against results                   │
│       └── TUI notification via project-notification event            │
└──────────────────────────────────────────────────────────────────────┘
```

### IPC Daemon Communication Flow

**Source:** infra-ipc-daemon, 04-desktop-gui: gui-integration

```
┌──────────────────────────────────────────────────────────────────────┐
│              DAEMON IPC COMMUNICATION                                │
│                                                                      │
│  TUI Client (0965.js)          GUI Renderer (direct WS)              │
│  LocalDaemonClient             window.electronAPI + WebSocket        │
│       │                              │                               │
│       └──────────┬───────────────────┘                               │
│                  │ WebSocket JSON-RPC 2.0                             │
│                  ▼                                                    │
│         ConnectionManager (3757.js)                                   │
│                  │                                                    │
│         RpcRouter (3755.js)                                           │
│           ├── DroidRequestHandler (3754.js)                           │
│           │   ├── daemon.initialize_session                          │
│           │   ├── daemon.send_message                                │
│           │   ├── daemon.request_permission                          │
│           │   ├── daemon.spawn_worker_session                        │
│           │   ├── daemon.kill_worker_session                         │
│           │   ├── daemon.interrupt_session                           │
│           │   └── 50+ more methods                                   │
│           └── TerminalHandler                                         │
│               ├── terminal.create                                     │
│               ├── terminal.resize                                     │
│               └── terminal.kill                                       │
│                  │                                                    │
│         Broadcaster (3756.js)                                         │
│           ├── session.notification (messages, status)                 │
│           ├── permission.request (user approval needed)              │
│           ├── terminal.data (stdout output)                           │
│           └── mission.progress (feature status changes)              │
└──────────────────────────────────────────────────────────────────────┘
```

## Shutdown Sequence

### Graceful Shutdown

```
1. Shutdown signal received:
   ├── SIGTERM / SIGINT → process signal handler
   ├── GUI: app 'will-quit' event / BrowserWindow close
   └── CLI: user Ctrl+C or session end

2. Active sessions drained:
   ├── Running tool executions cancelled (ToolAbortError, 0008.js)
   │   └── child_process.kill() for Execute tools
   │   └── AbortController signal for HTTP requests
   ├── Streaming LLM responses interrupted
   │   └── SSE stream.abort()
   └── Worker sessions (if mission running):
       ├── MissionRunner.pauseMissionRunner() (2543.js Hx)
       └── Worker state saved → workerSessionId persisted in features.json

3. Session persistence:
   ├── ConversationStateManager finalizes conversationHistory
   ├── SessionService.saveSessionSettings() → .settings.json
   ├── JSONL transcript flushed to disk
   ├── Cloud sync: pending sessions pushed to API (/api/v0/sessions/{id})
   └── Compaction state saved if mid-compact

4. Plugin/Service cleanup:
   ├── Plugin Manager: unregisterAllMcpTools() (1178.js)
   │   └── Each MCP server sent shutdown notification
   ├── MCP server processes killed (stdio-type)
   └── MCP HTTP connections closed

5. Daemon shutdown (if daemon mode):
   ├── Broadcaster notifies all clients: daemon shutting down
   ├── DroidRegistry clears active sessions
   ├── TerminalManager kills all PTY processes
   ├── CodeServerManager stops VS Code server instances
   ├── HeartbeatService stops
   ├── DueRunPoller stops
   ├── Bun.serve() closes WebSocket connections
   └── Process exits

6. GUI-specific cleanup:
   ├── DaemonManager.stop() → SIGTERM to opendroid.exe child
   │   └── If not responsive → SIGKILL after timeout
   ├── Electron IPC cleanup
   └── Auto-updater stop

7. Telemetry flush:
   ├── OTel: BasicTracerProvider.forceFlush() (1520.js)
   │   └── BatchSpanProcessor exports pending spans
   ├── OTLP metric exporter flush
   ├── ErrTracker: flush pending error events
   └── Logging: final structured log entries

8. Process exit:
   ├── Temp file cleanup (m3H temp manager)
   ├── File descriptors closed
   └── process.exit(0)
```

### Crash Recovery

```
1. Uncaught exception / unhandled rejection:
   ├── Global handler installed at startup (4071.js:1438-1450)
   ├── Error logged via uH() (0093.js) + ErrTracker capture
   └── Session state preserved in .jsonl files (crash-safe append)

2. Worker crash during mission:
   ├── MissionRunner detects via:
   │   ├── daemon session_inactivity notification
   │   ├── ProcessExitError from daemon
   │   └── Health check failure (5s polling)
   ├── Feature status reset to "pending" in features.json
   ├── Control returned to orchestrator session
   └── Orchestrator can retry by calling start_mission_run again

3. Daemon crash:
   ├── GUI DaemonManager detects via health poll failure
   ├── Exponential backoff restart (2s, 4s, 8s), max 3 attempts
   ├── Sessions recoverable from .jsonl files on disk
   └── GUI renderer reconnects WebSocket automatically
```

## Cross-System Integration Matrix

### Data Flow Connections Between All 5 Research Areas

| Source → Target | Connection | Key Modules | Description |
|----------------|-----------|-------------|-------------|
| M5-CLI → M1-TUI | Data Flow | 4071.js → 4070.js | CLI entry detects isTTY → launches Ink TUI renderer |
| M5-CLI → M2-Mission | API | 3710.js → 2542.js | --depth/--calling-session-id flags for subagent spawning |
| M5-CLI → M3-Tool | API | 3710.js → 1176.js | --enabled-tools/--disabled-tools filter tool registry |
| M5-CLI → M4-GUI | Data Flow | 3783.js → DaemonManager | `droid daemon` spawns daemon for GUI consumption |
| M5-Daemon → M1-TUI | WebSocket | 0965.js ↔ 3757.js | LocalDaemonClient for headless/worker mode |
| M5-Daemon → M2-Mission | RPC | 3754.js → 2542.js | spawnWorkerSession, kill_worker_session |
| M5-Daemon → M3-Tool | RPC | 3754.js → Tool Registry | Permission prompts routed through daemon IPC |
| M5-Daemon → M4-GUI | WebSocket | 3757.js ↔ Renderer | Primary data channel, bypasses Electron main process |
| M5-Session → M1-TUI | Data Flow | 3801.js, 4038.js | /compact slash command + TUI compaction orchestrator |
| M5-Session → M2-Mission | Data Flow | 3754.js → 1185.js | Automation run dispatch creates sessions via daemon IPC |
| M5-Session → M4-GUI | Events | 4065.js | Compact confirmation dialog, session list display |
| M5-Config → M2-Mission | Shared State | 2201.js | MissionFileService manages per-mission model-settings.json |
| M5-Config → M3-Tool | Shared State | 0939.js | Permission policies, command allowlists/denylists from config |
| M5-Config → M4-GUI | Shared State |: | Config-driven CSP whitelist, theme settings |
| M5-Plugin → M2-Mission | API | 1178.js → McpService | getAllTools() for dynamic tool discovery |
| M5-Plugin → M3-Tool | Bidirectional | 1178.js ↔ k0() | MCP tools registered in global tool registry |
| M5-Plugin → M4-GUI | API | 1178.js → Renderer | enableServer/disableServer for GUI MCP management |
| M5-Chat → M1-TUI | Data Flow | 2572.js → uiMessageCallback | scheduleReactUpdate() batches TUI rendering |
| M5-Chat → M3-Tool | Data Flow | 2572.js → toolExecutions Map | Tool calls tracked with lifecycle in conversation |
| M5-Chat → M4-GUI | Data Flow | 2572.js → 4065.js | ConversationStateManager consumed by GUI renderer |
| M5-Auth → M3-Tool | API/State | 1107.js, 1135.js | MCP client authenticates via OAuth2 before tool calls |
| M5-HTTP → M3-Tool | API | 0996.js | MCP OAuth flow for authenticating MCP tool servers |
| M5-GCP → M3-Tool | Data Flow | 1208.js | store_agent_readiness_report tool uses Firestore |
| M5-GCP → M4-GUI | Shared State | 0894.js | CSP whitelist for Google/CloudDB endpoints |
| M5-WASM → M1-TUI | Data Flow | 3351.js, 3345.js | highlight.js syntax highlighting in terminal |
| M5-WASM → M4-GUI | Data Flow | 2218.js | Yoga WASM flexbox for layout (if used by GUI) |
| M5-FSHelpers → M3-Tool | Data Flow | 0861.js | File tools use atomic write for safe modifications |
| M5-Errors → M2-Mission | API | 0008.js | ToolAbortError for worker cancellation |
| M5-Errors → M3-Tool | API | 0992.js | McpError wraps MCP protocol errors |
| M2-Mission → M1-TUI | Events | 2542.js → 3978.js | Mission progress UI component renders in TUI |
| M2-Mission → M4-GUI | Events | 2542.js → Renderer | Mission panel in Electron shows feature progress |
| M4-GUI → M5-Auth | IPC | preload.js → auth.signIn() | OAuth deep link handling (opendroid:// protocol) |

### Connection Statistics

| Category | Count |
|----------|-------|
| Total cross-system connections | 30+ |
| Data Flow (DF) | 12 |
| API Calls | 9 |
| Events (EV) | 4 |
| Shared State (SS) | 5 |
| Bidirectional | 1 |
| Daemon connections (hub) | 9 |
| Config connections (pervasive) | 4 |

## Key Findings

### 1. Daemon-Centric Architecture is the Backbone
The IPC Daemon (opendroidd/opendroid.exe) is the single most critical component in OpenDroid's architecture. With 9+ external connections spanning all 4 mission areas, it serves as the central communication hub. Both TUI and GUI connect via WebSocket JSON-RPC 2.0. The daemon manages session lifecycle, routes tool calls, handles permission prompts, broadcasts notifications, and spawns worker sessions for missions. **Any port of OpenDroid must preserve or replace this centralized daemon pattern.**

### 2. Dual Rendering Surfaces Share State via Callbacks
TUI (React/Ink) and GUI (Electron/React 19) are independent rendering surfaces that consume the same ConversationStateManager (2572.js). State consistency is maintained through callback-based UI updates (`uiMessageCallback` → `scheduleReactUpdate()`). Compaction events trigger UI callbacks in both TUI (4038.js) and GUI (4065.js). This clean separation means the session model can be ported independently of rendering.

### 3. Config Precedence Chain Pervades All Behavior
The hierarchical config loader (0267.js) with its 11-level precedence chain (env > CLI > dynamic > user > project > folder > org > builtin) influences every subsystem: tool permissions (M3), model selection (M2/M3), plugin configuration, session settings, and cloud SDK behavior. The MissionFileService (2201.js) extends this with per-mission model-settings.json overrides.

### 4. Plugin ↔ Tool Bridge Enables Dynamic Extensibility
The MCP Service (1178.js) acts as a dynamic bridge between infrastructure and the tool system. MCP servers discovered from config are registered in the global tool registry (`k0()`), making their tools available to LLM calls. The GUI controls this bridge through enableServer/disableServer APIs. This pattern allows OpenDroid to be extended at runtime without code changes.

### 5. Mission System is Deeply Integrated with Daemon
The mission/orchestration system (M2) depends heavily on daemon IPC for worker lifecycle management: spawning workers via `spawnWorkerSession` RPC, monitoring via progress_log.jsonl polling, killing via `kill_worker_session`, and crash recovery. The daemon's role as process manager makes it inseparable from the mission system.

### 6. Cross-Cutting Observability Without Direct Mission Connections
Telemetry (OTel SDK) and Logging (OTel DiagAPI + ErrTracker/winston bridge) have no direct external mission connections but are pervasive internal infrastructure. Every RPC call through the daemon creates an OTel span. Every error flows through ErrTracker transport. Every HTTP request is auto-instrumented. This transparent observability layer is essential for debugging the complex multi-agent system.

### 7. Cloud SDKs are Tightly Coupled with Auth
AWS SDK (S3 + STS + SSO OIDC) and GCP client (google-auth-library + gRPC) both integrate deeply with the auth subsystem for credential chains. Cross-cloud identity federation (AWS STS → GCP workload identity) adds complexity. The update checker depends on S3 for binary distribution, making cloud SDKs part of the critical startup path.

### 8. Foundation Layer is Mostly Vendor Libraries
The foundation layer: errors, semver, glob, WASM runtime: consists primarily of well-known vendor libraries (node-semver, picomatch, highlight.js, protobuf.js). The custom WASM component (Yoga layout engine) is a Facebook OSS project. Token counting uses a simple heuristic (charCount/4) plus Anthropic's server-side API rather than tiktoken WASM.

## Implementation Notes

### 1. Preserve Daemon-Centric Architecture
The IPC Daemon pattern is sound and well-tested. Port the JSON-RPC 2.0 protocol and WebSocket transport layer first. The daemon's role as session manager, tool router, and notification broadcaster should be maintained. Consider extracting the daemon as a standalone library.

### 2. Extract Config Loader as Standalone Module
The hierarchical merge algorithm (0264.js) and path discovery (0260.js) are well-designed and reusable. Make path conventions configurable for different OS (XDG, AppData). The 11-level precedence chain should be preserved but simplified where possible.

### 3. Unify Rendering Adapter Pattern
The ConversationStateManager (2572.js) already has clean separation from rendering via callbacks. Port it directly, then attach new rendering surfaces. The `uiMessageCallback` → `scheduleReactUpdate()` pattern can be replicated for any UI framework.

### 4. Make Cloud SDKs Pluggable
Abstract S3/GCS storage behind a common blob storage interface. Abstract AWS STS/Google Auth behind a credential provider interface. This allows OpenDroid to support different cloud backends or run fully offline.

### 5. Consolidate Observability
The current three-layer observability (OTel DiagAPI + gRPC logging + ErrTracker/winston) could be unified into a single logging/metrics/tracing stack. Use OpenTelemetry as the single framework for all three pillars.

### 6. Simplify Mission System Dependencies
The mission system's tight coupling with daemon RPC could be abstracted behind a worker management interface, allowing different process management strategies (containers, threads, separate machines).

### 7. Tool System is Highly Portable
The tool registry pattern (1176.js), tool factory (1201.js), and permission engine are well-modularized and can be ported with minimal changes. The Zod → JSON Schema conversion for tool definitions is framework-agnostic.

## Module Reference

### Cross-System Critical Modules

| Module | Lines | System | Role |
|--------|-------|--------|------|
| 4071.js | ~1700 | M5-CLI | Main CLI entry point, Commander program assembly |
| 3758.js | ~163 | M5-Daemon | DaemonServer: Bun.serve() HTTP + WebSocket |
| 3757.js | ~361 | M5-Daemon | ConnectionManager: WS auth + lifecycle |
| 3754.js | ~2197 | M5-Daemon | DroidRequestHandler: 50+ JSON-RPC methods |
| 3755.js | ~202 | M5-Daemon | RpcRouter + TerminalHandler |
| 3756.js | ~100 | M5-Daemon | Broadcaster: notification fan-out |
| 1185.js | ~1823 | M5-Session | SessionService: full session lifecycle |
| 2572.js | ~987 | M5-Chat | ConversationStateManager: conversation state + UI bridge |
| 0267.js | ~600 | M5-Config | SettingsManager: hierarchical config loading |
| 1178.js | ~691 | M5-Plugin | MCP Service: plugin lifecycle + tool bridge |
| 1176.js | ~30 | M3-Tools | Tool Registry: Map-backed keyed store |
| 2548.js | ~200 | M3-Tools | Built-in tool registration (19 tools) |
| 3134.js | ~1078 | M3-LLM | OpenAI adapter: dual API path (Responses + Chat) |
| 2542.js | ~570 | M2-Mission | MissionRunner: sequential feature execution |
| 2201.js | ~626 | M2-Mission | MissionFileService: state persistence |
| 4070.js | ~80 | M1-TUI | TUI bootstrapper: Ink render entry |
| 2228.js | ~7000 | M1-TUI | react-reconciler: custom terminal renderer |
| 2294.js | ~150 | M1-TUI | Ink renderer class: container lifecycle |
| 2218.js | ~1524 | M5-WASM | Yoga WASM: flexbox layout engine |
| 0965.js | ~188 | M5-Daemon | LocalDaemonClient: CLI-to-daemon WebSocket |

### Mission Report References

| Mission | Report Count | Key Reports Referenced |
|---------|-------------|----------------------|
| 01-terminal-ui (TUI) | 15 | tui-ink-core, tui-renderer-main-loop, tui-composer-input, tui-panel-session-selector, tui-status-bar |
| 02-orchestration (Mission) | 16 | mission-orchestrator-core, mission-worker-lifecycle, mission-state-machine, mission-task-tool, mission-validation-contract |
| 03-tool-agent-system (Tool) | 14 | tool-core-registry, tool-execute, tool-llm-openai, tool-llm-anthropic, tool-mcp-client, tool-permissions |
| 04-desktop-gui (GUI) | 16 | gui-main-process-core, gui-main-process-daemon, gui-preload-bridge, gui-renderer-core, gui-integration |
| 05-infrastructure (Infra) | 20 + 1 | All 20 infra reports + xref-cross-module-audit |
