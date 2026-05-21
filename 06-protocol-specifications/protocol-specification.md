# Protocol Specification: OpenDroid: Unified Cross-System Reference

## Status: DEFINITIVE (synthesis of 7 cross-validated protocol reports)

This document synthesizes all 7 individual protocol area reports into a unified specification for building OpenDroid. It documents cross-system data flows, invariants that span protocol boundaries, and the complete lifecycle from user interaction to mission completion.

### Source Reports

| # | Protocol Area | Report | Status |
|---|---------------|--------|--------|
| 1 | Handoff | `protocol-handoff.md` | DEFINITIVE |
| 2 | Feature State Machine | `protocol-feature-state.md` | DEFINITIVE |
| 3 | Validation System | `protocol-validation.md` | DEFINITIVE |
| 4 | Mission Lifecycle | `protocol-mission-lifecycle.md` | DEFINITIVE |
| 5 | Session Format | `protocol-session.md` | DEFINITIVE |
| 6 | Configuration System | `protocol-config.md` | DEFINITIVE |
| 7 | IPC & Daemon | `protocol-ipc.md` | PARTIAL (heavy RE, minimal runtime) |

### Core Principle

**Observed protocol samples override architecture notes.** All 7 reports were cross-validated against `protocol-samples/runtime-findings.md`. Discrepancies are documented in each individual report and summarized in §6.

---

## 1. Complete Cross-System Data Flow

### 1.1 End-to-End Mission Execution Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER (CLI / Desktop GUI)                          │
│  User types mission request → Chat session → Mission proposal generated    │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │ User accepts mission
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [CONFIG] Configuration Resolution                                          │
│  Precedence: builtin → org → folder → project → user → env → CLI flags     │
│  Loads: settings.json + config.json → merge → resolved config               │
│  Loads: feature-flags.json → gate features                                  │
│  Loads: model-settings.json → select worker & validator models              │
│  See: protocol-config.md §2-3                                               │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [Msample itemION] Mission Creation & Initialization                                │
│  state.json created → state: "initializing"                                 │
│  Mission directory populated: features.json, validation-state.json,          │
│    validation-contract.md, mission.md, agent-guidelines.md, working_directory.txt,    │
│    model-settings.json, runtime-custom-models.json                          │
│  progress_log.jsonl: mission_accepted event                                 │
│  state.json → state: "running"                                              │
│  See: protocol-mission-lifecycle.md §1, §3                                  │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [SESSION] Orchestrator Session Created                                     │
│  Session JSONL created with session_start event                             │
│  Settings sidecar: interactionMode="agi", tags with mission-session metadata│
│  sessions-index.json updated with entry for orchestrator session             │
│  See: protocol-session.md §1-3                                              │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [FEATURE] Orchestrator Selects Next Feature                                │
│  getNextPendingFeature() → first feature with status === "pending"          │
│  Feature: pending → in_progress (currentWorkerSessionId set)                │
│  progress_log.jsonl: worker_selected_feature event                          │
│  See: protocol-feature-state.md §State Machine                              │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [SESSION+IPC+CONFIG] Worker Session Spawned                                │
│  Orchestrator calls Task tool → daemon.initialize_session RPC               │
│  Worker session created with:                                               │
│    callingSessionId → orchestrator session UUID                             │
│    callingToolUseId → Task tool call ID                                     │
│    model → from model-settings.json.workerModel                             │
│    reasoningEffort → from model-settings.json.workerReasoningEffort         │
│  sessions-index.json updated with worker entry + tags ["exec","subagent"]   │
│  progress_log.jsonl: worker_started event (with spawnId)                    │
│  See: protocol-session.md §5, protocol-mission-lifecycle.md §4,             │
│       protocol-ipc.md §Daemon RPC                                           │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [SESSION] Worker Executes                                                  │
│  Worker invokes mission-worker-base skill (startup)                         │
│  Worker invokes assigned skill (implementation)                             │
│  All tool calls logged in session JSONL as message events                   │
│  Worker reads features, reads code, writes code, runs tests, commits        │
│  See: protocol-session.md §2                                                │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [HANDOFF] Worker Calls EndFeatureRun                                       │
│  Handoff JSON written to handoffs/{timestamp}__{featureId}__{sessionId}.json│
│  worker-transcripts.jsonl: skeleton entry written                           │
│  progress_log.jsonl: worker_completed event (with embedded handoff)         │
│  Feature status updated based on handoff result                             │
│  Session interrupted via daemon RPC                                         │
│  See: protocol-handoff.md §State Machine,                                   │
│       protocol-mission-lifecycle.md §4.5                                    │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
                         ┌────────────┴────────────┐
                         │ Handoff Result?          │
                         └────────────┬────────────┘
                                      │
              ┌───────────────────────┼──────────────────────┐
              │                       │                      │
    Clean success          Issues found              Worker failure
    returnToOrchestrator:  discoveredIssues or       (crash/timeout)
    false                  whatWasLeftUndone         │
              │              │                       │
              ▼              ▼                       ▼
    Feature → completed   Feature stays pending    Feature → pending
    State → running       State → orchestrator_    State → orchestrator_
    Next feature picked     turn                    turn
                           │                       │
                           ▼                       ▼
                    ┌──────────────────────────────────┐
                    │ Orchestrator reviews handoff      │
                    │ • Create fix features?            │
                    │ • Update descriptions?            │
                    │ • Dismiss items?                  │
                    │ Then: start_mission_run → running │
                    └──────────────────────────────────┘

              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [FEATURE+VALIDATION] Milestone Implementation Complete Check               │
