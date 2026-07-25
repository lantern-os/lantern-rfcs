---
rfc: 0006
title: Kernel concurrency model — single-stack, run-to-completion for Phase 1
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["kernel"]
domains: ["kernel", "hal"]
created: 2026-07-25
updated: 2026-07-25
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0006: Kernel concurrency model — single-stack, run-to-completion for Phase 1

## Summary

This RFC fixes the last of [RFC-0002](./0002-microkernel-architecture.md)/[ADR-0004](../adr/0004-kernel-responsibilities-and-tcb-boundary.md)'s
open questions not resolved by [RFC-0005](./0005-syscall-ipc-abi-and-phase1-scheduling.md):
the kernel's **concurrency model**. It adopts seL4's single-stack, run-to-completion
design for Phase 1 — one kernel stack per hart, no kernel-internal blocking, no
kernel-level threads of its own — and explicitly defers the multi-hart question
("big lock" vs. fine-grained locking) to a future SMP-scaling RFC, since `lantern-hal`'s
just-landed trap entries are single-hart only. This also makes explicit an assumption
[ADR-0008](../adr/0008-kernel-syscall-ipc-abi.md) already made implicitly: `TrapHandler`
being a plain `fn(&mut TrapFrame)` only makes sense under a kernel that never suspends
mid-call.

## Motivation

`lantern-kernel/STATUS.md` lists "concurrency model (single-stack vs. process-kernel)"
as the last unresolved question blocking Phase 1 prototype code (address spaces,
threads, IPC fast-path, capability mechanism). RFC-0005 deliberately left it untouched.
Without it, `lantern-kernel` cannot start writing its trap-dispatch code, because that
code's entire shape — whether a syscall can block partway through, whether the kernel
needs internal locks, how many kernel stacks exist at all — depends on this decision.

There is also a concrete forcing function that arrived after RFC-0002: `lantern-hal`'s
just-landed `riscv64`/`x86-64` trap entries (per ADR-0008) call into the portable kernel
core through `install_trap_handler(handler: TrapHandler)`, where
`TrapHandler = fn(frame: &mut TrapFrame)` — a plain, synchronous, non-suspendable
function, invoked directly from trap context with interrupts masked. That shape is only
sound if the kernel code it calls into runs to completion without blocking or switching
stacks mid-call. RFC-0005/ADR-0008 didn't spell this out as a decision — it fell out of
choosing a plain `fn` pointer over some continuation/coroutine-shaped callback. This RFC
makes that implicit assumption an explicit, reviewed one, rather than letting it be
discovered as an accident of the HAL's function-pointer signature.

