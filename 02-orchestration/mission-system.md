# Mission System: Aggregate Architecture Research Notes

## Overview

This report consolidates findings from **15 per-feature architecture notes** into a single end-to-end narrative of OpenDroid's mission/orchestration system. The mission system is a multi-agent execution framework where an orchestrator LLM plans features, spawns autonomous worker sessions via a daemon RPC layer, and validates results through auto-injected scrutiny and user-testing validators. All 182 `app-mission` classified modules in the compiled module set were analyzed across 15 feature domains, covering the complete lifecycle: **proposal → accept → spawn workers → feature execution → handoff → validation → restart/seal**.

The system is built around a sequential polling loop (`MissionRunner` in Module 2542.js), file-based state persistence (`MissionFileService` in Module 2201.js), and a structured handoff protocol (`EndFeatureRun` tool in Module 2539.js). Workers are daemon-managed sessions: not child processes: created via WebSocket RPC to the `opendroidd` daemon. Parallelism exists only at the tool level (multiple `Task` tool calls in a single LLM turn), not at the feature level.

---

## Per-Feature Report Cross-Reference

| # | Feature ID | Report | Key Modules | Core Finding |
|---|-----------|--------|-------------|--------------|
| 1 | mission-orchestrator-core | [mission-orchestrator-core.md](mission-orchestrator-core.md) | 2542, 2201, 2541, 3399, 1185, 3978, 2175, 0201 | MissionRunner (`toA`) is a sequential polling loop; workers are daemon RPC sessions |
| 2 | mission-worker-lifecycle | [mission-worker-lifecycle.md](mission-worker-lifecycle.md) | 2542, 2543, 2544, 0087, 1234, 2575, 3985, 2541 | Workers spawn via `spawnWorkerSession`, monitored via 1s polling + daemon notifications |
| 3 | mission-state-machine | [mission-state-machine.md](mission-state-machine.md) | 2201, 2542, 2175, 3970, 3973, 3971, 0919, 2857, 3384, 2856 | Dual-layer state: mission-level (`state.json`) + feature-level (`features.json`) |
| 4 | mission-validation-contract | [mission-validation-contract.md](mission-validation-contract.md) | 2542, 2541, 2175, 2176, 2201, 0082, 3383, 3978 | Three-file artifact model: validation-contract.md + validation-state.json + features.json |
| 5 | mission-handoff | [mission-handoff.md](mission-handoff.md) | 0082, 1234, 2539, 2544, 3973, 1237 | Zod-validated handoff with salientSummary (20-500 chars), orchestrator blocks on unaddressed items |
| 6 | mission-task-tool | [mission-task-tool.md](mission-task-tool.md) | 2522, 2521, 2519, 2518, 2176, 3983, 2175 | Subagents spawned as `droid exec` child processes, NDJSON stdout streaming |
| 7 | mission-feature-registry | [mission-feature-registry.md](mission-feature-registry.md) | 0082, 2175, 2201, 2542, 3971, 3973 | 12-field Zod schema; strict array-order execution; 4 statuses (pending/in_progress/completed/cancelled) |
| 8 | mission-progress-tracker | [mission-progress-tracker.md](mission-progress-tracker.md) | 0082, 2201, 2542, 2176, 3978, 3970, 0087 | Milestone completion excludes validators from counting; idempotent via `milestonesWithValidationPlanned` |
| 9 | mission-spec-mode | [mission-spec-mode.md](mission-spec-mode.md) | 1226, 1235, 0060, 0084, 0939, 1185, 2532, 3139, 2572, 3146, 3399, 2575 | `interactionMode === "spec"` blocks non-readonly tools; ExitSpecMode + ProposeMission for gated execution |
| 10 | mission-prompts-and-templates | [mission-prompts-and-templates.md](mission-prompts-and-templates.md) | 2175, 2541, 2526, 2517, 2519, 2857, 2522, 2312, 0060, 3399, 1181, 4065 | 4-layer prompt stack: identity → base → role → dynamic context |
| 11 | mission-reports | [mission-reports.md](mission-reports.md) | 3977, 3975, 1237, 2175, 2542, 3978, 1250, 3924 | LLM-prompt-driven validation; synthesis.json schema for scrutiny + user-testing reports |
| 12 | mission-parallel-runner | [mission-parallel-runner.md](mission-parallel-runner.md) | 1147, 0279, 2362, 2521, 2522, 0965, 2175, 1236, 0257, 3988 | Feature-level sequential; tool-level parallel via `Promise.all` over multiple `tool_use` blocks |
| 13 | mission-retry-policy | [mission-retry-policy.md](mission-retry-policy.md) | 2542, 2544, 0454, 2539, 0082, 0472, 2175 | No automatic retry; delegated to orchestrator; feature re-queuing via `S9H()` on crash |
| 14 | mission-logging | [mission-logging.md](mission-logging.md) | 0093, 2201, 2542, 0084, 2509, 1236, 1225, 1203, 1227, 1229 | Three-tier: console logger (BH/gH/uH) + JSONL progress log + event notification bus |
| 15 | mission-cli-ui-bridge | [mission-cli-ui-bridge.md](mission-cli-ui-bridge.md) | 0084, 2542, 2543, 2544, 2201, 2478, 1186, 0934 | 6 event types via in-process EventEmitter; 500ms polling + 180s heartbeat |