│  After each worker_completed:                                               │
│    checkMilestoneCompletionAndInjectValidation() fires                      │
│    Conditions: feature has milestone + all impl features complete +          │
│               validation not yet planned                                    │
│  See: protocol-validation.md §4, protocol-feature-state.md §Milestone       │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │ All impl features complete
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [VALIDATION+HANDOFF] Phase 1: Scrutiny Validation                         │
│  Auto-inject: scrutiny-validator-{milestone} feature at top of features.json│
│  progress_log.jsonl: milestone_validation_triggered event                   │
│  Worker spawned for scrutiny-validator skill                                │
│    → Runs hard gate: test, typecheck, lint                                 │
│    → Spawns scrutiny-feature-reviewer subagents (one per feature)           │
│    → Each subagent is a separate session (callingSessionId → validator)     │
│    → Writes synthesis.json to .opendroid/validation/{milestone}/scrutiny/     │
│    → EndFeatureRun with returnToOrchestrator: true                          │
│  See: protocol-validation.md §5.1                                           │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                              ┌───────┴───────┐
                              │ Result?       │
                              └───────┬───────┘
                                      │
                         Pass         │          Fail
                           │          │            │
                           ▼          │            ▼
                           │          │   Orchestrator creates fix features
                           │          │   Fix features executed → re-validate
                           │          │   (incremental: only failures reviewed)
                           │          │
                           ▼          │
┌─────────────────────────────────────────────────────────────────────────────┐
│  [VALIDATION+HANDOFF] Phase 2: User-Testing Validation                     │
│  Auto-inject: user-testing-validator-{milestone} feature                    │
│  Worker spawned for user-testing-validator skill                            │
│    → Collects testable assertions from features[].fulfills                  │
│    → Reads validation-state.json for "pending" assertions only              │
│    → Spawns user-testing-flow-validator subagents (parallel)                │
│    → Each subagent tests assertions through real UI/API surfaces            │
│    → Updates validation-state.json: assertion status → "passed"/"failed"    │
│    → Writes synthesis.json to .opendroid/validation/{milestone}/user-testing/ │
│    → EndFeatureRun with returnToOrchestrator: true                          │
│  See: protocol-validation.md §5.2                                           │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                              ┌───────┴───────┐
                              │ Result?       │
                              └───────┬───────┘
                                      │
                         Pass         │          Fail
                           │          │            │
                           ▼          │            ▼
                           │          │   Orchestrator creates fix features
                           │          │   Failed assertions reset → "pending"
                           │          │   Fix features executed → re-validate
                           │          │   (incremental: only failed re-tested)
                           │          │
                           ▼          │
