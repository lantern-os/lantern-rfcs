# lantern-rfcs — The LanternOS RFC Process

> Substantial changes to LanternOS are designed in the open, in writing, before they are
> built. This repository is where that happens.

If you are new to LanternOS, **this is the most important repository to understand**. It
encodes how the project thinks, decides, and remembers. The architecture documents in
[`lantern-docs`](../lantern-docs) describe the *current* design; the RFCs and ADRs here
describe *how and why* that design came to be, and how it is allowed to change.

## Two instruments, two jobs

| Instrument | Question it answers | Lives in | Mutable? |
| --- | --- | --- | --- |
| **RFC** (Request for Comments) | "Should we do this, and how?" — a proposal under discussion | `rfcs/` | Until accepted/rejected, then frozen |
| **ADR** (Architecture Decision Record) | "What did we decide, and why?" — the durable record of a decision | `adr/` | Append-only; superseded, never edited |

An RFC is the *deliberation*. An ADR is the *outcome*. A large RFC often produces one or
more ADRs when it is accepted. Small decisions that do not need a full proposal can be
captured directly as an ADR.

## When is an RFC required?

See [`GOVERNANCE.md`](../GOVERNANCE.md) for the authoritative list. In short, an RFC is
required for anything that is **hard to reverse** or **affects a trust boundary**:

- New or changed public interfaces: syscalls, capability types, IPC/wire formats, SDK APIs.
- Any change to the Trusted Computing Base or the threat model.
- Cryptographic primitive selection.
- New languages, toolchains, or TCB dependencies.
- Roadmap phase boundaries and governance changes.

Bug fixes, docs, tests, and interface-preserving refactors do **not** need an RFC.

## RFC lifecycle

```
  Draft ──▶ Proposed ──▶ Final Comment Period ──▶ Accepted ──▶ Active ──▶ Implemented
    │           │                                    │
    │           └──────────────▶ Rejected ◀──────────┘
    │
    └──▶ Withdrawn

  Accepted/Active ──▶ Superseded   (by a later RFC)
  Any state        ──▶ Postponed   (revisit later)
```

| Status | Meaning |
| --- | --- |
| **Draft** | Being written; not yet seeking broad review. |
| **Proposed** | Open for community review and discussion. |
| **Final Comment Period (FCP)** | Stewards intend to accept/reject; last call, minimum 1 week (2 weeks for governance/security). |
| **Accepted** | Approved. May not yet be implemented. Produces one or more ADRs. |
| **Active** | Accepted *and* describes an ongoing policy/process in force. |
| **Rejected** | Declined. Kept for the record with rationale. |
| **Withdrawn** | Pulled by the author before a decision. |
| **Postponed** | Good idea, wrong time; parked with a revisit condition. |
| **Implemented** | The accepted design now exists in code. |
| **Superseded** | Replaced by a later RFC (linked). |

## How to submit an RFC

1. Copy [`0000-template.md`](./0000-template.md) to `rfcs/0000-my-title.md` (keep `0000`
   until a number is assigned).
2. Fill it in. Be concrete about the **threat model impact** and **TCB impact** — these
   sections are mandatory and reviewers will not skip them.
3. Open a pull request. The PR number or the next free index becomes the RFC number;
   rename the file to `rfcs/NNNN-my-title.md`.
4. Discussion happens on the PR. The relevant **domain steward(s)** shepherd it.
5. When stewards are ready, they declare a **Final Comment Period**. If consensus holds,
   the RFC is **Accepted**, merged, and any resulting **ADRs** are filed in `adr/`.

## Index of RFCs

