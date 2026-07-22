---
adr: 0004
title: Kernel responsibility list and the TCB boundary
status: Accepted
date: 2026-07-22
deciders: ["TSC"]
rfc: ../rfcs/0002-microkernel-architecture.md
supersedes: []
superseded_by: null
---

# ADR-0004: Kernel responsibility list and the TCB boundary

## Context

[RFC-0002](../rfcs/0002-microkernel-architecture.md) proposed a capability-based
microkernel to minimise the Trusted Computing Base, the project's primary determinant of
its security ceiling. The RFC has been accepted; this ADR fixes the two durable decisions
it produces: the exact list of kernel responsibilities, and where the TCB boundary sits.

## Decision

**The kernel provides exactly five mechanisms and nothing else:**

1. Scheduling — threads and scheduling contexts; policy stays in user space.
2. Memory isolation — address spaces, page tables, untyped-memory retyping; no dynamic
   kernel heap allocation after boot.
3. IPC — synchronous call/reply (endpoints) and asynchronous signals (notifications) as
   the sole inter-component communication primitive.
4. Capability enforcement — every kernel object is named by a capability in a
   per-process CSpace; rights are checked on every operation.
5. Interrupt handling — interrupts delivered to user-space drivers as notifications
   bound to IRQ capabilities.

Drivers, filesystem, network stack, crypto services, the WASM and AI runtimes, the shell,
and all policy are explicitly **out** of the kernel and run as unprivileged, mutually
isolated user-space components.

**The TCB boundary is:**

```
   TCB (privileged / highest assurance)        Confined (unprivileged)
   ───────────────────────────────────         ───────────────────────────────
   lantern-boot  (root of trust, measured)      drivers, filesystem, network,
   lantern-kernel (microkernel)                 crypto service, wasm runtime,
   minimal HAL crate (machine layer)            ai runtime, shell, apps, agents
```

Any future addition to the TCB requires its own RFC and carries a verification cost.

## Consequences

- **Easier:** a bounded, auditable, eventually-verifiable privileged surface; driver bugs
  can no longer be total-system compromises.
- **Harder:** functionality that would be trivial in a monolithic kernel (e.g. a driver
  touching another driver's state directly) now requires explicit IPC and capabilities.
- **Committed to:** the kernel does no dynamic heap allocation after boot; `lantern-hal`'s
  machine layer is in the TCB and is held to the same scrutiny as the kernel itself; any
  TCB growth is an RFC-level decision, not an implementation detail.
- **Still open:** the scheduling-context model (full seL4-style MCS vs. a simpler initial
  scheme) and the concurrency model (single-stack vs. process-kernel) remain unresolved
  questions from RFC-0002 and are tracked as follow-on work in `lantern-kernel/STATUS.md`,
  not fixed by this ADR.