---

## End-to-End Mission Lifecycle

### Phase 1: Proposal (Spec Mode + ProposeMission)

The lifecycle begins in **spec mode** (`interactionMode === "spec"`), where the agent is constrained to read-only operations:

1. **Spec mode entry**: The user or orchestrator sets `interactionMode = "spec"` via `SessionService.cycleInteractionMode()` (Module 1185.js). A `specModeModel` may be configured for the planning phase.
2. **Research & planning**: The agent reads code, searches files, and crafts a multi-feature mission plan. Non-readonly tools are blocked by `3146.js` with error message `wRI`.
3. **Plan proposal**: The agent calls either:
   - **`ExitSpecMode`** (Module 1226.js) for single-feature plans, or
   - **`ProposeMission`** (Module 1235.js) for multi-feature mission proposals
4. **User approval**: The UI presents the plan with options: `proceed_once`, `proceed_edit`, or `cancel`. On `proceed_edit`, the plan is saved to `~/.opendroid/specs/` and opened in the editor.
5. **Mission creation**: On acceptance, the system creates `missionDir` at `~/.opendroid/missions/{sessionId}/` and writes:
   - `mission.md`: Mission proposal document
   - `features.json`: Feature array with status, skillName, milestone, fulfills
   - `validation-contract.md`: Assertion definitions per feature area
   - `validation-state.json`: Per-assertion tracking (all `"pending"`)
   - `agent-guidelines.md`: Worker/orchestrator guidance
   - `state.json`: Initial state: `{ state: "initializing", ... }`

> **Source reports**: [mission-spec-mode.md](mission-spec-mode.md), [mission-feature-registry.md](mission-feature-registry.md), [mission-state-machine.md](mission-state-machine.md)

### Phase 2: Accept & Initialize (start_mission_run)

The orchestrator calls the **`start_mission_run`** tool (Module 2544.js, `HtA` class):

1. **Orchestrator session upgrade**: `SessionService.upgradeToOrchestratorSession()` (Module 1185.js) promotes the current session to orchestrator mode with AGI-level permissions.
2. **MissionRunner instantiation**: `MissionRunner` class (`toA`, Module 2542.js) is created with a `MissionFileService` instance and abort signal.
3. **State initialization**: If `state.json` doesn't exist, `createInitialState()` creates it with `state: "initializing"`.
4. **Cleanup**: `cleanupOrphanedWorker()` resets any stale worker session referenced in state.json.
5. **Loop entry**: State transitions to `"running"` and the main `runLoop()` begins.

> **Source reports**: [mission-orchestrator-core.md](mission-orchestrator-core.md), [mission-worker-lifecycle.md](mission-worker-lifecycle.md)

### Phase 3: Spawn Workers (Sequential Feature Execution)

The `runLoop()` is a synchronous-style polling state machine:

```
while (isRunning):
  readState()
  if state === "running":
    feature = getNextPendingFeature()  // first pending in features.json
    if feature:
      spawnWorker(feature)
      result = waitForWorkerCompletion()
      if result.success:
        checkMilestoneCompletionAndInjectValidation(feature.id)
      else:
        state = "orchestrator_turn"
        break
    else:
      checkAllMilestonesForValidation()
      if areAllFeaturesCompleted():
        state = "completed"
        break
      else:
        state = "orchestrator_turn"
        break
  elif state in ["paused", "orchestrator_turn", "completed"]:
    break
```

**Worker spawning flow** (`spawnWorker()`, Module 2542.js):
1. Generate spawn ID (`worker_{random8chars}`)
2. Select model: validation workers use `getMissionValidationWorkerModel()`, implementation workers use `getMissionWorkerModel()`
3. Call `zX().spawnWorkerSession()`: daemon RPC creates a new autonomous session
4. Update feature status to `"in_progress"` with `currentWorkerSessionId`
5. Build bootstrap prompt via `GNI(missionDir, feature, sessionId)` (Module 2541.js): includes feature JSON, skill invocation order, session rules
6. Send bootstrap message via `zX().addUserMessage()`
7. Begin monitoring in `waitForWorkerCompletion()`

**Worker monitoring** (`waitForWorkerCompletion()`):
- Poll `progress_log.jsonl` every 1000ms for `worker_completed` entry
- Subscribe to daemon notifications: `session_inactivity` (timeout), `ProcessExitError` (crash)
- Daemon health check every 5s via TCP port probe (3 strikes = unreachable)
- Emit heartbeat every 180s for UI freshness

