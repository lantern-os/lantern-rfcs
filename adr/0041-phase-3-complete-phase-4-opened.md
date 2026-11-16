---
adr: 0041
title: Phase 3 complete; Phase 4 (hardware and assurance) opened
status: Accepted
date: 2026-11-16
deciders: ["TSC"]
rfc: ../rfcs/0036-phase-3-to-phase-4-transition.md
supersedes: []
superseded_by: null
---

# ADR-0041: Phase 3 complete; Phase 4 (hardware and assurance) opened

## Context

Phase 3's three exit criteria were met by [RFC-0031](../rfcs/0031-storage-a-block-driver-and-the-store-on-disk.md) and
[RFC-0033](../rfcs/0033-provenance-for-data-at-rest.md) (data at rest, encrypted and provenance-tracked),
[RFC-0034](../rfcs/0034-local-inference-and-a-file-assistant.md) (a local AI agent) and
[RFC-0035](../rfcs/0035-pairwise-network-identities.md) (a network identity without cross-context linkage). All three are
on `main`, verified from fresh clones on 2026-11-16. [RFC-0036](../rfcs/0036-phase-3-to-phase-4-transition.md) proposed
the transition. It has been accepted, and this ADR is the durable record.

## Decision

**Phase 3 is closed. Phase 4, hardware and assurance, is open.**

- **The narrow points are part of the record.**
  - The agent's model is tiny (15M parameters). What's proven is the boundary: consent, audit, attribution and
    revocation around a model.
  - Keys can't link two contexts; the network still can, by address and timing.
  - Data at rest is encrypted under software-held demo keys.
- **Carried forward.**
  - **Formal verification** of the IPC and capability paths, a Phase 3 deliverable that never started, is Phase 4's
    **first** work. It needs no hardware, and Phase 4's exit criterion requires machine-checked proofs. Its own RFC
    chooses the tool and the first properties.
  - DIDs and verifiable credentials, peer discovery, and an anonymity transport continue alongside Phase 4 as ordinary
    work, as earlier transitions' leftovers did.
- **RFC-0023's checklist is accounted for.** All 18 items are listed in RFC-0036, each with a work stream:
  - B, a hardware root of trust;
  - C, rollback resistance;
  - D, IOMMU-confined drivers;
  - F, signed models and publishers.

  Each item also notes whether it needs the board. None is dropped.
- **Minimum hardware has two tiers.**
  - To run LanternOS: RV64GC with Sv39, a PLIC or AIA, an SBI timer, a device tree, and storage the system can drive.
  - To exit Phase 4: additionally an IOMMU over every DMA-capable device, an entropy source (`Zkr` or a hardware
    generator), and a root of trust that holds keys and a monotonic counter without releasing them.
- **Sequencing.**
  1. Without hardware: formal verification first, then the hardware-independent parts of streams C and F.
  2. When a board arrives: bring-up (the installable track's step 6), then B and D as far as it allows.

  Which board is still open.

## Consequences

- Phase 4's exit criteria are unchanged: a hardware root of trust holding every key; device theft yields only
  ciphertext; the revocation and audit logs resist tampering and rollback; machine-checked proofs of the most critical
  kernel paths.
- Each Phase 4 work stream needs its own RFC. This ADR pre-approves no design.
- Hardware enters the trusted base for the first time in Phase 4, deliberately, by those RFCs.
