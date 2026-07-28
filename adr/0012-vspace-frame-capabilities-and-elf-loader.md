---
adr: 0012
title: VSpace/Frame capabilities and a minimal ELF loader
status: Accepted
date: 2026-07-27
deciders: ["TSC"]
rfc: ../rfcs/0008-vspace-frame-capabilities-and-elf-loader.md
supersedes: []
superseded_by: null
---

# ADR-0012: VSpace/Frame capabilities and a minimal ELF loader

## Context

RFC-0005 fixed the syscall/IPC ABI but explicitly deferred "VSpace/Frame invocation label
tables" as an implementation detail. `lantern-kernel/STATUS.md` named VSpace/Frame objects
as the next item, needed for actual confinement — the current QEMU demo proves the IPC
*mechanism*, not RFC-0004's actual Phase 1 exit criterion ("a confined user-space 'hello
service' reachable only via a granted capability"), since both demo threads are compiled
directly into `lantern-boot` and wired together by hand, and address-space assignment is a
direct `Tcb.address_space` field poke rather than anything capability-checked.
[RFC-0008](../rfcs/0008-vspace-frame-capabilities-and-elf-loader.md) proposed closing both
gaps together; it has been accepted. This ADR fixes the durable decision.

## Decision

**VSpace and Frame become real, retyped capability objects; a new `FrameInvoke` syscall
maps them; `lantern-boot` gains a minimal ELF64 loader that uses both.**

- **VSpace** names one Sv39 root page table (`root: usize`, a physical address).
  **Frame** names one physical page in one of two explicit size classes — `Small` (4 KiB)
  or `Mega` (2 MiB) — plus at most one current mapping (`mapped_at: Option<(VSpaceId,
  usize)>`; Phase 1 has no shared-frame IPC, so a Frame is never aliased into two VSpaces
  at once). Both are retyped from Untyped (`ObjectType` gains `VSpace`/`FrameSmall`/
  `FrameMega`), exactly like every other kernel object.
- **`FrameInvoke`** (13th syscall number) is invoked on a Frame capability, `Map`
  (`mr1` = target VSpace, `mr2` = vaddr, WRITE required on both) or `Unmap`, dispatched on
  the message tag's `label` the same way `CNodeInvoke` already is. `lantern-hal` gains
  matching unmap primitives (`riscv64_unmap`/`riscv64_unmap_megapage`) alongside its
  existing `map`/`map_megapage`.
- **`TCBConfigure`** gains a fourth argument (`mr3`): a VSpace capability, or `0` for a
  thread with no address space of its own (unchanged `Tcb.address_space: None` meaning) —
  retiring the direct field poke `lantern-boot`'s demo used.
- **Frame's size class defaults to `Mega` exclusively for now**, an explicit, documented
  workaround for the QEMU 3-level-Sv39-walk limitation `lantern-hal/STATUS.md` records —
  not a property of the object model, which keeps `Small` fully specified for when that
  environment issue is resolved.
- **`lantern-boot` gains a minimal, hand-written ELF64 loader** (no external parsing
  crate) — `PT_LOAD` segments only, strict header/bounds validation, everything else
  rejected rather than ignored. It loads a genuinely separate, independently built riscv64
  binary (own linker script, no dependency on `lantern-hal`/`lantern-kernel`) embedded via
  `include_bytes!`, and constructs each loaded thread's CSpace one capability at a time —
  the actual narrowing-waterfall root-task pattern RFC-0002 describes, not the symmetric
  hand-wiring the current demo does.
- **Untyped's real-memory backing is a narrow, additive change, not a general
  physical-memory-discovery subsystem.** The existing count-based `remaining` budget is
  unchanged and still applies to every object type. Specific Untyped capabilities
  `lantern-boot` constructs at boot may *additionally* carry a real physical bump range
  (`Untyped.memory: Option<(next_free, end)>`, sourced from `lantern-boot`'s existing
  `pmm` allocator) that `VSpace`/`Frame`/`FrameSmall`/`FrameMega` retype requires and bump-
  allocates from. Real DTB-based physical memory discovery remains separately tracked and
  out of scope here, per RFC-0008's own "Unresolved questions."

## Consequences

- **Easier:** address-space construction is now capability-checked and auditable the same
  way every other kernel operation already is; a future userspace root task can build
  address spaces over IPC using the exact same `FrameInvoke`/`TCBConfigure` path
  `lantern-boot`'s own loader uses, rather than needing kernel-internal access.
- **Harder:** the loader is real, meaningfully sized TCB code (ELF parsing, segment
  copying, per-segment Frame retype/map) — this is the cost of Phase 1's actual exit
  criterion, not incidental complexity. Building and embedding a genuinely standalone
  second riscv64 binary adds a build-tooling step `lantern-boot` didn't previously need.
- **Committed to:** `Frame`'s single-mapping invariant (no shared-frame IPC yet — deferred
  per RFC-0008); the hand-written, deliberately narrow ELF parser over a general-purpose
  crate; `FrameSize::Mega` as the loader's only size class until the QEMU limitation is
  resolved, at which point switching to `Small` is a one-line change, not an ABI break
  (both variants already exist).
- **Deferred, per RFC-0008:** real DTB-based physical memory discovery (Untyped's
  count-based budget itself is unchanged); shared-frame/zero-copy IPC; a standalone
  root-task crate (not warranted while `lantern-boot` loads only one program);
  `FrameSize::Giga`; whether the "client" reaching the loaded hello service is a second
  loaded program or `lantern-boot`'s own privileged boot code (left to implementation).