> **Source reports**: [mission-orchestrator-core.md](mission-orchestrator-core.md), [mission-worker-lifecycle.md](mission-worker-lifecycle.md), [mission-feature-registry.md](mission-feature-registry.md)

### Phase 4: Feature Execution (Worker Runtime)

Each worker runs as an autonomous daemon session with:
- `interactionMode: "auto"`, `autonomyLevel: "high"`
- Skill injection via the `Skill` tool (Module 2526.js loads SKILL.md files)
- A 4-layer prompt stack: identity → base system prompt → role prompt → dynamic context
- Tools: Read, Grep, Glob, Edit, Create, Execute, Task, etc.

**Prompt assembly** (Module 2541.js, 2175.js, 2857.js):
```
Layer 1: "You are Droid, an AI software engineering agent built by OpenDroid."
Layer 2: hMH: default interactive CLI behavior (2312.js)
Layer 3: GNI() output: worker bootstrap with feature JSON + skill instructions
Layer 4: <system-reminder> tags with agent-guidelines.md, mission context, skill content
```

**Tool-level parallelism**: When the LLM emits multiple `tool_use` blocks (e.g., multiple `Task` invocations), `Promise.all()` executes them concurrently (Module 2362.js). Each Task call spawns a `droid exec` child process (Module 2521.js) with isolated session, depth tracking, and abort propagation.

**Logging during execution**:
- Console: `BH()` info, `gH()` warn, `uH()` error (Module 0093.js) with ErrTracker breadcrumbs
- Progress: `appendProgressLog()` writes JSONL entries (Module 2201.js)
- Events: `v$.emit("project-notification", ...)` for TUI updates (Module 0084.js)

> **Source reports**: [mission-prompts-and-templates.md](mission-prompts-and-templates.md), [mission-task-tool.md](mission-task-tool.md), [mission-parallel-runner.md](mission-parallel-runner.md), [mission-logging.md](mission-logging.md)

### Phase 5: Handoff (EndFeatureRun)

When a worker completes, it calls **`EndFeatureRun`** (Module 2539.js):

1. **Validate handoff fields**: Zod schema `Z11` (Module 1234.js) validates:
   - `salientSummary`: 20-500 chars, 1-4 sentences, no newlines
   - `whatWasImplemented`: min 50 chars
   - `whatWasLeftUndone`: string (empty if complete)
   - `verification`: commands run + interactive checks
   - `tests`: added/updated test files with coverage summary
   - `discoveredIssues`: array of `{ severity, description, suggestedFix? }`
   - `skillFeedback`: `{ followedProcedure, deviations, suggestedChanges }`

2. **Compute escalation flag**: `returnToOrchestrator || hasDiscoveredIssues || hasUnfinishedWork`

3. **Update state**:
   - `success` → feature status = `"completed"`
   - Otherwise → feature status = `"pending"` (re-queued)
   - Append `worker_completed` to progress log
   - Write handoff JSON to `handoffs/{timestamp}__{featureId}__{sessionId}.json`
   - If escalation → `state.json` state = `"orchestrator_turn"`

4. **Generate transcript skeleton** from worker's message events

**Orchestrator consumption** (Module 2544.js):
- If `discoveredIssues` or `whatWasLeftUndone` exist → `start_mission_run` **blocks** with error
- Orchestrator must create follow-up features, update descriptions, or dismiss items via `dismiss-handoff-items` (Module 1237.js) before mission can resume

> **Source reports**: [mission-handoff.md](mission-handoff.md), [mission-state-machine.md](mission-state-machine.md)

### Phase 6: Validation (Auto-Injected Validators)

When all implementation features in a milestone complete, `MissionRunner` auto-injects validation:

1. **Milestone completion detection**: `isMilestoneImplementationComplete()` (Module 2201.js) filters out validator skill names (`HMH = ["scrutiny-validator", "user-testing-validator"]`) and checks all remaining features are `completed` or `cancelled`.

2. **Idempotent injection**: `hasValidationPlannerRun()` checks `state.json.milestonesWithValidationPlanned` to prevent duplicates.

3. **Feature insertion**: Two features are inserted at the **top** of `features.json`:
   - `scrutiny-validator-{milestone}`: reviews code quality per feature
   - `user-testing-validator-{milestone}`: tests user-facing behavior per assertion
   
   Scrutiny runs first (inserted after user-testing, so it ends up on top).

4. **Skip flags**: `model-settings.json` can set `skipScrutiny` and/or `skipUserTesting` to bypass validators.

**Scrutiny validator flow** (prompt in Module 2175.js):
1. Run hard-gate validators (test, typecheck, lint from `.opendroid/services.yaml`)
2. Spawn one `scrutiny-feature-reviewer` subagent per completed feature (parallel)
3. Synthesize findings into `.opendroid/validation/<milestone>/scrutiny/synthesis.json`

