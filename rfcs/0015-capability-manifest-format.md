---
rfc: 0015
title: The lantern-sdk capability manifest format
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["sdk", "runtime"]
domains: ["sdk", "runtime", "capabilities"]
created: 2026-08-29
updated: 2026-08-29
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0015: The lantern-sdk capability manifest format

## Summary

This RFC fixes the **capability manifest** a LanternOS app or agent ships with: the
declarative file a developer authors to state, per imported host interface, exactly what
authority the component needs. [RFC-0014](./0014-wit-handle-capability-mapping.md) fixed
the *runtime-side* contract this must satisfy — one badge per resource-scoped grant, one
yes/no per link-scoped facility, populated only at instantiation — and explicitly left the
file format to `lantern-sdk` (`lantern-sdk/STATUS.md`'s "design the capability manifest
format"). This RFC fixes: a **TOML** file (`lantern.capabilities.toml`); a schema keyed by
WIT interface with two declaration shapes mirroring RFC-0014's two mapping shapes; a
**mandatory human-readable justification** on every request; a **compile relationship** to
`lantern-runtime`'s `GrantManifest` in which resource-scoped declaration order *is* the
`open(slot)` index; the manifest as the **single source of truth** from which the SDK
generates the component's WIT `world`, so a component's imports can never exceed its
manifest; and the manifest bound into the **signed package** so code and manifest cannot be
recombined. It does **not** decide the launch-time *binder* (who resolves an abstract role
to a concrete object, and the consent UX — `lantern-shell`'s job), per-language binding
ergonomics, or an installed app's manifest-evolution/upgrade story.

## Motivation

`lantern-sdk` is the last Phase 2 component with no concrete artifact, and it is now the
**critical path** to the Phase 2 exit criterion — "a third-party Wasm app runs confined,
reads a file *only* via a granted capability, demonstrated adversarially"
([Roadmap](../../lantern-docs/wiki/Roadmap.md)). RFC-0014 gave `lantern-runtime` a host
surface; nothing yet lets a developer *declare* what a component may reach, and
`lantern-runtime`'s `GrantManifest` is a Rust struct built by hand in tests, not something
produced from a package.

The manifest is where two of `lantern-sdk`'s own threats are answered
(`lantern-sdk/THREAT_MODEL.md`): **S1** (developers ship over-privileged apps) and **S2**
(a dishonest or over-broad manifest). It is also where
[Principle 1](../../lantern-docs/wiki/Principles.md) (security by architecture — "explicit
permissions, least privilege") and [Principle 2](../../lantern-docs/wiki/Principles.md)
(privacy by default — "if the answer is 'something they did not consent to', it is a bug")
are made concrete for the app tier: "a component's declared capabilities are exactly what
it can be granted; the user/shell sees them before consenting"
(`lantern-sdk/ARCHITECTURE.md`). Fixing the format now — before the SDK's build tooling,
the WIT generator, and `lantern-shell`'s consent surface are all written against whatever
an implementation PR happened to pick — is the trust-boundary-adjacent decision
`GOVERNANCE.md` reserves for an RFC. The manifest is not itself an enforcement boundary
(enforcement is the runtime's and kernel's job), but it is the contract three components
must agree on.

The one point of tension is with the kernel's **mechanism-not-policy** doctrine
(RFC-0002/ADR-0004): the manifest is unavoidably policy-shaped — it encodes *what an app
wants*. This is acceptable because it lives entirely in tooling and package metadata,
never in the kernel, HAL, or the runtime's enforcement path; the runtime still only ever
sees a `GrantManifest` of concrete grants, exactly as RFC-0014 fixed.

## Guide-level explanation

A LanternOS app is a Wasm component plus a `lantern.capabilities.toml` next to its source.
The developer writes, for each host facility the app needs, one declaration with a plain
sentence explaining *why*:

```toml
# lantern.capabilities.toml
[app]
name    = "receipt-signer"
version = "0.3.1"

# Resource-scoped (RFC-0014): each [[keystore.key]] becomes one handle the component
# obtains via `keystore.open(N)`, where N is this list's order — first entry is slot 0.
[[keystore.key]]
role          = "receipt-signing"
ops           = ["sign"]
justification = "Signs each exported receipt so a recipient can verify it came from this app."

[[keystore.key]]
role          = "vault-at-rest"
ops           = ["encrypt", "decrypt"]
justification = "Encrypts the local receipt archive so another app cannot read it."

# Link-scoped (RFC-0014): present = requested, absent = not requested. No parameters —
# the facility is granted whole or refused whole.
[monotonic-clock]
justification = "Stamps each receipt with a monotonic sequence number."
```

Then `lantern-sdk build`:

1. **Validates** the manifest — every declaration names a real interface, every `ops`
   value is legal for that resource, every entry has a non-empty `justification`.
2. **Generates the component's WIT `world`** from the manifest and nothing else. The
   world above imports `lantern:crypto/keystore` and `lantern:host/monotonic-clock` — and
   *only* those. A component physically cannot import a facility its manifest didn't
   declare, because the world it's compiled against doesn't contain it.
3. **Packages** the component and the manifest together into one signed artifact
   (RFC-0013's `.cwasm` + Ed25519 signature), with the signature covering both, so the
   manifest that ships is the manifest that was reviewed.

At launch, `lantern-shell` (not this RFC) shows the user each `role`/facility with its
`justification` and `ops`, lets them bind each resource-scoped role to a concrete object
(*which* key) or decline it, obtains the real capabilities from the owning services, and
hands `lantern-runtime` a `GrantManifest`. A declined resource-scoped role leaves that
slot empty — `keystore.open(N)` returns `none`, exactly as RFC-0014 specifies. A declined
link-scoped facility is left unlinked — the component fails to instantiate, which the
shell surfaces as "this app requires X to run."

The manifest declares abstract **roles**, never concrete objects. `"receipt-signing"` is a
name the app uses to talk about *its own* need; the mapping from that name to an actual
`KeyId` is made by the user at launch, recorded by the shell, and never travels in the
package. This is what keeps the manifest signable and portable, and what keeps the
user — not the developer — in control of *which* of their keys an app touches.

## Reference-level explanation

### File: `lantern.capabilities.toml`

TOML, UTF-8, at the package root. TOML is chosen over JSON/YAML for the same reasons
`Cargo.toml` is TOML: comments (a `justification` is a comment-adjacent thing and the file
should read well), unambiguous parsing, stable diffs, and it is already the format every
Rust developer in this ecosystem authors by hand. A `lantern-sdk` crate owns the canonical
parser and serializer.

```
manifest        := app-table interface-table*
app-table       := "[app]" name version
interface-table := resource-scoped-array | link-scoped-table
```

#### `[app]`

| key       | type   | required | meaning |
| --------- | ------ | -------- | ------- |
| `name`    | string | yes      | Stable identifier, `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`. Shown in consent UX. |
| `version` | string | yes      | Semver. Advisory only in Phase 2 (no upgrade logic yet — see "Unresolved"). |

`[app]` carries no capabilities. It exists so a manifest read in isolation (by the shell,
by a registry later) identifies its app.

#### Resource-scoped declarations

One TOML array-of-tables per resource-scoped WIT resource, named `<interface-local-name>.<resource-name>`.
For RFC-0014's keystore that is `[[keystore.key]]`. Each entry:

| key             | type       | required | meaning |
| --------------- | ---------- | -------- | ------- |
| `role`          | string     | yes      | An app-local name for *this* need. Unique within the array. Shown in consent UX. `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`. |
| `ops`           | \[string]  | yes      | The requested operation subset. Each value must be legal for the resource (for `keystore.key`: `"encrypt"`, `"decrypt"`, `"sign"` — the `KeyOps` names, lower-cased). Empty is an error: a handle you can do nothing with is a bug, not a grant. |
| `justification` | string     | yes      | One or two sentences, non-empty after trimming, ≤ 280 chars. Written for the *user*, not the reviewer. |

**Order is load-bearing.** The Nth `[[keystore.key]]` entry (0-indexed, in file order)
is the handle the component receives from `keystore.open(N)`. The SDK-generated bindings
give each `role` a named accessor (`open_receipt_signing()`) that calls `open` with the
right constant, so the developer never writes the slot number — but the wire contract is
the index, and it is fixed by declaration order. Reordering entries in a shipped manifest
is a breaking change to the component.

#### Link-scoped declarations

One TOML table per link-scoped interface, named by its interface-local name. For RFC-0014's
clock that is `[monotonic-clock]`. The table's *presence* is the request.

| key             | type   | required | meaning |
| --------------- | ------ | -------- | ------- |
| `justification` | string | yes      | Same rules as above. |

No `ops`, no `role` — link-scoped facilities have no per-call object and no sub-operations
to attenuate (RFC-0014). Absent table = not requested = not linked.

#### The interface registry

Which interface names are legal, and which declaration shape and `ops` vocabulary each
has, is **not** open-ended: `lantern-sdk` ships a registry, one entry per interface
`lantern-runtime` implements, kept in lockstep with `lantern-runtime/wit/`. Phase 2 seeds
it with exactly the two RFC-0014 interfaces. A manifest naming an interface not in the
registry fails validation — a developer cannot request a facility the runtime cannot
satisfy. Adding an interface to the registry is a coordinated `lantern-runtime` +
`lantern-sdk` change; whether it also needs an RFC follows the normal rule (a new
trust-boundary-fixing interface does, per RFC-0014's own precedent).

### Compilation to `lantern-runtime::GrantManifest`

The manifest is a **request**. It contains no badges — badges are minted by the owning
services at grant time (RFC-0010) — and no `KeyId`s. Turning a validated manifest into the
`GrantManifest` `lantern-runtime` consumes is a launch-time step, performed by the binder
(`lantern-shell`, out of scope here) against user choices:

```
for keystore slot i = 0 .. N   (N = number of [[keystore.key]] entries):
    entry[i] = role, ops, justification
    user picks a concrete key K for entry[i].role, or declines:
        declined:  slot i = empty                 (open(i) → none)
        approved:  ask the keystore service to mint+grant a badge B scoped to K
                   and entry[i].ops;  slot i = HostCapability{ badge: B, key: K }

for the [monotonic-clock] table:
    present and user-approved  ──►  monotonic_clock = Some(the platform clock)
    absent or declined         ──►  monotonic_clock = None
```

**Declaration order is the permanent slot index, with holes.** The Nth `[[keystore.key]]`
entry is always `open(N)` — whether the user approved it, declined it, or hasn't been
asked yet. A declined role is an empty slot, and `open(N)` on it returns `none` exactly as
it does for an un-opened handle: at the WIT level a declined grant and an un-opened one are
indistinguishable to the guest, which is the property RFC-0014 wants. This means
`lantern-runtime`'s `GrantManifest.keystore_keys` is **positional with holes**
(`Vec<Option<HostCapability>>`, or an equivalent sparse map) rather than the dense
`Vec<HostCapability>` the RFC-0014 prototype currently uses — a small, mechanical change to
that prototype, called out here because it is the one place this RFC touches
`lantern-runtime`'s shape.

The SDK-generated bindings give each `role` a named accessor
(`open_receipt_signing() → open(0)`, `open_vault_at_rest() → open(1)`) so the developer
never writes a slot number and never sees the holes; the numeric index is purely the wire
contract, fixed by declaration order. Reordering or removing entries in a shipped manifest
is therefore a breaking change to the component (its compiled accessors call the old
indices) — an installed-app concern deferred with the rest of manifest evolution (see
"Unresolved questions").

### The signed package

RFC-0013 fixed a signed `.cwasm` artifact and left "the `.cwasm` artifact's signing-key
management story" to `lantern-sdk`. This RFC fixes the envelope shape only far enough to
bind the manifest:

- The package is `{ manifest_bytes, cwasm_bytes }`.
- What is signed (Ed25519, RFC-0007's primitive, as RFC-0013) is
  `BLAKE3(manifest_bytes) || BLAKE3(cwasm_bytes)` — a 64-byte value. Signing the two
  hashes rather than one concatenated blob lets the runtime verify the `.cwasm` it is
  about to `deserialize` against its own hash without having the manifest in hand, and
  lets the shell verify the manifest it is about to render without pulling the whole
  `.cwasm` — while still making it impossible to pair a manifest with a `.cwasm` it was
  not signed with.
- `compiler::precompile_and_sign` (`lantern-runtime`) gains a manifest argument or a
  sibling `sdk`-side function computes the combined digest; the exact API split is an
  implementation detail, not fixed here.

### Validation rules (an implementer's checklist)

1. Valid TOML; `[app]` present with valid `name`/`version`.
2. Every table/array name is either `[app]` or a registered interface's declaration name.
3. Resource-scoped: every entry has `role` (valid pattern, unique in its array), `ops`
   (non-empty, every value legal for that resource), `justification` (non-empty trimmed,
   ≤ 280 chars).
4. Link-scoped: table has exactly `justification`, same rules; no other keys.
5. No interface appears both as a resource-scoped array and a link-scoped table.
6. The generated WIT `world` imports exactly the declared interfaces — checked by
   regenerating and diffing, so a hand-edited world cannot drift from the manifest.

Any failure is a hard build error with the offending line.

### Error behaviour at the seams

- **Manifest fails validation:** `lantern-sdk build` fails; nothing is packaged.
- **Manifest/`.cwasm` signature mismatch at launch:** the binder refuses to launch; same
  posture as RFC-0013's tampered-artifact rejection.
- **Manifest declares a role the user declines:** slot stays empty; `open() → none`;
  the app decides whether it can proceed (it may treat the facility as optional).
- **Manifest declares a link-scoped facility the user declines:** interface unlinked;
  component fails to instantiate; shell reports "app requires X."
- **Manifest requests `ops` the chosen concrete object cannot support** (e.g. `sign`
  against an AEAD key): the *keystore service* rejects the mint (`KeystoreError::WrongPurpose`,
  already real) — the manifest layer does not pre-check object types it cannot see.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none moved. The manifest is package metadata and SDK
  tooling; enforcement stays entirely in `lantern-runtime` + the owning services + the
  kernel (ADR-0004). The manifest cannot grant anything — it can only *request*.
- **New assets introduced and who can reach them:** the manifest's integrity, and the
  binding between a manifest and its `.cwasm`. Reachable by anyone who can write the
  package file; protected by the combined-digest signature (a modified manifest, or a
  manifest paired with a different `.cwasm`, fails verification).
- **New adversary capabilities, if any:** none. A malicious developer can already write
  any Rust; the manifest constrains what that code can be *granted*, it does not expand
  what it can *do*. A tampered manifest that requests more is caught by the signature; a
  tampered manifest under a signature the attacker controls is just a different app the
  user still consents to per-role.
- **Mitigations:** mandatory per-request `justification` and abstract roles (S1 — broad
  or unexplained requests are visible friction); manifest = the exact grantable set, shown
  before consent, enforced by the runtime (S2); combined-digest signing binds manifest to
  code (S4, and system T10 supply-chain); WIT world generated *from* the manifest so
  imports cannot exceed declarations (defence in depth — even a runtime bug that linked an
  undeclared interface would face a component that never imported it).
- **Net change to attacker surface:** **reduces.** Today `GrantManifest` is assembled ad
  hoc; this fixes a reviewed, signed, least-privilege-by-default declaration as the only
  way authority reaches a confined app, and makes over-broad requests legible to the
  person granting them.

Cross-reference [`Threat-Model.md`](../../lantern-docs/wiki/Threat-Model.md) (T10) and
`lantern-sdk/THREAT_MODEL.md` (S1, S2, S4).

## TCB impact  *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No. The manifest parser,
  validator, and WIT generator are `lantern-sdk` (build-time tooling). The launch-time
  binder is `lantern-shell` (user space, outside the ADR-0004 TCB). `lantern-runtime`
  still only consumes a `GrantManifest` of concrete grants — unchanged by this RFC.
- **Does this add a dependency to the TCB?** No. A TOML parser enters `lantern-sdk`'s
  dependency set (not the TCB). `lantern-runtime` gains no dependency; the combined-digest
  signing reuses BLAKE3 + Ed25519, already present (RFC-0007).
- **Effect on TCB size and auditability:** neutral-to-positive. Nothing added to the TCB;
  the manifest makes *auditable* (by a human, before consent) what authority an app
  receives, which the TCB then enforces.

## Privacy impact

The manifest is static metadata, deliberately user-visible (honest manifests — S2). It
carries no telemetry and is not transmitted anywhere by the SDK. `role` names and
`justification` strings are authored by the developer and shown to the user at consent
time; they are not fingerprinting surface (they describe the app's needs, not the user or
device). The abstract-role design is itself privacy-positive: the package never names
*which* of the user's keys or objects an app uses, so a shared or published package leaks
nothing about the installing user's environment.

## Alternatives considered

- **Embed the manifest in a Wasm/component custom section instead of a sidecar file.**
  Rejected for authoring (a developer can't hand-write a custom section) and because
  `Engine::precompile_component` does not reliably carry arbitrary custom sections into
  the `.cwasm`. A sidecar the SDK authors, bound by signature, keeps the source of truth
  editable and the binding just as tight.
- **JSON or YAML.** Rejected: JSON has no comments and reads poorly for prose
  justifications; YAML's ambiguity (the Norway problem, indentation surprises) is the
  opposite of what a security-relevant declaration wants. TOML matches the ecosystem.
- **Let the WIT `world` be the source of truth; derive the manifest from it.** Rejected:
  a `world` cannot express `ops` subsets, per-role justifications, or the intent behind a
  request — it says *what* is imported, not *why* or *how narrowly*. The manifest is
  strictly richer, and generating the world from it (not vice versa) is what guarantees
  imports ≤ declarations.
- **Concrete objects in the manifest** (name the actual key/DID). Rejected: makes the
  package non-portable, unsignable-once (every user edits it), and puts the developer
  rather than the user in control of which objects are touched — the exact inversion
  `lantern-sdk/ARCHITECTURE.md`'s "honest manifests" principle exists to prevent.
- **No justification field** (just the capability list). Rejected: the justification is
  the S1 mitigation. A bare list normalises "just approve it all"; a sentence per request
  is the friction that makes least privilege the path of least resistance, and it is what
  `lantern-shell` needs to render a meaningful consent prompt.
- **Do nothing / hand-build `GrantManifest` per app.** That is the status quo, and it is
  fine for `lantern-runtime`'s own tests but is not a developer-facing story, is not
  signable, and gives `lantern-shell` nothing to render. It also leaves the format to be
  set by whichever implementation PR lands first.

## Prior art

- **Android / iOS permission manifests** (`AndroidManifest.xml`, `Info.plist` usage
  descriptions). Android's `<uses-permission>` got the coarse-grained, install-time,
  all-or-nothing model wrong (users click through); iOS's mandatory
  `NS…UsageDescription` string per capability got the "explain it in the app's own words,
  shown at point of consent" part right — this RFC's `justification` is that idea.
- **Fuchsia component manifests** (`.cml`) — capability *routing* declared per component,
  the runner enforces, nothing is ambient. The closest existing system to this design;
  Fuchsia's `use`/`offer`/`expose` split is heavier than Phase 2 needs but is the right
  long-term shape.
- **WASI Preview 2 / `wasi-virt`, Spin/`spin.toml`, Fastly `fastly.toml`** — Wasm hosts
  that already gate host access by per-component config. `spin.toml`'s `allowed_outbound_hosts`
  is the pattern of "declare it in a TOML next to the code or you don't get it."
- **`Cargo.toml` / `package.json`** — for the format choice and for "the tool generates
  the derived artifacts (lockfile, bindings) from the hand-authored file, never the
  reverse."
- **macaroons / RFC-0011** — the manifest's abstract-role-then-bind-at-launch flow is the
  same shape as issuing a narrow capability at use time rather than baking authority in.

## Unresolved questions

- **Manifest evolution for an installed app.** When `receipt-signer` 0.3.1 → 0.4.0 adds a
  capability, does the user re-consent to just the delta, all of it, or is the old grant
  set carried forward? Needs the binder and an install/upgrade model that don't exist
  yet — deferred, and the reason `[app].version` is advisory-only in Phase 2.
- **The binder and consent UX themselves** — `lantern-shell`'s job. This RFC fixes the
  data it consumes (the validated manifest) and the `GrantManifest` it must emit, nothing
  about how it asks.
- **Grouping / bundles of roles.** Real apps may want "these three keystore roles are one
  logical feature; approve or decline together." Not in Phase 2's schema; a future
  `[[feature]]` grouping is a candidate.
- **Per-language binding ergonomics** (`lantern-sdk/ARCHITECTURE.md`'s own open question)
  — how `role` accessors and attenuation feel in Rust vs. TypeScript. Out of scope.
- **Whether link-scoped `justification` should support a "degraded mode" hint** ("app
  still runs without this, with reduced function") so the shell can offer decline as a
  real choice rather than block launch. Named, not designed.
- **Registry versioning** — how a manifest targets a specific `lantern-runtime` interface
  set as that set evolves. Tied to the "governed public ABI" open question in
  `lantern-runtime/ARCHITECTURE.md`, a later phase.

## Future possibilities

- The `lantern:filesystem` interface (RFC-0014's deferred item) drops into the registry
  as another resource-scoped entry — `[[filesystem.file]]` with `ops = ["read", "write"]`
  — with no format change.
- A `lantern-sdk manifest explain` command that renders the manifest exactly as the shell
  will, so a developer sees their own consent prompt.
- Static analysis: warn when a component's code paths never exercise a declared capability
  ("you asked for `decrypt` but never call it").
- A signed, third-party **capability review** attestation attached to a package (RFC-0011
  sealed tokens as the carrier) — "auditor X confirms this manifest matches the code."
- Registry entries carrying a machine-readable risk tier so the shell can sort/highlight.

## Resulting ADRs

On acceptance, an ADR will record: TOML `lantern.capabilities.toml` as the format; the
`[app]` + per-interface schema with the two declaration shapes; mandatory `justification`
and abstract `role`s (no concrete objects in the manifest); resource-scoped declaration
order as the permanent `open(slot)` wire index (positional with holes — declined roles are
empty slots), which makes `GrantManifest.keystore_keys` sparse; the manifest as the sole
source of the generated WIT `world`; and the combined-digest
(`BLAKE3(manifest) || BLAKE3(cwasm)`, Ed25519-signed) package binding.
