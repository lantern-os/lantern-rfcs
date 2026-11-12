---
adr: 0027
title: The persistent policy store v0 — a narrow-only revocation log on flash, in a confined service
status: Accepted
date: 2026-11-03
deciders: ["TSC"]
rfc: ../rfcs/0022-persistent-revocation-store.md
supersedes: []
superseded_by: null
---

# ADR-0027: The persistent policy store v0 — a narrow-only revocation log on flash, in a confined service

## Context

[ADR-0026](./0026-grant-ledger-and-revocation.md) settled that a revocation must outlive a reboot before
the Phase 3 exit criterion counts as met. The project had no persistent storage, no design for a confined
program reaching a device, and no root of trust. [RFC-0022](../rfcs/0022-persistent-revocation-store.md)
proposed the smallest thing that serves this one need, and has been accepted; this ADR is the durable record.

## Decision

**A confined service, `policy-store`, keeps an append-only log of revoked roles on flash, and the launch binder
consults it before every grant. The store holds revocations, not grants, so it can only narrow what the
compiled-in policy allows.**

- **The medium:** QEMU's `virt.flash1`, chosen over `virtio-blk` because flash needs no DMA. Without an IOMMU, a
  driver that can program DMA wouldn't really be confined.
- **A launcher-named device Frame:** the launcher seeds an `Untyped` over one megapage of flash and retypes a
  `FrameMega` from it, mapped into `policy-store` alone. This needs no kernel mechanism, and it is deliberately not
  a general device-capability design, which needs its own RFC when a second device appears.
- **The protocol:** register-only `IS_REVOKED` and `RECORD`, on an endpoint only the binder holds.
- **The key:** the first 128 bits of `BLAKE3(publisher ‖ len(app) ‖ app ‖ interface ‖ len(role) ‖ role)`, which
  unlike a ledger index or badge stays the same across boots.
- **The log:** a header, then 32-byte slots, each erased, a record, a tombstone, or torn. A torn slot has an erased
  last word and comes just before erased space, and is zeroed into a tombstone on start. Anything else is corrupt.
- **Failing closed:** only `OK` from `IS_REVOKED` lets a grant proceed. A corrupt log or missing flash declines
  every grant.
- **Record first:** `RECORD`, then the service's `REVOKE`, then the ledger. A crash in between means declined next
  boot. A failed `RECORD` doesn't stop the revocation: it's reported `FAILED`, the entry lists as revoked but not
  durable, and a repeated `REVOKE` retries.

**Settled on acceptance:** `policy-store` is a separate program, because flash contents are untrusted bytes that
shouldn't be parsed by the holder of mint authority; and durable-but-not-tamper-proof is acceptable for v0.

**Capacity, decided alongside:** `MAX_FRAMES` 28, `MAX_CNODES` 7, `MAX_VSPACES` 6, `CNODE_SLOTS` 72.

Rejected alternatives (full reasoning in the RFC): `virtio-blk`; making `lantern_filesystem::Store` persistent first;
the binder owning the flash; a keyed MAC whose key is a public constant in the same image; storing whole policies, which
would make the store authoritative; failing open on corruption.

## Consequences

- **Easier:** a revocation now survives a reboot, and the protocol hides the medium, so the log can move onto real
  storage later.
- **Harder / committed to:** both runtime demos now need a flash image attached, or everything is declined. A corrupt
  store has no recovery beyond replacing the image.
- **Trust boundary:** one created. `policy-store` can read and write the flash, and holds nothing else. Its worst lie,
  "not revoked", restores only what the compiled-in policy already allows. The launcher, already trusted, names one more
  physical range. Nothing enters the ADR-0004 TCB.
- **Net attacker surface: slightly larger**, in exchange for a required property: one confined program, one parser, one
  device mapping.
- **Not provided:** resistance to offline tampering or rollback. Anyone able to write the flash from outside can erase it
  and restore every revoked role, and a test keeps this visible. Stopping it needs a hardware root of trust and a
  monotonic counter. Records are also unencrypted hashes, since encrypting under a public demo seed would protect nothing.
- **Still not met:** the Phase 3 exit criterion's "visible, revocable capability set". No person sees or decides anything,
  since `grant-control` is a test script. This is durable machinery, not a durable user decision, and it must not be
  described as one.

*(Its "narrow-only" property ends in a system with consent: [ADR-0032](./0032-trusted-path-and-consent.md), where a person's remembered "always" can skip a prompt, though never past the ceiling.)*