| # | Title | Status |
| --- | --- | --- |
| [0001](./rfcs/0001-rfc-process.md) | The RFC and ADR process | Active |
| [0002](./rfcs/0002-microkernel-architecture.md) | Microkernel architecture and TCB boundary | Accepted |
| [0003](./rfcs/0003-capability-model.md) | The LanternOS capability model | Accepted |
| [0004](./rfcs/0004-phase-0-to-phase-1-transition.md) | Closing Phase 0 and opening Phase 1 | Accepted |
| [0005](./rfcs/0005-syscall-ipc-abi-and-phase1-scheduling.md) | Kernel syscall/IPC ABI and the Phase 1 scheduling-context model | Accepted |
| [0006](./rfcs/0006-kernel-concurrency-model.md) | Kernel concurrency model — single-stack, run-to-completion for Phase 1 | Accepted |
| [0007](./rfcs/0007-cryptographic-primitive-set.md) | Cryptographic primitive set (Phase 1) | Accepted |
| [0008](./rfcs/0008-vspace-frame-capabilities-and-elf-loader.md) | VSpace/Frame capabilities and a minimal ELF loader | Accepted |
| [0009](./rfcs/0009-phase-1-to-phase-2-transition.md) | Closing Phase 1 and opening Phase 2 (capability runtime & first services) | Accepted |
| [0010](./rfcs/0010-cross-process-capability-transfer-and-brokering.md) | Cross-process capability transfer and the service-layer capability-brokering API | Draft |
| [0011](./rfcs/0011-sealed-capability-token-format.md) | Sealed capabilities — a macaroon-style cryptographic token format | Accepted |
| [0012](./rfcs/0012-monotonic-clock-primitive.md) | A monotonic clock read primitive for lantern-hal | Accepted |
| [0013](./rfcs/0013-wasm-engine-selection-and-aot-strategy.md) | Wasm engine selection and AOT execution strategy for lantern-runtime | Accepted |
| [0014](./rfcs/0014-wit-handle-capability-mapping.md) | WIT-handle ⇄ capability mapping for lantern-runtime | Accepted |
| [0015](./rfcs/0015-capability-manifest-format.md) | The lantern-sdk capability manifest format | Draft |

## Index of ADRs

| # | Decision | Status |
| --- | --- | --- |
| [0001](./adr/0001-rust-as-primary-language.md) | Rust is the primary implementation language | Accepted |
| [0002](./adr/0002-riscv-target-isa.md) | RISC-V is the long-term target ISA | Accepted |
| [0003](./adr/0003-wasm-as-portable-app-abi.md) | WebAssembly is the portable application ABI | Accepted |
| [0004](./adr/0004-kernel-responsibilities-and-tcb-boundary.md) | Kernel responsibility list and the TCB boundary | Accepted |
| [0005](./adr/0005-object-capabilities-as-universal-authority-model.md) | Object capabilities as the universal authority model | Accepted |
| [0006](./adr/0006-three-layer-capability-structure.md) | Three-layer capability structure — kernel / service / sealed | Accepted |
| [0007](./adr/0007-phase-0-complete-phase-1-opened.md) | Phase 0 complete; Phase 1 (microkernel prototype) opened | Accepted |
| [0008](./adr/0008-kernel-syscall-ipc-abi.md) | Kernel syscall/IPC ABI — capability invocation, message registers, IPC buffer | Accepted |
| [0009](./adr/0009-phase1-scheduling-context-model.md) | Phase 1 scheduling-context model — MCS-shaped object, simplified semantics | Accepted |
| [0010](./adr/0010-kernel-concurrency-model.md) | Kernel concurrency model — single-stack, run-to-completion (Phase 1) | Accepted |
| [0011](./adr/0011-cryptographic-primitive-set.md) | Cryptographic primitive set (Phase 1) | Accepted |
| [0012](./adr/0012-vspace-frame-capabilities-and-elf-loader.md) | VSpace/Frame capabilities and a minimal ELF loader | Accepted |
| [0013](./adr/0013-ipc-latency-benchmark.md) | IPC latency benchmark — methodology, Phase 1 target, and a known QEMU-only bug | Accepted |
| [0014](./adr/0014-phase-1-complete-phase-2-opened.md) | Phase 1 complete; Phase 2 (capability runtime & first services) opened | Accepted |
| [0015](./adr/0015-sealed-capability-token-format.md) | Sealed capabilities — macaroon-style BLAKE3-keyed-MAC token format | Accepted |
| [0016](./adr/0016-monotonic-clock-primitive.md) | A monotonic clock read primitive for lantern-hal | Accepted |
| [0017](./adr/0017-wasm-engine-selection-and-aot-strategy.md) | Wasm engine selection and AOT execution strategy for lantern-runtime | Accepted |
| [0018](./adr/0018-wit-handle-capability-mapping.md) | WIT-handle ⇄ capability mapping — resource-scoped vs link-scoped, first two interfaces | Accepted |

See [`adr/README.md`](./adr/README.md) for the ADR conventions.
