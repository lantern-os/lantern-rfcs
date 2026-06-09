# lantern-rfcs — The LanternOS RFC Process

> Substantial changes to LanternOS are designed in the open, in writing, before they are
> built. This repository is where that happens.

If you are new to LanternOS, **this is the most important repository to understand**. It
encodes how the project thinks, decides, and remembers. The architecture documents in
[`lantern-docs`](https://github.com/lantern-os/lantern-docs) describe the *current* design; the RFCs and ADRs here
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

See [`GOVERNANCE.md`](https://github.com/lantern-os/.github/blob/main/GOVERNANCE.md) for the authoritative list. In short, an RFC is
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
| [0002](./rfcs/0002-microkernel-architecture.md) | Microkernel architecture and TCB boundary | Proposed |
| [0003](./rfcs/0003-capability-model.md) | The LanternOS capability model | Proposed |

## Index of ADRs

| # | Decision | Status |
| --- | --- | --- |
| [0001](./adr/0001-rust-as-primary-language.md) | Rust is the primary implementation language | Accepted |
| [0002](./adr/0002-riscv-target-isa.md) | RISC-V is the long-term target ISA | Accepted |
| [0003](./adr/0003-wasm-as-portable-app-abi.md) | WebAssembly is the portable application ABI | Accepted |

See [`adr/README.md`](./adr/README.md) for the ADR conventions.