┌─────────────────────────────────────────────────────────────────────────────┐
│  [VALIDATION] Milestone Sealed                                              │
│  All milestone assertions → "passed" in validation-state.json               │
│  No new features can be added to sealed milestone                           │
│  Continue to next milestone's features                                      │
│  See: protocol-validation.md §6                                             │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  [Msample itemION] Mission Complete                                                 │
│  All features completed or cancelled                                        │
│  All assertions in validation-state.json → "passed"                         │
│  state.json → state: "completed"                                            │
│  progress_log.jsonl: final events logged                                    │
│  See: protocol-mission-lifecycle.md §1                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow Connections Between Protocol Areas

This table shows how data flows between protocol areas during mission execution:

| Data Artifact | Written By | Read By | Protocol Areas |
|---------------|-----------|---------|----------------|
| `features.json` | Orchestrator (planning) | MissionRunner (selection), Worker (bootstrap), Validation (assertion mapping) | Mission, Feature, Validation |
| `state.json` | MissionRunner | Orchestrator, MissionRunner (resume) | Mission |
| `validation-state.json` | User-testing-validator | Orchestrator (completion gate), User-testing-validator (re-run scoping) | Validation |
| `validation-contract.md` | Orchestrator (planning) | User-testing-validator (assertion definitions), Workers (feature context) | Validation, Mission |
| `progress_log.jsonl` | MissionRunner (events) | MissionRunner (completion polling), Orchestrator (history), TUI (display) | Mission, Handoff |
| `handoffs/*.json` | EndFeatureRun tool | Orchestrator (review), MissionRunner (gating) | Handoff, Mission |
| `worker-transcripts.jsonl` | EndFeatureRun tool | Orchestrator (audit trail) | Handoff, Mission |
| `sessions-index.json` | Session manager | Session reader, Orchestrator (session hierarchy) | Session |
| `*.settings.json` | Session manager | Session resumption, provider routing | Session, Config |
| `model-settings.json` | Orchestrator (mission creation) | MissionRunner (worker model selection) | Config, Mission |
| `settings.json` | Config loader (user edit) | All components (model resolution, plugin enablement) | Config |
| `feature-flags.json` | Remote API (org) | Feature gating (model availability, feature toggles) | Config |

---

## 2. Cross-System Invariants

These invariants span multiple protocol areas and MUST be maintained by any correct implementation:

### 2.1 Invariant 1: Every Worker Session Has a callingSessionId

**Protocol Areas:** Session, Mission, Feature

**Statement:** Every worker session created during mission execution has a `callingSessionId` field in `sessions-index.json` pointing to the orchestrator's session UUID. Similarly, every subagent session spawned by a worker has a `callingSessionId` pointing to the worker's session UUID.

**Evidence:**
- Runtime: `sessions-index.json` entries with `"tags": [{"name": "exec"}, {"name": "subagent"}]` have `callingSessionId` and `callingToolUseId` fields (protocol-session.md §4.3)
- RE: `daemon.initialize_session` accepts `callingSessionId` and `callingToolUseId` parameters (protocol-ipc.md)
- Feature: `workerSessionIds[]` in features.json tracks all sessions that worked on a feature (protocol-feature-state.md §Field Documentation)

**Enforcement:** The daemon session creation RPC requires `callingSessionId` for worker sessions. The session manager populates it in `sessions-index.json`.

**Importance:** This invariant enables session hierarchy reconstruction, which the orchestrator uses for worker monitoring and transcript reading. Without it, you cannot trace which worker belongs to which mission.

### 2.2 Invariant 2: Every Handoff commitId References a Git Commit Touching the Feature's Files

**Protocol Areas:** Handoff, Feature, Mission

**Statement:** When a worker calls `EndFeatureRun` with `successState: "success"`, the `commitId` field in the handoff must reference a real git commit that exists in the repository. The system validates this by checking `validatorsPassed: true` alongside the commit.

**Evidence:**
- Runtime: All successful handoffs contain valid commit hashes (e.g., `"<handoff-hash>"`) (protocol-handoff.md §Example 1)
- RE: The EndFeatureRun tool validates `commitId` is required when `successState === "success"` (protocol-handoff.md §EndFeatureRun Tool Parameters)
- Feature: The feature's `workerSessionIds` array tracks which sessions created commits for this feature (protocol-feature-state.md §Session Tracking)