**User-testing validator flow** (prompt in Module 2175.js):
1. Collect testable assertions from `features[].fulfills` cross-referenced with `validation-state.json`
2. Setup environment (start services, seed data)
3. Spawn one `user-testing-flow-validator` subagent per assertion group (parallel)
4. Update `validation-state.json` with pass/fail/blocked results
5. Synthesize into `.opendroid/validation/<milestone>/user-testing/synthesis.json`

5. **Sealing**: After validation passes, the milestone is "sealed" (policy enforced by orchestrator guidance, not hardcoded). No features should be added to sealed milestones.

> **Source reports**: [mission-validation-contract.md](mission-validation-contract.md), [mission-progress-tracker.md](mission-progress-tracker.md), [mission-reports.md](mission-reports.md)

### Phase 7: Restart / Seal

**On validation failure**:
- Failed assertions in `validation-state.json` are set to `"failed"`
- Orchestrator creates fix features targeting the failures
- Fix features go through the same lifecycle (spawn → execute → handoff)
- Re-run validation reads prior `synthesis.json` to scope reviews to fix features only

**On all-pass**:
- All assertions in `validation-state.json` are `"passed"`
- All features across all milestones are `completed` or `cancelled`
- Mission state transitions to `"completed"`
- Mission declared done

**On worker failure** (crash, timeout, daemon unreachable):
1. Feature is re-queued: status reset from `"in_progress"` to `"pending"` via `S9H()` (Module 2542.js)
2. `worker_failed` entry appended to progress log with reason
3. State transitions to `"orchestrator_turn"`
4. **No automatic retry**: orchestrator decides whether to re-run, fix, or abandon
5. On daemon failure: orchestrator is instructed "call `start_mission_run` once to retry, then stop"

**Pause/resume**:
- User SIGINT → `pauseMissionRunner()` → interrupt worker → state = `"paused"`
- Resume → `resumeWorker()` → load previous session → send continuation prompt

> **Source reports**: [mission-retry-policy.md](mission-retry-policy.md), [mission-worker-lifecycle.md](mission-worker-lifecycle.md), [mission-state-machine.md](mission-state-machine.md)

---

## Consolidated Module Map

### Core Modules (Cross-Feature, High Reuse)

| Module | Size (KB) | Primary Role | Referenced In Reports |
|--------|-----------|-------------|----------------------|
| **2542.js** | 19.0 | MissionRunner (`toA`): core run loop | orchestrator-core, worker-lifecycle, state-machine, validation-contract, feature-registry, progress-tracker, retry-policy, logging, cli-ui-bridge |
| **2201.js** | 13.6 | MissionFileService (`PMH`): state persistence | orchestrator-core, state-machine, validation-contract, feature-registry, progress-tracker, logging, cli-ui-bridge |
| **2175.js** | 135.8 | Prompt template library: ALL orchestrator/worker/validator prompts | orchestrator-core, state-machine, validation-contract, feature-registry, progress-tracker, prompts-and-templates, reports, parallel-runner, retry-policy |
| **2544.js** | ~20 | start_mission_run tool (`HtA`): orchestrator blocking call | worker-lifecycle, handoff, retry-policy, logging, cli-ui-bridge |
| **1185.js** | 42.3 | SessionService (`OGH`): session CRUD, orchestrator upgrade | orchestrator-core, spec-mode |
| **0082.js** | ~3 | Zod schemas: features, handoffs, events | validation-contract, handoff, feature-registry, progress-tracker, retry-policy |
| **0084.js** | ~7.5 | Event notification schemas | spec-mode, logging, cli-ui-bridge |
| **2539.js** | ~8 | EndFeatureRun tool (`roA`) | handoff, retry-policy |
| **0093.js** | ~3 | Core logging functions (BH/gH/uH) | logging |
| **0201.js** | ~0.2 | OpenDroid home resolver | orchestrator-core |

### Worker Lifecycle Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **2543.js** | ~4 | Worker lifecycle helpers: pause, kill, requeue | worker-lifecycle, cli-ui-bridge |
| **2541.js** | ~4 | Worker prompt builder (`GNI`) | orchestrator-core, worker-lifecycle, validation-contract, prompts-and-templates |
| **0087.js** | 8.1 | RPC schema definitions | worker-lifecycle, state-machine |
| **1234.js** | 8.0 | Tool parameter schemas | worker-lifecycle, handoff |
| **2575.js** | 9.9 | SessionController (`jtA`) | worker-lifecycle, spec-mode |

### Task Tool / Subagent Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **2522.js** | 5.5 | TaskCliExecutor: core Task tool | task-tool, parallel-runner, prompts-and-templates |
| **2521.js** | 11 | SubagentStreamProcessor: process runner | task-tool, parallel-runner |
| **2519.js** | 5.8 | OpenDroidManager: droid CRUD | task-tool, prompts-and-templates |
| **2518.js** | 8.5 | DroidValidator: metadata validation | task-tool |
| **2176.js** | 4.8 | Built-in droid definitions | validation-contract, task-tool, progress-tracker, parallel-runner |
| **1147.js** | ~1 | Task tool prompt template | parallel-runner |

