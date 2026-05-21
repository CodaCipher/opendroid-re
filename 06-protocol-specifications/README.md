# Protocol Specifications

Sanitized protocol specifications for long-running autonomous coding agent systems. These notes focus on state files, session records, handoff data, validation flow, and IPC boundaries used by multi-day CLI-native agent runtimes.

## Core Rule

**Observed protocol samples override inference.** When sanitized examples disagree with code-derived notes, the observed protocol shape is treated as the stronger signal. Discrepancies are documented with ⚠️ markers.

## Reports

| # | Protocol Area | Report |
|---|--------------|--------|
| 1 | Handoff Protocol | [protocol-handoff.md](protocol-handoff.md) |
| 2 | Feature State Machine | [protocol-feature-state.md](protocol-feature-state.md) |
| 3 | Validation Protocol | [protocol-validation.md](protocol-validation.md) |
| 4 | Mission Lifecycle | [protocol-mission-lifecycle.md](protocol-mission-lifecycle.md) |
| 5 | Session Protocol | [protocol-session.md](protocol-session.md) |
| 6 | Configuration Protocol | [protocol-config.md](protocol-config.md) |
| 7 | IPC Protocol | [protocol-ipc.md](protocol-ipc.md) |
| 8 | Cross-System Synthesis | [protocol-specification.md](protocol-specification.md) |

**Start with `protocol-specification.md`**: it synthesizes all 7 areas and documents cross-system data flows, invariants, and architectural patterns.

## Data Sources

- `protocol-samples/runtime-findings.md`: Sanitized protocol examples and schema observations
- `*.md`: Architecture notes used for cross-referencing

## Report Contents

Each report includes:
- **JSON schemas** with all fields, types, and required/optional status
- **Sanitized examples** from observed protocol samples
- **State machines** with transitions, triggers, and side effects
- **Discrepancy tables** documenting inferred vs observed behavior
- **Implementation notes** for equivalent system design

## Total Discrepancies Found

72 discrepancies documented across all protocol areas between inferred behavior and observed protocol samples.

## Methodology

1. Review sanitized protocol examples from `protocol-samples/runtime-findings.md`
2. Review corresponding architecture notes in this section
3. Cross-reference discrepancies, missing fields, and assumptions
4. Write implementation-oriented specifications with sanitized examples as the reference point