**Enforcement:** The EndFeatureRun tool schema requires `commitId` on success. The handoff gating system (2544.js) blocks `start_mission_run` if the commit cannot be found.

**Importance:** Without this invariant, the orchestrator cannot verify that work was actually committed, and validation would be testing stale code.

### 2.3 Invariant 3: Every Feature Status Change Is Logged in progress_log.jsonl

**Protocol Areas:** Mission, Feature, Handoff

**Statement:** Every transition in a feature's lifecycle is recorded as an event in `progress_log.jsonl`:
- `pending` → `in_progress`: logged as `worker_selected_feature` + `worker_started`
- `in_progress` → `completed`: logged as `worker_completed` (with embedded handoff)
- `in_progress` → `pending` (failure): logged as `worker_failed` or implied by `worker_paused` → `mission_paused`
- Feature creation (dynamic): logged as part of orchestrator's tool calls

**Evidence:**
- Runtime: 9 distinct event types observed in progress_log.jsonl covering all state transitions (protocol-mission-lifecycle.md §2)
- RE: `MissionRunner.runLoop()` logs events for every state change (protocol-mission-lifecycle.md §4)
- Feature: The progress log is the source of truth for worker completion detection (1-second polling interval)

**Enforcement:** The MissionRunner's event emission is integrated into every state transition function. The log is append-only (JSONL format).

**Importance:** The progress log is the audit trail for the entire mission. It is the mechanism by which the MissionRunner detects worker completion (polls the log every 1000ms). Without it, the orchestrator cannot know when workers finish.

### 2.4 Invariant 4: Every Assertion ID Appears in Exactly One Feature's fulfills Array

**Protocol Areas:** Validation, Feature

**Statement:** Each assertion ID in `validation-contract.md` (e.g., `VAL-HANDOFF-001`) must be claimed by exactly one feature's `fulfills` array in `features.json`. This is a coverage gate: unclaimed assertions indicate a planning gap, and duplicate claims indicate a conflict.

**Evidence:**
- Runtime: `validation-state.json` contains 26 assertions, each mapped to a single feature's `fulfills` (protocol-validation.md §1)
- RE: Orchestrator prompt explicitly states "Each assertion ID must appear in exactly one feature's `fulfills` across the entire features.json" (protocol-feature-state.md §Dynamic Feature Creation)
- Validation: The user-testing-validator collects assertions by reading features' `fulfills` arrays

**Enforcement:** Orchestrator planning ensures no duplicates. Validation fails if assertions are unclaimed.

**Importance:** Without this invariant, the validation system cannot determine which feature's implementation should be tested for each assertion, leading to gaps or duplicate testing.

### 2.5 Invariant 5: Validation Always Returns Control to Orchestrator

**Protocol Areas:** Validation, Handoff, Mission

**Statement:** Both scrutiny-validator and user-testing-validator features always complete with `returnToOrchestrator: true` in their handoffs. The orchestrator must always review validation results before proceeding, even on success.

**Evidence:**
- Runtime: Validator handoffs always have `returnToOrchestrator: true` (protocol-handoff.md §returnToOrchestrator Semantics)
- RE: Validator feature creation sets `returnToOrchestrator: true` as the expected pattern (protocol-validation.md §8)
- Mission: State transitions to `orchestrator_turn` after validator completion

**Enforcement:** The validator skill procedures explicitly set this flag. The system does not force it (unlike discoveredIssues forcing), but it is the required behavior.

**Importance:** This ensures the orchestrator always has the opportunity to create fix features or make decisions based on validation outcomes before continuing to the next milestone.

### 2.6 Invariant 6: Failed Features Reset to Pending (Not "failed")

**Protocol Areas:** Feature, Mission

**Statement:** There is no `"failed"` feature status. When a worker fails (crash, timeout, error), the feature is reset to `"pending"` via the `S9H()` function, making it eligible for re-execution.