### Spec Mode Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **1226.js** | ~1 | ExitSpecMode tool definition | spec-mode |
| **1235.js** | ~1 | ProposeMission tool definition | spec-mode |
| **3146.js** | ~20 | Tool execution hook: blocks tools in spec mode | spec-mode |
| **2532.js** | ~4 | ExitSpecMode executor: saves plan, opens editor | spec-mode |
| **3139.js** | ~6 | Confirmation UI factory | spec-mode |

### Prompt / Template Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **2526.js** | 5.6 | Skill injection engine (`WKH`) | prompts-and-templates |
| **2517.js** | 5.6 | SKILL.md parser (`km`) | prompts-and-templates |
| **2312.js** | 8.0 | Default system prompt (`hMH`) | prompts-and-templates |
| **2857.js** | 7.3 | System message builder (`QrI`) | prompts-and-templates, state-machine |
| **3399.js** | 28.6 | System prompt builder + context injection | orchestrator-core, spec-mode, prompts-and-templates |

### TUI / UI Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **3978.js** | ~30 | Mission status UI | orchestrator-core, progress-tracker, reports, cli-ui-bridge |
| **3985.js** | 7.7 | Worker status TUI component | worker-lifecycle |
| **3971.js** | 7.1 | Feature detail viewer | state-machine, feature-registry |
| **3973.js** | 6.6 | Handoff viewer + status filters | state-machine, handoff, feature-registry |
| **3977.js** | ~5 | Session preview UI | reports |
| **3975.js** | ~3 | Transcript reader | reports |
| **3970.js** | ~4 | React hook: file watching for live UI | state-machine, progress-tracker |
| **3983.js** | ~12 | TUI model selector | task-tool, parallel-runner |

### Infrastructure Modules

| Module | Size (KB) | Role | Referenced In |
|--------|-----------|------|--------------|
| **0279.js** | ~60 | Git scheduler + task queue (concurrency=2) | parallel-runner |
| **2362.js** | ~50 | LLM client / tool runner: `Promise.all` for parallel tools | parallel-runner |
| **0257.js** | ~22 | Model configuration registry: `parallelToolCalls` per model | parallel-runner |
| **0454.js** | ~8 | AWS SDK retry strategy (infra-level, not mission-level) | retry-policy |
| **0939.js** | ~28 | Settings service: specModeModel, specSaveDir | spec-mode |
| **0965.js** | ~6 | LocalDaemonClient: WebSocket to opendroidd | parallel-runner |
| **1237.js** | ~1 | dismiss-handoff-items tool | handoff, reports |

### Coverage Summary

- **Total unique modules referenced across all 15 reports**: ~50+
- **Core module cluster**: 2542, 2201, 2175, 2544, 1185, 0082 (appear in 6+ reports each)
- **90 module slots** specified in mission brief (6 per feature × 15 features): ~60 unique modules actually analyzed, with ~30 stale mappings substituted via grep-first discovery
- **Module substitution rate**: ~40% of assigned modules were stale; actual implementations found via keyword grep

---

## Cross-System Dependency Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SYSTEMS                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐│
│  │ 01-terminal-ui  │  │ 03-tool-agent-system  │  │ 04-desktop-gui  │  │ 05-infrastructure  │  │User  ││
│  │ (TUI)     │  │ (Tool)    │  │ (GUI)     │  │ (Infra)   │  │      ││
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘  └──┬───┘│
│        │              │              │              │           │    │
└────────┼──────────────┼──────────────┼──────────────┼───────────┼────┘
         │              │              │              │           │
    ┌────▼──────────────▼──────────────▼──────────────▼───────────▼──┐
    │                   Msample itemION SYSTEM (02-orchestration)                     │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                  MissionRunner (2542.js)                 │   │
    │  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │   │
    │  │  │ Spec Mode │  │ Feature  │  │  Worker Spawn/Monitor│  │   │
    │  │  │  Gate     │  │ Registry │  │  (daemon RPC)        │  │   │
    │  │  └──────────┘  └────┬─────┘  └──────────┬───────────┘  │   │
    │  │                      │                    │              │   │
    │  │  ┌───────────────────▼────────────────────▼───────────┐ │   │
    │  │  │          MissionFileService (2201.js)               │ │   │
    │  │  │  state.json │ features.json │ progress_log.jsonl   │ │   │
    │  │  └────────────────────────────────────────────────────┘ │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
    │  │  Handoff      │  │  Validation   │  │  CLI-UI Bridge      │ │
    │  │  Protocol     │  │  Contracts    │  │  (Event Emitter)    │ │
    │  │  (2539.js)    │  │  (2542+2201)  │  │  (v$ emitter)       │ │
    │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
    │         │                  │                      │             │
    │  ┌──────▼──────────────────▼──────────────────────▼──────────┐│
    │  │              Prompt Templates (2175.js)                    ││
    │  │  orchestrator │ worker │ scrutiny │ user-testing prompts  ││
    │  └───────────────────────────────────────────────────────────┘│
    │                                                                 │
    │  ┌────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
    │  │ Task Tool       │  │ Logging (0093.js)│  │ Retry Policy   │ │
    │  │ (2522+2521.js)  │  │ + Progress Log   │  │ (S9H in 2542)  │ │
    │  └────────────────┘  └─────────────────┘  └────────────────┘ │
    └─────────────────────────────────────────────────────────────────┘
         │              │              │              │           
    ┌────▼──────────────▼──────────────▼──────────────▼───────────┐
    │                    OPENDROIDD DAEMON                           │
    │  spawnWorkerSession │ loadSession │ interruptSession │ ...   │
    └─────────────────────────────────────────────────────────────┘
