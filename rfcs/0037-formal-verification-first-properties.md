---
rfc: 0037
title: Formal verification — the tool, and the first properties of the capability and IPC paths
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel", "capabilities", "governance"]
domains: ["kernel", "capabilities", "hal"]
created: 2026-11-16
updated: 2026-11-18
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0037: Formal verification — the tool, and the first properties of the capability and IPC paths

> **Accepted 2026-11-18, implemented and recorded in [ADR-0042](../adr/0042-kani-proofs-of-the-kernel.md).** All nine properties are
> proved by Kani within the bounds below, in 84 harnesses, each shown able to fail. Built as proposed, with the changes in
> "As implemented". The proofs found three kernel bugs, each now fixed with a host test.
>
> **Direction agreed 2026-11-16:**
> - The tool is Kani.
> - All nine properties are in the first implementation.
> - W is also checked at run time, after every syscall, in debug builds and host tests. Release kernels are unchanged.

## Summary

[RFC-0036](./0036-phase-3-to-phase-4-transition.md) made formal verification Phase 4's first work. Phase 4 can't exit
without "machine-checked proofs of the most critical kernel paths", and none of it needs hardware. This RFC chooses:

- **a tool**;
- **the first properties** to prove, about the capability operations, `Reset`, and the scheduler's and IPC's queues;
- **how the proofs live**: beside the kernel's code, and run as part of its checks, so they can't fall behind the code they
  describe.

It also says what these proofs will *not* establish, so that "verified" is never read as more than it is.

## Motivation

The kernel is about 6,200 lines of Rust (`lantern-kernel/src`). Its tests are thorough, and still they missed three bugs
that QEMU found the hard way:

- a thread could sit in a ready queue twice;
- the first thread to run was never removed from the ready queue, so it could pop itself;
- `Yield` wrote its result into the *next* thread's tag register, which changed one bit of an app's `a4` about one boot
  in four (fixed in RFC-0035's round).

Each broke an invariant that holds in every reachable state, and each needed one particular interleaving to show. Tests
check the interleavings someone thought of. A model checker checks all of them, up to a bound.

The capability model is the other reason. ADR-0005 rests the whole system on monotone attenuation: authority can be
narrowed, never widened or conjured. Today that is enforced by a few `if` statements in `cnode.rs` and `ipc.rs`. It
should be proved.

## The kernel is a good fit

- **No heap and no pointers between objects.** Every object lives at an index in a fixed-size pool (`pool.rs`), and a
  capability is a `Copy` value naming an index. The whole kernel state is a finite value.
- **The operations are short.** `mint`, `copy`, `move`, `delete` and `copy_cross` are each under 30 lines, functions of
  `&mut KernelState` and a few words.
- **The hardware is already behind a seam.** `handle_trap<H: Hal>` is tested with a `FakeHal`, and the kernel already
  reads the clock once per trap. Proofs can use the same seam.
- **Little `unsafe`.** There are 37 uses, in 7 files: page-table writes, CSR access and the trap path. The properties
  here don't depend on them, and those calls are replaced by stubs in proofs (below).

## The tool

| Tool | What it proves | Cost |
|---|---|---|
| **Kani** (bounded model checking of Rust as written, via CBMC) | Every execution up to a bound: within pools of a given size, *all* states and inputs, not samples. Also panics, overflow and undefined behaviour in safe code | Proof harnesses beside the code. No rewrite. Runs from `cargo kani` |
| **Verus** (SMT-based verification of Rust with specifications) | Unbounded functional correctness: for all pool sizes | The verified code is written in Verus's dialect, with ghost state and loop invariants. Substantial rewriting, and a second toolchain for kernel code |
| **Aeneas → Lean**, or an seL4-style refinement proof | The strongest: a proof against an abstract specification | Months to years. seL4's proof took about 20 person-years |

**Proposed: Kani first.**

- It checks the code that actually runs, not a copy written in another dialect, so the proofs can't drift from the
  kernel.
- It finds the class of bug that has actually hurt here.
- It can start this week.

Its limit is the bound: a proof covers pools up to the sizes the harness uses. The kernel's code is generic in those
sizes. It loops over pools and never special-cases a size. That makes the "small-scope hypothesis" reasonable: bugs show
up in small configurations. It is still a hypothesis, not a proof, and this RFC records it as such.

Verus or Lean stay open for the properties where a bound isn't enough. Phase 4's exit RFC decides whether bounded proofs
meet "machine-checked proofs of the most critical kernel paths", or which properties need unbounded ones.

## The first properties

Each is a Kani harness. Each starts from a **nondeterministic** kernel state: any contents of small pools and CNodes that
satisfy the kernel's well-formedness invariant (W, below). It applies one operation with nondeterministic arguments, and
asserts the property.

**W, well-formedness.** The invariant every other property assumes and must preserve:

- every capability names a live object of its type;
- a pool's `len` matches its occupied slots;
- every thread is in at most one queue: the ready queue, one endpoint's queue, or a notification's waiters;
- the current thread is in none of them;
- a thread's `state` matches the queue it is in;
- every object's recorded origin is a live `Untyped`, or none.

**Capabilities (`cnode.rs`, `ipc.rs`):**

1. **No amplification.** After any `CNodeInvoke` (Mint, Copy, Move, Delete, CopyCross), every capability in every CNode
   either was there before, or names an object that some capability already named, with rights that capability already
   had. Minting may change a badge; it can never add a right.
2. **No forgery.** No operation makes a capability to an object that no capability named before, except
   `UntypedRetype`, whose new object is fresh.
3. **Transfer is granted, and exact.** An IPC `Send`, `Call` or `Reply` moves a capability only if the capability being
   sent has `GRANT` (RFC-0010). `Null`, CNode and reply capabilities can never be sent. What arrives is exactly what was
   sent, into the slot the receiver named, and only if that slot was empty.
4. **Lookup is confined.** A thread reaches capabilities only through its own CSpace: `lookup_cap(t, _)` never reads
   another thread's CNode.

**Reclamation (`reclaim.rs`):**

5. **`Reset` leaves nothing dangling.** Afterwards:
   - no capability in any surviving CNode names a destroyed object;
   - no queue holds a destroyed thread;
   - no notification is bound to a destroyed thread;
   - no surviving Frame records a mapping in a destroyed VSpace;
   - the `Untyped` is empty.

   W holds again.
6. **`Reset` destroys only what it should.** Every object it frees came from that `Untyped` or one of its descendants.
   Everything else is unchanged.

**Scheduling and IPC (`state.rs`, `scheduler.rs`, `ipc.rs`, `syscall.rs`):**

7. **Every syscall preserves W.** This is the property that would have caught the two queue bugs.
8. **A result reaches its caller.** A syscall writes its result only into the trap frame of the thread that made it. The
   other threads' saved registers are unchanged. This is the property that would have caught `Yield`.
9. **No panics.** No syscall, with any arguments in any well-formed state, panics, overflows, or indexes out of bounds.
   The kernel can't recover from a panic.

Properties 1–6 are the "capability path". Properties 7–9 are the "IPC path" that Phase 4's exit criterion names.

## How the proofs live

- **Beside the code.** Harnesses go in `lantern-kernel/src/proofs/`, compiled only under `cfg(kani)`. Building and
  testing the kernel is unchanged.
- **Reduced limits.** Under `cfg(kani)`, `limits.rs` uses small pools (the exact bounds are in "As implemented"). The
  kernel's logic is the same at both sizes; how it reaches its objects changed, as described there.
- **Hardware stubs.** Page-table walks, RAM zeroing and the host HAL's initial register frame are replaced, under Kani,
  by stubs with the same signatures that answer arbitrarily (a mapping exists or doesn't) and otherwise do nothing; the
  trap harnesses use a Kani HAL whose clock, pending interrupt and timer are arbitrary. The `unsafe` code behind those
  stubs is outside these proofs.
- **Run as their own step.** `scripts/prove.sh` runs every harness, one at a time; a harness that runs out of time or
  memory counts as a failure, not a pass. A full run takes hours, so it runs before a kernel change is merged rather than
  in the quick regression.
- **W at run time too.** In debug builds and host tests, the kernel checks W after every syscall
  (`debug_assertions`), so every test and debug QEMU run checks it too, at full pool sizes. Release kernels are
  unchanged.
- **Mutations, as for every RFC.** Each property is shown to catch a deliberately reintroduced bug. Examples: `mint`
  without its subset check, `Reset` skipping one CNode, and the original `Yield` bug. A proof that can't fail proves
  nothing.

## As implemented

**The kernel changed to become provable.** The draft said only the constants would change. Kani encodes a reference
kept past a lookup with an arbitrary index as a pointer with a symbolic offset into the whole kernel state, and every
later read becomes a selection over all of it: a trivial assertion after a CNode `Copy` needed more than 16 GB. So
(user decision, 2026-11-17):

- every kernel read and write of a pooled object goes through `Pool::with` / `with_mut`, which apply a closure inside
  the lookup, and every CNode slot write through `CNode::set`; under Kani these find their target by comparing indices,
  outside Kani they are the old lookups. About 90 call sites changed. The host tests caught two behaviour changes the
  conversion introduced (an early `return` inside a closure, and a call moved ahead of its check), and both were fixed.
- queue element accesses go through `get_at`/`set_at`; `Pool::alloc` and `free` write in place; a small array inside an
  object is copied, changed and written back whole rather than indexed in place.
- under Kani there is no `Pool::get`/`get_mut` or `CNode::slot_mut` at all, so the compiler finds any new such lookup.
- that each Kani-only accessor agrees with the one that runs, for any contents and index, is itself proved
  (`pool_lookups_agree`, `cnode_lookups_agree`, `queue_access_agrees`).

**Harnesses start from a skeleton, not from everything.** The draft's "any well-formed state" didn't fit: generating every
pool at once exhausted memory before any operation ran. Each harness starts from a small fixed state (two CNodes; one
each of `Untyped`, endpoint, notification, scheduling context; a running caller and two other threads, every thread's
registers arbitrary) and makes arbitrary exactly what its operation touches: every CNode slot, the other threads' states
and queues, device objects (a VSpace, a Frame, an interrupt line, a deadline), physical memory behind the `Untyped`, the
origins of the objects `Reset` destroys. Each harness says what it leaves fixed. Operations with many paths are a harness
per path: per CNode label, per `extra_caps` value of `Send`/`Recv`/`Call`, per object type retyped, per device label, per
object `Reset` destroys and, for a destroyed thread, per state it is in.

**The properties, as proved.**

- 1–2 (no amplification, no forgery) are checked directly for CNode operations, retype (where only fresh objects may
  appear) and the device calls. For IPC the stronger and cheaper statement is checked instead: every slot is unchanged
  except at most one empty slot that now holds, exactly, a capability with `GRANT` that was sent (which is property 3).
  For `Reset`: every slot is unchanged or now empty.
- 3 holds as stated; W gained a clause for it: a capability in flight with a blocked `Send` has `GRANT` and names a live
  object.
- 4: whatever `lookup_cap` returns, for any thread and slot, is what that thread's own CNode holds there.
- 5 is W after `Reset`; 6 is checked object by object: an object is destroyed exactly when it came from the reset
  `Untyped` or its child, and the `Untyped` is empty again.
- 7 holds for every system call and label, and for the timer, fault and external-interrupt traps.
- 8 is stated precisely: a thread that was ready or inactive before the call, and isn't named by it through a thread
  capability (`TcbInvoke` sets an inactive thread's first registers), has exactly its registers afterwards.
- 9 is Kani's own checks throughout.

**Bounds.** 3 CNodes of 4 slots, 3 threads, 2 endpoints, 2 notifications, 2 `Untyped`s, 2 scheduling contexts, 2
VSpaces, 2 Frames, 1 interrupt line, 1 deadline; 2 opaque register words (the kernel copies them only as a block); no
kernel page-table pages, so making a VSpace is covered only as far as its refusal. Loops unwind 9 times (5 for `Reset`),
and Kani's unwinding assertions fail any harness whose bound is too small. Each harness runs under a 16 GB cap
(`systemd-run`, not `ulimit -v`, which SAT solvers' address reservations defeat) and a 40-minute limit; either counts as a
failure.

**Assumptions a harness makes, beyond the skeleton.** Boot code's `Untyped` memory ranges don't wrap the address space.
The trap harnesses have another thread ready, so the trap never reaches the idle loop: that loop waits until an
interrupt makes a thread ready, which no bound covers. The idle loop is outside these proofs.

**What the proofs found.**

1. `Untyped::bump` rounded up with an unchecked addition: a range ending near the top of the address space panicked
   the kernel. Now checked (property 9).
2. A thread that reset the `Untyped` its own scheduling context, CSpace or address space came from was suspended but
   stayed current, and kept naming freed pool slots. It now stops being current, its registers and the call's answer are
   saved, and the lost references are cleared (W).
3. A capability attached to a blocked `Send` survived `Reset` of the object it named. That pool slot could be reused by
   the next retype, and the receiver would have got a capability to an object nobody granted. The send now fails (W,
   with its in-flight clause).

The tests missed all three; each now has a host test that fails without its fix.

**Each property can fail.** Deliberately reintroduced bugs, each caught by the property meant to catch it:

| Mutation | Caught by |
|---|---|
| `mint` without its subset check | "a capability with authority nothing held before" (Mint) |
| the original `Yield` bug: success written after the switch | "a bystander's registers changed" (Yield) |
| `make_ready` without its guard | W: a thread queued twice (thread configure) |
| capability transfer without `GRANT` | "a capability without GRANT moved", and W (Send with a capability) |
| `Reset` skipping one CNode | W: a dangling capability (`Reset` of an endpoint) |

**Running them.** `lantern-kernel/scripts/prove.sh` runs every harness, listed from Kani's own metadata so that
macro-generated ones aren't missed, and refuses to start if the list is shorter than the harnesses in the source. The
full run takes about 9 hours.

## What these proofs do not establish

- **Bounded, not universal.** They cover pools up to the stated sizes. The full-size kernel is covered by the small-scope
  hypothesis, not by proof.
- **Not the `unsafe` code, the HAL or the trap entry assembly.** Those are trusted, and stay reviewed rather than proved.
- **Not the compiler, the hardware, or QEMU.**
- **Not that the specification is the right one.** The properties are stated in this RFC so they can be reviewed as
  carefully as the code.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none. The proofs strengthen the evidence for the existing boundary (ADR-0004, ADR-0005):
  kernel threats K1 (capability forgery or amplification) and K3 (memory corruption) gain machine-checked evidence within
  the bound.
- **New assets, adversaries:** none.

## TCB impact  *(mandatory)*

- **Code added to the TCB:** none. Harnesses and stubs compile only under `cfg(kani)`. The limits change only in that
  configuration.
- **Dependencies:** Kani, a development tool, not linked into anything.

## Privacy impact

None.

## Alternatives considered

- **Verus from the start.** Unbounded, but it means rewriting the kernel's verified parts in Verus's dialect before
  proving anything. The cost comes first, and the payoff is uncertain. Kept for later, where a bound isn't enough.
- **An abstract model (TLA+, Alloy) of the capability system.** Cheap and expressive, but it checks a model, not the
  kernel. The bugs found here were in the code, not the design.
- **More tests instead.** The three bugs above survived thorough tests.

## Unresolved questions

- Whether bounded proofs meet Phase 4's exit criterion, or which properties need unbounded ones (decided by the exit
  RFC).
- Kani's run time at these bounds. Property 7 over every syscall may need splitting per syscall, or smaller bounds.

## Resulting ADRs

On acceptance: an ADR fixing the tool, the property set, and what the proofs do and don't establish.
