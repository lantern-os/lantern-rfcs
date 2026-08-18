---
rfc: 0009
title: Closing Phase 1 and opening Phase 2 (capability runtime & first services)
status: Accepted
authors: ["LanternOS founding stewards"]
stewards: ["capabilities", "runtime", "crypto", "filesystem", "sdk"]
domains: ["governance", "capabilities", "runtime", "crypto", "filesystem", "sdk", "kernel"]
created: 2026-08-18
updated: 2026-08-18
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0009: Closing Phase 1 and opening Phase 2 (capability runtime & first services)

## Summary

This RFC proposes formally declaring **Phase 1 — Microkernel prototype** complete and
opening **Phase 2 — Capability runtime & first services**, per the gates defined in
[`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md). Phase 1's exit criterion — a confined
"hello service" reachable only via a granted capability, with IPC latency benchmarked and
within target — is now fully met. This RFC is the phase-boundary decision the Roadmap
requires before Phase 2 work — `lantern-capabilities`, `lantern-runtime`,
`lantern-crypto`'s keystore, `lantern-filesystem`, and `lantern-sdk` moving from
design-only to prototype code — may begin, following the same pattern
[RFC-0004](./0004-phase-0-to-phase-1-transition.md) established for Phase 0 → Phase 1.

## Motivation

`GOVERNANCE.md` requires an RFC for "Roadmap phase boundaries." The Roadmap's gate is
*quality of foundation*, not the calendar: "we do not advance until they are met." Two
things follow, exactly as they did for RFC-0004:

1. We should not silently flip phase markers as a side effect of editing docs.
2. Once the criteria genuinely are met, we should say so explicitly and unblock Phase 2,
   rather than leaving five repositories idling in "Phase 0 (design only)" after the
   kernel primitive they were each explicitly *blocked on* now exists and is proven.

This serves **security by architecture** directly, the same way RFC-0004 did: Phase 2 is
where genuinely untrusted code runs for the first time (a third-party Wasm app, per this
phase's own exit criterion below) against a capability boundary that has so far only been
exercised between two mutually first-party, cooperating programs. The project's own rule
— architecture, threat modelling, and RFC review before implementation — applies with more
force here than it did for Phase 1's kernel-internal prototype, not less.

## Guide-level explanation

For a contributor, this RFC means:

- Phase 1 is closed: the confined-IPC exit criterion is met, both parts of it, with
  evidence below. No further Phase 1 kernel/boot/HAL work is *required* before Phase 2
  proceeds, though Phase 1's own recorded "Next" items (idle thread, cross-CNode capability
  transfer, capability-derivation tree, `x86-64` boot, DTB memory discovery) remain open
  and continue as ordinary engineering work in parallel — RFC-0004 set this same precedent
  for Phase 0's open items.
- Phase 2 is open: `lantern-capabilities`, `lantern-runtime`, `lantern-crypto`, and
  `lantern-filesystem` may now accept prototype code toward a capability-brokering service
  framework, a capability-backed WASM runtime, and the first two real services (a
  content-addressed filesystem v0 and a crypto keystore); `lantern-sdk` may begin its v0
  so a developer can actually build and run a confined Wasm app against them.
- Nothing about the accepted architecture changes. This RFC does not modify RFC-0002,
  RFC-0003, RFC-0005, RFC-0006, or any accepted ADR — it only unblocks implementation work
  those already authorised.
- Normal RFC gates still apply within Phase 2: a new public interface (the WIT/capability
  mapping, the sealed-cap token wire format, the WASI capability surface), a TCB change, or
  an external dependency (a Wasm engine, most concretely) still needs its own RFC per
  `GOVERNANCE.md`. This RFC does not pre-approve any of those.
- Unlike Phase 1's kernel prototype, Phase 2's first services are named as *v0*s of the
  real thing (`Roadmap.md`: "a content-addressed store (Filesystem v0)"), not
  explicitly throwaway. Contributors should not assume Phase 2 code carries the same
  "prototype, may be discarded" license RFC-0004 granted Phase 1 — see "Unresolved
  questions."

## Reference-level explanation

### Phase 1 exit-criteria evidence

Per `Roadmap.md`, Phase 1's exit criteria are: *"a confined user-space 'hello service'
reachable only via a granted capability, with IPC latency benchmarked and within target."*

| Item | Status | Evidence |
| --- | --- | --- |
| Boot to a minimal kernel on `riscv64` under QEMU | Done | `lantern-boot`/`lantern-hal` STATUS.md |
| Address spaces, threads, a scheduler | Done | `lantern-kernel`'s object model, round-robin scheduler, real Sv39 VSpaces |
| Kernel capability mechanism (CSpace, untyped retyping) | Done | RFC-0003/ADR-0005/ADR-0006; `lantern-kernel/src/cap.rs`, `admin.rs` |
| Narrowing-waterfall root task starting one confined service | Done | `lantern-boot/src/loader.rs` — real ELF loader, RFC-0008/[ADR-0012](../adr/0012-vspace-frame-capabilities-and-elf-loader.md) |
| Confined "hello service" reachable only via a granted capability | Done | Two independently-built, mutually confined programs exchange IPC under real QEMU, each granted *only* the one shared endpoint capability it needs |
| IPC latency benchmarked and within target | Done | 2000-round-trip benchmark, [ADR-0013](../adr/0013-ipc-latency-benchmark.md) — ~26–28k cycles/round-trip avg against a ~40,000 target, this environment |

All items are satisfied. No item is waived, and — unlike some Phase 0 checklist items —
none is deferred to "ordinary engineering work" to call this criterion met: both halves
Phase 1 actually asked for are done.

### A carried-forward risk, not waived, not blocking

ADR-0013 also recorded a real, reproducible, **unresolved** bug found while building the
IPC benchmark: the first `Call` a thread issues immediately after being resumed via
`ipc::reply`'s direct `state.switch_to` occasionally never completes its switch to the
receiver — silently dropping that one message, with no panic and no anomalous trap. It
was investigated and several plausible causes were ruled out (the kernel's own dispatch
logic, a spurious interrupt, TLB staleness, a stack-overflow artefact of the benchmark's
own instrumentation) without finding the actual root cause. It is worked around with a
tolerated safety margin in the benchmark itself, not fixed.

This RFC does **not** treat that bug as blocking the phase transition — Phase 1's exit
criterion asked for a benchmarked, in-target latency number, which now exists and is
reproducible, not a formally verified fault-free IPC path (Phase 1's own scope explicitly
excludes that; see [ADR-0009](../adr/0009-phase1-scheduling-context-model.md)). But it
*is* recorded here, explicitly, as a carried-forward risk Phase 2 inherits and should not
forget: Phase 2 is where IPC volume goes up sharply (every capability-brokering
mint/grant/revoke, every WASI call a confined Wasm app makes, crosses this same fast
path), which is exactly the kind of load that could turn a once-in-2000-round-trips
anomaly into a real reliability problem. `lantern-kernel/STATUS.md` and
`lantern-boot/STATUS.md` continue to track it; this RFC adds no new tracking mechanism,
it just makes explicit that opening Phase 2 does not implicitly wave it away.

### Phase 2 scope (unchanged from Roadmap, restated for clarity)

- Service framework: badged endpoints, capability brokering (mint/grant/revoke) —
  `lantern-capabilities`, which is currently blocked *only* on "kernel capability
  mechanism," now available.
- The WASM runtime with capability-backed WASI (ADR-0003) — `lantern-runtime`, currently
  blocked on the capability-brokering API and kernel IPC/endpoints, both now available.
- First real services: a content-addressed store (`lantern-filesystem`, Filesystem v0) and
  the crypto keystore (`lantern-crypto`) — each currently blocked on capability brokering,
  and `lantern-filesystem` additionally on the keystore/AEAD `lantern-crypto` provides.
- The SDK v0 (`lantern-sdk`) so a developer can build and run a confined Wasm app —
  currently blocked on stable WIT interfaces from `lantern-runtime`.

**Phase 2 exit criteria** (unchanged, restated): a third-party Wasm app runs confined,
reads a file *only* via a granted capability, and cannot touch anything it wasn't
granted — demonstrated adversarially.

### What this RFC does *not* do

- It does not resolve any "Next"/"Blocked on" item already recorded in any Phase 2
  component's own `STATUS.md` (the rights lattice per object type, the sealed-cap token
  format, Wasm engine/AOT selection, the WIT-handle ⇄ capability mapping, the capability
  manifest format, the block-store/GC strategy). Those are ordinary Phase 2 engineering
  work, tracked where they already live, exactly as RFC-0004 left Phase 1's open questions
  in place rather than resolving them here.
- It does not pre-approve any specific implementation choice. A Wasm engine selection in
  particular is likely to cross `GOVERNANCE.md`'s "new languages, toolchains, or external
  dependencies in the TCB" bar (`lantern-runtime` sits *outside* the TCB per RFC-0002's
  boundary — see "TCB impact" below — but an engine choice is still consequential enough,
  and precedented enough by ADR-0001–0003, that it should get its own ADR at minimum).
- It does not root-cause or fix the IPC round-trip-loss bug above.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none *newly defined* — RFC-0002/ADR-0004 already fixed
  the TCB boundary, and RFC-0003/ADR-0005/ADR-0006 already fixed the capability model,
  that Phase 2 work operates inside of. What changes is *exercise*, not *definition*:
  Phase 2's own exit criterion is the first time this project runs genuinely untrusted
  third-party code (a Wasm app) against that boundary, rather than two mutually
  cooperating, first-party test programs (Phase 1's actual demo). This is the boundary
  being used as designed for the first time, not a new one.
- **New assets introduced and who can reach them:** a capability-brokering service holding
  mint/grant/revoke authority over service-layer capabilities; a crypto keystore holding
  key material; a content-addressed filesystem holding user data — each reachable *only*
  via capabilities granted through the broker, by design (RFC-0003).
- **New adversary capabilities, if any:** for the first time, a confined but genuinely
  untrusted Wasm app can execute. Its blast radius is bounded, by construction, to
  whatever capabilities it was explicitly granted — this is exactly what Phase 2's exit
  criterion requires demonstrating *adversarially*, not merely asserting.
- **Mitigations:** unchanged from the already-accepted capability model — deny by default,
  explicit least-privilege grants, capability-backed WASI rather than ambient WASI
  authority (ADR-0003). Phase 2 code stays subject to each component's already-reviewed
  `THREAT_MODEL.md`; any change to those still requires its own RFC.
- **Net change to attacker surface:** increases, deliberately and by design — this is the
  expected, planned-for expansion RFC-0002/RFC-0003 already justified, exercised for the
  first time rather than an unreviewed one. The carried-forward IPC round-trip-loss risk
  (above) is the one genuinely new, *unplanned* wrinkle this RFC surfaces rather than
  waives.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No. By RFC-0002/ADR-0004's
  already-fixed TCB boundary, every Phase 2 component (`lantern-capabilities`,
  `lantern-runtime`, `lantern-crypto`, `lantern-filesystem`, `lantern-sdk`) is a confined,
  unprivileged user-space service. This RFC authorises prototype code in components that
  sit *outside* the TCB by construction; it does not touch or expand the boundary itself.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** none directly. (A Wasm engine choice will add a
  large dependency to `lantern-runtime` — but `lantern-runtime` is not in the TCB, so that
  dependency's own review, whenever it happens, is about confined-component trust and
  supply-chain hygiene, not TCB growth.)

## Privacy impact

None directly — this is a governance/process decision affecting no user data, telemetry,
or identity surface by itself. It does open work (the crypto keystore, the filesystem)
that will have real privacy impact once implemented; that impact belongs to those
components' own RFCs and threat models as they're written, not this one.

## Alternatives considered

- **Remain in Phase 1 indefinitely.** Rejected: exit criteria are fully met; the
  Roadmap's own philosophy is to gate on quality, not to gate arbitrarily once quality is
  achieved — the same reasoning RFC-0004 used for Phase 0.
- **Wait for the IPC round-trip-loss bug to be root-caused first.** Rejected: Phase 1's
  exit criterion is a benchmarked, in-target latency number, which exists; holding the
  entire project's roadmap hostage to a fully root-caused explanation of a rare, worked-
  around anomaly is a stronger bar than Phase 1 itself ever set (see [ADR-0009](../adr/0009-phase1-scheduling-context-model.md)
  on why formal fault-freedom is explicitly out of Phase 1's scope). The right response is
  to carry the risk forward loudly, which this RFC does, not to stall on it silently.
- **Skip the phase-boundary RFC and just edit `Roadmap.md`'s checkboxes.** Rejected, same
  reasoning as RFC-0004: `GOVERNANCE.md` requires an RFC for roadmap phase boundaries.
- **Fold this into RFC-0003's or RFC-0005's own status.** Rejected: those RFCs are about
  *what* the capability model and syscall ABI are, not *when* Phase 2 implementation
  starts; mixing the two would make each harder to review and reuse independently — same
  reasoning RFC-0004 gave for not folding into RFC-0002/RFC-0003.

## Prior art

N/A — this is a project-internal process decision, not a technical design, exactly as
RFC-0004 was.

## Unresolved questions

- **Is Phase 2 code prototype/throwaway, like Phase 1's, or held to a higher bar because
  it's named "v0" of a real service?** `Roadmap.md` doesn't say explicitly. This RFC takes
  no position beyond flagging the question (see "Guide-level explanation") — worth a
  steward decision, possibly as a short standalone ADR, before Phase 2 work goes very far,
  so contributors aren't guessing.
- The IPC round-trip-loss bug's actual root cause (`lantern-boot/STATUS.md`'s "Next" has
  candidate next steps) — not blocking, but genuinely open.
- Each Phase 2 component's own open design questions (rights lattice, sealed-cap token
  format, Wasm engine selection, WIT-handle mapping, capability manifest format, block-
  store/GC strategy) — tracked in their own `STATUS.md`s, not here.

## Future possibilities

A parallel Phase 2 → Phase 3 transition RFC, following the same pattern, once Phase 2's
exit criteria (a third-party Wasm app demonstrated confined, adversarially) are met.

## Resulting ADRs

[ADR-0014](../adr/0014-phase-1-complete-phase-2-opened.md) records the phase transition
and the exit-criteria evidence as a durable decision, independent of this RFC file,
following the RFC-0004/ADR-0007 pattern.
