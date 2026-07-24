---
adr: 0009
title: Phase 1 scheduling-context model — MCS-shaped object, simplified semantics
status: Accepted
date: 2026-07-24
deciders: ["TSC"]
rfc: ../rfcs/0005-syscall-ipc-abi-and-phase1-scheduling.md
supersedes: []
superseded_by: null
---

# ADR-0009: Phase 1 scheduling-context model — MCS-shaped object, simplified semantics

## Context

[RFC-0002](../rfcs/0002-microkernel-architecture.md) left "full seL4-style MCS, or a
simpler initial scheme?" as an open question, deferring the scheduling-context ADR until
resolved (see RFC-0002, "Resulting ADRs"). [RFC-0005](../rfcs/0005-syscall-ipc-abi-and-phase1-scheduling.md)
proposed an answer sized for Phase 1's actual exit criterion — a benchmarked IPC latency
for one confined "hello service," not a temporal-isolation proof — and it has been
accepted. This ADR fixes the durable decision.

## Decision

**Scheduling context is a distinct kernel object, separate from the thread (TCB) object,
with two fields: `budget` and `period` — matching seL4 MCS's object shape.** Phase 1
restricts the semantics: `budget == period` always, giving plain round-robin scheduling
within a priority band. There is no sporadic-server replenishment, no admission control,
and no temporal-isolation guarantee or proof obligation in Phase 1.

- **Why a distinct object now, not fields on the TCB:** folding budget/period into the
  thread object is cheaper today but is an ABI break to undo later — every
  `TCBConfigure` caller would need updating to thread a new capability type through.
  Keeping the object separate costs nothing now and avoids that break.
- **Why not full MCS now:** full MCS's complexity (replenishment queues, sporadic-server
  admission control) is justified by real-time/mixed-criticality guarantees and is a named
  formal-verification target (Roadmap Phase 3+), not by Phase 1's exit criteria.

## Consequences

- **Easier:** Phase 1 kernel prototype work can implement plain round-robin scheduling
  without building replenishment/admission-control machinery that Phase 1 doesn't need;
  `TCBConfigure` (ADR-0008) can bind a scheduling context to a thread now, with the same
  capability shape that full MCS will later use.
- **Harder:** none directly for Phase 1; the deferred cost is that full MCS semantics
  (and any verification built against the simplified model) will need dedicated follow-on
  work later rather than being available from the start.
- **Committed to:** the scheduling-context object's shape (`budget`, `period` fields, a
  capability type distinct from TCB) will not change when full MCS semantics are
  introduced — only the semantics upgrade (replenishment, admission control), not the
  object or the `TCBConfigure` invocation shape from ADR-0008. Upgrading to full MCS is
  planned for the Phase 1 → Phase 2 transition RFC once real service/process counts
  justify it, following the same RFC + ADR pattern as this decision.
- **Still open:** full MCS budget/replenishment semantics and any associated verification
  proof obligations remain deferred to Roadmap Phase 3+, tracked in
  `lantern-kernel/STATUS.md`.