**Evidence:**
- Runtime: Only `pending`, `in_progress`, `completed`, and `cancelled` statuses observed (protocol-feature-state.md §FeatureStatus Enum)
- RE: Zod schema defines only 4 states. The `S9H()` function explicitly resets to `"pending"` (protocol-feature-state.md §State Machine)
- Mission: The delegated retry policy means the orchestrator decides whether to retry, not the feature state machine

**Enforcement:** The `S9H()` function is the only code path for failure handling. It always sets status to `"pending"`.

**Importance:** This design choice means features are always retryable. There is no terminal "failed" state: only the orchestrator can decide to stop retrying (by cancelling the feature or the mission).

### 2.7 Invariant 7: Feature Execution Is Sequential, Not Parallel

**Protocol Areas:** Mission, Feature, Session

**Statement:** Workers execute one at a time in feature array order. There is no feature-level parallelism. The `MissionRunner.runLoop()` picks the first `pending` feature and waits for completion before picking the next.

**Evidence:**
- Runtime: Progress log shows sequential `worker_started` → `worker_completed` events with no overlap (protocol-mission-lifecycle.md §7.1)
- RE: Module 1236.js states "The tool call remains open while the mission runner executes workers sequentially" (protocol-mission-lifecycle.md §7.1)
- Feature: `getNextPendingFeature()` returns the first `pending` feature in array order

**Enforcement:** The runLoop is a single-threaded sequential loop. One worker at a time.

**Importance:** This simplifies the implementation significantly: no concurrent file access issues, no feature ordering conflicts, no race conditions in features.json updates.

---

## 3. Protocol Area Cross-Reference Matrix

### 3.1 Protocol Area Responsibilities

| Protocol Area | Primary Files | Key Operations | System Component |
|---------------|--------------|----------------|------------------|
| **Mission Lifecycle** | `state.json`, `progress_log.jsonl`, `worker-transcripts.jsonl` | State machine, event logging, worker spawn/monitor | MissionRunner |
| **Feature State** | `features.json` | Feature CRUD, state transitions, priority ordering, dynamic creation | MissionFileService |
| **Validation** | `validation-state.json`, `validation-contract.md`, `synthesis.json` | Assertion tracking, two-phase validation, milestone sealing | ValidationRunner |
| **Handoff** | `handoffs/*.json` | Worker completion reporting, orchestrator gating | EndFeatureRun tool |
| **Session** | `sessions/{slug}/*.jsonl`, `*.settings.json`, `sessions-index.json` | Chat history, provider management, session hierarchy | SessionManager |
| **Configuration** | `settings.json`, `config.json`, `feature-flags.json`, `model-settings.json` | Config resolution, feature flags, plugin management | ConfigLoader |
| **IPC** | (in-memory) | Electron IPC, daemon JSON-RPC, auth flow | Electron main + daemon |

### 3.2 Cross-Area Dependencies

```
Config ──────────► Mission (model selection, feature flags)
   │                   │
   │                   ▼
   │              Session (session creation, provider selection)
   │                   │
   │                   ▼
   │              Feature (feature selection from features.json)
   │                   │
   │                   ▼
   │              Session (worker session spawned with callingSessionId)
   │                   │
   │                   ▼
   │              Handoff (worker completion, commitId)
   │                   │
   │                   ▼
   │              Feature (status update, next feature selection)
   │                   │
   │                   ▼
   │              Validation (milestone completion check)
   │                   │
   │                   ▼
   │              Session (validator session spawned)
   │                   │
   │                   ▼
   │              Validation (scrutiny → user-testing)
   │                   │
   │                   ▼
   └────────────► Mission (state update, milestone sealed or orchestrator_turn)
```

### 3.3 Shared Data Elements

These data elements appear in multiple protocol areas:

| Data Element | Used In | Format |
|-------------|---------|--------|
| `workerSessionId` (UUID) | Session, Feature, Handoff, Mission (progress_log) | UUID string |
| `featureId` (kebab-case) | Feature, Handoff, Mission (progress_log), Validation | String |
| `milestone` (string) | Feature, Mission, Validation, Handoff | String |
| `missionId` (`mis_` prefix) | Mission (state.json), Session (tags metadata) | String |
| `callingSessionId` (UUID) | Session (index), IPC (daemon RPC params) | UUID string |
| `model` (model ID) | Config (settings.json), Session (*.settings.json), Mission (model-settings.json) | String |
| `commitId` (SHA) | Handoff, Mission (progress_log worker_completed) | String (40-char hex) |
| `assertionId` (`VAL-*`) | Feature (fulfills), Validation (validation-state.json, contract) | String |
| `successState` enum | Handoff, Mission (progress_log worker_completed) | `"success" \| "partial" \| "failure"` |
| `status` enum (feature) | Feature (features.json), Mission (completion checks) | `"pending" \| "in_progress" \| "completed" \| "cancelled"` |

