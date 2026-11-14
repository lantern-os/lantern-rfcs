---
rfc: 0030
title: Process lifecycle — building a process from user space, surviving its faults, and reclaiming it
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel", "hal", "boot"]
domains: ["kernel", "hal", "abi", "boot"]
created: 2026-11-13
updated: 2026-11-13
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-13, implemented and recorded in [ADR-0034](../adr/0034-process-lifecycle-faults-and-reclamation.md).**
> The unresolved questions were settled as proposed:
>
> - A fault is reported through a notification, with `ReadFault` for the details.
> - **The kernel zeroes every Frame when it's retyped**, through an S-mode-only direct map of RAM.
> - `Reset` is one bounded call. Under QEMU, a child's 4 MiB slot resets in 30–50 µs, and a whole 8 MiB `Untyped` with its
>   child in 45–60 µs.
>
> **Found while implementing:** `FrameInvoke::Map` and `Unmap` never flushed the TLB. A page a program unmapped from its own
> address space stayed reachable through a cached translation. They now flush when they change the running table.
> Section 4's "memory isn't cleared by `Reset`" stands: memory is cleared when it's next retyped instead.

# RFC-0030: Process lifecycle — building a process from user space, surviving its faults, and reclaiming it

## Summary

[RFC-0029](./0029-a-confined-launcher.md)'s launcher was agreed on condition that reclamation comes first. Reclamation turns
out to need two more things, so this RFC makes the kernel ready for programs that come and go:

1. **Building a process from user space.** A new VSpace gets the kernel's own mappings from a template, and a new `TCB`
   invocation, `SetEntry`, sets a thread's starting registers. Today boot code does both by writing kernel state directly.
2. **Surviving a program's faults.** Today any fault in a user program, a page fault or an illegal instruction, **parks the
   whole hart**. After this RFC, the faulting thread is suspended, and whoever holds its fault notification is told. The rest
   of the system carries on.
3. **Reclaiming.** An `Untyped` can be split into bounded children, and **`Reset`** on one destroys every object made from it,
   clears every capability to them in every CNode, unwinds every kernel reference to them, and returns their pool slots, page
   tables and memory. The launcher gives each app its own child `Untyped`, and resets it when the app ends.

It's deliberately not a general capability-derivation tree. Objects die with the memory they were made from, which is
seL4's model for memory, and it is bounded: every step is a scan of fixed-size pools.

## Motivation

- **One faulty app stops everything.** The HAL parks on any trap it doesn't understand. Under RFC-0029, a person runs apps of
  their choosing, and one stray pointer in any of them would freeze the machine. Fault isolation is the minimum for
  running programs you don't fully trust, and a microkernel's whole argument is that a component failing doesn't take down
  the rest.
- **Nothing can be given back.** `Revoke` is refused and `Delete` frees no object (`lantern-kernel/STATUS.md`). Pools,
  page tables and memory only ever shrink, so the launcher could run each of its four slots once per boot.
- **Process creation is a boot-time privilege.** The loader writes the kernel's mappings into page tables, and registers into
  TCBs, directly.

Principles served: **security by architecture** (a fault is contained to its program, and the authority to destroy is the
authority over the memory), and the small-kernel thesis (a bounded scan instead of a derivation tree). In tension: **TCB size**.
This is the largest kernel change since Phase 1.

## Guide-level explanation

```
 launcher: Untyped(app memory) ── Retype(Untyped, 16 MiB) ──▶ slot k's Untyped
           slot k's Untyped ── Retype ──▶ VSpace, CNode, TCB, Frames ...   (the kernel copies its mappings into the VSpace)
           TCB ── SetFaultNotification(launcher's notification, badge k) ;  TCB ── SetEntry(pc, sp, arg0) ;  TCBConfigure ──▶ runs

 the app: finishes and signals ... or faults:  kernel suspends it, signals badge k ── launcher wakes
 launcher: slot k's Untyped ── Reset ──▶ every object made from it is gone; slot k is ready again
```

## Reference-level explanation

### 1. Building a process from user space

