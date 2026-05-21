# OpenDroid

## Anatomy of Long-Running Autonomous Coding Agents

OpenDroid is an architecture research project focused on a new class of coding systems: CLI-native agents that can plan work, spawn workers, operate tools, validate milestones, persist state, and continue across long-running autonomous development sessions.

These notes map the mechanics behind multi-hour and multi-day coding agents: how they keep context alive, delegate work, recover from interruptions, coordinate tools, and expose their state through terminal and desktop surfaces.

## What This Maps

| Layer | What It Covers |
|-------|----------------|
| Terminal UI | React Ink-style terminal surfaces, panels, prompts, status, themes, and interaction loops |
| Orchestration | Long-running mission lifecycle, worker spawning, state transitions, progress tracking, and validation gates |
| Tool Runtime | Tool registry, execution flow, permissions, sandbox policy, agent state, MCP, web tools, and file operations |
| Desktop GUI | Electron-style shell, renderer/main/preload boundaries, IPC topology, onboarding, sessions, and settings surfaces |
| Infrastructure | Config loading, session persistence, plugin metadata, auth boundaries, telemetry, crypto, logging, cache, and daemon lifecycle |
| Protocol Specs | Sanitized protocol examples for sessions, configuration, feature state, handoff, validation, and runtime files |

## Repository Map

```text
opendroid-re/
├── 01-terminal-ui/              # Terminal UI system notes
├── 02-orchestration/            # Mission orchestration and worker lifecycle
├── 03-tool-agent-system/        # Tool runtime, permissions, agents, MCP
├── 04-desktop-gui/              # Desktop shell, renderer/main/preload IPC
├── 05-infrastructure/           # Config, session, telemetry, crypto, daemon, synthesis
└── 06-protocol-specifications/  # Sanitized protocol specifications and examples
```

## Research Focus

OpenDroid is especially interested in the systems design problems behind unattended coding sessions:

- How a CLI agent plans work that may span hours or days.
- How workers and subagents are spawned, tracked, and retired.
- How tool calls move through schema validation, permission checks, sandbox policy, execution, and result serialization.
- How progress, validation, handoff, and session state survive restarts.
- How terminal and desktop interfaces observe and control the same underlying agent runtime.
- How protocol files can serve as durable coordination surfaces between humans, agents, and validators.

## Where To Start

1. `06-protocol-specifications/protocol-specification.md`: end-to-end protocol synthesis.
2. `05-infrastructure/final-integration-report.md`: cross-system architecture map.
3. `03-tool-agent-system/tool-agent-system.md`: tool execution and agent loop overview.
4. `02-orchestration/mission-system.md`: long-running mission lifecycle.
5. Individual subsystem reports for detailed component notes.

## Scale

| Area | Reports |
|------|--------:|
| Terminal UI | 15 |
| Orchestration | 16 |
| Tool & Agent System | 19 |
| Desktop GUI | 15 |
| Infrastructure | 22 |
| Protocol Specifications | 8 |

Total: 87 subsystem reports, 8 protocol specifications, and 2 integration syntheses.

## Methodology

This repository contains original architectural notes derived from static analysis of compiled application artifacts and sanitized protocol examples. The emphasis is on system behavior, data flow, protocol shape, and implementation patterns rather than reproducing source code.

The reports use generalized names, placeholder identifiers, approximate bundle descriptions, and pseudocode summaries where needed. Examples are sanitized to avoid exposing personal data, credentials, secrets, proprietary assets, or exact build fingerprints.

## What Is Not Included

- No binaries.
- No extracted application assets.
- No proprietary source code.
- No credentials, API keys, tokens, cookies, or secrets.
- No personal machine paths or user-identifying local data.
- No exact build hashes, runtime IDs, or machine identifiers.

## License

This project is released under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License. See `LICENSE` for details.

## Disclaimer

OpenDroid is an independent educational and interoperability research project. It documents architectural observations about autonomous coding agent systems using sanitized examples and original analysis. It is not affiliated with, endorsed by, or connected to any commercial entity.
