---
adr: 0020
title: The lantern-sdk capability manifest format
status: Accepted
date: 2026-08-31
deciders: ["TSC"]
rfc: ../rfcs/0015-capability-manifest-format.md
supersedes: []
superseded_by: null
---

# ADR-0020: The lantern-sdk capability manifest format

## Context

[ADR-0018](./0018-wit-handle-capability-mapping.md) fixed the runtime-side contract for
how a Wasm component's WIT-typed imports become LanternOS capabilities — one badge per
resource-scoped grant, one yes/no per link-scoped facility, populated only at
instantiation — and left the developer-facing declaration format to `lantern-sdk`. Until
now `lantern-runtime`'s `GrantManifest` has only ever been built by hand in tests and in
the `lantern-example-signer` runner; there is no way for a developer to *declare* what a
component needs, and nothing to sign or to show a user before consent.

The manifest is the contract three components must agree on (`lantern-sdk` authors it,
`lantern-runtime` consumes the `GrantManifest` it compiles to, `lantern-shell` renders it
for consent), and it is where `lantern-sdk`'s S1 (over-privileged apps) and S2 (dishonest
manifest) threats are answered. Fixing it in the open — rather than letting the first
implementation PR set it — is the trust-boundary-adjacent decision `GOVERNANCE.md`
reserves for an RFC. [RFC-0015](../rfcs/0015-capability-manifest-format.md) proposed it and
has been accepted; this ADR is the durable record.

## Decision

**A LanternOS app ships a `lantern.capabilities.toml` — TOML, UTF-8, at the package root —
declaring, per imported host interface, exactly what authority it needs.**

- **`[app]`** — `name` (`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`) and `version` (semver, advisory
  in Phase 2). Carries no capabilities.
- **Resource-scoped declarations** — one TOML array-of-tables per resource, named
  `<interface-local-name>.<resource-name>` (`[[keystore.key]]`, `[[filesystem.file]]`).
  Each entry: `role` (app-local name, unique in its array, same charset as `name`), `ops`
  (non-empty; every value legal for that resource per the registry), `justification`
  (non-empty trimmed, ≤ 280 chars, written for the user).
- **Link-scoped declarations** — one TOML table per interface (`[monotonic-clock]`),
  presence = request, `justification` the only key.
- **The manifest declares abstract `role`s, never concrete objects.** The mapping from a
  role to a `KeyId` / `FileId` is made at launch by the user, recorded by the shell, and
  never travels in the package — keeping it portable, signable once, and the user (not the
  developer) in control of which objects an app touches.
- **`lantern-sdk` ships an interface registry**, one entry per interface `lantern-runtime`
  implements, kept in lockstep with `lantern-runtime/wit/`. A manifest naming an interface
  or `ops` value the registry does not know fails validation. Phase 2 seeds it with
  `keystore`, `filesystem`, and `monotonic-clock`.
- **Declaration order is the permanent `open(slot)` wire index, positional with holes.**
  The Nth `[[…]]` entry is always `open(N)` — approved, declined, or not-yet-asked. A
  declined role is an empty slot; `open(N)` returns `none`, indistinguishable at the WIT
  level from an un-opened handle. This makes `lantern-runtime`'s
  `GrantManifest.keystore_keys` / `.filesystem_files` **sparse** (`Vec<Option<…>>`) rather
  than dense — a mechanical change applied to the prototype in the same round.
- **The manifest is the sole source of the generated WIT `world`.** `lantern-sdk`
  generates the world the component compiles against from the manifest and nothing else,
  so a component's imports can never exceed its declarations (defence in depth — even a
  runtime bug that linked an undeclared interface would face a component that never
  imported it).
- **The package is `{ manifest_bytes, cwasm_bytes }`**, and what is Ed25519-signed
  (RFC-0007's primitive) is `BLAKE3(manifest_bytes) ‖ BLAKE3(cwasm_bytes)` — a 64-byte
  value. Two hashes rather than one blob so the runtime can verify the `.cwasm` without
  the manifest and the shell can verify the manifest without the `.cwasm`, while still
  making a manifest un-pairable with a `.cwasm` it was not signed with.

**Out of scope (unchanged from the RFC):** the launch-time binder and consent UX
(`lantern-shell`'s — this ADR fixes only the validated manifest it consumes and the
`GrantManifest` it emits); per-language binding ergonomics; and an installed app's
manifest-evolution / upgrade story (the reason `[app].version` is advisory only).

Rejected alternatives (full reasoning in the RFC): embedding the manifest in a Wasm custom
section (unauthorable, not carried by `precompile_component`); JSON/YAML (no comments / the
Norway problem); deriving the manifest from the WIT `world` (a world can't express `ops`
subsets, justifications, or intent); concrete objects in the manifest (non-portable,
inverts who controls which objects are touched); no `justification` field (removes the S1
friction).

## Consequences

- **Easier:** a developer can now declare capabilities in a file, and that file — not the
  runner's hard-coded Rust — is the authority. `lantern-example-signer` is reworked to be
  driven by its `lantern.capabilities.toml`. The Phase 2 "SDK v0" deliverable exists.
- **Harder / committed to:** the interface registry is now a maintained artifact that must
  move in lockstep with `lantern-runtime/wit/`; adding an interface is a coordinated
  change. Declaration order in a shipped manifest is a breaking change to the component
  (its compiled `role` accessors call fixed indices) — an installed-app concern deferred
  with manifest evolution.
- **Trust boundary:** none moved (ADR-0004 TCB unchanged). `lantern-sdk` gains a TOML
  parser and sibling-crate dependencies (not TCB); `lantern-runtime` gains no dependency —
  the `Vec<Option<…>>` change is a shape tweak, and the combined-digest signing reuses
  BLAKE3 + Ed25519 already present.
- **Net attacker surface: reduces.** A reviewed, signed, least-privilege-by-default
  declaration becomes the only way authority reaches a confined app, and over-broad
  requests become legible to whoever grants them.
- **Still open (tracked in the RFC):** manifest evolution / re-consent on upgrade; the
  binder and consent UX; role bundles (`[[feature]]`); per-language ergonomics; a
  degraded-mode hint for declinable link-scoped facilities; registry versioning against a
  governed public ABI.
