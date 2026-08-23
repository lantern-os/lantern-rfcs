---
adr: 0015
title: Sealed capabilities — macaroon-style BLAKE3-keyed-MAC token format
status: Accepted
date: 2026-08-23
deciders: ["TSC"]
rfc: ../rfcs/0011-sealed-capability-token-format.md
supersedes: []
superseded_by: null
---

# ADR-0015: Sealed capabilities — macaroon-style BLAKE3-keyed-MAC token format

## Context

RFC-0003/ADR-0006 named sealed capabilities as LanternOS's third capability layer — signed,
attenuable bearer tokens for delegation that crosses machines or persists — and gave them two
operations (`seal`/`unseal`), but left the actual token format as an open question tracked in
both `lantern-capabilities/STATUS.md` and `lantern-crypto/STATUS.md`. That format could not
be specified earlier because no keyed-MAC or signature primitive existed to build it on;
`lantern-crypto`'s `Keystore` (Ed25519 signing, BLAKE3-keyed `MacKey`) closed that gap.
[RFC-0011](../rfcs/0011-sealed-capability-token-format.md) proposed a concrete design and has
been accepted; this ADR is the durable record.

## Decision

**Sealed capabilities are macaroon-style tokens, chained with BLAKE3's keyed-hash mode as the
MAC, per RFC-0011.**

- A `SealedToken` is `{ identifier, caveats (a small fixed-capacity, closed set for Phase 2:
  `RightsSubset`, `ExpiresAt`), mac }`.
- The MAC chains: `mac_0 = MacKey(root_key).compute(identifier)`,
  `mac_i = MacKey(mac_{i-1}).compute(encode(caveat_i))`. A root key is an ordinary
  `Keystore` `KeyPurpose::Mac` key, never exposed outside the badge-gated `Keystore`
  interface.
- **Attenuation needs no root key**: appending a caveat and rechaining from the token's own
  current `mac` is sufficient, which is what lets any holder narrow a token offline without
  contacting the issuer. This is monotone-only — attenuating cannot remove an earlier caveat,
  since doing so would require an earlier link's MAC value, which an attenuator never has.
- **`unseal` success never bypasses the service layer.** A verified, un-revoked,
  caveat-satisfying token only ever justifies the verifying service calling an ordinary
  `lantern_capabilities::Broker::mint`/`grant` on the live capability `identifier` designates,
  attenuated to the caveats' intersection. This is the resolution to RFC-0003's open "mapping
  between sealed and live kernel caps" question: sealed capabilities are a credential that
  justifies a fresh live mint, never a second path to authority that skips kernel/service-layer
  checks.
- **Revocation is a deny-by-default `identifier → revoked` table**, consulted before any MAC
  computation, mirroring `Broker::is_revoked` exactly. Revoking an identifier invalidates every
  token descended from it in one write, since every descendant's chain bottoms out at the same
  root key.
- Third-party caveats, an `Audience`/identity-bound caveat type, and Ed25519-signed root
  issuance are explicitly deferred (Phase 3, pending an identity/DID system) — this format is
  designed to extend to them later without a wire-format break, not to include them now.

## Consequences

- **Easier:** `lantern-crypto` and `lantern-capabilities` can now implement `seal`/`attenuate`/
  `unseal` against a fixed, reviewed design instead of improvising one the first time a
  concrete consumer (cross-device sharing, P2P sync) needs it. The construction reuses
  primitives already accepted (RFC-0007's BLAKE3 keyed mode, RFC-0010's `Broker`) — no new
  cryptographic primitive, no kernel or ABI change, so implementation carries none of
  RFC-0010's TCB-adjacent review burden.
- **Harder:** bearer tokens cannot be cryptographically un-issued once handed out — the
  revocation table and the recommended-default `ExpiresAt` caveat are the only mitigations,
  and both are policy the issuing service must actually apply, not something this format
  enforces structurally. Offline attenuation is a deliberate new adversary capability (a token
  holder can narrow a token without the issuer's knowledge at the moment of attenuation) that
  every future consumer of this format needs to keep in its own threat model, not just this
  ADR's.
- **Committed to:** the token structure and MAC-chaining algorithm above are the canonical
  sealed-capability format; any future revision (e.g. adding third-party caveats) is a new RFC
  that supersedes this one, not a silent extension. `unseal` mapping onto an ordinary
  `Broker::mint`/`grant` — never a separate privileged path — is fixed as the sealed-to-live
  boundary for every future sealed-capability consumer.
- **Still open:** exact caveat byte encoding, a clock/timestamp source for `ExpiresAt`
  (`lantern-hal` has none yet), whether verification must stay single-issuer, and which crate
  (`lantern-crypto` or `lantern-capabilities`) owns the implementation — all left to
  implementation per RFC-0011's "Unresolved questions", tracked in `lantern-crypto/STATUS.md`.
