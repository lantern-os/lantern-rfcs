---
adr: 0016
title: A monotonic clock read primitive for lantern-hal
status: Accepted
date: 2026-08-24
deciders: ["TSC"]
rfc: ../rfcs/0012-monotonic-clock-primitive.md
supersedes: []
superseded_by: null
---

# ADR-0016: A monotonic clock read primitive for lantern-hal

## Context

[RFC-0011](../rfcs/0011-sealed-capability-token-format.md)/[ADR-0015](./0015-sealed-capability-token-format.md)
added `Caveat::ExpiresAt` to sealed capabilities with `Keystore::unseal`'s `now: Option<u64>`
deliberately unsourced — no real clock existed in the project. `lantern-hal/ARCHITECTURE.md`
has listed "Timekeeping" in the HAL's abstraction table since Phase 0 but never implemented
it. [RFC-0012](../rfcs/0012-monotonic-clock-primitive.md) proposed a narrow, read-only slice
of that responsibility and has been accepted; this ADR is the durable record.

## Decision

**`lantern-hal`'s `Hal` trait gains `monotonic_time_ns() -> u64`, a read-only monotonic
clock, with no timer-interrupt or scheduler-tick surface added alongside it.**

- `riscv64`: implemented via the `time` CSR (`rdtime`), directly readable from S-mode under
  standard OpenSBI `mcounteren` configuration — no trap, no SBI call, no new privilege
  boundary. QEMU `virt`'s 10 MHz timer frequency is a hardcoded constant for now;
  device-tree-sourced frequency discovery remains open (`lantern-hal/ARCHITECTURE.md`'s
  "platform discovery").
- `x86-64`: `unimplemented!()`, matching `initial_trap_frame`/`enter_thread`'s existing
  precedent for methods with no real boot work to exercise them against yet.
- Explicitly out of scope: timer interrupts, scheduler ticks, and any syscall exposing time
  to confined user-space code. These remain separate, larger future work
  (RFC-0012's "Future possibilities").
- This is the first real implementation of an already-scoped HAL responsibility (per
  `lantern-hal/ARCHITECTURE.md`'s existing "Timekeeping" table entry and RFC-0002's
  "Scheduling"/"Interrupt handling" kernel responsibilities), not a newly invented one — the
  same relationship [ADR-0012](./0012-vspace-frame-capabilities-and-elf-loader.md) had to
  "Memory isolation" when it first implemented VSpace/Frame capabilities.

## Consequences

- **Easier:** `Caveat::ExpiresAt` can move from a permanently-unsatisfiable placeholder
  (`now: None` always denies) to an actually-enforceable expiry, once a caller wires
  `monotonic_time_ns()` into `Keystore::unseal`. Any future component needing "how long since
  some earlier point" (rate limiting, timeouts, benchmarking) has a portable, contract-defined
  primitive instead of reaching for a raw CSR read of its own.
- **Harder:** nothing new — the addition is small enough that it doesn't materially change
  what's hard about working in this codebase.
- **Committed to:** `monotonic_time_ns()`'s semantics (monotonic, nanosecond-scaled, arbitrary
  epoch, never wall-clock time) are fixed as the HAL's clock contract; a future wall-clock or
  timer-interrupt capability is new, separate work, not a silent redefinition of this one.
- **Still open:** device-tree-sourced timer frequency (currently hardcoded for QEMU `virt`),
  `x86-64`'s real implementation (needs TSC calibration), and — tracked in
  `lantern-crypto/STATUS.md` and `lantern-filesystem/STATUS.md`, not here — actually wiring
  this into `Keystore::unseal`'s call sites.
