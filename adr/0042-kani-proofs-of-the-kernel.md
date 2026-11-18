---
adr: 0042
title: Kani proofs of the kernel's capability and IPC paths, within stated bounds
status: Accepted
date: 2026-11-18
deciders: ["TSC"]
rfc: ../rfcs/0037-formal-verification-first-properties.md
supersedes: []
superseded_by: null
---

# ADR-0042: Kani proofs of the kernel's capability and IPC paths

## Context

[RFC-0036](../rfcs/0036-phase-3-to-phase-4-transition.md) made formal verification Phase 4's first work. Phase 4 can't
exit without machine-checked proofs of the most critical kernel paths.
[RFC-0037](../rfcs/0037-formal-verification-first-properties.md) chose the tool and the first properties. It has been
accepted, and this ADR is the durable record.

## Decision

**The kernel's capability and IPC paths are proved with Kani (bounded model checking of the Rust as written), within
stated bounds, by harnesses that live beside the code and run before a kernel change merges.**

- **The invariant W** (`lantern-kernel/src/wf.rs`). Every capability names a live object of its type, and none is a stored
  reply. Every reference a thread holds is live, and so is any capability it has in flight, which also has `GRANT`. Pool
  and queue bookkeeping is exact. Every thread is in at most one queue, the one its state names. The current thread is
  `Running` and in none, and no other thread is `Running`. Origins and bindings are live. W is checked after every
  syscall and trap in debug builds and host tests. Release kernels don't check it.
- **The properties** (RFC-0037's nine):
  1. no capability is amplified;
  2. none is forged;
  3. a capability moves over IPC only with `GRANT`, and exactly as sent;
  4. a lookup reads only the caller's own CSpace;
  5. `Reset` leaves nothing dangling;
  6. `Reset` destroys exactly what came from the `Untyped`;
  7. W holds after every syscall and every trap;
  8. a bystander's registers are untouched;
  9. nothing panics, overflows or indexes out of bounds.

  Each holds, within the bounds, in every harness it applies to.
- **How the kernel is written.** Pooled objects are read and written through `Pool::with` and `with_mut`, slots through
  `CNode::set`, and queue elements through `get_at` and `set_at`. Under Kani these find their target by comparison, and
  that they agree with the direct lookups is proved. Under Kani there is no `get`, `get_mut` or `slot_mut`, so the
  compiler flags any new code that returns a reference from a lookup.
- **Bounds and assumptions.**
  - Pool sizes are those in RFC-0037: 3 CNodes of 4 slots, 3 threads, and 1 or 2 of everything else.
  - Two opaque register words.
  - No kernel page-table pages, so making a VSpace is covered only as far as its refusal.
  - Boot-time memory ranges don't wrap the address space.
  - The trap harnesses have another thread ready, so the idle loop is outside the proofs.
  - Each harness fixes what its operation doesn't touch.
  - The hardware, the `unsafe` page-table code, the trap entry assembly, the compiler and QEMU are outside the proofs.
- **Each property can fail.** Five reintroduced bugs are each caught by the property meant to catch them.
  `lantern-kernel/scripts/prove.sh` runs every harness under a 16 GB cap and a 40-minute limit; either counts as a
  failure.

## Consequences

- **Found and fixed:**
  - `Untyped::bump` could overflow, which would have panicked the kernel.
  - A thread whose own resources `Reset` destroyed stayed current, naming freed slots.
  - A capability in flight survived `Reset` of its object, so a receiver could have been handed an object nobody
    granted.

  Each has a host test.
- **Harder:**
  - Kernel code must keep the accessor discipline above.
  - A new operation needs a harness, or several, before Phase 4's exit RFC can claim it.
  - A full proof run takes hours.
- **Not done:**
  - Unbounded proofs: the small-scope hypothesis covers full-size pools, but it isn't proof. Phase 4's exit RFC
    decides which properties, if any, need Verus or similar.
  - The idle loop.
  - VSpace creation beyond its refusal.
  - The HAL and the boot path.
