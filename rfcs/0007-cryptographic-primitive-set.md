---
rfc: 0007
title: Cryptographic primitive set (Phase 1)
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["crypto"]
domains: ["crypto", "boot", "capabilities"]
created: 2026-07-26
updated: 2026-07-26
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0007: Cryptographic primitive set (Phase 1)

## Summary

This RFC ratifies a concrete cryptographic primitive set for Phase 1: **BLAKE3** (+SHA-256
for interop) for hashing/content-addressing, **XChaCha20-Poly1305** (AES-256-GCM where
hardware-accelerated) for AEAD, **HKDF** + **Argon2id** for key derivation, **Ed25519** for
signatures, **X25519** for key exchange, and an OS **CSPRNG seeded from hardware entropy**
for randomness — the same defaults already sketched, but not yet RFC'd, in
`lantern-crypto/ARCHITECTURE.md` and `lantern-docs/wiki/Cryptography.md`. Every value
produced (key, ciphertext, signature, credential) is versioned and algorithm-tagged from
day one, per the project's crypto-agility principle, and the signature/key-exchange
choices each reserve an identifier for a named PQC-hybrid successor (ML-DSA, ML-KEM) —
but the exact hybrid construction is deliberately deferred, since Phase 1 (RFC-0004:
prototype, QEMU-only, throwaway) has no PQC exit criterion.

## Motivation

`lantern-crypto/STATUS.md` lists "RFC to ratify the primitive set (→ ADR)" as its "Next"
item, and `GOVERNANCE.md` is explicit that "cryptographic primitive selection or changes"
always requires an RFC — this is the one domain where the project's own governance
forbids settling it ad hoc in an implementation PR, no matter how settled the defaults
already look, written down in `ARCHITECTURE.md`.

