---
adr: 0007
title: Phase 0 complete; Phase 1 (microkernel prototype) opened
status: Accepted
date: 2026-07-22
deciders: ["TSC"]
rfc: ../rfcs/0004-phase-0-to-phase-1-transition.md
supersedes: []
superseded_by: null
---

# ADR-0007: Phase 0 complete; Phase 1 (microkernel prototype) opened

## Context

[RFC-0004](../rfcs/0004-phase-0-to-phase-1-transition.md) proposed declaring Phase 0 —
Foundations complete, per the exit criteria in
[`Roadmap.md`](https://github.com/lantern-os/lantern-docs/blob/main/wiki/Roadmap.md), and opening Phase 1 — Microkernel
prototype. `GOVERNANCE.md` requires an RFC for roadmap phase boundaries specifically so
this transition is a recorded decision, not a status edit. The RFC has been accepted; this
ADR is the durable record.

## Decision

**Phase 0 is complete. Phase 1 is open.**

All Phase 0 exit criteria are satisfied:

- Organisation structure, principles, vision, and system threat model — done at founding.
- Layered architecture documented end to end — done at founding.
- RFC + ADR process and governance model — done at founding.
- Seed RFCs and ADRs (Rust, RISC-V, Wasm) — ADR-0001–0003.
- Per-component `THREAT_MODEL.md` reviewed by domain stewards — all 11 components.
- Per-component `ARCHITECTURE.md` reviewed by domain stewards — all 11 components (fixed a
  TCB-terminology collision in `lantern-kernel`, added a missing auditability invariant in
  `lantern-capabilities`).
- RFC-0002 and RFC-0003 advanced to Accepted — producing
  [ADR-0004](./0004-kernel-responsibilities-and-tcb-boundary.md),
  [ADR-0005](./0005-object-capabilities-as-universal-authority-model.md), and
  [ADR-0006](./0006-three-layer-capability-structure.md).

`lantern-boot`, `lantern-hal`, and `lantern-kernel` may now accept prototype/throwaway
code toward Phase 1's goal: booting a minimal kernel on `riscv64`/x86-64 under QEMU, with
address spaces, threads, a scheduler, IPC fast-path, the kernel capability mechanism, and a
narrowing-waterfall root task starting one trivial user-space service. No other component
repository's phase status changes by this decision.

## Consequences

- **Easier:** `lantern-boot`, `lantern-hal`, and `lantern-kernel` can now start
  implementation work without a per-PR debate over whether Phase 0 is "really" done — this
  ADR is the answer.
- **Harder:** none directly; the accepted architecture (RFC-0002, RFC-0003,
  ADR-0001–0006) is unchanged, so no existing design work needs to be redone.
- **Committed to:** Phase 1 code in `lantern-boot`, `lantern-hal`, and `lantern-kernel` is
  explicitly prototype/throwaway and does not need to meet production stability or
  API-freeze expectations; it still must respect the already-accepted threat models, and
  any new public interface, TCB addition, or crypto primitive choice still requires its own
  RFC per `GOVERNANCE.md`. Phase 1's exit criterion — a confined "hello service" reachable
  only via a granted capability, with IPC latency benchmarked and within target — gates the
  next transition (Phase 1 → Phase 2), which will follow this same RFC + ADR pattern.
