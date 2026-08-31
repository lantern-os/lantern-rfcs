---
adr: 0022
title: Confined-service model and the badged-endpoint + shared-memory call transport
status: Accepted
date: 2026-09-01
deciders: ["TSC"]
rfc: ../rfcs/0018-confined-execution-port.md
supersedes: []
superseded_by: null
---

# ADR-0022: Confined-service model and the badged-endpoint + shared-memory call transport

## Context

[ADR-0021](./0021-phase-2-complete-phase-3-opened.md) closed Phase 2 but carried one large
piece of work forward as Phase 3's foundational prerequisite: **the Phase 2 services
(`Broker`, `Keystore`, `Store`) and `lantern-runtime` run as in-process stand-ins against a
real `KernelState` on a native host target, not as confined processes on the kernel.**
Concretely, `Keystore::sign`, `Store::read`, and `Broker::mint` all take
`&mut lantern_kernel::state::KernelState` — a same-address-space privileged handle valid
only for TCB-resident code — and `lantern-example-signer`'s host constructs the services in
its own address space, so the "IPC round trip" between a confined component and the
keystore is an ordinary function call.

Every Phase 3 deliverable depends on closing this.
[`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md)'s "capability-scoped agents with a
faithful audit log" cannot be faithful while the runtime and the services it calls share an
address space — a compromised runtime would *be* the keystore. Encrypted-at-rest storage
"tied to hardware-backed keys" needs the keystore as a process an attacker who owns the app
still cannot read. This is the work that turns Phase 2's *demonstration* of isolation into
*deployed* isolation —
[Principle 1](../../lantern-docs/wiki/Principles.md) (security by architecture): the blast
radius of a fully compromised component must be exactly the capabilities it held.

[RFC-0018](../rfcs/0018-confined-execution-port.md) proposed the model and the contracts —
rather than discovering them inside the first porting PR — because this touches the
kernel/HAL trust boundary's *use*, spans six components, and sets the substrate every later
confined service is built on. It has been accepted. This ADR is the durable record of its
Parts 1 and 2; [ADR-0023](./0023-wasmtime-no-std-pulley-hosting.md) records Part 3.

## Decision

**The Phase 2 services become confined U-mode programs, and a confined component's host
calls reach them over a badged endpoint capability plus a shared memory region — never
`&mut KernelState`, never a `u64` badge passed as data, and deliberately not the in-memory
IPC buffer.**

### Services as confined programs

A service (keystore, store) is a standalone `#![no_std]` `riscv64` program, loaded the way
`lantern-boot`'s `hello-service` / `broker-service` already are (RFC-0008/ADR-0012), with:

- its own **CSpace** (a flat CNode) and **VSpace** (Sv39, built from `Frame` retypes);
- one **request endpoint** it `Recv`s on in a run-to-completion loop (ADR-0010);
- only its **retained authority** — for the keystore, key material in its own BSS/heap plus
  a GRANT-able source capability its composed `Broker` attenuates from; for the store, the
  block array in its own memory plus a keystore endpoint badged for its single v0 AEAD key;
- **no** capability to any other component's objects.

`Broker`, `Keystore`, and `Store` keep their exact Phase 2 *logic* — the badge/grant
tables, `check_access`, the mint→grant sequence, the AEAD/signing/refcounting — but their
Rust substrate changes from `&mut KernelState` to **issuing syscalls**, through a new small
component (working name **`lantern-abi`**): typed wrappers for the ADR-0008 syscall table
(`CNodeInvoke`, `FrameInvoke`, `Send`/`Recv`/`Call`/`Reply`, `Signal`/`Wait`) plus a
minimal program runtime (entry point, an allocator over a granted heap `Frame`, a
`#[panic_handler]` that `Signal`s a fault notification and halts). Every confined program —
services and the runtime — links `lantern-abi`. **It is not in the TCB**: it issues
syscalls; it does not implement them.

### The launcher

The narrowing-waterfall root task, extended: `lantern-boot`'s loader gains the ability to
load several programs and, for each, place exactly the capabilities named by a **launch
description** into its CSpace via `CNodeInvoke::CopyCross` (RFC-0010). For Phase 3 v0 the
launch description is hard-wired in the launcher, exactly as `lantern-example-signer`'s
runner hard-wires its `GrantManifest` today. Resolving a manifest `role` to a concrete
key/file with user consent is `lantern-shell`'s job (RFC-0015) and layers on top later.

### The call transport

- **Badged endpoint, not a `u64`.** Each resource-scoped handle is backed by a **badged
  capability to the owning service's endpoint** — minted by that service's `Broker`
  (`CNodeInvoke::Mint`, scoped to one `(KeyId, KeyOps)` / `(FileId, FileOps)`) and
  transferred into the runtime's CSpace at grant time (the RFC-0010 `extra_caps == 1` live
  transfer). The runtime never sees or handles the badge value; it invokes the capability,
  and the kernel delivers the badge to the service on `Recv` (`mr0`, ADR-0008). A guest
  cannot forge a handle for a key it was not granted because the corresponding badged
  capability simply is not in the runtime's CSpace — the property RFC-0014 already relies on
  one layer up.

- **Shared memory for payloads, not the IPC buffer.** The kernel's in-memory IPC buffer is
  designed (RFC-0005/ADR-0008) but not implemented (`length > 0` is rejected). Rather than
  implement it, this decision uses **a shared `Frame`** per `(runtime, service)` pair,
  mapped read-write into both VSpaces at launch:

  ```
  runtime:  write args into shared[..n]
            Call(endpoint_badged_for_this_handle, tag{ label = op, mr1 = n })
  service:  Recv → badge identifies (KeyId|FileId, ops); copy-in shared[..n] to a
            private buffer FIRST, then check_access(badge, op), then operate;
            write result to shared[..m]; Reply(tag{ label = OK|ACCESS|INVALID, mr1 = m })
  runtime:  read shared[..m], return to the guest
  ```

  The service always **copies in before validating** (TOCTOU). The shared `Frame` is
  reachable only by that one runtime and that one service, so it is no worse than the
  arguments already crossing the boundary. Payloads larger than one `Frame` chunk across
  multiple `Call`s.