It also sits on the critical path for two other components. `lantern-boot/STATUS.md`
lists "crypto verification primitives" as a blocker: the loader cannot fix its
kernel-image signature-verification scheme (RFC-0002's boot flow: "verify kernel image
signature against a trust anchor") without a ratified signature algorithm and hash
function. And RFC-0003/[ADR-0006](../adr/0006-three-layer-capability-structure.md)
already committed to sealed, cryptographic capabilities ("macaroon-style attenuable,
revocable tokens") as the third capability layer; a concrete signing/MAC primitive is a
precondition for `lantern-capabilities` to later specify that token format — itself a
separate, later `lantern-crypto/STATUS.md` item this RFC does not attempt.

Ratifying now, rather than waiting for `lantern-crypto`'s own Phase 1/2 implementation
work to get around to it, means `lantern-boot` can unblock immediately rather than
waiting on a service that has no code yet.

## Guide-level explanation

`lantern-crypto` doesn't invent cryptography — it picks, from established, audited
primitives, one option per job (hash, symmetric encryption, key derivation, signatures,
key exchange, randomness), and commits to shipping every one of them tagged with a
stable identifier so a signature made today can still be verified, and a ciphertext
encrypted today can still be decrypted, after the underlying primitive is eventually
replaced. Nothing here is novel cryptographic design; the interesting decision is *which*
well-reviewed primitive to point at for each job, and how the system stays able to
change its mind later without breaking anything already on disk or in flight.

## Reference-level explanation

### The primitive table (Phase 1 default)

| Purpose | Primitive | Notes |
| --- | --- | --- |
| Hashing / content addressing | BLAKE3 (SHA-256 for interop) | Fast, parallelizable; used by `lantern-filesystem`'s CAS and `lantern-boot`'s measured-boot records. |
| AEAD (symmetric encryption) | XChaCha20-Poly1305; AES-256-GCM where hardware-accelerated | 192-bit nonce avoids the reuse pitfalls a 96-bit AEAD nonce invites under random generation. |
| KDF | HKDF (key derivation); Argon2id (password-based) | |
| Signatures | Ed25519 | Fixes `lantern-boot`'s kernel-image trust-anchor algorithm. |
| Key exchange | X25519 | |
| Randomness | OS CSPRNG, seeded from hardware entropy, health-checked | Never app-supplied (X3, `lantern-crypto/THREAT_MODEL.md`). |

These match `lantern-crypto/ARCHITECTURE.md`'s and `lantern-docs/wiki/Cryptography.md`'s
existing draft table; this RFC's job is to make that draft a reviewed, durable decision,
not to invent new choices.

### Crypto-agility: versioning and algorithm tagging

Every key, ciphertext, signature, and sealed-capability token carries a stable algorithm
identifier from the moment it's created — this RFC fixes the *principle*, not a specific
byte-level wire encoding (left to `lantern-crypto`'s implementation, itself not yet
started). Rotating a primitive later (deprecating BLAKE3 for its eventual successor, say)
never requires re-interpreting old data without knowing which algorithm produced it.

### PQC: hybrid slots reserved, construction deferred

Signatures and key exchange each reserve an identifier for a PQC-hybrid successor —
ML-DSA (Dilithium) alongside Ed25519, ML-KEM (Kyber) alongside X25519 — matching the
"harvest now, decrypt later" stance already committed to project-wide
(`lantern-docs/wiki/Cryptography.md`, "Post-quantum readiness"). **Phase 1 does not
implement the hybrid construction**: it ships the classical primitive alone, tagged in a
way that adding the hybrid later is a new algorithm identifier, not a breaking change to
anything Phase 1 produced. This mirrors RFC-0005's "keep the object shape now, defer full
semantics" move for the MCS scheduling context
([ADR-0009](../adr/0009-phase1-scheduling-context-model.md)).

### What this RFC does not decide

- The sealed-capability token format (`lantern-capabilities`, RFC-0003/ADR-0006's
  "sealed" layer) — a separate, later `lantern-crypto/STATUS.md` item. This RFC only
  ensures the primitives that future work would need (a signature or keyed-hash/MAC
  scheme — BLAKE3 supports a native keyed mode, available to it without adding another
  primitive) already exist in the ratified set.
- Key backup/recovery, the attestation-vs-privacy tension, and whether any low-level
  primitive is ever exposed directly to apps — all remain open questions in
  `lantern-crypto/ARCHITECTURE.md`, untouched here.
- The exact hybrid-PQC construction and its default-on/opt-in timing.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none moved; this ratifies which primitives back the
  existing crypto-service boundary already fixed by RFC-0002/ADR-0004 (crypto is a
  confined user-space service, not in the kernel TCB) — it does not change who can reach
  what.
- **New assets introduced and who can reach them:** none new; this fixes the algorithms
  protecting assets already named in `lantern-crypto/THREAT_MODEL.md` (private key
  material, signing/decryption integrity, sealed-capability/attestation integrity,
  randomness quality).
- **New adversary capabilities, if any:** none intended.
- **Mitigations:** directly answers X3 (weak/biased randomness → hardware-seeded,
  health-checked CSPRNG), X4 (nonce reuse → XChaCha20-Poly1305's large nonce), and X5
  (algorithm obsolescence/harvest-now-decrypt-later → crypto-agility tagging + reserved
  PQC-hybrid slots), all from `lantern-crypto/THREAT_MODEL.md`.
- **Net change to attacker surface:** neutral — no new surface; this closes off X5 more
  concretely than the un-ratified draft table did (a draft in an `ARCHITECTURE.md` file
  has no governance weight; an accepted RFC/ADR does, per `GOVERNANCE.md`'s "no
  security-relevant change... without review").

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No — `lantern-crypto` is a
  confined user-space service (RFC-0002/ADR-0004's TCB boundary: kernel + HAL + boot
  only); this RFC does not move that boundary.
- **Does this add a dependency to the TCB?** No; it fixes external crate/algorithm
  choices for a non-TCB service. `lantern-boot`, which *is* in the TCB, depends on the
  signature/hash algorithms chosen here for its verification step — but the boundary
  itself, and boot's existing commitment to being minimal and in the TCB, are unchanged;
  this RFC just tells boot which algorithms to call.
- **Effect on TCB size and auditability:** positive for `lantern-boot` specifically — its
  verification code can now be written against a fixed, reviewed algorithm pair (Ed25519
  + BLAKE3/SHA-256) instead of an open question, keeping that TCB component's scope
  bounded.

## Privacy impact

Mixed, flagged rather than resolved: the primitive choices themselves are
privacy-neutral, but **attestation** (`lantern-crypto/THREAT_MODEL.md` X7,
`lantern-boot/THREAT_MODEL.md` B5) — signing measured-boot state with the Ed25519 key
this RFC ratifies — is a potential device fingerprint. This RFC fixes the *algorithm*,
not attestation's exposure policy (who can request it, how much detail it reveals); that
remains an open question in both `lantern-crypto/ARCHITECTURE.md` and
`lantern-boot/ARCHITECTURE.md`, tracked separately.

## Alternatives considered

- **Defer this RFC until `lantern-crypto` starts real implementation work.** Rejected:
  `lantern-boot` is blocked on it *now* per its `STATUS.md`, and `GOVERNANCE.md` requires
  the RFC regardless of implementation timing — waiting only delays `lantern-boot`, which
  has nothing else standing in its way besides this and the HAL bring-up contract (now
  further along).
- **NIST P-256/P-384 (NIST curves) instead of Ed25519/X25519.** Rejected:
  Curve25519-family primitives have simpler, more widely audited constant-time
  implementations and avoid the historical NIST-curve nonce/parameter-generation
  concerns; Ed25519/X25519 are also what most comparable modern systems (age, Signal,
  WireGuard) converged on for the same reasons.
- **Skip reserving PQC-hybrid slots in Phase 1; add them only when actually implementing
  PQC.** Rejected: the whole point of fixing crypto-agility now is that *not* reserving
  the slot is exactly the kind of decision that becomes a breaking change later — the
  same reasoning ADR-0009 used for keeping the MCS scheduling-context object shape now
  even though full MCS semantics are deferred.
- **Roll a custom AEAD/KDF construction tuned to LanternOS.** Rejected outright:
  `lantern-crypto/ARCHITECTURE.md`'s "Assurance and `unsafe`" principle already commits
  to audited, well-reviewed implementations over rolling our own; this RFC does not
  reconsider that commitment.

## Prior art

**age** and **libsodium** for the "misuse-resistant, high-level operations over raw
primitives" API philosophy already committed to in `lantern-docs/wiki/Cryptography.md`.
**Signal** and **WireGuard** for the Curve25519-family (Ed25519/X25519) default and its
constant-time-implementation track record at scale. **TLS 1.3 / the broader PQC-hybrid
migration literature** (NIST's ML-DSA/ML-KEM standardization) for the hybrid
classical+PQC pattern this RFC reserves space for without committing to a specific
construction yet.

## Unresolved questions

- Exact hybrid-PQC construction (concatenation vs. nested vs. combiner-function hybrid)
  and when it becomes default rather than opt-in.
- Key backup/recovery mechanism (social recovery, hardware tokens, sharding) — untouched
  by this RFC.
- Attestation's privacy exposure policy — algorithm fixed here, policy not.
- Whether any low-level primitive is ever exposed directly to apps, or only high-level
  sealed operations — `lantern-crypto/ARCHITECTURE.md`'s existing open question, not
  resolved here.
- The sealed-capability token format and its exact use of the primitives ratified here —
  deferred to a joint `lantern-crypto`/`lantern-capabilities` RFC.

## Future possibilities

- The PQC-hybrid construction RFC, once ML-DSA/ML-KEM implementations mature enough for
  the project's audited-implementation bar.
- The sealed-capability token-format RFC (`lantern-crypto` + `lantern-capabilities`),
  building directly on this primitive set.
- Hardware-enclave-backed key custody design, once a concrete hardware target is chosen
  (ties to `lantern-hal`/`lantern-boot`'s hardware-root-of-trust open question).

## Resulting ADRs

[ADR-0011](../adr/0011-cryptographic-primitive-set.md) fixes the Phase 1 primitive table
and the crypto-agility/PQC-reservation stance.
