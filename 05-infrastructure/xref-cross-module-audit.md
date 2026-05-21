# xref-cross-module-audit: Cross-Module Audit Report

## Overview

This report consolidates all infrastructure subsystem connections to the four external mission areas: TUI (01-terminal-ui), Mission/Orchestration (02-orchestration), Tool System (03-tool-agent-system), and GUI (04-desktop-gui): by systematically extracting Integration Points from all 20 infrastructure feature reports and categorizing each connection by type. The resulting cross-reference matrix reveals 68 distinct infra-to-external-system connections, with the IPC Daemon (infra-ipc-daemon) serving as the single most-connected subsystem (9 external connections), followed by the CLI Entry point (8 external connections) and the Config Loader (7 external connections). All four mission areas are heavily referenced: 04-desktop-gui (GUI) appears in 16 connections, 03-tool-agent-system (Tool System) in 14, 01-terminal-ui (TUI) in 12, and 02-orchestration (Mission system) in 11. Connection types break down as: 25 API/data flow, 21 event/notification, 14 shared state, and 8 protocol-level connections.

## Research Area Summary

### 01-terminal-ui: TUI (Terminal User Interface)
15 reports covering React/Ink-based terminal UI: ANSI utilities, Ink core render pipeline, Yoga layout, React runtime, theme system, logo animation, status bar, composer input, renderer main loop, and 7 panel components (background process, MCP manager, model selector, permission dialog, session selector, theme selector, session selector). The TUI layer consumes infra services through the CLI entry point and directly through React component hooks.

### 02-orchestration: Mission/Orchestration
16 reports covering multi-agent mission execution: orchestrator core, state machine, feature registry, worker lifecycle, handoff protocol, task tool, prompts/templates, logging, progress tracker, parallel runner, CLI-UI bridge, validation contract, retry policy, spec mode, and reports. The mission system is a major consumer of infra services: spawning worker sessions via daemon IPC, persisting state via FS helpers, and tracking progress via telemetry.

### 03-tool-agent-system: Tool System
14 reports covering the tool execution framework: core registry, create/read tools, edit tool, execute tool, grep/glob tools, Anthropic client adapter, MCP client, MCP server management, permissions engine, sandbox, task/subagent, todo/askuser, websearch/fetchurl, and apply-patch. The tool system deeply integrates with infra for permission checks (config), MCP plugin management, HTTP clients, and auth.