This also serves the project's principles directly: **TCB minimalism** (a single-stack,
non-preemptible kernel core needs no internal lock machinery yet, keeping the
`unsafe`/concurrency-reasoning surface minimal — see K3 below) and the standing
**formal-verification target** named in `lantern-kernel/ARCHITECTURE.md` ("Concurrency &
assurance") and `THREAT_MODEL.md` ("Verification intent") — seL4's verification was only
tractable because its kernel is single-stack and (mostly) non-preemptible; diverging from
that now would foreclose reusing that verification strategy later without a rewrite.

## Guide-level explanation

When a thread traps into the kernel (via the HAL trap entry ADR-0008 fixed), the kernel
runs the entire syscall on the interrupting hart's own kernel stack, start to finish,
with interrupts masked, and returns directly to user space — the same thread, its
scheduled successor, or back to itself. The kernel never blocks partway through a
syscall waiting on another hart or on I/O; every syscall either finishes or is
explicitly bounded and restartable (see "Preemption points" below). A `Send`/`Call`/
`Recv` that has nothing to rendezvous with immediately puts the calling thread on a wait
queue and returns to the scheduler — the kernel itself does not sit there waiting.

There is exactly one kernel stack in Phase 1, because `lantern-hal` (per its just-landed
trap-entry implementations) is single-hart only right now. The multi-hart case (one
kernel stack per hart, plus whatever hart-to-hart synchronization that requires) is real
future work, not decided by this RFC.

## Reference-level explanation

### The model: single-stack, run-to-completion

- **One kernel stack per hart** (Phase 1: one hart total, so one kernel stack total).
  Set up once at boot; never allocated dynamically per-thread.
- **No kernel-level threads.** The kernel itself is not a scheduled entity; it only ever
  executes on top of whichever user thread trapped in (or during boot/idle). This
  matches `TrapHandler`'s `fn(&mut TrapFrame)` shape exactly: the call stack when
  `TrapHandler` runs *is* the kernel stack, and it unwinds back to the HAL's trap-exit
  path (`sret`/`iretq`) when the syscall is done — there is no way to suspend a `fn`
  call and resume it later on a different stack without CPS-transforming the whole
  kernel, which this RFC does not adopt.
- **Non-preemptible by default.** Traps arrive with interrupts masked (per
  `lantern-hal`'s `riscv64` `sstatus.SIE`-clearing and `x86-64` interrupt-gate
  behavior, both already implemented this way) and the kernel does not re-enable them
  until the trap handler returns. A syscall either completes or hits an explicit
  preemption point (below); it is never interrupted by a timer tick partway through in
  a way the kernel has to reason about reentrantly.
- **No kernel-internal locks in Phase 1.** Because there is one hart and the kernel is
  non-preemptible, kernel data structures (CNodes, endpoints, scheduler queues, ...)
  need no synchronization primitives at all — ordinary Rust `&mut` access, checked by
  the borrow checker, is sufficient. This is the direct payoff of the model, and it's
  why K3 (`THREAT_MODEL.md`, "Memory corruption in the kernel" — mitigated by "safe
  Rust; `unsafe` isolated, justified, reviewed") gets easier to hold to, not harder.

### Preemption points

Some kernel operations are not O(1) in the worst case — the clearest example is
`CNodeInvoke`'s `Revoke` (ADR-0008's syscall table, entry 9), which can fan out over an
attacker-influenced number of derived capabilities. An unbounded-latency kernel
operation is a K8-class problem (`THREAT_MODEL.md`, "Timing/covert channels... partial
mitigation via scheduling isolation; not fully solved") turned into a denial-of-service
vector, and it undercuts RFC-0002's "Performance posture" commitment to benchmarked IPC
latency.

Per seL4's precedent, this RFC adopts the *principle* — not yet the mechanism — that any
kernel operation whose cost scales with caller-influenced state must be checked against
a preemption budget and, if exceeded, save enough state to resume (or safely restart)
the operation on the next trap. **Phase 1 does not need to implement this**: its exit
criterion (RFC-0004: one confined "hello service", benchmarked IPC latency) has no
operation that fans out over enough objects to matter. Fixing the principle now means a
Phase 1 implementer doesn't have to choose between shipping an unbounded `Revoke` and
blocking on a mechanism Phase 1 doesn't need yet; the actual restart mechanism is
deferred, tracked as unresolved below.

### Multi-hart: explicitly out of scope

RFC-0002 posed this as "single-stack vs. process-kernel," not "single-hart vs.
multi-hart" — but `lantern-hal`'s current implementation (both `riscv64` and `x86-64`
trap entries) is single-hart only (a single static trap-frame/IDT, no per-hart storage).
This RFC does not change that. When multi-hart support lands, "single-stack" needs a
hart-indexed answer: seL4 historically shipped a single "big kernel lock" (BKL) held for
the duration of any hart's time in the kernel — the simplest extension of this RFC's
model to multiple harts — before later work moved parts of seL4 to finer-grained locking
for scalability. That choice (BKL vs. fine-grained, and exactly where) is deferred to
the SMP-scaling RFC this document does not attempt to pre-empt.

### Interaction with the scheduling-context model (ADR-0009)

ADR-0009's Phase 1 scheduling context (`budget`/`period`, `budget == period`,
round-robin) assumes a kernel that can always tell "whose turn is it" synchronously at
the point a thread blocks or a syscall completes — true only under a single-stack,
run-to-completion kernel where scheduling decisions happen inline in the trap-return
path, not in some concurrently-running kernel scheduler thread. This RFC is consistent
with, and in fact required for, ADR-0009 to mean what it currently says.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none moved; this concerns the kernel's *internal*
  execution model, not the kernel/user-space boundary RFC-0002/ADR-0004 already fixed.
- **New assets introduced and who can reach them:** none.
- **New adversary capabilities, if any:** none intended. If anything, this model
  *reduces* K3 (memory corruption) and K8 (timing/covert channel) exposure versus a
  process-kernel model: no kernel-internal locking means no lock-ordering or
  priority-inversion class bugs to introduce in the first place, and the "no preemption
  mid-syscall, explicit preemption points for unbounded ops" principle is aimed
  precisely at K8's DoS-via-unbounded-latency variant.
- **Mitigations:** run-to-completion bounds most syscalls' worst-case latency by
  construction (they're small, fixed operations per ADR-0008's syscall table); the
  preemption-point principle (not yet mechanism) is fixed now so an unbounded operation
  like `Revoke` cannot ship without a designed answer later.
- **Net change to attacker surface:** neutral-to-reducing, for the reasons above — no
  new surface, and the model is chosen partly *because* it closes off two classes of
  kernel bugs a process-kernel model would otherwise have to defend against from day
  one.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No new component; it constrains
  how existing/future kernel TCB code (already in the TCB per ADR-0004) is structured
  internally.
- **Does this add a dependency to the TCB?** No — if anything it removes a prospective
  one: a process-kernel model would likely need synchronization primitives (spinlocks,
  or worse) inside the kernel that this model does not require in Phase 1.