- **The kernel-mapping template.** Boot registers, once, which root-table entries every VSpace must share: the kernel image,
  the UART and the PLIC, S-mode-only. `UntypedRetype` of a `VSpace` copies them into the new root. The kernel learns no
  layout; it copies what it was given at boot. `launch::map_kernel_shared` uses the same template.
- **`TCB::SetEntry(pc, sp, arg0)`**, valid only while the thread is `Inactive`. `TCBConfigure` then admits it, as now.
- **`TCB::SetFaultNotification(ntfn)`**, with a nonzero badge, as for `IrqHandler` and `Deadline`.

### 2. Faults

- The HAL reports a synchronous exception from U-mode as a new `TrapCause::Fault { cause, value }` (`scause` and `stval`),
  instead of parking. A trap from S-mode still parks: that's a kernel bug, and continuing would be worse.
- The kernel makes the faulting thread `Inactive`, takes it out of every queue, records the cause and value on it (a
  `TCB::ReadFault` invocation returns them), and signals its fault notification, if set. Then it schedules the next thread.
- **A fault never resumes the thread.** There's no handler that fixes it and retries: no demand paging, no copy-on-write.
  A program that faults is finished, and whoever started it decides what happens next.

### 3. Child Untypeds

`UntypedRetype` gains `Untyped` as a target, with a size: it carves that much from the parent's remaining memory, aligned to
the size, into a new `Untyped`. Every object retyped from an `Untyped` records it as its **origin**, so the kernel knows what
was made from what, as a tree of `Untyped`s with objects at its leaves. The pool of `Untyped`s grows from 4 to 16.

### 4. `Reset`

An invocation on an `Untyped` with `WRITE`. It **destroys every object whose origin is this `Untyped` or a descendant of it**,
then rewinds its memory to empty. For each destroyed object:

| Object | Unwinding |
|---|---|
| Every capability to it, in every CNode | Cleared to `Null`. A scan of every CNode's slots |
| TCB | Removed from the ready queue, every endpoint and notification queue, and every `reply_to` link that names it. If it's the current thread, the kernel switches away without saving it |
| Endpoint, Notification | Every thread blocked on it resumes with an error, `InvalidCapability`. An `IrqHandler`, `Deadline` or fault binding to it is cleared |
| VSpace | Its page tables return to the kernel's page-table pool, freed rather than bump-allocated. A thread still using it, which can only be one made elsewhere, is suspended |
| Frame | Unmapped from every VSpace it's mapped in, including another program's. See "A Frame shared across the boundary" |
| CNode, SchedContext | Their pool slots are freed |
| Child `Untyped` | Reset first, recursively, then freed |

Objects that boot code created, and objects made from an `Untyped` nobody resets, are never touched. `IrqHandler`s and
`Deadline`s are boot-created, so a `Reset` can't destroy them.

**Bounded:** a `Reset` scans every pool and every CNode slot once per destroyed object at most. That's a few thousand
steps, so the kernel stays non-preemptible (ADR-0010), and the cost is stated and measured.

**Memory isn't cleared by `Reset`.** The kernel clears each Frame when it's next retyped (unresolved question 2, as
decided), so no program ever sees what the last one left.

### 5. A Frame shared across the boundary

A `Reset` unmaps a destroyed Frame from every VSpace, including one belonging to a program that outlives it. With page faults
now contained, a program that touches such a page faults and is suspended, rather than freezing the machine. But a service
must never be put in that position: **a service's shared Frames must not come from an app's memory.** RFC-0029 is amended
to say so. The launcher allocates each slot's channel Frames once, from its own memory, maps them into the services for good,
and maps them into whichever app occupies the slot. It clears them between apps.

### 6. The proof

- **Host tests** for each row of section 4's table, for faults, and for the page-table pool's freeing. They use the fake
  controller the trap glue is already tested with.
- **A QEMU demo, `lantern-boot-lifecycle-demo`**: a confined "nursery" program, holding an `Untyped` and nothing of the
  boot loader's, builds a child process from an embedded ELF. It does this **more times than any pool has room for**, so that
  without reclamation it must fail. Some children finish; some fault on purpose: a page fault, an illegal instruction, a store
  to a read-only page. The nursery is told each time, `Reset`s, and goes on. A spinner runs throughout, and the system must not
  stop.
