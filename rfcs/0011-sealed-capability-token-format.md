---
rfc: 0011
title: Sealed capabilities — a macaroon-style cryptographic token format
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["capabilities", "crypto"]
domains: ["capabilities", "crypto"]
created: 2026-08-23
updated: 2026-08-23
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0011: Sealed capabilities — a macaroon-style cryptographic token format

## Summary

This RFC fixes the concrete design of the third capability layer RFC-0003/ADR-0006 already
named but left unspecified: **sealed capabilities** — signed, attenuable bearer tokens used
when delegation must cross machines or persist beyond a live kernel/service capability's
lifetime. It defines the token structure, the caveat-chaining construction (BLAKE3-keyed-MAC
macaroons, per [RFC-0007](./0007-cryptographic-primitive-set.md)'s reserved keyed-hash/MAC
use), how `seal`/`unseal` map onto the existing kernel/service capability layers, and the
revocation model. It resolves RFC-0003's own "mapping between sealed and live kernel caps at
trust boundaries" unresolved question and closes out
[`lantern-crypto`](../../lantern-crypto)'s and
[`lantern-capabilities`](../../lantern-capabilities)'s long-standing "sealed-cap token
format" Next item.

## Motivation

RFC-0003 names sealed capabilities as LanternOS's third layer and gives them a job
("delegation that must cross machines or persist") and two operations
(`seal(cap)`/`unseal(token)`) — but never specifies what a sealed token actually *is*. That
gap has sat as an open item since Phase 0: `lantern-capabilities/STATUS.md` has carried "the
sealed-cap token format (RFC-0003's third layer)" as a blocked Next item since its first
prototype, and `lantern-crypto/STATUS.md` has carried "specify the sealed-capability token
format" the same way. Both were genuinely blocked — no keyed-MAC or signature primitive
existed to build the format on. That gap just closed: `lantern-crypto` now has real Ed25519
signing and a real BLAKE3 keyed-mode MAC key
([`lantern_crypto::hash::MacKey`](../../lantern-crypto/src/hash.rs)), gated through
`Keystore` exactly like every other secret key. [RFC-0010](./0010-cross-process-capability-transfer-and-brokering.md)'s
own "Future possibilities" named this as the next natural step once that existed.

This is squarely an RFC-required decision under [`GOVERNANCE.md`](../../GOVERNANCE.md): it
fixes a new trust boundary (authority that can leave a single kernel instance and be
presented later, by a different party, potentially on a different machine) and a
cryptographic wire format, not an interface-preserving refactor.

This serves **security by architecture** (a specified, reviewable format instead of an
ad hoc one improvised the first time some service needs cross-machine delegation) and
**local-first ownership** (sealed caps are exactly what lets delegation work when the network
is unavailable or the counterparty isn't reachable synchronously — the token, once issued, is
self-contained).

## Guide-level explanation

A sealed capability is a **bearer token**: whoever holds the bytes can present them. It is
built the way a physical wax seal works — anyone can check a seal wasn't broken, but only the
sealer's signet ring could have made it, and anyone downstream can press their own signet
into the still-soft wax to add a restriction without needing the original ring back.

Concretely, using a macaroon-style construction:

1. An **issuer** — a confined service that already holds a live capability (kernel or
   service layer) it wants to delegate — calls `seal(cap, caveats)`. This mints a fresh
   secret **root key** (via `Keystore::generate_mac_key`), computes a chained MAC over an
   opaque `identifier` and the initial `caveats` (e.g. "read-only", "expires in 1 hour"), and
   hands back a `SealedToken`: `{ identifier, caveats, mac }`. The root key never leaves the
   issuer's `Keystore`.
2. Anyone holding a `SealedToken` can **attenuate** it further — append a new caveat (e.g.
   "and also: only between these hours") and re-chain the MAC — without contacting the
   issuer or ever seeing the root key. This is the same "can only narrow, never widen"
   discipline `CNodeInvoke::Mint` already enforces for live capabilities, just usable
   offline.
3. Presenting the token back to the issuer (or a party the issuer trusts to verify — see
   "Unresolved questions") calls `unseal(token)`: the verifier recomputes the same MAC chain
   from its own copy of the root key (looked up by `identifier`) and checks it matches, then
   evaluates every caveat against the current context (is it actually still before the
   expiry? does the request actually stay within the attenuated rights?). Only if both checks
   pass does the verifier do anything privileged — and what it does is **mint and grant a
   fresh live service capability** via the existing [`lantern_capabilities::Broker`](../../lantern-capabilities/src/lib.rs),
   attenuated to the caveats' intersection. A sealed token by itself authorises nothing
   inside this machine; it only ever gets to *ask* a broker to mint something, exactly as any
   other client would.

Example: `lantern-filesystem` (once it exists) holds a live capability to a document object.
A user wants to share read access with a collaborator's device that isn't currently
reachable over IPC. The filesystem service seals a `READ`-only, 24-hour-expiry capability,
hands the resulting bytes to `lantern-network` for delivery. The collaborator's device
presents the token back later; the filesystem service unseals it, checks the expiry hasn't
passed, and mints a real, live, badged capability for that session — the same mechanism a
same-machine `Broker::grant` client gets, just reached via a token that survived being
disconnected in between.

## Reference-level explanation

### Token structure

```
SealedToken {
    identifier: [u8; 16],       // opaque; issuer's lookup key for the root key + underlying
                                 // live capability this token designates. Not secret.
    caveat_count: u8,           // 0..=MAX_CAVEATS
    caveats: [Caveat; MAX_CAVEATS],  // fixed-capacity, no_std/no-alloc, matching every other
                                     // pool in this project (Broker::MAX_GRANTS,
                                     // Keystore::MAX_KEYS)
    mac: [u8; 32],               // lantern_crypto::hash::HASH_LEN — the running keyed-MAC
}
```

`MAX_CAVEATS` is small (proposed: 4) — Phase 2's actual delegation needs are narrow; raising
it later is additive, not a format break, the same "narrowest slice that closes the concrete
gap" precedent RFC-0008 set shipping `Mega`-only frames.

### Caveat set (Phase 2)

A closed, small enum — deliberately **not** a general expression language yet:

```
enum Caveat {
    RightsSubset(Rights),   // narrows the rights the eventual minted live cap may carry
    ExpiresAt(u64),         // a monotonic timestamp source TBD by lantern-hal; see
                             // "Unresolved questions"
}
```

Third-party caveats (macaroons' mechanism for "this restriction must be discharged by a
*different* party") are explicitly out of scope — see "Future possibilities". Every caveat
here is a **first-party caveat**: checked locally by whichever service unseals the token,
against context it already has.

### MAC chaining

Using [`lantern_crypto::hash::MacKey`](../../lantern-crypto/src/hash.rs) exactly as it exists
today, no new primitive:

```
mac_0 = MacKey(root_key).compute(identifier)
mac_i = MacKey(mac_{i-1}).compute(encode(caveat_i))     for i in 1..=caveat_count
```

`encode(caveat)` is a fixed, small byte encoding of the `Caveat` enum (tag byte + fixed
payload — no variable-length fields, so no framing ambiguity). The token's `mac` field is
`mac_{caveat_count}`. This is the standard macaroon HMAC-chaining construction with BLAKE3's
native keyed mode standing in for HMAC — the same substitution RFC-0007 reserved BLAKE3's
keyed mode for.

**Attenuation** (`attenuate(token, new_caveat) -> SealedToken`) appends `new_caveat` and
computes `mac_{n+1} = MacKey(token.mac).compute(encode(new_caveat))` — using the *previous
MAC value itself* as the next link's key. This is what lets any holder attenuate without the
root key: they have `mac_n` (it's right there in the token), which is exactly the key the
next link needs. They do **not** have `root_key` or any `mac_i` for `i < n`, so they cannot
*remove* a caveat or forge a token with fewer restrictions — only ever append.

### `unseal` and the live-capability boundary

`unseal(token, context) -> Result<(), SealedCapError>` (verification only) plus a
service-specific "and then mint" step:

1. Look up `root_key` and the underlying live capability/rights by `token.identifier`. Unknown
   `identifier` → reject, deny-by-default (same convention as
   `Broker::is_revoked`/`Keystore::check_access`).
2. Consult a local **revocation table** keyed by `identifier` (see "Revocation" below).
   Revoked → reject before touching the MAC at all — cheaper and avoids leaking timing
   information about *why* a token failed.
3. Recompute the MAC chain from `root_key` through every caveat in order; constant-time
   compare (`MacKey::verify`, already real) against `token.mac`. Mismatch at any point →
   reject.
4. Evaluate every `Caveat` against `context` (current time vs. `ExpiresAt`; the operation
   being attempted vs. `RightsSubset`). Any unsatisfied caveat → reject.
5. Only on success: the unsealing service calls `Broker::mint`/`Broker::grant` (or
   `grant_via_reply`) on the live capability `identifier` designates, with rights narrowed to
   the intersection of every `RightsSubset` caveat present. This is the resolution to
   RFC-0003's open "mapping between sealed and live kernel caps" question: **a sealed
   capability never bypasses the service layer — it only ever justifies a fresh, ordinary,
   broker-mediated mint.** No new kernel- or service-layer trust path is introduced.

### Root-key custody

A root key is an ordinary `lantern_crypto::Keystore` MAC key
(`KeyPurpose::Mac`/`KeyOps::MAC`, already implemented) — generated via
`Keystore::generate_mac_key`, never returned to a caller as raw bytes (X1,
`lantern-crypto/THREAT_MODEL.md`), used only through `Keystore::mac`/`verify_mac`'s existing
badge-gated interface. Sealing/unsealing therefore requires holding an appropriately-scoped
`Keystore` badge exactly like every other MAC operation — no new authority model, just a new
consumer of the one that already exists.

### Revocation

Bearer tokens cannot be cryptographically un-issued once handed out — this is inherent to the
construction, not a gap in it. The mandated mitigation, mirroring `Broker::revoke`'s already
sanctioned Phase 2 pattern exactly:

- Every `identifier` the issuer has ever sealed a root token under is tracked in a
  `identifier → revoked: bool` table, consulted first in `unseal` (step 2 above).
- Issuers **should** default to attaching a short `ExpiresAt` caveat at seal time (a
  recommended default, not a format requirement) so an un-revoked-but-abandoned token decays
  on its own rather than requiring active revocation to become harmless.
- Revoking an `identifier` invalidates every token derived from it, including every
  attenuated descendant — because every descendant's MAC chain still bottoms out at the same
  `root_key`/`identifier`, revoking the root revokes the whole family in one write, with no
  derivation tree to walk (unlike the still-unimplemented kernel-level `Revoke`,
  `lantern-kernel/STATUS.md`).

## Threat model impact

- **Trust boundaries affected:** this is the first mechanism by which authority can leave a
  single kernel instance's live capability space and be presented later, potentially by a
  different party or after a network round trip. It does not weaken the kernel or
  service-layer boundaries RFC-0003/RFC-0010 already fixed — `unseal` success only ever
  produces an ordinary, broker-mediated, rights-checked live capability.
- **New assets introduced and who can reach them:** (1) root keys — `Keystore`-custodied MAC
  keys, reachable only through the same badge-gated interface as every other key `Keystore`
  holds; (2) outstanding `SealedToken` bytes — bearer-secret by construction, so their
  confidentiality *in transit* matters (a network eavesdropper who captures a token can
  present it); this RFC does not itself provide transport confidentiality — that is
  [`lantern-network`](../../lantern-network)'s job (authenticated encrypted channels,
  `wiki/Networking.md`), out of scope here but load-bearing for real deployment; (3) the
  issuer's `identifier → root_key/revoked` table, a security-critical asset with the same
  criticality `Broker`'s own `badge → revoked` table already has.
- **New adversary capabilities, if any:** a token holder can attenuate offline, without the
  issuer's involvement or knowledge, at the moment of attenuation. This is intentional and
  bounded by monotone attenuation (only ever narrows) — flagged explicitly because it is a
  real, deliberate capability a purely broker-mediated live capability does not have (every
  live `mint` is synchronous and broker-observed; every sealed-cap attenuation is not).
- **Mitigations:** BLAKE3-keyed MAC chaining (forging a valid token without `root_key` is
  exactly as hard as forging a MAC under an unknown key); monotone-attenuation-only chaining
  (cannot strip a caveat without an earlier link's MAC, which an attenuator never has);
  mandatory deny-by-default revocation-table check before any MAC work; constant-time MAC
  comparison (X6); root keys never leave `Keystore` in the clear (X1).
- **Net change to attacker surface:** increases, deliberately — this is Phase 2/3's sanctioned
  way to do cross-machine delegation at all, the same framing RFC-0010 used for cross-process
  `grant`. The increase is bounded by the mitigations above and by `unseal`'s hard requirement
  to still go through an ordinary `Broker::mint`/`grant` call — a compromised or forged token
  can never do more than a legitimately narrowly-scoped live capability could.

## TCB impact

- **Does this add code to the Trusted Computing Base?** No. Sealing, attenuation, and
  unsealing are all confined user-space operations (`lantern-crypto`, and whichever service
  does the unsealing) built entirely on `Keystore`/`Broker`, neither of which is in the TCB
  (ADR-0006). `lantern-kernel` is untouched by this RFC — unlike RFC-0010, no syscall, ABI, or
  kernel object is involved.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** none. This is entirely a service-layer/crypto-layer
  design; it composes primitives and mechanisms already accepted (RFC-0007's BLAKE3 keyed
  mode, RFC-0010's `Broker`), so it introduces no new TCB-adjacent surface to audit.

## Privacy impact

`identifier` is opaque but not secret, and is the one piece of a sealed token every verifier
sees — it should not itself be linkable across contexts (e.g. not a raw user ID or a reused
value across unrelated delegations), matching **privacy by default**'s "what does this let
someone learn" test. This RFC recommends `identifier` be a fresh random value per `seal` call,
not a stable per-object or per-user identifier; enforcing that is left to whichever service
issues tokens, the same way `Broker`'s badge counter is already fresh-per-grant rather than
identity-derived. Caveats themselves may carry sensitive context (an `Audience` caveat, if
added later, names a party) — deferred to "Future possibilities" alongside third-party
caveats generally, so this RFC's Phase 2 caveat set (`RightsSubset`, `ExpiresAt`) carries no
new privacy-sensitive fields.

## Alternatives considered

- **Plain asymmetric signatures (Ed25519) instead of MAC chaining for every caveat.**
  Rejected as the *chaining* mechanism: a signature can prove "the issuer authorised exactly
  this token," but re-signing on every attenuation step requires the private signing key at
  each step — defeating the whole point of offline, root-key-free attenuation. Ed25519 stays
  available in `lantern-crypto` and is the right tool for `seal`'s own root issuance
  certification, should a future revision want the issuer's *identity* attested (see "Future
  possibilities"); it is the wrong tool for the caveat chain itself.
- **A general caveat expression language (arbitrary predicates) now.** Rejected for Phase 2:
  no concrete consumer needs more than "narrower rights" and "expires by," and an expression
  language is real design and TCB-adjacent-parsing surface with no current justification —
  same "narrowest slice" precedent as the fixed `Caveat` enum above.
- **Third-party caveats (macaroons' discharge-macaroon mechanism) now.** Rejected: needs a
  network identity/DID story ([Identity](../../lantern-docs/wiki/Identity.md)) that is
  explicitly Phase 3, not built yet. Deferred, not designed out — the chaining construction
  here is compatible with adding it later without a format break (macaroons are designed for
  exactly this kind of incremental extension).
- **Do nothing — keep delegation kernel/service-layer-only, no cross-machine story.**
  Rejected: directly blocks the Phase 3 exit criterion ("a network identity can be presented
  without cross-context linkage") and every P2P-sync use `wiki/Networking.md`/
  `wiki/Filesystem.md` already name as a design goal.

## Prior art

Google's original macaroons paper (Birgisson et al., "Macaroons: Cookies with Contextual
Caveats for Decentralized Authorization") — this RFC is a direct application of that
construction, substituting BLAKE3 keyed mode for HMAC-SHA256 per this project's own
RFC-0007/ADR-0011 primitive choice. Biscuit tokens (a more recent macaroon-family design using
public-key chaining) were considered as a richer alternative but rejected for Phase 2 as more
machinery than the current caveat set justifies. Capability-based distributed systems
generally (Fuchsia's cross-device story, EROS/KeyKOS's original sealed-capability framing)
motivate treating this as a distinct layer rather than stretching the live kernel/service
capability model across a network boundary.

## Unresolved questions

- Exact `Caveat` byte encoding (`encode()`) — left to implementation, to be documented
  alongside `lantern-crypto`'s existing fixed-encoding conventions
  (`cnode::LABEL_MINT`'s packed-argument precedent).
- A monotonic timestamp source for `ExpiresAt` — `lantern-hal` has no clock abstraction yet;
  this RFC assumes one will exist before `ExpiresAt` caveats are enforceable, and does not
  block acceptance on it (`RightsSubset`-only sealed tokens are still useful without it).
- Whether verification (`unseal`) must always happen at the original issuer, or whether a
  root key can be safely shared with a small set of mutually-trusted verifying replicas (a
  multi-device local-first sync scenario `lantern-filesystem/ARCHITECTURE.md` already flags
  as an open question). This RFC assumes single-issuer verification as the Phase 2 baseline.
- Which crate owns the actual `SealedToken`/`seal`/`unseal` implementation —
  `lantern-crypto` (closest to the primitives) or `lantern-capabilities` (closest to
  `Broker`, which `unseal` must call into) — left to implementation to decide once it's
  written against both; either choice is compatible with this RFC's design.

## Future possibilities

- Third-party caveats once an identity/DID system exists (Phase 3) — "this caveat must be
  discharged by party X" delegated verification, the full macaroon model.
- An `Audience`/identity-bound caveat type, once linkable-identity trade-offs are worked out
  (**privacy by default** tension to resolve explicitly, per this project's principle-conflict
  discipline).
- Ed25519-signed root issuance (issuer authenticity, not just root-key-holder authenticity) —
  useful once tokens might be verified by a party other than the original issuer.
- Wiring `seal`/`unseal` into `lantern-sdk` so application/agent code gets an ergonomic API
  rather than hand-building `SealedToken`s.

## Resulting ADRs

On acceptance, an ADR will fix (a) the macaroon-style, BLAKE3-keyed-MAC-chained token
structure as sealed capabilities' canonical format, and (b) `unseal` success mapping only ever
onto an ordinary `Broker::mint`/`grant` call as the resolution to RFC-0003's sealed-to-live
capability mapping question.
