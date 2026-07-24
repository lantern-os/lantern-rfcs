---
rfc: 0005
title: Kernel syscall/IPC ABI and the Phase 1 scheduling-context model
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel"]
domains: ["kernel", "hal", "boot"]
created: 2026-07-23
updated: 2026-07-24
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0005: Kernel syscall/IPC ABI and the Phase 1 scheduling-context model

## Summary

This RFC fixes two of the four open questions left by
[RFC-0002](./0002-microkernel-architecture.md)/[ADR-0004](../adr/0004-kernel-responsibilities-and-tcb-boundary.md):
the **syscall/IPC ABI and message-register layout**, and the **scheduling-context model**
for Phase 1. It defines a small, seL4-inspired syscall table built entirely on capability
invocation, a fixed-size fast-path message-register convention backed by a per-thread IPC
buffer, and a deliberately simplified (non-MCS) scheduling context whose *object shape* is
forward-compatible with the full MCS model deferred to a later phase. Nothing here is final
implementation code — Phase 1 is prototype/throwaway per RFC-0004 — but it is the concrete
interface `lantern-hal` and `lantern-boot` are currently blocked on.

## Motivation

`lantern-kernel/STATUS.md` lists this as the next item, and it sits on the critical path:
`lantern-hal`'s "Next" is "define the minimal HAL trait/contract the kernel depends on,"
which cannot be written until the kernel's trap-entry and register-passing convention
exists; `lantern-boot`'s "Next" is blocked on the HAL bring-up contract, which is blocked
on this. Without this RFC, Phase 1 cannot produce its exit criterion — a confined
"hello service" reachable via a granted capability, with IPC latency benchmarked — because
there is no fixed convention for invoking a capability across the syscall boundary at all.

This serves **security by architecture** directly: RFC-0003/ADR-0005 already committed the
project to "designation = authority, no ambient authority, no identity check." That is only
true if it is true *at the ABI level* — the syscall boundary is where a user-space
component's capabilities become the only thing it can act through. Getting this boundary
wrong (e.g. letting a syscall number stand in for authority, or trusting a caller-supplied
length field without kernel validation) would quietly reintroduce ambient authority at the
lowest level, undermining every layer above it.

## Guide-level explanation

