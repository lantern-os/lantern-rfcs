---
adr: 0008
title: Kernel syscall/IPC ABI — capability invocation, message registers, IPC buffer
status: Accepted
date: 2026-07-24
deciders: ["TSC"]
rfc: ../rfcs/0005-syscall-ipc-abi-and-phase1-scheduling.md
supersedes: []
superseded_by: null
---

# ADR-0008: Kernel syscall/IPC ABI — capability invocation, message registers, IPC buffer

## Context

[RFC-0002](../rfcs/0002-microkernel-architecture.md) left the syscall/IPC ABI and
message-register layout as an open question. `lantern-hal` and `lantern-boot` were both
blocked on it per their `STATUS.md` files: the HAL cannot define its trap-entry contract,
and boot cannot finalise its bring-up contract, without a fixed convention for invoking a
capability across the syscall boundary. [RFC-0005](../rfcs/0005-syscall-ipc-abi-and-phase1-scheduling.md)
proposed that convention; it has been accepted. This ADR fixes the durable decision.

## Decision

**There is one syscall shape: invoke a capability.** A thread names a capability by an
integer CPtr into its (Phase 1: flat, single-level) CSpace, packs a message tag plus up to
four fast-path message registers (`mr0`–`mr3`) into registers, and traps. The kernel
dispatches on capability type, then on the tag's `label` field.

- **Syscall table** (12 entries): `Send`, `NBSend`, `Recv`, `Call`, `Reply`, `Signal`,
  `Wait`, `Poll`, `CNodeInvoke`, `UntypedRetype`, `TCBConfigure`, `Yield` — as specified in
  RFC-0005's reference-level explanation.
- **Message tag:** one word — `{ label: u32, length: u12, extra_caps: u4, flags: u16 }`.
  `flags` bit 0 carries the error flag on return.
- **IPC buffer:** one page per thread, holding extended message words beyond `mr3` (up to
  `MAX_MSG_WORDS`) and a capability-transfer list (up to `MAX_EXTRA_CAPS`), both validated
  against fixed maxima **before** the kernel reads buffer contents.
- **Error model:** a fixed enum (`InvalidCapability`, `InvalidArgument`,
  `IllegalOperation`, `RangeError`, `AlignmentError`, `FailedLookup`, `TruncatedMessage`,
  `NotEnoughMemory`, `Timeout`) returned in `mr0` on error; no syscall panics the kernel on
  caller-supplied input.
- **HAL contract:** the portable kernel core never names a physical register; `lantern-hal`
  maps `mr0..mr3`, the tag register, and the syscall-number register to the ISA's actual
  calling convention (per target: `riscv64`, `x86-64`) and handles trap entry/exit.
- **CPtr resolution is a flat, single-level CNode for Phase 1** — a deliberate
  simplification of seL4's guarded multi-level CSpace. The invocation ABI itself (CPtr +
  tag + MRs) does not encode CNode depth/guard bits, so moving to multi-level CSpace later
  is a kernel-internal change, not an ABI break.

## Consequences

- **Easier:** `lantern-hal` can now write its trap-entry contract and register mapping
  against a fixed target instead of guessing; `lantern-boot`'s bring-up contract has one
  fewer unknown; the syscall surface is small (~12 entries) and uniform, which helps both
  review now and the formal-verification target later (Roadmap Phase 3+).
- **Harder:** every capability-typed object needs its own `label`-dispatch table designed
  before it can be invoked (IRQ handlers, VSpace/Frame objects — left to implementation);
  the bounds-checked IPC buffer adds a small amount of fixed per-thread memory and
  validation logic that a naive design might have skipped.
- **Committed to:** the portable kernel core will not reference ISA register names
  directly, only `mr0..mr3`/tag/syscall-number by fixed name; message length and
  capability-transfer count are always validated against `MAX_MSG_WORDS`/`MAX_EXTRA_CAPS`
  before the IPC buffer is read; any future syscall-table addition or change to this ABI is
  itself an RFC-level decision (`GOVERNANCE.md`, "new or changed public interfaces").
- **Still open:** final `MR` count, whether reply capabilities become first-class
  grantable objects, the multi-level CSpace design, and IRQ-handler/VSpace `label` tables
  are unresolved questions carried from RFC-0005, tracked in `lantern-kernel/STATUS.md`.
