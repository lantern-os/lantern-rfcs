---
rfc: 0017
title: Closing Phase 2 and opening Phase 3 (privacy, identity, networking, and AI)
status: Accepted
authors: ["LanternOS founding stewards"]
stewards: ["governance", "runtime", "sdk", "capabilities", "network", "ai-runtime", "identity"]
domains: ["governance", "runtime", "sdk", "capabilities", "crypto", "filesystem", "kernel", "network", "ai-runtime", "identity"]
created: 2026-08-31
updated: 2026-08-31
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0017: Closing Phase 2 and opening Phase 3 (privacy, identity, networking, and AI)

> **Accepted 2026-08-31.** Phase 2 is closed and Phase 3 is open;
> [ADR-0021](../adr/0021-phase-2-complete-phase-3-opened.md) is the durable record of the
> transition, the exit-criteria evidence, and the carried-forward confined-execution gap
> that is Phase 3's foundational prerequisite.

## Summary

This RFC proposes formally declaring **Phase 2 — Capability runtime & first services**
complete and opening **Phase 3 — Privacy, identity, networking, and AI**, per the gates in
[`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md). Phase 2's exit criterion — *a
third-party Wasm app runs confined, reads a file only via a granted capability, and cannot
touch anything it wasn't granted, demonstrated adversarially* — is now met: the
`lantern-example-signer` app is compiled and signed to a `.lpkg` by `lantern-sdk build`,
run confined by a host that verifies the package, binds the manifest, and traps everything
ungranted, and its `probe` export demonstrates adversarially that ungranted capability
slots return `none` and a write through a read-only handle is refused. This RFC is the
phase-boundary decision the Roadmap requires, following the pattern
[RFC-0004](./0004-phase-0-to-phase-1-transition.md) and
[RFC-0009](./0009-phase-1-to-phase-2-transition.md) established. It carries one large piece
of unfinished work forward, loudly and by name — **the services and the runtime run as
in-process stand-ins on a host target, not yet as confined processes on the kernel** — and
frames it as Phase 3's foundational prerequisite rather than a reason to stall the
roadmap.

## Motivation

`GOVERNANCE.md` requires an RFC for "Roadmap phase boundaries," and the Roadmap's gate is
*quality of foundation*, not the calendar. Two things follow, exactly as they did for
RFC-0004 and RFC-0009:

1. We do not silently flip phase markers as a side effect of editing docs.
2. Once the criteria genuinely are met, we say so explicitly and open the next phase,
   rather than leaving the differentiating work — identity, networking, local AI — idling
   because the demonstration that was supposed to unblock it has quietly already happened.

This serves the [Core Principles](../../lantern-docs/wiki/Principles.md) directly. Phase 2
proved the capability boundary holds for genuinely untrusted third-party code (Principle 1,
security by architecture). Phase 3 is where the **differentiating** principles get built
for the first time: Principle 2 (privacy by default — partitionable identities, no global
identifiers) becomes the DID/wallet work and the anonymity-capable transport; Principle 3
(local-first ownership) becomes encrypted-by-default storage tied to hardware-backed keys;
Principle 6 (AI-native) becomes the local inference runtime and capability-scoped agents.
Each is a new trust-boundary-defining effort that needs its own RFC — this one only opens
the phase in which those RFCs are written.

## Guide-level explanation

For a contributor, this RFC means:

- **Phase 2 is closed.** Its exit criterion is met, both halves ("reads a file only via a
  granted capability" *and* "cannot touch anything it wasn't granted, demonstrated
  adversarially"), with evidence below. Phase 2 components' own recorded "Next" items
  (per-object rights lattices, chunking, per-object AEAD keys, a `lantern-sdk` build CLI
  that also drives `cargo`, WIT `world` composition, resource accounting) remain open and
  continue as ordinary engineering work in parallel — RFC-0004 and RFC-0009 set this same
  precedent.
- **Phase 3 is open.** `lantern-network`, `lantern-ai-runtime`, and an identity component
  may move from design-only toward prototype code; `lantern-crypto` / `lantern-filesystem`
  may take on encrypted-at-rest and provenance work; formal-verification effort on the IPC
  and capability paths may begin.
- **Nothing about the accepted architecture changes.** This RFC modifies no earlier RFC or
  ADR. Normal RFC gates still apply within Phase 3: DID method selection, the OS wallet's
  trust model, the anonymity transport, the AI runtime's agent/capability model, and any
  new cryptographic primitive each need their own RFC per `GOVERNANCE.md`. This RFC
  pre-approves none of them.
- **One large item is carried forward, not waived.** The Phase 2 services and runtime are
  proven *as mechanisms* but run in-process on a host target, not confined on the kernel.
  Closing this Phase depends on the exit-criterion *demonstration*, which is complete;
  making it *real* is the first thing Phase 3 builds. See "A carried-forward gap" below.

## Reference-level explanation

### Phase 2 deliverables and exit-criteria evidence

Per `Roadmap.md`, Phase 2's deliverables and exit criterion:

| Deliverable | Status | Evidence |
| --- | --- | --- |
| Service framework: badged endpoints, capability brokering (mint/grant/revoke) | Done | [RFC-0010](./0010-cross-process-capability-transfer-and-brokering.md) — kernel-side `extra_caps==1` transfer, `CopyCross`, reply-leg transfer, all QEMU-validated; `lantern-capabilities`' `Broker` (mint/grant/`grant_via_reply`/revoke) proven against a real `KernelState` *and* under real confined U-mode `ecall`s (`lantern-boot`'s broker demo) |
| The WASM runtime with capability-backed WASI (ADR-0003) | Done | [RFC-0013](./0013-wasm-engine-selection-and-aot-strategy.md)/[ADR-0017](../adr/0017-wasm-engine-selection-and-aot-strategy.md) (Wasmtime, runtime/compiler split, signature-checked `.cwasm`); [RFC-0014](./0014-wit-handle-capability-mapping.md)/[ADR-0018](../adr/0018-wit-handle-capability-mapping.md) + [RFC-0016](./0016-filesystem-wit-interface.md)/[ADR-0019](../adr/0019-filesystem-wit-interface.md) — the WIT-handle ⇄ capability mapping, resource-scoped (`lantern:host/keystore`, `lantern:host/filesystem`) and link-scoped (`monotonic-clock`), deny-by-default relayed from each owning service, no `wasmtime-wasi` ambient authority |
| First real services: a content-addressed store (Filesystem v0) and the crypto keystore | Done | `lantern-crypto`'s `Keystore` (real XChaCha20-Poly1305 / Ed25519 / BLAKE3-MAC, sealed capabilities per [RFC-0011](./0011-sealed-capability-token-format.md)); `lantern-filesystem`'s `Store` (content-addressed, AEAD-encrypted, refcounted, `FileId`-capability-gated) — both exercised end to end against a real `KernelState` with real `Broker`-minted badges |
| The SDK v0 so a developer can build and run a confined Wasm app | Done | [RFC-0015](./0015-capability-manifest-format.md)/[ADR-0020](../adr/0020-capability-manifest-format.md) — `lantern-sdk`'s manifest parser + validator, interface registry, WIT-`world` generation, `GrantPlan`, `bind`, combined-digest package signing, and the `lantern-sdk build` CLI (validate → import-check → AOT-compile → sign → `.lpkg`) |
| **Exit criterion:** a third-party Wasm app runs confined, reads a file *only* via a granted capability, and cannot touch anything it wasn't granted — demonstrated adversarially | **Met** | `lantern-example-signer` (its own repo): a Wasm component whose entire import surface is `lantern:host/{monotonic-clock, keystore, filesystem}`, packaged by `lantern-sdk build`, run by a host that verifies the `.lpkg` signature, binds the manifest's `GrantPlan`, and `define_unknown_imports_as_traps` for everything else. `attest()` reads a file **only** via the granted `filesystem` handle; `probe()` shows adversarially that `keystore.open(1)` / `filesystem.open(9)` (ungranted slots) return `none` and a write through the read-only file handle returns `error-code::access`. The `--deny-sign` run additionally shows a granted key handle whose badge lacks the operation being refused by the owning service |

All deliverables are satisfied and the exit criterion is met, both halves, with an
adversarial demonstration — not merely an assertion.

### A carried-forward gap: confined execution on the kernel

Every Phase 2 service (`Broker`, `Keystore`, `Store`) and `lantern-runtime` itself run
today as **in-process stand-ins against a real `KernelState`, on a native `std` host
target** — not as confined processes on `riscv64`. Concretely:

- `Broker` / `Keystore::*` / `Store::*` take `&mut lantern_kernel::state::KernelState`
  directly — valid only for privileged, same-address-space code. `lantern-boot`'s broker
  demo hand-reimplements the mint/grant sequence as raw `ecall`s to prove it *can* run
  confined, but the Rust services themselves do not.
- `lantern-runtime` does not build for `riscv64gc-unknown-none-elf`: Wasmtime assumes
  OS-level mmap / threads / signal-based traps that LanternOS does not provide. Its
  custom-platform embedding hooks against `lantern-hal` / VSpace-Frame capabilities are
  named as required work in ADR-0017 but not done.
- `lantern-example-signer`'s runner therefore builds real `Keystore` / `Store` instances
  in its own address space; the badge checks, the crypto, the storage, and the
  error relays are all real, but the **transport** between the confined component and
  those services is an in-process call, not a kernel IPC round trip.

This RFC does **not** treat that as blocking the phase transition, for the same reason
RFC-0009 did not treat the IPC round-trip-loss bug as blocking Phase 1's: Phase 2's exit
criterion is a *demonstration* that the capability boundary holds adversarially, and that
demonstration is complete. What Phase 2 proved is the **mechanism** — the mapping, the
deny-by-default relay, the manifest → plan → grant pipeline, the trap-on-undeclared. What
remains is putting a real IPC transport under it and getting the runtime hosted on the
kernel.

It **is** carried forward, explicitly, as the first thing Phase 3 builds — because Phase
3's own deliverables require it. "Capability-scoped agents" with "a faithful audit log"
(`Roadmap.md`, Phase 3) cannot be faithful if the runtime and the services it calls share
an address space; encrypted-at-rest storage "tied to hardware-backed keys" needs the
keystore reachable as a confined service. So this is not "Phase 2 left something
undone" — it is shared infrastructure that Phase 3 both needs and is the natural place to
build, and it should get its own design RFC as Phase 3's opening move.

### Other risks Phase 3 inherits

- **The IPC round-trip-loss bug** (Phase 1, via RFC-0009 / ADR-0013) — still reproducible,
  still worked around, not root-caused. Phase 3 raises IPC volume again: every agent
  capability call, every network-service request crosses the same fast path.
- **`wasi:*` imports trapped, not absent.** The demo component is a `std` `wasm32-wasip2`
  build that imports a handful of unused `wasi:cli/*` / `wasi:io/*` interfaces the runner
  stubs as traps; a `#![no_std]` build importing *only* `lantern:host/*` hit a toolchain
  snag. Cosmetic for the demonstration (the app never calls them), worth closing.
- **Formal verification has not started.** `Roadmap.md` lists "begin formal verification
  of the IPC and capability paths" under Phase 3, not Phase 2 — so this is planned Phase 3
  work, not a Phase 2 miss, but it is the mitigation the carried-forward IPC bug most
  wants.

### Phase 3 scope (unchanged from Roadmap, restated)

- **Identity:** DIDs, the OS wallet service, verifiable credentials.
- **Networking:** authenticated encrypted channels; P2P discovery; a first
  anonymity-capable transport.
- **AI runtime:** local inference, capability-scoped agents, the audit log.
- **Storage:** encrypted-by-default, tied to hardware-backed keys; provenance tracking.
- **Assurance:** begin formal verification of the IPC and capability paths.

**Phase 3 exit criteria** (unchanged, restated): a user can run a capable local AI agent
with a visible, revocable capability set and a faithful audit log; a network identity can
be presented without cross-context linkage; data at rest is encrypted and
provenance-tracked.

### What this RFC does *not* do

- It does not resolve any "Next" / "Blocked on" item already recorded in any component's
  own `STATUS.md`.
- It does not pre-approve any Phase 3 design. DID method, wallet trust model, anonymity
  transport, the AI runtime's agent/capability model, encrypted-storage key hierarchy —
  each needs its own RFC.
- It does not perform the confined-execution port or root-cause the IPC bug.
- It takes no position on the still-open "is Phase 2/3 code prototype or held to a higher
  bar" question RFC-0009 raised (see "Unresolved questions").

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none *newly defined*. RFC-0002/ADR-0004 (the TCB
  boundary) and RFC-0003/ADR-0005/ADR-0006 (the capability model) still bound everything.
  What changes is again *exercise*, not *definition*: Phase 3 introduces the **network**
  attack surface for the first time (`lantern-network`), and hands **capability sets to AI
  agents** — both are the planned expansions RFC-0002's and the architectural-context
  vision anticipated, to be reviewed in their own component RFCs, not new boundaries this
  RFC creates.
- **New assets introduced and who can reach them:** (opened, not built here) network
  identities and the OS wallet's key material; an AI agent's model weights and its audit
  log; encrypted-at-rest storage keyed to hardware. Each reachable only via granted
  capabilities, by the already-accepted model.
- **New adversary capabilities, if any:** none *from this RFC*. Phase 3's work will add a
  remote adversary (the network) and a semi-trusted local one (an agent acting on the
  user's behalf but capability-bounded) — the defining threat-model work for those belongs
  to `lantern-network` / `lantern-ai-runtime` RFCs.
- **Mitigations:** unchanged capability model — deny by default, least-privilege grants,
  capability-backed interfaces. The carried-forward confined-execution gap is the one
  genuinely-unfinished mitigation this RFC surfaces: until the runtime and services are
  confined on the kernel, the demo's isolation is *demonstrated* but not *deployed*.
- **Net change to attacker surface:** increases in Phase 3, deliberately and by design
  (network, agents) — the planned expansion. This RFC itself changes nothing; it opens the
  phase where that expansion is designed and reviewed.

## TCB impact  *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No. Every Phase 2 component sits
  outside the ADR-0004 TCB boundary by construction, and so will `lantern-network` /
  `lantern-ai-runtime` / the identity component. This RFC authorises prototype work in
  non-TCB components; it does not touch the boundary.
- **Does this add a dependency to the TCB?** No. (Phase 4, not Phase 3, is where a secure
  enclave enters the picture — a real TCB-adjacent dependency with its own RFC then.)
- **Effect on TCB size and auditability:** none directly. The confined-execution port,
  when it happens, adds Wasmtime custom-platform code to `lantern-runtime` — still outside
  the TCB — and its review is about confined-component trust and supply-chain hygiene, not
  TCB growth.

## Privacy impact

None directly — this is a governance decision affecting no user data, telemetry, or
identity surface by itself. It opens the phase in which the project's core privacy
work — partitionable identities, no global identifiers, an anonymity-capable transport,
encrypted-by-default storage — is designed; that impact belongs to those components' own
RFCs and threat models, not this one.

## Alternatives considered

- **Remain in Phase 2 until the runtime and services are confined on the kernel.**
  Rejected: that work is Phase 3 infrastructure (Phase 3's own deliverables require it and
  it is the natural place to build it), the exit criterion is a demonstration and the
  demonstration is complete and adversarial, and holding the roadmap hostage to a
  multi-month platform port is a stronger bar than Phase 2 ever set — the same reasoning
  RFC-0004 used for Phase 0's open items and RFC-0009 for the IPC bug.
- **Wait for the IPC round-trip-loss bug to be root-caused first.** Rejected, identically
  to RFC-0009: it is worked around, reproducible, and Phase 3 lists "begin formal
  verification of the IPC and capability paths" as its own deliverable — the right
  response is to carry the risk forward loudly, which this RFC does.
- **Skip the phase-boundary RFC and edit `Roadmap.md`'s markers.** Rejected: `GOVERNANCE.md`
  requires an RFC for roadmap phase boundaries.
- **Fold the confined-execution port into this RFC as a blocking Phase 2 task.** Rejected:
  it is large enough and consequential enough (a new dependency surface in
  `lantern-runtime`, a `lantern-hal` contract for Wasmtime's platform hooks) to deserve
  its own RFC, and it is Phase 3 work by scope. This RFC names it; it does not design it.

## Prior art

N/A — a project-internal process decision, following RFC-0004 and RFC-0009.

## Unresolved questions

- **The confined-execution port's own design** — how `Broker` / `Keystore` / `Store`
  become deployable confined services reachable over kernel IPC, and how `lantern-runtime`
  implements Wasmtime's custom-platform hooks against `lantern-hal` / VSpace-Frame
  capabilities. This is the intended first Phase 3 RFC.
- **Is Phase 2/3 code prototype/throwaway or held to a "v0 of the real thing" bar?**
  RFC-0009 raised this; still unanswered. Worth a short standalone ADR before Phase 3 work
  goes far.
- **Phase 3 sequencing** — whether the AI runtime, networking, or identity work goes
  first, given they partly depend on each other (an agent wants an identity; an identity
  wants a transport).
- The IPC round-trip-loss bug's actual root cause.

## Future possibilities

- A Phase 3 → Phase 4 transition RFC, same pattern, once a user can run a capability-bounded
  local AI agent with a faithful audit log and present an unlinkable network identity.
- The standalone ADR on the prototype-vs-v0 bar.

## Resulting ADRs

On acceptance, an ADR will record the phase transition and the exit-criteria evidence as a
durable decision independent of this RFC file, following the RFC-0004/ADR-0007 and
RFC-0009/ADR-0014 pattern, and will name the carried-forward confined-execution gap as
Phase 3's foundational prerequisite.