- **Effect on TCB size and auditability:** positive. A single-stack, non-preemptible,
  lock-free kernel core is smaller and more directly auditable/verifiable than the
  equivalent process-kernel design, and keeps the door open to the formal-verification
  target both `lantern-kernel/ARCHITECTURE.md` and `THREAT_MODEL.md` already name — this
  is the same execution model seL4's verification was built against.

## Privacy impact

None directly — this is a kernel execution-model decision with no user data or
telemetry surface. Indirectly positive in the same way RFC-0005 is: a kernel that's
easier to reason about and verify is a stronger foundation for every privacy-relevant
guarantee built above it (AI runtime confinement, consent-scoped capability grants).

## Alternatives considered

- **Process-kernel model (per-thread kernel stack, kernel can block internally).**
  Rejected for Phase 1: requires a per-thread kernel stack (memory cost, and in tension
  with "no dynamic kernel heap allocation after boot" from ADR-0004 unless kernel
  stacks are statically sized and pre-allocated per TCB — itself an unresolved design
  question this RFC would rather not smuggle in unexamined) and kernel-internal
  synchronization primitives that neither Phase 1's exit criterion nor `lantern-hal`'s
  current single-hart implementation need. It buys the ability for kernel code to block
  waiting on another hart or on I/O mid-syscall — a capability Phase 1's five-
  responsibility kernel (ADR-0004: scheduling, memory isolation, IPC, capability
  enforcement, interrupt handling — nothing I/O-bound) has no use for. Revisit once real
  multi-core, multi-service workloads make single-stack's simplicity actually cost
  something.
- **Fully event-based/async kernel (continuation-passing, explicit suspend/resume of
  in-kernel operations).** Rejected: strictly more machinery than either alternative
  above for no Phase 1 benefit, and incompatible with `TrapHandler`'s already-shipped
  `fn(&mut TrapFrame)` shape (ADR-0008) without either changing that ABI or building a
  continuation mechanism underneath it the ABI can't see — worse of both worlds.
- **Decide nothing yet; let the first kernel prototype PR settle it implicitly.**
  Rejected: this is exactly the ad hoc "decide the public interface inside an
  implementation PR instead of in the open" pattern `GOVERNANCE.md` and RFC-0001 exist
  to prevent, and RFC-0005 already made the same call for the ABI rather than leaving it
  to a HAL PR.

## Prior art

**seL4** is again the primary influence (as for RFC-0002/RFC-0005): its verified kernel
is single-stack per core, (mostly) non-preemptible, and uses explicit, bounded
preemption points for the small number of operations that aren't naturally O(1)
(revocation being the canonical example) — this RFC adopts that model directly rather
than reinventing one. **L4**'s lineage generally favors small, non-blocking kernel-mode
execution for the same latency reasons cited in RFC-0002's "Performance posture."
Traditional Unix-like monolithic kernels (and hybrid kernels like XNU) are closer to the
"process-kernel" alternative — kernel-mode code that blocks freely on locks and I/O —
which is exactly the complexity RFC-0002 rejected that class of kernel for in the first
place (see RFC-0002, "Alternatives considered").

## Unresolved questions

- The concrete preemption-point mechanism (how a long operation like `Revoke` checks its
  budget and resumes/restarts) — principle fixed here, implementation deferred to
  whichever Phase 1 (or later) PR first needs it.
- Multi-hart concurrency model (big-lock vs. fine-grained) — explicitly out of scope,
  deferred to an SMP-scaling RFC once multi-core `lantern-hal` support exists.
- Whether/how kernel stacks are sized and where they live in the memory map —
  implementation detail, not fixed here.
- Interrupt handling latency under a fully non-preemptible kernel (a long-running kernel
  operation delays IRQ delivery to the eventual user-space driver) — acknowledged, same
  class of concern as the preemption-point question above, not separately solved here.

## Future possibilities

- The SMP-scaling RFC: extends this model to multiple harts, starting from (per seL4's
  own history) a single big kernel lock before any fine-grained locking is justified by
  measured contention.
- Machine-checked verification of the trap-dispatch/scheduling path, using this RFC's
  execution model as a precondition — consistent with the Roadmap Phase 3+ verification
  target already named in `lantern-kernel/ARCHITECTURE.md` and `THREAT_MODEL.md`.
- A concrete preemption-point design for `Revoke` and any other future fan-out
  operation, once Phase 1's prototype work reaches the capability-derivation-tree code
  that needs it.

## Resulting ADRs

None yet — this RFC is currently a Draft. On acceptance, expect a single ADR fixing the
single-stack/run-to-completion decision (and, if warranted, a short note on the
preemption-point principle), analogous to how RFC-0005 produced ADR-0008/ADR-0009.
