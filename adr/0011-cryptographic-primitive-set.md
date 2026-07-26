---
adr: 0011
title: Cryptographic primitive set (Phase 1)
status: Accepted
date: 2026-07-26
deciders: ["TSC"]
rfc: ../rfcs/0007-cryptographic-primitive-set.md
supersedes: []
superseded_by: null
---

# ADR-0011: Cryptographic primitive set (Phase 1)

## Context

`lantern-crypto/ARCHITECTURE.md` and `lantern-docs/wiki/Cryptography.md` both sketched a
candidate primitive set, but `GOVERNANCE.md` requires an RFC for any cryptographic
primitive selection — a draft table in an `ARCHITECTURE.md` file carries no governance
weight on its own. `lantern-boot/STATUS.md` was blocked on this: it cannot fix its
kernel-image signature-verification scheme without a ratified signature algorithm and
hash function. [RFC-0007](../rfcs/0007-cryptographic-primitive-set.md) proposed ratifying
the existing draft table as a reviewed decision; it has been accepted. This ADR fixes the
durable decision.

## Decision

**Phase 1 ships the following primitive set, each versioned and algorithm-tagged from
creation:**

| Purpose | Primitive | Notes |
| --- | --- | --- |
| Hashing / content addressing | BLAKE3 (SHA-256 for interop) | Used by `lantern-filesystem`'s CAS and `lantern-boot`'s measured-boot records. |
| AEAD (symmetric encryption) | XChaCha20-Poly1305; AES-256-GCM where hardware-accelerated | Large (192-bit) nonce. |
| KDF | HKDF (key derivation); Argon2id (password-based) | |
| Signatures | Ed25519 | Fixes `lantern-boot`'s kernel-image trust-anchor algorithm. |
| Key exchange | X25519 | |
| Randomness | OS CSPRNG, seeded from hardware entropy, health-checked | Never app-supplied. |

- **Crypto-agility is a fixed principle, not yet a fixed wire format:** every key,
  ciphertext, signature, and sealed-capability token carries a stable algorithm
  identifier from creation. The exact byte-level encoding is left to `lantern-crypto`'s
  implementation.
- **PQC-hybrid identifier slots are reserved, not implemented:** signatures and key
  exchange each reserve an identifier for a future hybrid successor (ML-DSA alongside
  Ed25519, ML-KEM alongside X25519). Phase 1 ships the classical primitive alone; adding
  the hybrid later is a new algorithm identifier, not a breaking change.
- **Not decided here:** the sealed-capability token format, key backup/recovery,
  attestation's privacy-exposure policy, and whether any low-level primitive is ever
  exposed directly to apps — all remain open, tracked in `lantern-crypto/STATUS.md`.

## Consequences

- **Easier:** `lantern-boot` can now write its kernel-image verification step against a
  fixed algorithm pair (Ed25519 + BLAKE3/SHA-256) instead of an open question;
  `lantern-crypto` has a reviewed target to implement against once its Phase 2 keystore
  work starts; any future sealed-capability token-format work already has a keyed-hash/
  signature primitive available (BLAKE3's native keyed mode, or Ed25519) without needing
  to add one.
- **Harder:** none directly — this is a selection among existing, audited primitives, not
  new engineering work.
- **Committed to:** every cryptographic output is versioned and algorithm-tagged from
  creation; rolling a custom primitive instead of using an audited implementation is
  ruled out per `lantern-crypto/ARCHITECTURE.md`'s existing "Assurance and `unsafe`"
  stance; any future change to this primitive set is itself an RFC-level decision
  (`GOVERNANCE.md`, "cryptographic primitive selection or changes").
- **Still open:** the exact hybrid-PQC construction and its default-on/opt-in timing, key
  backup/recovery, attestation's privacy-exposure policy, low-level-primitive exposure to
  apps, and the sealed-capability token format — all carried forward from RFC-0007,
  tracked in `lantern-crypto/STATUS.md`.