```

### Integration Points by Sister Mission

#### 01-terminal-ui (TUI Rendering)
- **Event consumption**: TUI subscribes to `project-notification` events (6 mission event types)
- **Status rendering**: Module 3978.js formats mission state, feature counts, progress logs
- **Worker timeline**: Module 3985.js reconstructs worker status from progress log entries
- **Handoff display**: Module 3973.js renders handoff fields with color-coded severity
- **File watching**: Module 3970.js polls `state.json`/`features.json`/`progress_log.jsonl` for live UI updates
- **Session preview**: Module 3977.js tails worker JSONL transcripts for real-time display

#### 03-tool-agent-system (Tool System)
- **`start_mission_run`**: Streaming tool that yields status updates (Module 2544.js)
- **`EndFeatureRun`**: Worker submission tool with Zod validation (Module 2539.js)
- **`dismiss-handoff-items`**: Orchestrator tool for resolving handoff items (Module 1237.js)
- **`ExitSpecMode`**: Plan approval tool (Module 1226.js)
- **`ProposeMission`**: Mission proposal tool (Module 1235.js)
- **`Task`**: Subagent invocation via `droid exec` CLI (Module 2522.js)
- **Skill injection**: Skill tool loads SKILL.md content into worker context (Module 2526.js)

#### 04-desktop-gui (GUI / Electron)
- **Daemon session schemas**: Module 0919.js bridges state to Electron app layer
- **Worker status data shape**: `soA()` composite status payload consumable by GUI components
- **Real-time notifications**: `mission_state_changed`, `mission_heartbeat`, `mission_worker_completed`
- **Spec plan editing**: `ideClient.openFile()` opens plan in editor on `proceed_edit`
- **Model selection UI**: Module 3983.js for orchestrator/worker/validator model config

#### 05-infrastructure (Infrastructure)
- **Daemon RPC layer**: `zX()` provides `spawnWorkerSession`, `loadSession`, `interruptSession`, `subscribeToSessionNotifications`
- **TCP health check**: Daemon reachability monitoring
- **Environment variables**: `OPENDROID_HOME_OVERRIDE`, `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS`
- **Session storage**: `{opendroidHome}/sessions/{cwdHash}/{sessionId}.jsonl`
- **ErrTracker integration**: Error tracking via breadcrumbs and exception capture
- **AWS SDK retry**: Standard + adaptive retry modes for API calls (Module 0454.js)
- **Model configuration**: Per-model settings including `parallelToolCalls`, `reasoningEffort`

---

## OpenDroid Port Roadmap

This section provides a concrete, prioritized implementation guide for porting the Mission System to OpenDroid.

### Phase 1: Foundation (P0: Must-Have)

#### 1.1 MissionFileService
**Modules to port**: 2201.js (~626 lines)
**Implementation steps**:
1. Create a `MissionFileService` class with async read/write methods
2. Implement file-based state persistence for `state.json`, `features.json`, `validation-state.json`
3. Implement `progress_log.jsonl` append-only logging with incremental read cache
4. Implement `handoffs/` directory management (write per-worker JSON, read summaries)
5. Add `getNextPendingFeature()`: returns first feature with `status === "pending"`
6. Add `isMilestoneImplementationComplete()`: filters validators, checks completion
7. Add `insertFeatureAtTop()`: array unshift for priority features
8. Add `normalizeFeature()`: default missing arrays, default status to pending
9. Use atomic file writes (write to temp, rename)
10. Emit change notifications on every mutation

#### 1.2 MissionRunner
**Modules to port**: 2542.js (~811 lines)
**Implementation steps**:
1. Create `MissionRunner` class with `start(abortSignal, resumeWorkerSessionId?)`
2. Implement `runLoop()` state machine: running → spawn → wait → check → repeat
3. Implement `spawnWorker()`: select model, create session, build bootstrap, send
4. Implement `waitForWorkerCompletion()`: 1s polling + timeout detection
5. Implement `checkMilestoneCompletionAndInjectValidation()`: idempotent via state
6. Implement `cleanupOrphanedWorker()`: reset stale sessions on startup
7. Implement `pause()`: set `isRunning = false`, interrupt worker
8. Implement `resumeWorker()`: load session, send continuation prompt
9. Feature re-queue function (`S9H`): reset `in_progress` → `pending` on failure
10. SIGINT handler for graceful shutdown

#### 1.3 Feature Registry Schema
**Modules to port**: 0082.js (~120 lines)
**Implementation steps**:
1. Define feature schema with 12 fields: id, description, status, skillName, preconditions, expectedBehavior, verificationSteps, fulfills, milestone, workerSessionIds, currentWorkerSessionId, completedWorkerSessionId
2. Define status enum: `pending | in_progress | completed | cancelled`
3. Define mission state enum: `initializing | running | paused | orchestrator_turn | completed`
4. Implement Zod-style validation (or equivalent runtime validation)
5. Ensure `features.json` is always valid after every mutation

#### 1.4 Handoff Protocol
**Modules to port**: 1234.js (~240 lines), 2539.js (~280 lines)
**Implementation steps**:
1. Define handoff JSON schema: salientSummary, whatWasImplemented, whatWasLeftUndone, verification, tests, discoveredIssues, skillFeedback
2. Validate `salientSummary`: 20-500 chars, 1-4 sentences, no newlines
3. Compute escalation: `returnToOrchestrator || hasIssues || hasUnfinishedWork`
4. Update feature status: success → completed, otherwise → pending
5. Write handoff JSON to per-worker file in `handoffs/` directory
6. Block mission resume if unaddressed items exist

### Phase 2: Quality Gates (P1: Should-Have)

#### 2.1 Validation Contract System
**Modules to port**: 2542.js (validation injection), 2175.js (validator prompts)
**Implementation steps**:
1. Define `validation-contract.md` format: assertion IDs grouped by feature area
2. Define `validation-state.json` schema: assertions map with pending/passed/failed status
3. Implement milestone completion detection (exclude validator skill names from counting)
4. Implement auto-injection: create scrutiny + user-testing features, insert at top
5. Add idempotency guard via `milestonesWithValidationPlanned` in state.json
6. Implement skip flags in model settings
7. Design synthesis.json schema for validation reports

#### 2.2 Spec Mode / Plan Approval
**Modules to port**: 3146.js (tool blocking), 1226.js (ExitSpecMode), 2532.js (executor)
**Implementation steps**:
1. Define `interactionMode` enum: `auto | spec`
2. Implement tool blocking: maintain readonly allowlist, block all others in spec mode
3. Implement plan approval flow: present plan → approve/cancel
4. Save approved plans to disk
5. On approval: transition to execution mode

#### 2.3 Retry / Failure Handling
**Modules to port**: 2542.js (S9H function), 2544.js (error messaging)
**Implementation steps**:
1. Implement feature re-queue: reset `in_progress` → `pending` on any failure
2. Log `worker_failed` to progress log with reason
3. Set state to `orchestrator_turn` for manual intervention
4. No automatic retry counter: delegate retry decision to orchestrator
5. Implement daemon health monitoring (periodic TCP check, 3-strike threshold)

### Phase 3: UX / Developer Experience (P2: Nice-to-Have)

#### 3.1 Event Notification System
**Modules to port**: 0084.js (~242 lines)
**Implementation steps**:
1. Define 6 event types with typed payloads
2. Implement EventEmitter-based notification bus
3. Emit on state changes, feature updates, progress entries, worker lifecycle
4. Heartbeat every 180s during active worker monitoring

#### 3.2 Worker Bootstrap Prompt Builder
**Modules to port**: 2541.js (~100 lines)
**Implementation steps**:
1. Build `GNI(missionDir, feature, sessionId)` function
2. Include feature JSON, skill invocation order, session rules
3. Differentiate validator vs implementation worker prompts
4. Include agent-guidelines.md content as context

#### 3.3 Logging Infrastructure
**Modules to port**: 0093.js (~100 lines), 2201.js (progress log)
**Implementation steps**:
1. Implement console logger with BH/gH/uH levels
2. Add message sanitization (strip sensitive data)
3. Optional ErrTracker breadcrumb integration
4. Structured JSONL progress log with typed entries
5. Incremental read cache for large log files

#### 3.4 Task Tool / Subagent Spawning
**Modules to port**: 2522.js (~150 lines), 2521.js (~394 lines)
**Implementation steps**:
1. Implement droid config loading from `.opendroid/droids/` and `~/.opendroid/droids/`
2. Validate droid metadata (name, tools, model)
3. Build command args for subagent execution
4. Spawn child process with NDJSON stdout parsing
5. Handle abort signal propagation
6. Implement depth tracking and session ID linking

---

## Key Architectural Insights

### 1. Sequential Feature Execution with Tool-Level Parallelism
The system deliberately limits feature-level parallelism to simplify state management. Only one worker runs at a time per mission. Parallelism exists at the tool level when the LLM emits multiple `Task` tool calls: these execute concurrently via `Promise.all()`. This hybrid approach balances simplicity with throughput.

### 2. File-Based State with Notification Events
All state is persisted to JSON/JSONL files in `missionDir`. There is no database. The `MissionFileService` reads/writes files directly and emits events on mutations. This design is simple and crash-resilient: the system can recover from any failure by reading the file state.

### 3. Delegated Retry (No Automatic Retry Counter)
Failed features are immediately escalated to the orchestrator with no retry counter or backoff strategy at the mission level. The orchestrator (LLM) has the intelligence to diagnose whether a failure is transient or fundamental, making manual retry more effective than blind automatic retry.

### 4. Prompt-Driven Validation
Validation reports are generated entirely by LLM prompts, not by code. The scrutiny and user-testing validator behaviors are encoded as multi-page prompt templates in Module 2175.js. This means validation logic can be updated without code changes: just prompt edits.

### 5. Daemon-Managed Workers (Not Child Processes)
Workers are not spawned as child processes. They are daemon sessions created via WebSocket RPC. This provides process isolation, session persistence, and independent lifecycle management. The tradeoff is dependency on the opendroidd daemon for all worker operations.

### 6. Idempotent Validation Injection
The `milestonesWithValidationPlanned` array in `state.json` ensures validation features are only injected once per milestone, even across pause/resume cycles. This prevents duplicate validation runs that would waste LLM tokens.

### 7. Handoff Gating as Quality Gate
The system blocks mission continuation if a worker's handoff contains unaddressed `discoveredIssues` or non-empty `whatWasLeftUndone`. This forces the orchestrator to either create follow-up features or explicitly dismiss items before the mission can proceed, preventing issues from being silently ignored.

---

## Validation Assertion Coverage

| Assertion ID | Description | Fulfilled By | Status |
|-------------|-------------|-------------|--------|
| VAL-MIS-001 | Orchestrator Core Report | mission-orchestrator-core | ✓ All 15 reports verified present |
| VAL-MIS-002 | Worker Lifecycle Report | mission-worker-lifecycle | ✓ Documents spawn, monitor, crash recovery |
| VAL-MIS-003 | State Machine Report | mission-state-machine | ✓ Dual-layer state machine documented |
| VAL-MIS-004 | Validation Contract Report | mission-validation-contract | ✓ Three-file artifact model documented |
| VAL-MIS-005 | Handoff Protocol Report | mission-handoff | ✓ Full schema with Zod validation |
| VAL-MIS-006 | Task Tool Report | mission-task-tool | ✓ API, prompt construction, droid selection |
| VAL-MIS-007 | Feature Registry Report | mission-feature-registry | ✓ 12-field schema, ordering algorithm |
| VAL-MIS-008 | Progress Tracker Report | mission-progress-tracker | ✓ Milestone detection, validator injection |
| VAL-MIS-009 | Spec Mode Report | mission-spec-mode | ✓ Entry/exit, plan approval flow |
| VAL-MIS-010 | Prompts & Templates Report | mission-prompts-and-templates | ✓ 4-layer prompt stack |
| VAL-MIS-011 | Reports System Report | mission-reports | ✓ Validation report generation pipeline |
| VAL-MIS-012 | Parallel Runner Report | mission-parallel-runner | ✓ Feature sequential, tool parallel |
| VAL-MIS-013 | Retry Policy Report | mission-retry-policy | ✓ Delegated retry, S9H re-queue |
| VAL-MIS-014 | Logging Report | mission-logging | ✓ Three-tier logging architecture |
| VAL-MIS-015 | CLI-UI Bridge Report | mission-cli-ui-bridge | ✓ 6 event types, EventEmitter pattern |
| VAL-MIS-CROSS-001 | Aggregate Report | mission-system (this report) | ✓ End-to-end lifecycle narrative |
## Known Gaps

- **Module 3978.js full analysis**: Only partially read (~200 of 1045 lines). Contains full TUI rendering for mission status panel. 01-terminal-ui territory.
- **Module 1185.js full session lifecycle**: Only orchestrator upgrade/downgrade and spawn functions analyzed. Full 1823-line module includes session persistence, token tracking, cloud sync: 05-infrastructure territory.
- **Module 3399.js agent streaming**: Only context injection portion analyzed. Full 1303 lines contain LLM streaming, tool execution, message formatting: 03-tool-agent-system territory.
- **Cloud session sync** in SessionService: Not explored. Likely 05-infrastructure territory.
- **Worker transcript management**: `worker-transcripts.jsonl` write path identified but not fully traced.
- **Constants `GUf`/`QUf`** (inactivity/spawn timeouts): Referenced but exact definitions not verified from source.
- **`dismiss-handoff-items` tool** full handler: Only registration found (Module 1237.js). Execution handler not traced.
- **Live validation**: This is a read-only RE mission. No code was written. All findings are observational.