---

## 4. Key Architectural Patterns

### 4.1 File-Based State with JSONL Event Sourcing

The entire mission system uses files as the persistence layer:

- **JSON files** for current state (`state.json`, `features.json`, `validation-state.json`)
- **JSONL files** for append-only event logs (`progress_log.jsonl`, `worker-transcripts.jsonl`, `*.jsonl` sessions)
- **JSON handoff files** for worker completion reports

No database is used. All state is file-based. This simplifies implementation but requires careful file locking or sequential access patterns.

### 4.2 Sequential Feature Execution with FIFO-Priority

Features execute one at a time in array order. New features (fixes, validators) are inserted at the top for highest priority. Completed/cancelled features are moved to the bottom. This implements a FIFO-with-priority queue using simple array reordering.

### 4.3 Delegated Retry (No Automatic Retry)

Failed workers do NOT automatically retry. Every failure escalates to the orchestrator LLM for decision. The orchestrator can:
1. Re-run the same feature (just calls `start_mission_run`)
2. Create fix features and then re-run
3. Cancel the feature
4. Abort the mission

The `workerSessionIds[]` array in features.json retains the full retry history.

### 4.4 Two-Phase Validation with Incremental Re-Runs

Validation is a two-phase process:
1. **Scrutiny** (code quality): test + typecheck + lint + code review
2. **User Testing** (behavioral): assertion testing through real surfaces

Both phases support **incremental re-runs**: on retry, they only validate what previously failed, reading the prior `synthesis.json` to scope work. The `round` field tracks validation iterations.

### 4.5 Session Hierarchy for Traceability

Sessions form a parent-child tree via `callingSessionId`:
```
Orchestrator Session
├── Worker Session 1 (callingSessionId = orchestrator)
│   └── Subagent Session (callingSessionId = worker1)
├── Worker Session 2 (callingSessionId = orchestrator)
└── Validator Session (callingSessionId = orchestrator)
    └── Flow Validator Subagent (callingSessionId = validator)
```

This enables:
- Tracing which worker belongs to which mission
- Reading worker transcripts from the orchestrator
- Understanding the full execution tree

### 4.6 Dual Configuration Formats

Two config formats coexist:
- `config.json` (snake_case, legacy): custom models only
- `settings.json` (camelCase, primary): all settings including custom models

The system reads both and reconciles via the merge algorithm. OpenDroid should normalize to a single format.

### 4.7 Handoff as the Core Communication Mechanism

The handoff JSON is the primary communication mechanism between workers and the orchestrator. It flows through three persistence points:
1. **Handoff file** (`handoffs/*.json`): persistent record
2. **Progress log** (`worker_completed` event with embedded handoff): timeline event
3. **Worker transcript** (`worker-transcripts.jsonl` skeleton): audit trail

The orchestrator consumes handoffs via `workerHandoffs` array on `start_mission_run`, with `lastReviewedHandoffCount` tracking progress.

---

## 5. Critical File Format Quick Reference

### 5.1 Mission Directory Structure

```
~/.opendroid/missions/{uuid}/
├── state.json                 # Mission state machine (6 possible states)
├── features.json              # Feature array (FIFO-priority queue)
├── validation-state.json      # Assertion status map (pending/passed/failed)
├── validation-contract.md     # Behavioral assertions (VAL-{CAT}-{NUM})
├── mission.md                 # Mission description
├── agent-guidelines.md                  # Agent guidance and boundaries
├── model-settings.json        # Per-mission model selection
├── runtime-custom-models.json # Snapshotted model definitions
├── working_directory.txt      # Project path
├── progress_log.jsonl         # Append-only event stream (9 event types)
├── worker-transcripts.jsonl   # Full tool call audit trail
├── handoffs/                  # Worker completion reports
│   └── {ts}__{featureId}__{sessionId}.json
├── contract-work/             # Validation surface descriptions
└── evidence/                  # Test evidence (screenshots, etc.)
```