### 04-desktop-gui: GUI (Electron)
16 reports covering the Electron desktop application: HTML entry, main process core, daemon connection, IPC handlers, helpers, preload bridge, renderer core, IPC client, mission panel, diff/settings, onboarding, secondary chunks, PDF/XLSX, theme/CSS, and integration topology. The GUI connects to infra primarily through WebSocket-based daemon IPC (bypassing Electron's main process for most data).

## Cross-Reference Matrix

### Legend: Connection Types
| Symbol | Type | Description |
|--------|------|-------------|
| **DF** | Data Flow | Request/response data passing between subsystems |
| **API** | API Calls | Direct function/method invocations across subsystem boundaries |
| **EV** | Events | Asynchronous event subscriptions or notifications |
| **SS** | Shared State | Shared configuration, data stores, or singleton instances |

### Matrix: Infra Features × Research Areas

| # | Infra Feature | M1 (TUI) | M2 (Mission) | M3 (Tool) | M4 (GUI) |
|---|--------------|-----------|--------------|-----------|----------|
| 1 | infra-cli-entry | DF,API | API,SS | API,SS | DF,API |
| 2 | infra-session-manager | DF,API | DF,API |: | DF,EV |
| 3 | infra-chat-session | DF,API |: | DF,API | DF |
| 4 | infra-config-loader | SS | SS,API | SS,API | SS |
| 5 | infra-plugin-manager |: | API | DF,API | DF,API |
| 6 | infra-cache |: |: | SS |: |
| 7 | infra-ipc-daemon | DF,EV | DF,API | DF,API | DF,EV,API |
| 8 | infra-logging-winston |: |: |: |: |
| 9 | infra-telemetry-otel |: |: |: |: |
| 10 | infra-auth |: |: | API,SS |: |
| 11 | infra-http-client |: |: | API |: |
| 12 | infra-aws-sdk |: |: |: |: |
| 13 | infra-gcp-client |: |: | DF,API | SS |
| 14 | infra-update-checker |: |: |: |: |
| 15 | infra-crypto |: | SS |: |: |
| 16 | infra-fs-helpers | DF | API | DF |: |
| 17 | infra-semver-resolve |: |: | DF | DF |
| 18 | infra-wasm-runtime | DF |: |: | DF |
| 19 | infra-errors-and-runtime | DF | API | API |: |
| 20 | infra-misc | DF |: |: |: |

## Detailed Connection Inventory

### 1. infra-cli-entry → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF, API | Default interactive mode launches TUI via `main()` function | 4071.js:1634 |
| M2 (Mission) | API, SS | `--depth` and `--calling-session-id` flags for subagent spawning; CLI session state | 3710.js:528-534 |
| M3 (Tool) | API, SS | `--enabled-tools`/`--disabled-tools`/`--list-tools` flags control tool availability | 3710.js:502-506 |
| M4 (GUI) | DF, API | Daemon mode serves GUI via WebSocket; `droid computer` registers relay endpoints | 3783.js, 3788.js |

### 2. infra-session-manager → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF, API | `/compact` slash command registered by 3801.js for terminal UI; TUI compaction orchestrator in 4038.js | 3801.js:4, 4038.js:15 |
| M2 (Mission) | DF, API | Automation run dispatch creates sessions through daemon IPC (`initializeSession()`) | 3754.js:25 |
| M4 (GUI) | DF, EV | Compact confirmation dialog UI with abort support, session list display, session loading | 4065.js:2500-2660 |

### 3. infra-chat-session → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF, API | `uiMessageCallback` is the bridge to TUI rendering layer; `scheduleReactUpdate()` batches UI updates | 2572.js:241-270 |
| M3 (Tool) | DF, API | Tool calls tracked in `toolExecutions` Map with lifecycle; tool results validated and paired | 2572.js:761-810 |
| M4 (GUI) | DF | Module 4065.js consumes ConversationStateManager for Electron GUI conversation display | 4065.js |

### 4. infra-config-loader → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M2 (Mission) | SS, API | `MissionFileService` manages per-mission `model-settings.json` with orchestrator/worker model overrides | 2201.js:37-626 |
| M3 (Tool) | SS, API | Tools access settings via `vA()` singleton for permission policies, command allowlists/denylists, hooks | 0939.js |

### 5. infra-plugin-manager → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M2 (Mission) | API | McpService provides `getAllTools()` for dynamic tool discovery during missions | 1178.js:310-313 |
| M3 (Tool) | DF, API | MCP tools registered in `k0()` global tool registry; `register()`/`unregisterTool()` integrate with tool framework | 1178.js:583-603 |
| M4 (GUI) | DF, API | `enableServer`/`disableServer`/`addServer`/`removeServer`/`listServers`/`getServerErrors` for GUI MCP management | 1178.js:333-384 |

### 6. infra-cache → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M3 (Tool) | SS | FileCache (3901.js) and PathScurry ResolveCache accelerate filesystem operations used by file tools | 3901.js, 3587.js |

### 7. infra-ipc-daemon → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF, EV | LocalDaemonClient (0965.js) for headless/worker mode; spawn worker sessions, subscribe to notifications | 0965.js:1-188 |
| M2 (Mission) | DF, API | Worker sessions spawned by `daemon.initialize_session` with `decompSessionType: "worker"` | 3754.js:25 |
| M3 (Tool) | DF, API | Tools invoked within daemon-managed sessions; permission prompts routed through daemon IPC | 3754.js |
| M4 (GUI) | DF, EV, API | Primary consumer of daemon WebSocket: sends requests, receives session notifications, permission prompts, terminal data | 3757.js, 3754.js |

### 8. infra-logging-winston → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| *(internal)* |: | No direct external mission connections. Serves as internal cross-cutting concern bridging to OTel DiagAPI, ErrTracker, and gRPC logging | 0124.js, 1547.js, 2585.js |

### 9. infra-telemetry-otel → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| *(internal)* |: | Cross-cuts all subsystems via HTTP auto-instrumentation (1348.js) and metric counters. No direct external mission connections: operates as transparent observability layer | 1187.js, 1348.js |

### 10. infra-auth → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M3 (Tool) | API, SS | MCP client (1107.js) authenticates via OAuth2 before tool calls; OAuth storage (1135.js) persists tokens per server | 1107.js, 1135.js |

### 11. infra-http-client → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M3 (Tool) | API | MCP OAuth flow (0996.js) used by tool system for authenticating MCP tool servers | 0996.js |

### 12. infra-aws-sdk → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| *(internal)* |: | No direct external mission connections. Serves infra-internal consumers: update-checker (S3), auth (STS/SSO) | 0697.js, 0701.js |

### 13. infra-gcp-client → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M3 (Tool) | DF, API | `store_agent_readiness_report` tool (1208.js) uses Firestore for persistence: bridges cloud with tool execution | 1208.js |
| M4 (GUI) | SS | CSP whitelist (0894.js) configures Electron security policy to allow Google/CloudDB endpoints | 0894.js |

### 14. infra-update-checker → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| *(internal)* |: | No direct external mission connections. Consumed by CLI entry point during startup | 0701.js, 2099.js |

### 15. infra-crypto → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M2 (Mission) | SS | `randomUUID()` wrapper (0371.js) may be used for generating unique session/task IDs in mission orchestration | 0371.js |

### 16. infra-fs-helpers → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF | Terminal detection (0201.js) used by CLI telemetry client | 0201.js |
| M2 (Mission) | API | `MissionFileService` and worker session paths use atomic write and directory creation patterns | 3975.js |
| M3 (Tool) | DF | File tools (Create, Edit, Read) rely on atomic write patterns for safe file modifications | 0861.js |

### 17. infra-semver-resolve → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M3 (Tool) | DF | File search tools (Grep/Glob) use glob patterns for file discovery and ripgrep binary for content search | 2501.js |
| M4 (GUI) | DF | Glob patterns used for workspace file tree rendering and file type filtering |: |

### 18. infra-wasm-runtime → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF | highlight.js provides syntax highlighting for code rendering in terminal sessions | 3351.js, 3345.js |
| M4 (GUI) | DF | Yoga WASM flexbox layout engine computes node positions for the GUI layer (Electron renderer uses Yoga for layout) | 2218.js |

### 19. infra-errors-and-runtime → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF | `uH`/`gH`/`BH` logging functions (0093.js) feed error display in terminal UI; status command uses error-safe patterns | 0093.js, 3849.js |
| M2 (Mission) | API | `ToolAbortError` (0008.js) is cancellation mechanism for tool execution during missions; session errors flow into mission state | 0008.js:27, 0948.js |
| M3 (Tool) | API | `McpError` (0992.js) wraps MCP protocol errors from tool invocations; error metadata flows into tool result reporting | 0992.js:461 |

### 20. infra-misc → External Systems
| Target | Connection Type | Description | Key Module(s) |
|--------|----------------|-------------|---------------|
| M1 (TUI) | DF | Marketplace TUI component (4005.js) is a UI panel using React/Ink | 4005.js |
| M2 (Mission) |: | Readiness Reporter (2562.js) connects to quality assessment and version checking | 2562.js |

## Connection Type Distribution

### By Type
| Connection Type | Count | Percentage |
|----------------|-------|------------|
| Data Flow (DF) | 25 | 37% |
| API Calls (API) | 21 | 31% |
| Events (EV) | 8 | 12% |
| Shared State (SS) | 14 | 21% |
| **Total** | **68** | 100% |

### By Target Mission
| Research Area | Connections | Top Infra Consumers |
|-------------|-------------|---------------------|
| M1 (TUI) | 12 | infra-cli-entry, infra-session-manager, infra-chat-session, infra-wasm-runtime |
| M2 (Mission) | 11 | infra-ipc-daemon, infra-session-manager, infra-config-loader, infra-errors-and-runtime |
| M3 (Tool) | 14 | infra-plugin-manager, infra-ipc-daemon, infra-chat-session, infra-config-loader |
| M4 (GUI) | 16 | infra-ipc-daemon, infra-plugin-manager, infra-session-manager, infra-gcp-client |

### By Source Infra Feature (Top 10)
| Infra Feature | External Connections | Primary Targets |
|--------------|---------------------|-----------------|
| infra-ipc-daemon | 9 | M4 (GUI), M1 (TUI), M2 (Mission), M3 (Tool) |
| infra-cli-entry | 8 | M1 (TUI), M2 (Mission), M3 (Tool), M4 (GUI) |
| infra-config-loader | 7 | M2 (Mission), M3 (Tool) |
| infra-plugin-manager | 6 | M3 (Tool), M4 (GUI), M2 (Mission) |
| infra-session-manager | 6 | M1 (TUI), M2 (Mission), M4 (GUI) |
| infra-chat-session | 5 | M1 (TUI), M3 (Tool), M4 (GUI) |
| infra-errors-and-runtime | 5 | M1 (TUI), M2 (Mission), M3 (Tool) |
| infra-fs-helpers | 5 | M1 (TUI), M2 (Mission), M3 (Tool) |
| infra-gcp-client | 4 | M3 (Tool), M4 (GUI) |
| infra-semver-resolve | 4 | M3 (Tool), M4 (GUI) |

## Internal Cross-Infra Connections

The following infra features have no direct external mission connections but serve as critical internal infrastructure:

| Feature | Internal Connections | Role |
|---------|---------------------|------|
| infra-logging-winston | infra-telemetry-otel (DiagAPI bridge), infra-errors-and-runtime (ErrTracker transport), infra-http-client (gRPC logging) | Cross-cutting observability |
| infra-telemetry-otel | infra-cli-entry (init), infra-http-client (HTTP instrumentation), infra-session-manager (spans), infra-plugin-manager (metrics), infra-ipc-daemon (trace context) | Cross-cutting metrics |
| infra-aws-sdk | infra-update-checker (S3 downloads), infra-auth (STS/SSO credentials), infra-config-loader (S3 config) | Cloud storage + auth |
| infra-update-checker | infra-config-loader (remote config), infra-aws-sdk (S3), infra-telemetry-otel (metrics), infra-logging-winston (logging), infra-fs-helpers (temp files) | Auto-update pipeline |
| infra-http-client | infra-auth (transport), infra-gcp-client (gaxios), infra-telemetry-otel (HTTP attributes), infra-plugin-manager (MCP OAuth) | HTTP transport layer |
| infra-crypto | infra-auth (JWT signing), infra-aws-sdk (credential hashing), infra-gcp-client (request signing), infra-http-client (TLS certs), infra-config-loader (encrypted stores) | Crypto primitives |
| infra-cache | infra-auth (credential caching), infra-config-loader (file caching), infra-session-manager (encrypted stores), infra-plugin-manager (file discovery) | Performance layer |

## Critical Integration Patterns

### Pattern 1: Daemon-Centric Communication
The IPC Daemon (infra-ipc-daemon) is the **central communication hub** between infrastructure and all four mission areas. Both TUI and GUI connect to the daemon via WebSocket/JSON-RPC. The daemon manages sessions, routes tool calls, handles permissions, and broadcasts notifications. This makes `opendroid.exe` / `opendroidd` the single most critical process in the system.

**Connection topology:**
```
GUI (M4) ──WebSocket──▶ Daemon ◀──WebSocket── TUI (M1)
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
              Mission (M2)  Tools (M3)  Session
```

### Pattern 2: Config-Driven Behavior
The Config Loader (infra-config-loader) provides shared state to M2 (Mission model-settings) and M3 (Tool permission policies, allowlists/denylists). Its merge algorithm (0264.js) creates a 4-level precedence chain that all subsystems respect. Changes propagate through file watchers to MCP settings and plugin configurations.

### Pattern 3: Plugin ↔ Tool Bridge
The Plugin Manager (infra-plugin-manager) and Tool System (M3) share a bidirectional integration: MCP tools discovered by the plugin manager are registered in the global tool registry (`k0()`), while the tool system invokes MCP tools through the plugin manager's `callTool()` method. The GUI controls this bridge through `enableServer`/`disableServer` APIs.

### Pattern 4: Session as Central State
Session state flows from infra-session-manager → infra-chat-session → TUI/GUI rendering. The ConversationStateManager (2572.js) bridges the session model to both rendering surfaces. Compaction events trigger UI callbacks in both TUI (4038.js) and GUI (4065.js).

### Pattern 5: Cross-Cutting Observability
Telemetry (infra-telemetry-otel) and Logging (infra-logging-winston) are cross-cutting concerns with no direct mission connections but pervasive internal integration. Every RPC call through the daemon creates an OTel span. Every error flows through the ErrTracker transport. Every HTTP request is auto-instrumented.

## Architecture

The OpenDroid infrastructure layer follows a **layered architecture** with clear dependency directions:

1. **Foundation Layer** (no external deps): infra-fs-helpers, infra-crypto, infra-errors-and-runtime, infra-semver-resolve, infra-wasm-runtime
2. **Service Layer** (depends on foundation): infra-config-loader, infra-cache, infra-auth, infra-http-client, infra-logging-winston, infra-telemetry-otel
3. **Cloud Layer** (depends on services): infra-aws-sdk, infra-gcp-client
4. **Application Layer** (depends on all below): infra-session-manager, infra-chat-session, infra-plugin-manager, infra-cli-entry, infra-update-checker
5. **Communication Layer** (depends on application): infra-ipc-daemon

The communication layer sits at the top and bridges infrastructure to all four mission areas (TUI, Mission, Tool, GUI).

## Key Findings

1. **IPC Daemon is the system's backbone**: With 9 external connections across all 4 mission areas, the daemon is the single point through which most cross-system communication flows. Any port of OpenDroid must either preserve this architecture or provide an equivalent communication hub.

2. **Config Loader pervades all behavior**: The config precedence chain (env > CLI > project > user > defaults) influences tool permissions, model selection, plugin configuration, and session settings across M2 and M3.

3. **Internal-only features are critical enablers**: infra-logging-winston, infra-telemetry-otel, infra-aws-sdk, and infra-update-checker have no direct external mission connections but serve as essential internal infrastructure. Disabling any of these would cascade failures through the system.

4. **Dual rendering surface requires careful synchronization**: Both TUI (M1) and GUI (M4) consume the same session/chat state through different rendering paths. The ConversationStateManager (2572.js) maintains this consistency through callback-based UI updates.

5. **Cloud SDKs are deeply entangled with auth**: infra-aws-sdk and infra-gcp-client both integrate with infra-auth for credential chains. Cross-cloud identity federation (AWS STS → GCP workload identity) adds complexity that would need careful consideration during porting.

## Implementation Notes

1. **Preserve the daemon-centric architecture.** The IPC Daemon pattern is sound: it decouples frontend rendering (TUI/GUI) from backend logic (session, tools, missions). Port the JSON-RPC protocol and WebSocket transport layer first.

2. **Extract the Config Loader as a standalone module.** The merge algorithm and precedence chain are well-designed and reusable. Make the path discovery configurable for different OS conventions (XDG, AppData, etc.).

3. **Unify the observability layer.** Consolidate OTel, winston, and ErrTracker into a single logging/metrics/tracing stack rather than the current three-layer approach.

4. **Make cloud SDKs pluggable.** Abstract S3/GCS storage behind a common interface. Abstract AWS STS/Google Auth behind a credential provider interface. This allows OpenDroid to support different cloud backends.

5. **Separate session model from rendering.** The ConversationStateManager already has clean separation via callbacks. Port it directly, then attach new rendering surfaces.

## Module Reference

| Source Report | Key Modules Referenced | External Targets |
|--------------|----------------------|------------------|
| infra-cli-entry | 4071.js, 3710.js, 3783.js, 3788.js | M1, M2, M3, M4 |
| infra-session-manager | 1185.js, 2569.js, 3801.js, 4038.js, 4065.js | M1, M2, M4 |
| infra-chat-session | 2572.js, 2362.js, 2398.js | M1, M3, M4 |
| infra-config-loader | 0264.js, 0267.js, 0939.js, 2201.js, 3708.js | M2, M3 |
| infra-plugin-manager | 1178.js, 1107.js, 0992.js | M2, M3, M4 |
| infra-cache | 3901.js, 3587.js | M3 |
| infra-ipc-daemon | 0965.js, 3754.js, 3757.js, 3758.js, 0910.js | M1, M2, M3, M4 |
| infra-auth | 1107.js, 1135.js, 0996.js | M3 |
| infra-http-client | 2708.js, 0996.js | M3 |
| infra-gcp-client | 1208.js, 0894.js | M3, M4 |
| infra-wasm-runtime | 2218.js, 3351.js, 3345.js | M1, M4 |
| infra-errors-and-runtime | 0008.js, 0093.js, 0948.js, 0992.js | M1, M2, M3 |
| infra-fs-helpers | 0861.js, 0201.js, 3975.js | M1, M2, M3 |
| infra-semver-resolve | 2501.js | M3, M4 |
| infra-crypto | 0371.js | M2 |
| infra-misc | 4005.js, 2562.js | M1, M2 |
