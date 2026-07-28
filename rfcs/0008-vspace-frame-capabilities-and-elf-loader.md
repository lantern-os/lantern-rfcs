---
rfc: 0008
title: VSpace/Frame capabilities and a minimal ELF loader for Phase 1's confined hello service
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel", "boot"]
domains: ["kernel", "hal", "boot"]
created: 2026-07-27
updated: 2026-07-27
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0008: VSpace/Frame capabilities and a minimal ELF loader for Phase 1's confined hello service

## Summary

This RFC fixes the last open item RFC-0005 deliberately deferred ("VSpace/Frame invocation
label tables — implementation detail, not fixed by this RFC") and defines a minimal ELF
loader in `lantern-boot` that uses it. Two new retypable capability object types —
**VSpace** and **Frame** — make address-space construction capability-mediated instead of
the direct `Tcb.address_space` field poke `lantern-boot`'s demo currently uses. A new
`FrameInvoke` syscall (`Map`/`Unmap`, dispatched the same way `CNodeInvoke` already is)
lets a Frame capability be mapped into a VSpace. `TCBConfigure` gains an optional VSpace
argument. `lantern-boot` gains an ELF loader that parses a real, independently-built
riscv64 ELF binary, retypes Frames to hold its segments, maps them into a fresh VSpace, and
grants the resulting thread only the capabilities it's explicitly given — replacing the
current demo, where both threads' bodies are compiled directly into `lantern-boot` itself
and wired together by hand.

This is what RFC-0004 calls Phase 1's actual exit criterion: "a confined user-space
'hello service' reachable only via a granted capability." The two-thread IPC demo already
proves the syscall/IPC mechanism; it does not yet prove confinement of a genuinely separate
program, or that reaching it requires a capability the loader chose to grant rather than
one the demo wired in by construction.

## Motivation

`lantern-kernel/STATUS.md`'s "Next" lists VSpace/Frame objects "needed for actual
confinement, not just the IPC mechanism the QEMU demo already proves." `lantern-hal`
already has real Sv39 page-table primitives (`riscv64_map_page`/`riscv64_map_megapage`);
what's missing is the capability layer on top of them. Without it, address-space
construction stays a privileged `lantern-boot`-internal operation with no rights check, no
budget accounting, and no way for a future userspace root task to do it over IPC — quietly
reintroducing ambient authority at exactly the layer RFC-0003/ADR-0005 committed to
gating, the same concern RFC-0005's motivation raised about the syscall boundary generally.

This also directly motivates the ELF loader: today's `client_thread`/`server_thread` are
Rust functions compiled into `lantern-boot`'s own binary and placed in a shared
`.user_text` linker section (`lantern-boot/paging.rs`'s module doc). That's a genuine
step toward real U-mode execution, but it cannot demonstrate *confinement of a separate
program* — there is no separate program. A real ELF loader, loading an independently
built and linked binary nothing else in the tree references at compile time, is the only
way to actually exercise "a component cannot reach what it wasn't handed a capability to."

## Guide-level explanation

**VSpace** is a capability naming one address space (one Sv39 root page table). **Frame**
is a capability naming one physical page usable as a mapping target, in one of two
explicit sizes: `Small` (4 KiB) or `Mega` (2 MiB). Both are retyped from Untyped, exactly
like every other kernel object (`UntypedRetype` gains two new `ObjectType` variants) —
spending real budget, capability-checked, no shortcut.

A new syscall, `FrameInvoke`, is invoked on a Frame capability (`mr0`, matching every other
capability-invocation syscall's convention) with two operations, dispatched on the message
tag's `label` exactly like `CNodeInvoke` already is:

- **Map** — `mr1` names a VSpace capability, `mr2` is the target virtual address. Requires
  `WRITE` rights on both the Frame and the VSpace. Fails if the VSpace already maps that
  address (Phase 1 has no remap/COW).
- **Unmap** — removes the Frame's mapping from whichever VSpace it's currently mapped
  into, if any.

`TCBConfigure` gains a fourth, optional argument (`mr3`): a VSpace capability, or `0`
(`Capability::Null`, matching the existing "empty slot" convention) for a thread that
stays kernel-resident with no address space of its own — the same meaning
`Tcb::address_space: None` already has today, now reachable through the syscall interface
instead of only through `lantern-boot` writing the field directly.

`lantern-boot` gains a minimal ELF64 loader: given a `&[u8]` (Phase 1: embedded at build
time via `include_bytes!`, not read from any storage device — see "Reference-level
explanation"), it validates the ELF header, walks `PT_LOAD` program headers, retypes one
Frame per page each segment spans, copies the segment's bytes in, maps each Frame into a
fresh VSpace with that segment's permissions, and configures a fresh TCB with that VSpace,
an entry point taken from the ELF header, and a CSpace containing *only* the capabilities
the loader explicitly places there. The loader plays the "root task" role RFC-0002's
narrowing waterfall describes: it starts with (nearly) unlimited Untyped budget and hands
out exactly as much authority as each loaded program needs, not more.

## Reference-level explanation

### VSpace/Frame object shapes

```rust
pub struct VSpace {
    /// Physical root page-table address — riscv64: an Sv39 root, built via
    /// lantern-hal's riscv64_map_page/riscv64_map_megapage.
    pub root: usize,
}

pub enum FrameSize {
    Small, // 4 KiB — lantern_hal::RISCV64_PAGE_SIZE
    Mega,  // 2 MiB — lantern_hal::RISCV64_MEGAPAGE_SIZE
}

pub struct Frame {
    pub paddr: usize,
    pub size: FrameSize,
    /// Which VSpace (if any) currently maps this Frame, and at what address —
    /// Unmap's target, and what stops a Frame being mapped twice at once
    /// (Phase 1 has no shared-frame/zero-copy IPC yet; one mapping at a time
    /// keeps ownership unambiguous).
    pub mapped_at: Option<(VSpaceId, usize)>,
}
```

`FrameSize` is a real, forward-compatible size class, not a QEMU-specific hack: `Small`
uses `lantern-hal`'s already-correct, host-tested `map`/4 KiB path.
`lantern-hal/STATUS.md` records an empirically-confirmed limitation in this project's
current QEMU environment (Debian's `qemu-system-riscv64` 10.2.1) where a full 3-level Sv39
walk reliably faults, while a one-branch-hop walk (`Mega`) does not — so **this RFC's
loader uses `FrameSize::Mega` exclusively for now**, as an environment workaround recorded
here and in `lantern-boot/STATUS.md`, not as a property of the VSpace/Frame object model
itself. `Small` stays fully specified and implemented so switching is a one-line change
once the environment issue is resolved.

### `ObjectType` extension

`UntypedRetype`'s `ObjectType` enum (`lantern-kernel/src/cap.rs`) gains:

```rust
VSpace,
FrameSmall,
FrameMega,
```

(Two `Frame*` variants rather than packing a size argument into `UntypedRetype`'s already-
fully-used `mr1`/`mr2` — `mr1` is the object type, `mr2` the destination slot; there is no
third register free for a size argument without breaking `UntypedRetype`'s existing shape,
which this RFC has no reason to touch.)

### `FrameInvoke` syscall

A 13th syscall number (`lantern-kernel/src/syscall.rs`'s `SyscallNumber`), dispatched the
same way `CNodeInvoke` is (`crate::frame::invoke`, mirroring `crate::cnode::invoke`'s
shape):

```rust
pub const LABEL_MAP: u32 = 1;
pub const LABEL_UNMAP: u32 = 2;
```

`Map`: resolves `mr0` to a `Capability::Frame` (WRITE required), `mr1` to a
`Capability::VSpace` (WRITE required), reads `mr2` as the target vaddr. Fails
(`IllegalOperation`) if the Frame is already mapped anywhere, or if the target vaddr in
that VSpace is already occupied (Phase 1 does not walk the target table to merge/replace —
same "fail rather than silently do something surprising" discipline `admin.rs` already
uses for `UntypedRetype`'s destination-slot check). Calls
`lantern_hal::riscv64_map_megapage`/`riscv64_map_page` per the Frame's size, then records
`mapped_at`.

`Unmap`: resolves `mr0` to a `Capability::Frame` (WRITE required). No-op (success) if not
currently mapped. Phase 1 has no PTE-clearing primitive in `lantern-hal` yet
(`riscv64_paging.rs`'s `map`/`map_megapage` only ever write a *new* leaf) — this RFC adds
one (`riscv64_unmap`/`riscv64_unmap_megapage`, walks to the leaf and clears it, then the
caller is responsible for `sfence.vma` on next activation, same as every other mapping
change today).

### `TCBConfigure` extension

`mr3` (previously unused) becomes the VSpace argument: `0` means
`Capability::Null` → `Tcb.address_space = None` (unchanged default, unchanged meaning);
any other value must resolve to a `Capability::VSpace` (WRITE required) →
`Tcb.address_space = Some(vspace.root)`. `Tcb::address_space`'s field itself is unchanged
(`lantern-kernel/src/object.rs`, added this session) — this RFC is what finally gives it a
capability-mediated setter, retiring the doc comment that currently says "not yet
capability-mediated... `lantern-boot` sets this directly."

### The ELF loader

Lives in `lantern-boot` as a new module (`elf.rs` — parsing — plus a rewritten
`demo.rs`/`loader.rs` that replaces today's hand-built two-thread setup), not a new crate.
`lantern-boot/ARCHITECTURE.md` already states "the loader is part of the TCB"; this is
that same TCB, gaining a new capability, not a new component with its own trust boundary.
A future phase may split a genuine root-task crate out of `lantern-boot` once there's more
than one thing it needs to load (see "Future possibilities") — not warranted yet.

Minimal ELF64 support only: validates `e_ident` (magic, `ELFCLASS64`, `ELFDATA2LSB`,
`EM_RISCV`), walks `e_phoff`/`e_phnum` program headers, handles `PT_LOAD` only (anything
else — `PT_DYNAMIC`, `PT_INTERP`, `PT_NOTE`, ... — is rejected, not silently ignored: this
loader only ever needs to run a statically linked, position-dependent, no-libc binary,
and silently ignoring a segment type it doesn't understand is exactly the kind of "parse
untrusted structure permissively" mistake `lantern-boot/THREAT_MODEL.md` should not
reintroduce). Per-segment: round the `[p_vaddr, p_vaddr + p_memsz)` range out to whole
Frames, retype one `FrameMega` per Frame-sized chunk, copy `p_filesz` bytes from the ELF
image starting at `p_offset` (zero-filling the `p_memsz - p_filesz` BSS tail, and the
partial Frame at each end), map each with `PteFlags` derived from `p_flags`
(`PF_R`/`PF_W`/`PF_X` → `READ`/`WRITE`/`EXECUTE`, always plus `USER` — this loader only
ever builds thread-owned, U-accessible mappings) into a freshly retyped VSpace.

**Where the ELF bytes come from:** embedded via `include_bytes!` at build time, exactly
matching how RFC-0004/`lantern-boot/STATUS.md` already justify deferring measured boot —
"Phase 1 links `lantern-kernel` directly as a library rather than loading/verifying a
separate signed image... since there's no separate image and no exit criterion needing
that trust chain yet." The image being *embedded* rather than *linked in as a Rust
library* is exactly what makes this a real ELF-loading exercise rather than a repeat of
today's demo: `lantern-boot` never sees the hello service's source, only its compiled
bytes, walked exactly the way it would need to be walked if those bytes had come from a
disk image instead. Concretely: a new, genuinely standalone `#![no_std] #![no_main]`
riscv64 binary (own linker script, no dependency on `lantern-hal`/`lantern-kernel` — it
only ever issues raw `ecall`s, the same convention `demo.rs`'s `syscall()` helper already
uses) built as its own cargo target, whose output `lantern-boot`'s build embeds. Exact
crate/build-script layout is implementation detail, not fixed by this RFC.

**Capability granting:** the loader constructs each thread's CSpace slot-by-slot,
explicitly, the same way `demo.rs`'s `spawn` does today — nothing is ambient. For the
initial "confined hello service" milestone this RFC targets, that means: one loaded
program gets the server-side Endpoint capability (to `Recv`/`Reply`), a second loaded
program (or `lantern-boot`'s own still-privileged boot code, playing the "client" —
open question, see below) gets the client-side one (to `Call`), and neither gets a
capability to the other's VSpace, Frames, or CNode.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** sharpens, rather than moves, the kernel/user-space
  boundary RFC-0005 already made checkable — extends the same capability-invocation
  discipline to address-space construction, closing the one place (`Tcb.address_space`)
  that currently bypasses it.
- **New assets introduced and who can reach them:** VSpace/Frame capabilities, reachable
  only by whoever holds them (retyped from Untyped, subject to the same budget/rights
  discipline as every other object). The loaded ELF image itself is a new parsed-untrusted-
  structure surface inside `lantern-boot`'s existing TCB (see below).
- **New adversary capabilities, if any:** a malformed or malicious ELF is a new input
  `lantern-boot` parses. Mitigated by the "minimal ELF64 support only, reject anything not
  understood" stance above (no `PT_DYNAMIC`/relocation processing, no `PT_INTERP` — the
  two biggest classes of real-world ELF-loader exploits), and by every offset/size derived
  from the header being range-checked against the actual byte slice before use (no
  trusting `p_offset`/`p_filesz` to be in-bounds). Phase 1's threat model already scopes
  the loaded image as *not* adversarial in the strong sense (it's this project's own
  build output, embedded at compile time, not fetched over a network or read from
  removable media) — this RFC does not change that scoping, only the parsing code that
  will eventually need to hold up against a stronger one once boot moves off
  `include_bytes!`.
- **Mitigations:** capability rights checks on every Map/Unmap/TCBConfigure-VSpace call
  (`WRITE` required, matching every other mutating invocation in the kernel); Frame
  single-mapping invariant (`mapped_at`) prevents a Frame being aliased into two VSpaces at
  once, avoiding an accidental confused-deputy/aliasing bug before Phase 1 ever needs real
  shared-frame IPC; strict ELF acceptance criteria as above.
- **Net change to attacker surface:** a real increase, and an intentional one — this is
  the RFC that makes address-space confinement *load-bearing* rather than aspirational.
  It is scoped as tightly as Phase 1's exit criterion allows (one loader, one trusted
  embedded image, minimal ELF subset) specifically so that increase is reviewable.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** Yes: `FrameInvoke`'s dispatch and
  the VSpace/Frame retype paths in `lantern-kernel` (small, capability-checked, following
  existing patterns exactly); the ELF parser and loader logic in `lantern-boot` (new,
  meaningfully sized — this is the RFC's biggest TCB addition).
- **Does this add a dependency to the TCB?** No external crate — the ELF parser is
  hand-written against the minimal subset above, not a general-purpose `elf`/`goblin`-style
  crate, matching ADR-0001's preference for minimal, auditable, from-scratch TCB code over
  pulling in a general-purpose parser this project would only use 5% of.
- **Effect on TCB size and auditability:** grows the TCB by a bounded, reviewable amount —
  one syscall, two object types, one minimal from-scratch ELF64 parser scoped to reject
  everything it doesn't need to handle. This is the cost of Phase 1's actual exit
  criterion; RFC-0004 already authorized Phase 1 to produce TCB code.

## Privacy impact

None directly — no user data, telemetry, or identity surface. Indirectly the same as
RFC-0005: capability-gated address-space construction is a prerequisite for the later
privacy-relevant confinement guarantees (e.g. the AI runtime) this project's Roadmap
depends on.

## Alternatives considered

- **Keep `Tcb.address_space` as a direct field poke indefinitely, add no VSpace/Frame
  capabilities.** Rejected: this is exactly the ambient-authority gap RFC-0003/ADR-0005
  committed to closing, and `lantern-kernel/STATUS.md` already names it as the next item.
- **A single `VSpaceInvoke` syscall (Map/Unmap invoked on the VSpace capability, naming
  the Frame as an argument) instead of `FrameInvoke`.** Rejected in favor of invoking on
  the Frame: matches seL4's actual convention (`Frame_Map`, not `VSpace_MapFrame`), and
  keeps the "WRITE rights on the thing being invoked" pattern `CNodeInvoke` already
  established, rather than introducing a new "rights on the *argument*, not the invoked
  cap" shape into the ABI.
- **Fold Map/Unmap into `CNodeInvoke`'s label space instead of a new syscall.**
  Rejected: `CNodeInvoke` is explicitly scoped to "slots within a single CNode"
  (`cnode.rs`'s own module doc) — Frame/VSpace operations aren't CNode operations, and
  overloading one syscall number across unrelated object types would make dispatch harder
  to audit, not easier.
- **A general-purpose Rust ELF-parsing crate (e.g. `goblin`, `object`).** Rejected per the
  TCB-impact section: Phase 1 needs to parse a small, self-controlled ELF subset; a
  general parser is a larger, less-audited dependency for less benefit than 100–150 lines
  of hand-written, narrowly-scoped parsing.
- **Read the ELF image from a real storage device (virtio-blk) instead of embedding it.**
  Rejected for this RFC: needs a block-device driver and filesystem/partition parsing,
  neither of which exist and both of which are separable, larger pieces of work with their
  own threat-model consequences. `include_bytes!` proves the loader itself works without
  that prerequisite; swapping the byte source later doesn't change anything this RFC
  specifies.
- **Have Frame objects always be page-size-generic (a runtime size field with arbitrary
  values) rather than a closed `FrameSize` enum.** Rejected: Sv39 only has three legal
  leaf sizes (4 KiB/2 MiB/1 GiB) tied to specific table levels; an open-ended size field
  would let a caller request a nonsensical value the kernel then has to reject anyway,
  for no expressiveness gained over an enum matching the hardware's actual choices
  (`FrameSize::Giga`/1 GiB is not included yet since nothing needs it — trivial to add
  later without an ABI break, since it's a new enum variant, not a reinterpreted field).

## Prior art

**seL4** again: `Frame_Map`/`Frame_Unmap` invoked on the frame capability, VSpace as a
first-class retypable object, and page-size-as-object-type (`seL4_ARM_SmallPageObject` vs.
`seL4_ARM_LargePageObject` etc.) are all seL4 conventions this RFC follows directly, for
the same reason RFC-0005 gave: it's the most analyzed design in this space and RFC-0002
already anchors the object model on it. **This project's own `lantern-hal` Sv39 work**
(this session, prior to this RFC) is the other major input — `FrameSize::Mega`'s
environment-workaround framing, and the decision to add an unmap primitive, both come
directly from what that work already discovered about this QEMU environment's behavior
and `lantern-hal`'s existing primitive shapes.

## Unresolved questions

- Who plays the "client" role reaching the loaded hello service — a second loaded ELF
  program, or `lantern-boot`'s own still-privileged boot code issuing the `Call` directly?
  Either satisfies "confined hello service reachable only via a granted capability" for
  the *service* side; a second loaded program is more architecturally honest (two mutually
  confined programs, neither trusting the other) but roughly doubles the loader work for
  this first pass. Left to implementation to decide once the loader itself exists;
  revisit if it turns out to matter for the exit-criterion benchmark.
- Exact crate/build-script mechanics for building and embedding the standalone hello-
  service ELF — implementation detail, not fixed here (see "Reference-level explanation").
- Whether `FrameSize::Giga` (1 GiB) is ever needed before Phase 2 — not included; trivial
  to add later.
- Real physical-memory-backed Untyped (from the DTB) vs. today's count-based budget is
  unchanged by this RFC and remains tracked separately in `lantern-kernel/STATUS.md`.
- Shared-frame (zero-copy) IPC, which would need relaxing the single-mapping invariant
  this RFC gives Frame — explicitly deferred; Phase 1's exit criterion doesn't need it.

## Future possibilities

- A genuine root-task crate, once `lantern-boot` needs to load more than one program or
  make loading policy configurable rather than embedded — not warranted by this RFC's
  single hello-service loader.
- Loading from a real block device/filesystem once one exists, replacing `include_bytes!`
  without changing anything this RFC specifies about the loader's internals.
- Shared-frame IPC (relaxing Frame's single-mapping invariant) for bulk data transfer,
  per `lantern-kernel/ARCHITECTURE.md`'s "Bulk: zero-copy via shared frames granted by
  capability" line, already anticipated but not implemented.
- `FrameSize::Giga` and dynamic (non-embedded) untyped-memory-backed Frame allocation,
  once real physical-memory discovery lands.

## Resulting ADRs

[ADR-0012](../adr/0012-vspace-frame-capabilities-and-elf-loader.md) fixes the VSpace/Frame
object shapes, the `FrameInvoke` syscall, and the ELF loader's scope as a durable
decision.
