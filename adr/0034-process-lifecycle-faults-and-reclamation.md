---
adr: 0034
title: Process lifecycle — building, faults and reclamation
status: Accepted
date: 2026-11-13
deciders: ["TSC"]
rfc: ../rfcs/0030-process-lifecycle-faults-and-reclamation.md
supersedes: []
superseded_by: null
---

# ADR-0034: Process lifecycle — building, faults and reclamation

## Context

[RFC-0029](../rfcs/0029-a-confined-launcher.md)'s launcher was agreed on condition that reclamation came first. Three things
stood in its way:

- Only boot code could build a process, by writing kernel state directly.
- Any fault in any user program parked the whole hart.
- Nothing could be given back: `Revoke` was refused, and `Delete` freed nothing.

[RFC-0030](../rfcs/0030-process-lifecycle-faults-and-reclamation.md) proposed all three, and has been accepted. This ADR is the
durable record.

## Decision

**A program can build a process from an `Untyped` with system calls alone. A fault ends the program, not the system. `Reset`
on an `Untyped` destroys everything made from it, and returns its pool slots, page tables and memory.**

- **Building.**
  - Boot registers a kernel-mapping template once: the kernel's megapages, the UART, the PLIC, and a direct map of RAM. The
    direct map is S-mode-only, at `0xffff_ffc0_0000_0000`, in one root entry shared by every VSpace.
  - Every VSpace retype installs the template. The kernel learns no layout: it copies what it was given.
  - `TcbInvoke` is syscall 16:
    - `SetEntry(pc, sp, arg0)`, only while the thread is `Inactive` and has never faulted.
    - `SetFaultNotification`, with a nonzero badge.
    - `ReadFault`, which returns `(scause, stval)`, or `(0, 0)` if the thread hasn't faulted.
- **Faults.**
  - The HAL reports a synchronous exception from U-mode as `TrapCause::Fault`. A trap from S-mode still parks: that's a kernel
    bug.
  - The kernel makes the thread `Inactive`, records the fault, and fails any caller it owed a reply. It signals the thread's
    fault notification, and runs the next thread.
  - A faulted thread is never resumed.
- **Child `Untyped`s and origins.**
  - `UntypedRetype` to an `Untyped` carves a megapage-multiple child, aligned to a megapage, which inherits device-ness. Device
    memory is never written.
  - Every retyped object records the `Untyped` it came from. Objects boot code makes directly have no origin, so `Reset`
    never reaches them.
  - `MAX_UNTYPEDS` grows from 4 to 16.
- **`Reset`** is `CNodeInvoke::Revoke` on an `Untyped` with `WRITE`. On anything else, `Revoke` is still refused. It:
  - marks the `Untyped`'s descendants, and every object whose origin is one of them;
  - switches to a kernel-only root if the hart is running on a VSpace that's going;
  - unlinks dying threads from every queue and `reply_to`;
  - suspends a surviving thread that lost its CSpace, scheduling context or VSpace;
  - fails threads blocked on a dying endpoint or notification with `InvalidCapability`;
  - clears `IrqHandler`, `Deadline` and fault bindings to a dying notification;
  - unmaps dying Frames from surviving VSpaces;
  - returns dying VSpaces' page tables to the kernel's page-table free list;
  - nulls every capability to anything dying, in every surviving CNode;
  - frees the pool slots;
  - rewinds the `Untyped`;
  - and flushes the TLB.

  It's a scan of fixed-size pools, bounded by their sizes, so the kernel stays non-preemptible (ADR-0010).
- **The kernel clears memory.** Every RAM Frame is zeroed through the direct map when it's retyped, whoever holds the
  `Untyped`. No program is ever handed another's old memory, and boot code no longer clears Frames itself.
- **Map and Unmap flush the TLB** when they change the table the hart is running on. Found while building the proof: before
  this, a page a program unmapped from its own VSpace stayed reachable through a cached translation, and a page it mapped
  could read as its previous mapping.
- **A service's shared Frames never come from an app's memory**, since resetting the app unmaps them. RFC-0029 is amended
  to say so.

## Consequences

- **Easier:** a launcher can run apps without limit, from bounded memory, and tear down each one completely. A crashing
  program costs nothing but itself, and whoever started it is told why.
- **Measured, under QEMU** (`lantern-boot-lifecycle-demo`, a nursery building 40 children of 11 objects each). A
  `Reset` of one child's 4 MiB slot takes 30–50 µs. One of the nursery's whole 8 MiB, with its child `Untyped`, takes
  45–60 µs. The first `Reset` of a boot takes about 1.5 ms. That looks like QEMU translating the code for the first time,
  and isn't counted. Nothing has been measured on hardware.
- **Harder / committed to:**
  - Every object carries an origin.
  - A `Reset` scans every pool and every CNode slot.
  - Page tables are freed, so they need a free list.
  - The kernel can now write any RAM page.
- **TCB:** the largest growth since Phase 1. `lantern-hal` gains fault reporting and the direct map. `lantern-kernel` gains
  `lifecycle.rs`, `vspace.rs` and `reclaim.rs`, origins, child `Untyped`s, the page-table free list, and the TLB flush. Still
  no derivation tree: objects die with their memory.
- **Still not done:**
  - A general `Revoke` over a derivation tree.
  - Resumable faults, for demand paging and copy-on-write.
  - A `Reset` that's resumable, or capped per call.
  - Per-app memory accounting visible to a person.