### 5.2 Session Directory Structure

```
~/.opendroid/sessions/{cwd-slug}/
├── {session-uuid}.jsonl           # Message transcript
└── {session-uuid}.settings.json   # Session config sidecar
```

### 5.3 Configuration Directory Structure

```
~/.opendroid/
├── settings.json              # Primary user settings (camelCase)
├── config.json                # Legacy custom models (snake_case)
├── feature-flags.json         # Org feature flags (50+ flags)
├── computer.json              # Machine registration
├── background-processes.json  # Running processes
├── changelog.json             # CLI version tracking
├── sessions-index.json        # Global session metadata (52 entries)
├── history.json               # Chat history (440 entries)
├── plugins/
│   ├── installed_plugins.json # Plugin registry
│   └── known_marketplaces.json# Marketplace sources
└── droids/
    └── {name}.md              # YAML frontmatter + Markdown prompt
```

---

## 6. Discrepancies Summary

All discrepancies between architecture notes and observed protocol samples are documented in individual protocol reports. This section provides a cross-area summary of the most significant ones:

### 6.1 High-Impact Discrepancies

| # | Area | Discrepancy | Resolution |
|---|------|------------|------------|
| 1 | Mission | `missionId` format: RE expected UUID, runtime shows `mis_` prefix + hex | Runtime is authoritative. Directory UUID ≠ missionId |
| 2 | Feature | No `"failed"` status exists. RE and runtime both confirm 4 states only | Failed features always reset to `"pending"` |
| 3 | Config | Two config formats coexist (snake_case + camelCase) | config.json is legacy; settings.json is primary |
| 4 | IPC | 27 IPC channels + 35 daemon RPC methods are analysis-only, no runtime traces | IPC is in-memory; file artifacts cannot confirm. Trust RE |
| 5 | Session | Directory naming: RE says hash, runtime shows path encoding | Runtime is correct: simple character replacement |
| 6 | Handoff | `validatorsPassed` in progress_log but NOT in handoff file | Different persistence formats for same data |
| 7 | Validation | `"blocked"` collapsed to `"failed"` in validation-state.json | Runtime would show `"failed"` even for blocked assertions |

### 6.2 Total Discrepancy Count by Area

| Protocol Area | Discrepancy Count | Key Finding |
|---------------|------------------|-------------|
| Handoff | 12 | skillFeedback optional, validatorsPassed location, missionId format |
| Feature State | 9 | No "failed" state, completedWorkerSessionId unused, validation injection order |
| Validation | 8 | "blocked"=failed, "skipped" status unclear, synthesis.json unconfirmed in runtime |
| Mission Lifecycle | 11 | missionId format, spawnId/validatorsPassed runtime-only, state.json fields gap |
| Session | 15 | Directory naming, tool result role normalization, OpenAI reasoning format |
| Config | 8 | Dual config formats, feature flags schema, droid-vs-skill distinction |
| IPC | 9 | Almost entirely analysis-only, minimal runtime confirmation possible |
| **Total** | **72** | |

---

## 7. OpenDroid Implementation Priority

Based on cross-system analysis, these implementation components should be built in order:

### Phase 1: Foundation
1. **ConfigLoader**: Read/write settings.json, resolve precedence chain, merge algorithm
2. **SessionManager**: JSONL session files, settings sidecars, sessions-index.json
3. **MissionFileService**: File CRUD for mission directory (features.json, state.json, etc.)
4. **Daemon**: JSON-RPC 2.0 server with session management RPCs

### Phase 2: Execution Engine
5. **MissionRunner**: State machine (6 states), runLoop with 1s polling, pause/resume
6. **FeatureStateMachine**: 4 states, FIFO-priority ordering, S9H failure reset
7. **EndFeatureRun**: Handoff schema validation, file persistence, progress log event emission
8. **ProgressLog**: Append-only JSONL with 9+ event types

