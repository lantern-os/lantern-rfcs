---
adr: 0010
title: Kernel concurrency model — single-stack, run-to-completion (Phase 1)
status: Accepted
date: 2026-07-26
deciders: ["TSC"]
rfc: ../rfcs/0006-kernel-concurrency-model.md
supersedes: []
superseded_by: null
---

# ADR-0010: Kernel concurrency model — single-stack, run-to-completion (Phase 1)

## Context

[RFC-0002](../rfcs/0002-microkernel-architecture.md) left "single-stack vs.
process-kernel" as an open question. `lantern-kernel/STATUS.md` listed it as the last
item blocking Phase 1 prototype code, and `lantern-hal`'s `riscv64`/`x86-64` trap entries
already committed the HAL side to a plain, synchronous `TrapHandler = fn(&mut TrapFrame)`
callback ([ADR-0008](./0008-kernel-syscall-ipc-abi.md)) — a shape only a single-stack
kernel can honor. [RFC-0006](../rfcs/0006-kernel-concurrency-model.md) proposed fixing
the model explicitly; it has been accepted. This ADR fixes the durable decision.

## Decision

**The kernel is single-stack and run-to-completion, seL4-style: one kernel stack per
hart, no kernel-level threads, no kernel-internal blocking.** A trapped-into syscall
runs to completion (or to an explicit, bounded preemption point for the rare
unbounded-cost operation) entirely on the interrupting hart's own kernel stack, with
interrupts masked throughout, then returns directly to user space.

- **Phase 1 is single-hart** (matching `lantern-hal`'s current implementation), so there
  is exactly one kernel stack. The multi-hart question (a single big kernel lock vs.
  fine-grained locking) is explicitly deferred to a future SMP-scaling RFC.
- **No kernel-internal locks in Phase 1:** with one hart and a non-preemptible kernel,
  ordinary `&mut` access checked by the borrow checker is sufficient for kernel data
  structures; no synchronization primitives are needed yet.
- **Preemption points (principle, not yet mechanism):** any kernel operation whose cost
  scales with caller-influenced state (e.g. `CNodeInvoke`'s `Revoke` fanning out over
  derived capabilities) must eventually be checked against a preemption budget and made
  resumable/restartable. Phase 1's exit criterion doesn't exercise this, so only the
  principle is fixed now; the mechanism is deferred.

## Consequences

- **Easier:** kernel data structures need no synchronization primitives in Phase 1,
  keeping the `unsafe`/concurrency-reasoning surface — and thus the audit and eventual
  formal-verification burden — minimal; `lantern-kernel`'s trap-dispatch code can now be
  written directly against `TrapHandler`'s existing `fn(&mut TrapFrame)` shape
  (ADR-0008) with no redesign.
- **Harder:** kernel code can never block waiting on another hart or on I/O mid-syscall;
  any future need for that (multi-hart rendezvous, blocking on slow hardware) must be
  designed around the run-to-completion model rather than assumed.
- **Committed to:** kernel stacks are set up once at boot, never allocated dynamically
  per-thread; the kernel is non-preemptible by default (interrupts masked for the
  duration of a trap) except at explicit, bounded preemption points reserved for
  unbounded-cost operations; multi-hart concurrency is out of scope for this ADR and
  requires its own RFC before `lantern-hal` gains multi-hart support.
- **Still open:** the concrete preemption-point mechanism, the multi-hart concurrency
  model, and kernel-stack sizing/placement in the memory map remain unresolved, tracked
  in `lantern-kernel/STATUS.md`.
