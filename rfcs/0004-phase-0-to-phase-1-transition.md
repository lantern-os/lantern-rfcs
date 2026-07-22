---
rfc: 0004
title: Closing Phase 0 and opening Phase 1 (microkernel prototype)
status: Accepted
authors: ["LanternOS founding stewards"]
stewards: ["kernel", "hal", "boot"]
domains: ["governance", "kernel", "hal", "boot"]
created: 2026-07-22
updated: 2026-07-22
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0004: Closing Phase 0 and opening Phase 1 (microkernel prototype)

## Summary

This RFC proposes formally declaring **Phase 0 — Foundations** complete and opening
**Phase 1 — Microkernel prototype**, per the gates defined in
[`Roadmap.md`](https://github.com/lantern-os/lantern-docs/blob/main/wiki/Roadmap.md). Every Phase 0 checklist item now has
recorded evidence; this RFC is the phase-boundary decision the Roadmap itself requires
before Phase 1 work — including the first lines of kernel/boot/HAL code — may begin.

## Motivation

`GOVERNANCE.md` requires an RFC for "Roadmap phase boundaries," and the Roadmap says the
gate between phases is *quality of foundation*, not the calendar: "we do not advance until
they are met." Two things follow from that:

1. We should not silently flip phase markers as a side effect of editing docs — that would
   defeat the purpose of having a gate at all.
2. Once the criteria genuinely are met, we should say so explicitly and unblock Phase 1,
   rather than leaving the project idling in Phase 0 after its exit criteria are satisfied.

This serves the **security by architecture** principle directly: Phase 1 is where TCB code
starts to exist, and the project's own rule is that architecture, threat modelling, and
RFC review happen *before* that code is written — not after.

## Guide-level explanation

For a contributor, this RFC means:

- Phase 0 is closed: the architecture, threat models, and foundational RFCs are accepted
  and reviewed. No further Phase 0 work is expected before Phase 1 proceeds.
- Phase 1 is open: `lantern-boot`, `lantern-hal`, and `lantern-kernel` may now accept
  **prototype/throwaway** code aimed at booting a minimal kernel under QEMU on
  `riscv64`/x86-64, per their existing `ARCHITECTURE.md` and RFC-0002/ADR-0004.
- Nothing about the accepted architecture changes. This RFC does not modify RFC-0002,
  RFC-0003, or ADR-0001–0006 — it only unblocks the implementation work they already
  authorized.
- Normal RFC gates still apply within Phase 1: any new public interface, TCB addition, or
  crypto primitive choice still needs its own RFC (`GOVERNANCE.md`, "What requires an
  RFC"). Phase 1 prototype code is explicitly allowed to be thrown away — it is not held
  to production stability or API-freeze expectations.

## Reference-level explanation

### Phase 0 exit-criteria evidence

Per `Roadmap.md`, Phase 0's exit criteria are: *"the architecture is coherent, the threat
model is agreed, and the open foundational RFCs are accepted. A new contributor can
understand the why of the system from the docs alone."* Checklist status:

| Item | Status | Evidence |
| --- | --- | --- |
| Organisation structure and the 14 repositories | Done | Founding commit |
| Core principles, vision, system threat model | Done | Founding commit |
| Layered architecture documented end to end | Done | Founding commit |
| RFC + ADR process and governance model | Done | Founding commit |
| Seed RFCs and ADRs (Rust, RISC-V, Wasm) | Done | ADR-0001–0003 |
| Per-component `THREAT_MODEL.md` reviewed by domain stewards | Done | All 11 components reviewed |
| Per-component `ARCHITECTURE.md` reviewed by domain stewards | Done | All 11 reviewed; fixed a TCB-terminology collision in `lantern-kernel` and a missing auditability invariant in `lantern-capabilities` during review |
| RFC-0002, RFC-0003 advanced to Accepted | Done | Both `status: Accepted`; produced [ADR-0004](../adr/0004-kernel-responsibilities-and-tcb-boundary.md), [ADR-0005](../adr/0005-object-capabilities-as-universal-authority-model.md), [ADR-0006](../adr/0006-three-layer-capability-structure.md) |

All items are satisfied. No item is waived or deferred.

### Phase 1 scope (unchanged from Roadmap, restated for clarity)

- Boot to a minimal kernel on `riscv64`/x86-64 under QEMU (`lantern-boot`, `lantern-hal`).
- Address spaces, threads, and a scheduler.
- IPC fast-path (endpoints + notifications) with benchmarks.
- Kernel capability mechanism (CSpace, untyped retyping) per RFC-0003/ADR-0005/ADR-0006.
- The narrowing-waterfall root task starting one trivial user-space service.

**Phase 1 exit criteria** (unchanged, restated): a confined user-space "hello service"
reachable only via a granted capability, with IPC latency benchmarked and within target.

### What this RFC does *not* do

- It does not resolve any of the "Open questions" already recorded in `lantern-kernel`,
  `lantern-hal`, or `lantern-boot`'s `ARCHITECTURE.md`/`STATUS.md` (e.g. scheduling-context
  model, HAL/TCB split details). Those remain open and are Phase 1 work, tracked where
  they already live.
- It does not pre-approve any specific implementation choice (build system, target config,
  toolchain beyond ADR-0001/0002). Those are ordinary engineering decisions or, if they
  meet the RFC bar (e.g. a new TCB dependency), separate RFCs.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none newly defined — RFC-0002/ADR-0004 already fixed the
  TCB boundary this RFC operates inside of.
- **New assets introduced and who can reach them:** none directly; this RFC authorizes
  writing code, it does not itself introduce a running system. Once Phase 1 code lands, it
  runs only under QEMU in development, not on any production or user-facing target.
- **New adversary capabilities, if any:** none from this RFC itself.
- **Mitigations:** Phase 1 code is scoped to prototype/throwaway status and stays subject
  to the already-accepted threat models for `lantern-kernel`, `lantern-hal`, and
  `lantern-boot`; any change to those threat models still requires its own RFC per
  `GOVERNANCE.md`.
- **Net change to attacker surface:** neutral. This is a process decision, not a code or
  interface change.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** Not by itself. It authorizes
  future Phase 1 work (kernel, boot, minimal HAL) to begin producing TCB code, which was
  already scoped and justified by RFC-0002/ADR-0004.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** none yet; TCB size starts moving from zero once
  Phase 1 implementation begins, and stays tracked per-component in each `STATUS.md`.

## Privacy impact

None. This is a governance/process decision affecting no user data, telemetry, or identity
surface.

## Alternatives considered

- **Remain in Phase 0 indefinitely.** Rejected: exit criteria are fully met; the Roadmap's
  own philosophy is to gate on quality, not to gate arbitrarily once quality is achieved.
- **Skip the phase-boundary RFC and just edit `Roadmap.md`'s checkboxes.** Rejected:
  `GOVERNANCE.md` explicitly requires an RFC for roadmap phase boundaries; silently editing
  status defeats the point of having a documented, reviewable gate.
- **Fold this into RFC-0002/RFC-0003 acceptance directly.** Rejected: those RFCs are about
  *what* the microkernel and capability model are, not *when* implementation starts; mixing
  the two would make each RFC harder to review and reuse independently.

## Prior art

N/A — this is a project-internal process decision, not a technical design.

## Unresolved questions

None blocking. Phase 1's own open technical questions (scheduling-context model,
concurrency model, HAL/TCB split specifics) are tracked in the relevant components'
`ARCHITECTURE.md`/`STATUS.md`, not here.

## Future possibilities

A parallel Phase 1 → Phase 2 transition RFC, following the same pattern, once Phase 1's
exit criteria (confined "hello service" over a granted capability, IPC latency within
target) are met.

## Resulting ADRs

[ADR-0007](../adr/0007-phase-0-complete-phase-1-opened.md) records the phase transition
and the exit-criteria evidence as a durable decision, independent of this RFC file.