### Phase 3: Validation System
9. **ValidationInjector**: Auto-injection on milestone completion, idempotency guard
10. **ScrutinyValidator**: Hard gate (test/typecheck/lint) + subagent code review
11. **UserTestingValidator**: Assertion testing through real surfaces, validation-state.json updates
12. **MilestoneSealer**: Prevent scope creep after validation passes

### Phase 4: GUI & IPC
13. **Electron IPC**: 27 channels (15 handle + 6 on + 6 push), preload bridge
14. **Daemon Auto-Spawn**: TCP probe → spawn → exponential backoff → ready
15. **Auth Flow**: OAuth integration with deep link callback

---

## 8. Key Metrics from Observed Protocol Samples

| Metric | Value | Source |
|--------|-------|--------|
| Total sessions observed | 52 | sessions-index.json |
| Total chat history entries | 440 | history.json |
| Total mission assertions (SampleApp) | 118 | validation-state.json |
| Total mission assertions (this mission) | 26 | validation-state.json |
| Total handoff files observed | 11 | handoffs/ directory |
| Total progress event types observed | 9 | progress_log.jsonl |
| Total IPC channels (RE) | 27 | preload.js analysis |
| Total daemon RPC methods (RE) | 35+ | 3754.js + 3755.js |
| Feature flags observed | 50+ | feature-flags.json |
| Total discrepancies documented | 72 | All 7 protocol reports |
| Config precedence levels | 11 | Module 0267 |
| Feature states | 4 (pending, in_progress, completed, cancelled) | Analysis + Samples |
| Mission states | 6 (awaiting_input, initializing, running, paused, orchestrator_turn, completed) | Analysis + Samples |
| Validation phases | 2 (scrutiny → user-testing) | Analysis + Samples |
| Max report length | 800 lines | Context safety rule |

---

## 9. Open Questions Across Protocol Areas

These questions arise from cross-system analysis and were not fully answered by individual reports:

1. **State persistence gap**: `state.json` in runtime contains only 6 fields, but RE code tracks many more (currentFeatureId, completedFeatures, milestonesWithValidationPlanned). Where are these tracked: in memory only, or in a separate file?

2. **Concurrent milestone validation**: Can two milestones be in validation simultaneously? The sequential feature execution model suggests no, but the `milestonesWithValidationPlanned` set could theoretically track multiple milestones.

3. **Max retry circuit breaker**: No evidence of a maximum retry count for features. The `workerSessionIds[]` array grows unboundedly. Is there a limit?

4. **Handoff file cleanup**: No evidence of old handoff files being pruned. Do they accumulate indefinitely?

5. **Session compaction during missions**: Compaction was never triggered during observed runtime, but the mechanism exists (auto-compaction at 250K tokens). How does compaction interact with worker sessions that need full context?

6. **Droid model resolution**: When a droid specifies `model: "inherit"`, does it inherit from the spawning session's model or from the mission's model-settings.json? This affects validation worker subagents.

7. **Cloud sync interaction**: RE documents cloud sync endpoints for sessions. How do local mission files interact with remote state? Are missions synced?

---

## Appendix A: Assertion Coverage Map

| Validation Assertion | Protocol Area | Feature |
|---------------------|---------------|---------|
| VAL-CROSS-001 | Cross-Area | protocol-synthesis (this feature) |
| VAL-CROSS-002 | Cross-Area | protocol-synthesis (this feature) |
| VAL-CROSS-003 | Cross-Area | protocol-synthesis (this feature) |
| VAL-HANDOFF-001 to 004 | Handoff | protocol-handoff |
| VAL-FEATURE-001 to 003 | Feature State | protocol-feature-state |
| VAL-VALID-001 to 003 | Validation | protocol-validation |
| VAL-Msample itemION-001 to 004 | Mission Lifecycle | protocol-mission-lifecycle |
| VAL-SESSION-001 to 003 | Session | protocol-session |
| VAL-CONFIG-001 to 003 | Configuration | protocol-config |
| VAL-IPC-001 to 003 | IPC | protocol-ipc |

**Total: 26 assertions across 7 areas + 1 synthesis area**