- **The trait impls.** `lantern-runtime`'s `KeystoreService` / `FilesystemService` traits
  (ADR-0018 / ADR-0019) are unchanged as *interfaces*. The Phase 2 `InProcessFilesystem`
  stand-in is joined by an `IpcKeystore` / `IpcFilesystem` that hold `(endpoint CPtr,
  shared Frame view)` and implement each method as the marshal → `Call` → unmarshal
  sequence above. The host-target demo keeps working with the in-process stand-in; the
  `riscv64` build uses the IPC ones.

### Not decided here

- **The per-service wire protocols** — the exact `label` values and `mr` / shared-region
  layout for `keystore` (SIGN/ENCRYPT/DECRYPT) vs `store` (READ/WRITE). Each is its own
  small follow-up, the way ADR-0018 fixed the two mapping *shapes* and left each interface
  separate.
- **The launch binder and consent UX** — `lantern-shell`'s, per RFC-0015. This decision
  fixes the launcher *mechanism*; the policy that produces the launch description is
  separate.
- **Shared-`Frame` sizing and lifecycle** — one fixed-size `Frame` per pair, a small pool,
  or a per-call transient mapping; chunking large file I/O needs a decided answer.
- **`lantern-abi`'s exact name, crate layout, and allocator** — one crate or two (syscall
  wrappers vs. program runtime); an implementation decision.

### Rejected alternatives (full reasoning in the RFC)

- **Implement the in-memory IPC buffer and pass payloads through it** — kernel code (TCB
  growth); file reads/writes want more than any fixed buffer size anyway; shared memory is
  zero-copy for the large cases.
- **One merged "services" process** holding the keystore and the store together —
  re-creates the Phase 2 problem one level up (a store bug reaches key material), and the
  store↔keystore call is exactly the boundary this work exists to make real.
- **A shared runtime process hosting many apps' components** — a runtime bug then crosses
  every hosted app; one runtime process per app keeps ADR-0004's "bounded by *its*
  capabilities" literal. A pooled runtime is a later density optimisation.
- **Keep `&mut KernelState` and add a "kernel proxy" that marshals it over IPC** —
  `&mut KernelState` is a same-address-space privileged handle by construction; there is
  nothing to proxy. The services genuinely have to be rewritten onto syscalls.
- **Wait for multi-hart / a preemptive timer first** — the synchronous request/reply mesh
  works under ADR-0010's single-stack model with fuel-based interruption; SMP and timer
  preemption are refinements, not blockers.

## Consequences

- **Easier:** Phase 2's *demonstrated* isolation becomes *deployed* — a runtime compromise
  stops being a service compromise, and key material / user data sit behind real kernel
  address-space and IPC boundaries a reviewer can point at. `lantern-network`'s
  capability-scoped sockets get the pattern to reach a confined component the way
  `lantern:host/keystore` already does.
- **Harder / committed to:** the services are genuinely rewritten onto syscalls, not
  wrapped; `lantern-abi` is new user-space surface to build and maintain (outside the TCB);
  the per-`(runtime, service)` shared `Frame` is a new asset, defended by copy-in-before-
  validate. **`Revoke` is still absent** — withdrawing *one* granted capability from a
  still-running app (the Phase 3 exit criterion's "revocable capability set") means tearing
  the process down until a real `Revoke` + capability-derivation tree lands. Recorded, not
  waived.
- **TCB impact: none.** ADR-0004's TCB is kernel + HAL + boot. This design needs **no new
  kernel object, no new syscall, and no new `lantern-hal` surface**: it composes the
  existing IPC fast path, `CNodeInvoke` (incl. `Mint` / `CopyCross`, RFC-0010),
  `FrameInvoke` (`Map` / `Unmap`, RFC-0008), and badged endpoints (ADR-0008). The in-memory
  IPC buffer — the one kernel addition this could have needed — is deliberately not built.
  `lantern-abi`, `IpcKeystore` / `IpcFilesystem`, and the extended launcher are all
  confined user-space code. No defined trust boundary moves; a designed one is enforced for
  the first time.
- **Prerequisites (ordinary engineering, no new RFC):** DTB-based physical-memory discovery
  (`lantern-boot/pmm.rs` seeds one hardcoded range today — already `lantern-boot`'s "Next");
  a `talc`-style allocator for `lantern-abi`'s program heap; `Frame`-region reservation
  discipline so a program's heap, image, and Wasm memories do not collide.
- **Becomes more pressing (each a `lantern-kernel` "Next"), not blocking v0:** `Revoke` +
  a capability-derivation tree; an idle thread (a synchronous request/reply mesh with no
  external I/O never reaches "all threads blocked", so v0 is fine, but `lantern-network`'s
  first blocking socket read will need it); delivering CPU faults to a U-mode handler
  (required only before native-AOT execution, [ADR-0023](./0023-wasmtime-no-std-pulley-hosting.md)).
- **Open (tracked in `lantern-runtime` / `lantern-kernel` `STATUS.md`):** shared-`Frame`
  sizing/lifecycle; how the launch description is expressed before `lantern-shell` exists;
  whether the services port and the Wasmtime port proceed as one work item or two, and in
  which order (Part 3 has no hard dependency on Part 1).