There is effectively **one syscall**: *invoke a capability*. A user-space thread names a
capability by an integer slot in its CSpace (fixed by ADR-0006's kernel layer), packs a
small header plus up to four words of payload into registers, and traps into the kernel.
The kernel looks up the capability, checks the requested operation against its rights, and
either performs a fast in-register operation (an IPC send/receive/call on an endpoint) or
dispatches to a slower, still-bounded operation (retyping untyped memory, configuring a
thread, minting a capability).

A thread never asks "am I allowed to do X" in the abstract — it can only ever *invoke a
capability it holds*, per method. If it holds no capability to something, there is no
syscall that can reach it: the ABI has no "open by name" primitive anywhere.

For the common case — a client calling a service and waiting for a reply — a single `Call`
syscall sends a message to an endpoint capability and blocks until the service `Reply`s,
using a one-shot reply capability the kernel manufactures automatically. This is the fast
path the whole system's latency budget depends on (RFC-0002, "Performance posture").

## Reference-level explanation

### Capability addressing (CPtr)

A **CPtr** is an unsigned integer naming a slot in the invoking thread's CSpace root
CNode.

- **Phase 1 simplification:** the CSpace is a single flat CNode (a fixed-depth array of
  slots), not seL4's guarded multi-level CSpace. This removes a substantial chunk of
  lookup-path complexity from the prototype.
- **Forward compatibility:** the *invocation* ABI (CPtr + label + message, see below) does
  not encode CNode depth or guard bits anywhere visible to callers. Moving to a
  multi-level, guarded CSpace later (Phase 2, once address-space/process count justifies
  it) changes only kernel-internal slot resolution, not this ABI. This is deferred, not
  designed away.

### Message registers and the IPC buffer

- **Fast-path message registers (MRs):** 4 machine words, `mr0`–`mr3`, passed directly in
  general-purpose registers. `lantern-hal` fixes which physical register corresponds to
  each `mrN` per target (this is exactly the "minimal HAL trait" `lantern-hal/STATUS.md`
  is waiting to write); the portable kernel core never names a physical register directly.
- **IPC buffer:** each thread has one page, mapped read/write into that thread only,
  containing:
  - Extended message words beyond `mr3`, up to a fixed `MAX_MSG_WORDS` (bounds IPC cost and
    closes off unbounded-copy attacks).
  - A capability-transfer list: up to `MAX_EXTRA_CAPS` CPtrs naming capabilities to be
    copied to the receiver as part of this message (this is how `grant` from RFC-0003 is
    realised at the ABI level).
  - The kernel validates `length` and `extraCaps` against these fixed maxima **before**
    touching the buffer contents; a caller cannot claim a longer message than the buffer
    can hold.
- **Message tag:** one word, packed as `{ label: u32, length: u12, extra_caps: u4,
  flags: u16 }`, carried in a designated register alongside the MRs. `label` is
  protocol-specific (interpreted by the receiving service, meaningless to the kernel,
  except for the reserved CNode/Untyped/TCB invocation labels below). `flags` carries the
  error bit on return (see Error model) and is reserved otherwise.

### Syscall table

All syscalls take an implicit CPtr (the capability being invoked) plus the message tag and
MRs; the kernel dispatches on capability *type*, then on `label` where a type supports
multiple methods.

| # | Name | Applies to | Semantics |
| --- | --- | --- | --- |
| 1 | `Send` | Endpoint | Blocking send; blocks if no receiver is waiting. |
| 2 | `NBSend` | Endpoint | Non-blocking send; dropped if no receiver is ready. |
| 3 | `Recv` | Endpoint | Block until a `Send`/`Call` arrives. |
| 4 | `Call` | Endpoint | `Send` + block for `Reply`; generates a one-shot reply capability. |
| 5 | `Reply` | (implicit reply cap) | Replies to the most recent unanswered `Call` this thread received. |
| 6 | `Signal` | Notification | Set signal bits; non-blocking. |
| 7 | `Wait` | Notification | Block until signalled. |
| 8 | `Poll` | Notification | Non-blocking check. |
| 9 | `CNodeInvoke` | CNode | `label` selects `Mint` / `Copy` / `Move` / `Delete` / `Revoke` (RFC-0003's core operations). |
| 10 | `UntypedRetype` | Untyped | Retype a region into a typed object — the narrowing-waterfall primitive. |
| 11 | `TCBConfigure` | TCB (thread) | Set VSpace root, CSpace root, scheduling context, register state. |
| 12 | `Yield` | (thread, no cap) | Voluntarily surrender the remainder of the current scheduling-context budget. |

This list is intentionally close to seL4's, per "Prior art" below. It is not exhaustive —
IRQ-handler and VSpace/Frame invocations exist but reuse the same `label`-dispatch pattern
under `CNodeInvoke`-style capability-typed invocation rather than needing their own syscall
numbers, and are left to implementation, not ABI design.

### Error model

Every syscall returns through the same tag register it was invoked with: bit 0 of `flags`
is the error bit. On error, `mr0` carries one of a fixed error enum: `InvalidCapability`,
`InvalidArgument`, `IllegalOperation`, `RangeError`, `AlignmentError`, `FailedLookup`,
`TruncatedMessage`, `NotEnoughMemory`, `Timeout`. No syscall panics the kernel on caller
error — every input is validated before any state change (ties to K3 in
`lantern-kernel/THREAT_MODEL.md`).

### Scheduling-context model (Phase 1)

RFC-0002 left "full seL4-style MCS, or a simpler initial scheme?" open. This RFC decides:

- **Phase 1 ships a simplified scheduling context**: a distinct kernel object (not folded
  into the thread object) with two fields, `budget` and `period`, exactly like MCS — but
  Phase 1 semantics require `budget == period` (plain round-robin within a priority band,
  no sporadic-server replenishment, no temporal-isolation proof obligation).
- **Why keep the MCS object shape now:** if scheduling contexts were *not* a distinct
  object in Phase 1, introducing them later would be an ABI break (existing `TCBConfigure`
  callers would need a new capability type threaded through). Keeping the object shape and
  restricting the semantics costs nothing today and avoids an ABI-breaking RFC later.
- **Why not full MCS now:** full MCS's complexity (replenishment queues, sporadic-server
  admission control) is justified by real-time / mixed-criticality guarantees and is a
  named formal-verification target (Roadmap Phase 3+). Phase 1's exit criterion is a
  benchmarked IPC latency for one confined service, not a temporal-isolation proof;
  building that machinery now would be speculative for a prototype explicitly allowed to
  be thrown away (RFC-0004).

### HAL contract this ABI requires

This RFC is what unblocks `lantern-hal/STATUS.md`'s "Next" item. Concretely, the HAL must
provide, per target (`riscv64`, `x86-64`):

- Trap entry that saves the full register file into the interrupted thread's TCB and
  exposes the syscall-number register, the tag register, and `mr0`–`mr3` to the portable
  kernel core by fixed name (not raw register number).
- The physical-register ↔ `mrN` mapping (calling-convention-driven; e.g. `a1`–`a4` on
  RISC-V vs. the equivalent x86-64 SysV convention register set).
- Restoring that state on return, including the error tag.

The portable kernel core never references `a0`, `rdi`, or any ISA register name directly —
only `mr0..mr3`, the tag, and the syscall number, per the existing HAL-seam discipline in
`lantern-kernel/ARCHITECTURE.md`.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** concretises the kernel/user-space boundary from RFC-0002
  into a checkable interface; does not move the boundary itself.
- **New assets introduced and who can reach them:** the per-thread IPC buffer (readable/
  writable only by its owning thread and the kernel) and the message tag/MR registers
  during a trap. Reachable only by the thread that owns them; never shared.
- **New adversary capabilities, if any:** none intended. A malicious or buggy component can
  only ever invoke capabilities it already holds, with `length`/`extra_caps` validated
  against fixed maxima before the kernel reads the IPC buffer — this closes the obvious
  "claim a huge message, force an OOB kernel read" failure mode at the ABI level.
- **Mitigations:** bounds-checked message length and cap-transfer count; capability-type
  dispatch (a `label` meaningless to a CNode cannot be reinterpreted as a CNode operation);
  one-shot, non-forgeable reply capabilities scoped to a single outstanding `Call`.
- **Net change to attacker surface:** neutral-to-reducing — this is RFC-0002/0003's
  abstract commitments made concrete and checkable, not a new surface. The one addition
  worth naming plainly is the IPC buffer itself as a new per-thread memory object; it is
  mitigated by being exclusively mapped and bounds-checked as above.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** It specifies an interface to
  existing TCB code (the kernel and the minimal HAL crate, both already in the TCB per
  ADR-0004); it does not add a new TCB component.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** positive for auditability — a fixed ~12-entry
  syscall table dispatched entirely through capability-typed invocation is small and
  uniform to review, versus each implementer inventing ad hoc conventions per object type.

## Privacy impact

None directly — this is a mechanism-layer ABI with no user data, telemetry, or identity
surface. Indirectly positive, in the same way RFC-0003 is: bounds-checked, capability-gated
IPC is what makes the later privacy-relevant guarantees (confinement of the AI runtime,
consent-scoped capability grants) enforceable at all.

## Alternatives considered

- **Adopt seL4's ABI verbatim, including full MCS and multi-level CSpace from day one.**
  Rejected for Phase 1: the additional complexity (replenishment queues, guarded CSpace
  lookup) is not justified by Phase 1's exit criteria and would slow the prototype without
  buying anything a throwaway/QEMU-only "hello service" needs yet. Revisit in the Phase
  1→2 transition RFC once real service counts exist.
- **A fully custom ABI diverging from seL4 patterns.** Rejected: seL4's IPC/capability ABI
  is the most analysed design in this space (including under formal verification), and
  RFC-0002 already anchors the object model on it; diverging without a specific reason
  would discard that prior art for no benefit.
- **Fold scheduling context fields into the TCB object instead of a separate capability
  type.** Rejected: cheaper today, but forecloses per-thread scheduling-context reuse and
  budget/priority separation later without an ABI break — the exact problem this RFC is
  trying to avoid by fixing the object shape now.
- **Do nothing / leave the ABI undefined until someone writes kernel code.** Rejected:
  `lantern-hal` and `lantern-boot` are both explicitly blocked on this per their
  `STATUS.md` files; leaving it implicit would mean the HAL contract gets defined
  ad hoc inside a HAL PR instead of reviewed as the public interface it is
  (`GOVERNANCE.md` requires an RFC for exactly this).

## Prior art

**seL4** is the primary influence: the capability-invocation-as-sole-syscall pattern, the
IPC buffer, badged endpoints, and the MCS scheduling-context split are all seL4 concepts;
this RFC's main departure is deliberately simplifying MCS semantics and CSpace structure
for a throwaway Phase 1 prototype while keeping seL4's object shapes for forward
compatibility. **Zircon** (Fuchsia) uses handles rather than CSpace slots and a richer
per-object syscall surface; considered and not followed, since RFC-0002/ADR-0004 already
committed to the seL4-style CSpace/CNode model. **Barrelfish**'s multikernel message-passing
informs the "no shared mutable state across the fast path" discipline already noted in
RFC-0002.

## Unresolved questions

- Final MR count (4 is a starting point; may change based on RISC-V/x86-64 calling-
  convention register pressure once benchmarked).
- Whether reply capabilities become first-class, storable/grantable objects (needed for
  some multi-stage service patterns) — deferred to Phase 2.
- Multi-level guarded CSpace design — deferred to Phase 2, as stated above.
- Full MCS budget/replenishment semantics and any associated verification proof
  obligations — deferred to Phase 3 per the Roadmap's formal-verification phase.
- IRQ-handler and VSpace/Frame invocation `label` tables — implementation detail, not
  fixed by this RFC.
- The IOMMU/interrupt-controller HAL/portable-core split remains a separate open question
  from `lantern-kernel/ARCHITECTURE.md`, out of scope here.

## Future possibilities

- Migrating CPtr resolution to a CHERI-backed hardware capability representation on capable
  RISC-V cores, without changing this RFC's invocation-level ABI (RFC-0002/0003 both flag
  CHERI as future work).
- The Phase 1→2 transition RFC upgrading scheduling contexts to full MCS and CSpace to
  multi-level, once real service/process counts justify the added complexity.
- Machine-checked verification of the `Call`/`Reply` fast path and the capability-check
  logic, using this ABI as the specification surface (Roadmap Phase 3+).

## Resulting ADRs

[ADR-0008](../adr/0008-kernel-syscall-ipc-abi.md) fixes the syscall table, message-register
convention, and IPC-buffer layout. [ADR-0009](../adr/0009-phase1-scheduling-context-model.md)
fixes the Phase 1 scheduling-context model (object shape now, simplified semantics, MCS
deferred).
