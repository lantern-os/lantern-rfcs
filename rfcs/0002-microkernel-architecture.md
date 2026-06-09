---
rfc: 0002
title: Microkernel architecture and the TCB boundary
status: Proposed
authors: ["LanternOS founding stewards"]
stewards: ["kernel", "capabilities"]
domains: ["kernel", "capabilities", "hal"]
created: 2026-06-09
updated: 2026-06-09
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0002: Microkernel architecture and the TCB boundary

## Summary

LanternOS adopts a **capability-based microkernel**. The kernel provides only address-space
isolation, thread scheduling, inter-process communication (IPC), capability enforcement,
and interrupt dispatch. Everything else — drivers, filesystems, networking, crypto
services, the AI runtime — runs as unprivileged, mutually isolated user-space components
communicating exclusively by IPC over capabilities.

## Motivation

The size of the Trusted Computing Base is the single biggest determinant of a system's
security ceiling. Monolithic kernels place millions of lines — including every device
driver — inside the most privileged execution context, so any driver bug is a total
compromise. seL4 demonstrated that a microkernel small enough to *formally verify*
(~10k LoC) is achievable and that the resulting isolation is strong and performant.

LanternOS's first principle is **security by architecture**. A microkernel is the most
direct expression of it: minimise the TCB, deny ambient authority, mediate everything
through capabilities, and push policy out of the kernel.

## Guide-level explanation

The kernel is a *mechanism, not a policy* engine. It knows how to switch threads, map
memory, deliver a message, check a capability, and route an interrupt. It does **not**
know what a file is, what a network is, or what an AI model is — those concepts live in
user-space services that the kernel cannot distinguish from any other process.

A component can do exactly what its capabilities permit and nothing more. There is no
"root", no ambient filesystem, no global namespace. To talk to the filesystem service you
must hold a capability (an endpoint) to it; to read a file you must hold a capability to
that file's object.

## Reference-level explanation

### Kernel responsibilities (the entire list)

1. **Scheduling** — threads and a hierarchy of scheduling contexts; the policy of *who
   gets CPU* is constrained by kernel mechanism but configurable from user space.
2. **Memory isolation** — address spaces, page tables, and untyped-memory retyping; user
   space manages its own memory from granted untyped regions (seL4-style), so the kernel
   performs **no dynamic heap allocation** after boot.
3. **IPC** — synchronous call/reply (endpoints) and asynchronous signals (notifications);
   the sole inter-component communication primitive.
4. **Capability enforcement** — every kernel object is named by a capability held in a
   per-process capability space (CSpace); the kernel checks rights on every operation.
5. **Interrupt handling** — interrupts are delivered to user-space drivers as
   notifications bound to IRQ capabilities.

### Explicitly *out* of the kernel

Device drivers, the filesystem, the network stack, cryptographic services, the WASM and
AI runtimes, the shell, and all policy. These are ordinary user-space components.

### The TCB boundary

```
   TCB (privileged / highest assurance)        Confined (unprivileged)
   ───────────────────────────────────         ───────────────────────────────
   lantern-boot  (root of trust, measured)      drivers, filesystem, network,
   lantern-kernel (microkernel)                 crypto service, wasm runtime,
   minimal HAL crate (machine layer)            ai runtime, shell, apps, agents
```

A flaw in a confined component is bounded by the capabilities that component holds. A flaw
in the TCB is, by definition, severe — hence the TCB must stay small, be written in safe
Rust where possible (see [ADR-0001](../adr/0001-rust-as-primary-language.md)), and be a
verification target.

### Performance posture

IPC is the kernel's hot path; its design dictates whole-system performance (the classic
microkernel lesson from L4). We commit to: register-passing fast-path IPC, zero-copy bulk
transfer via shared memory granted by capability, and core-local scheduling to avoid lock
contention. Benchmarks gate IPC changes.

## Threat model impact

- **Trust boundaries affected:** defines the primary boundary of the entire system.
- **New assets:** kernel objects (CNodes, endpoints, untyped memory, page tables).
- **New adversary capabilities:** none introduced; the design *removes* the "driver = ring
  0" class of attack entirely.
- **Mitigations:** isolation by default, capability mediation, no ambient authority.
- **Net change:** large **reduction** — the exploitable privileged surface shrinks from a
  whole monolithic kernel to a small, auditable core.

## TCB impact

This RFC *defines* the TCB. It commits to keeping it minimal and to treating any addition
to it as an RFC-level decision with a verification cost.

## Privacy impact

Indirect but strong: confinement of services limits how far a compromise can spread before
it can reach user data, and removes the global-access patterns that enable silent
exfiltration.

## Alternatives considered

- **Monolithic kernel (Linux-like).** Rejected: enormous TCB, ambient authority, contrary
  to first principle. Also explicitly a non-goal of the project.
- **Hybrid/macrokernel (XNU-like).** Rejected: blurs the boundary we most want crisp.
- **Unikernel.** Rejected as the base model: excellent isolation *between* VMs but pushes
  a fat library OS into each, and does not give us in-system multi-component capability
  confinement. May still be used to host LanternOS during bring-up.
- **Library OS on a hypervisor.** Considered as a bring-up path, not the end state.

## Prior art

seL4 (verified microkernel, capability model), L4 family (IPC performance), Fuchsia/Zircon
(object-capability microkernel at scale), Barrelfish (multikernel ideas), Genode
(component composition), CHERI (hardware capabilities, complementary — see future work).

## Unresolved questions

- Scheduling-context model: full seL4-style MCS, or a simpler initial scheme?
- Single-stack vs. process-kernel concurrency model.
- How much of the HAL is "in the TCB" vs. a thin shim.
- Verification strategy and timing (see Roadmap Phase 3+).

## Future possibilities

- Formal verification of the IPC and capability paths.
- Exploiting CHERI capabilities in hardware to back software capabilities.
- Multikernel scaling across many-core and heterogeneous (NPU/FPGA) compute.

## Resulting ADRs

On acceptance: an ADR fixing the kernel responsibility list and the TCB boundary, and an
ADR selecting the scheduling-context model once Q1 above is resolved.