- **Regression:** every existing demo and suite.
- **Mutation tests:** a `Reset` that leaves one capability in place (the next child must see a stale capability, and the test
  must catch it); one that doesn't free a pool slot (the demo runs out); and a fault that parks the hart (the demo stops).

## Threat model impact

- **Trust boundaries affected:** a fault is contained to its program. The holder of an `Untyped` can destroy everything made
  from it, which is the authority over that memory, made explicit.
- **New adversary capabilities:** a launcher, or any holder of an `Untyped`, can tear down what it made, including a Frame
  mapped into another program. Section 5's rule keeps services out of that position, and faults are now contained either way.
- **Mitigated:** a faulting program can no longer stop the system. Pools and memory are no longer exhausted by launching.
- **Net change to attacker surface: smaller** in practice (no single-fault halt, no exhaustion), for a larger kernel.

## TCB impact

Yes, the largest since Phase 1. In `lantern-hal`: reporting U-mode faults instead of parking. In `lantern-kernel`: origins on
every object, child `Untyped`s, `Reset` with its unwinding, the page-table free list, `SetEntry`, fault notifications, and the
mapping template. All of it is Rust, bounded, and allocation-free. It's still far smaller than a derivation tree.

## Privacy impact

None, once the kernel clears every Frame on retype: a Frame reused after `Reset` never shows a program what its last
owner left in it.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** Larger. Safer in the way that matters most for apps: a fault ends a
   program, not the system. Reclamation is a scan, not a tree.
2. **Can an agent use this capability without being unnecessarily trusted?** An agent holds no `Untyped`. It can be torn
   down, but can't tear anything down.
3. **Same interface for humans and agents?** Yes. A person's app and an agent are built, contained and reclaimed the same way.

## Alternatives considered

- **A full capability-derivation tree**, as seL4 has: `Revoke` on any capability removes everything derived from it. It's more
  general, but it's a linked structure threaded through every slot. Service-side badge revocation (RFC-0021) already covers
  grants, so this RFC doesn't need it.
- **Reference counting**, where deleting the last capability destroys the object. It's natural, but every copy becomes a count
  to keep right, and memory still can't be returned from a bump allocator piecemeal.
- **Fault handlers that resume**, as seL4's fault endpoints allow. Needed for demand paging one day, but nothing needs it now,
  and "a fault ends the program" is easier to get right.
- **Kill the whole system on any fault**, as now. Not acceptable once a person chooses what runs.

## Prior art

- seL4: retype from `Untyped`, revoking an `Untyped` to reclaim, and fault endpoints.
- Genode: a parent that owns its children's resources and is told when they fault.
- Fuchsia: process exception channels, and job hierarchies whose teardown frees everything below them.

## Unresolved questions

1. **Faults: a notification, as proposed, or an endpoint carrying the details?** This RFC proposes a notification plus
   `ReadFault`, which keeps one waiting pattern everywhere.
2. **Who clears reused memory?** This RFC proposes **the kernel**: it zeroes a Frame when it's retyped. For that, the kernel
   maps all RAM S-mode-only in its own half (a direct map, through the same template), and no program can ever be handed old
   contents. The alternative is the launcher, as the boot loader does today, which keeps general memory out of the kernel's
   view but makes every holder of an `Untyped` responsible for clearing.
3. **`Reset` cost:** state a bound and measure it, or cap the work per call and make `Reset` resumable? This RFC proposes the
   former for v0, since the pools are small.

## Future possibilities

- A derivation tree and a general `Revoke`, if a need appears that `Untyped` origins can't meet.
- Resumable faults, for demand paging and copy-on-write.
- Per-app memory accounting visible to a person.

## Resulting ADRs

On acceptance, an ADR will record: the kernel-mapping template, `SetEntry`, fault suspension and notification, child
`Untyped`s and origins, `Reset` and its unwinding, the page-table free list, the rule that services' shared Frames never
come from an app's memory, and the decision on clearing memory.
