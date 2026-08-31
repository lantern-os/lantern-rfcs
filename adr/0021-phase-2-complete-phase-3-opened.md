---
adr: 0021
title: Phase 2 complete; Phase 3 (privacy, identity, networking, and AI) opened
status: Accepted
date: 2026-08-31
deciders: ["TSC"]
rfc: ../rfcs/0017-phase-2-to-phase-3-transition.md
supersedes: []
superseded_by: null
---

# ADR-0021: Phase 2 complete; Phase 3 (privacy, identity, networking, and AI) opened

## Context

[RFC-0017](../rfcs/0017-phase-2-to-phase-3-transition.md) proposed declaring **Phase 2 —
Capability runtime & first services** complete, per the exit criterion in
[`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md), and opening **Phase 3 — Privacy,
identity, networking, and AI**. `GOVERNANCE.md` requires an RFC for roadmap phase
boundaries specifically so this is a recorded decision, not a status edit. The RFC has
been accepted; this ADR is the durable record, following the RFC-0004/ADR-0007 and
RFC-0009/ADR-0014 pattern.

## Decision

**Phase 2 is complete. Phase 3 is open.**

### Phase 2 deliverables — all satisfied

| Deliverable | Where |
| --- | --- |
| Service framework: badged endpoints, capability brokering (mint/grant/revoke) | [RFC-0010](../rfcs/0010-cross-process-capability-transfer-and-brokering.md); `lantern-capabilities`' `Broker`, proven against a real `KernelState` and under real confined U-mode `ecall`s |
| The WASM runtime with capability-backed WASI (ADR-0003) | [RFC-0013](../rfcs/0013-wasm-engine-selection-and-aot-strategy.md)/[ADR-0017](./0017-wasm-engine-selection-and-aot-strategy.md); [RFC-0014](../rfcs/0014-wit-handle-capability-mapping.md)/[ADR-0018](./0018-wit-handle-capability-mapping.md) + [RFC-0016](../rfcs/0016-filesystem-wit-interface.md)/[ADR-0019](./0019-filesystem-wit-interface.md) — the WIT-handle ⇄ capability mapping (`lantern:host/keystore`, `lantern:host/filesystem`, `monotonic-clock`), deny-by-default, no ambient `wasmtime-wasi` |
| First real services: a content-addressed store (Filesystem v0) and the crypto keystore | `lantern-filesystem`'s `Store`, `lantern-crypto`'s `Keystore` (with sealed capabilities, [RFC-0011](../rfcs/0011-sealed-capability-token-format.md)) — both exercised end to end with real `Broker`-minted badges |
| The SDK v0 so a developer can build and run a confined Wasm app | [RFC-0015](../rfcs/0015-capability-manifest-format.md)/[ADR-0020](./0020-capability-manifest-format.md) — `lantern-sdk`'s manifest parser/validator, interface registry, WIT-`world` generation, `GrantPlan`, combined-digest package signing, and the `lantern-sdk build` CLI (validate → import-check → AOT-compile → sign → `.lpkg`) |

### Phase 2 exit criterion — met

*A third-party Wasm app runs confined, reads a file only via a granted capability, and
cannot touch anything it wasn't granted — demonstrated adversarially.*

`lantern-example-signer` (its own repo): a Wasm component whose entire import surface is
`lantern:host/{monotonic-clock, keystore, filesystem}`, packaged by `lantern-sdk build`,
run by a host that verifies the `.lpkg` signature, binds the manifest's `GrantPlan`, and
`define_unknown_imports_as_traps` for everything else. `attest()` reads a file **only** via
the granted `filesystem` handle; `probe()` shows adversarially that `keystore.open(1)` /
`filesystem.open(9)` (ungranted slots) return `none` and a write through the read-only file
handle returns `error-code::access`. The `--deny-sign` run additionally shows a granted key
handle whose badge lacks the operation being refused by the owning service. The
demonstration is complete and adversarial — not an assertion.

### Carried forward — not waived, and Phase 3's foundational prerequisite

**The Phase 2 services (`Broker`, `Keystore`, `Store`) and `lantern-runtime` run as
in-process stand-ins against a real `KernelState` on a native host target — not as confined
processes on the kernel.** Concretely: those services' methods take `&mut KernelState`
directly (valid only for privileged, same-address-space code); `lantern-runtime` does not
build for `riscv64gc-unknown-none-elf` (Wasmtime's custom-platform embedding hooks against
`lantern-hal`/VSpace-Frame capabilities, named in ADR-0017, are not done); and
`lantern-example-signer`'s host builds real service instances in its own address space, so
the transport between the confined component and those services is an in-process call, not
a kernel IPC round trip.

What Phase 2 proved is the **mechanism** — the mapping, the deny-by-default relay, the
manifest → plan → grant pipeline, the trap-on-undeclared. This does not block the
transition, for the same reason RFC-0009/ADR-0014 did not let the IPC round-trip-loss bug
block Phase 1: the exit criterion is a demonstration, and the demonstration is complete.
Putting a real IPC transport under the services and hosting the runtime on the kernel is
**Phase 3's first work** — its own deliverables ("capability-scoped agents" with "a
faithful audit log", encrypted storage "tied to hardware-backed keys") require it — and
gets its own design RFC as Phase 3's opening move. It is not "Phase 2 left something
undone"; it is shared infrastructure that Phase 3 both needs and is the right place to
build.

Also carried forward: the Phase 1 IPC round-trip-loss bug (RFC-0009/ADR-0014, ADR-0013) —
still reproducible, still worked around; and the demo's `wasi:*`-imports-trapped-not-absent
toolchain wrinkle. Formal verification of the IPC and capability paths has not started and
is a listed Phase 3 deliverable, not a Phase 2 miss.

### What opens

`lantern-network`, `lantern-ai-runtime`, and an identity component may move from
design-only toward prototype code; `lantern-crypto` / `lantern-filesystem` may take on
encrypted-at-rest and provenance work; formal-verification effort on the IPC and
capability paths may begin. Phase 2 components' own recorded "Next" items continue as
ordinary engineering work in parallel. No accepted RFC or ADR is modified by this
decision.

## Consequences

- **Easier:** the differentiating work — identity, networking, local AI — can start
  without a per-PR debate over whether Phase 2 is "really" done; this ADR is the answer.
- **Harder:** Phase 3 introduces the **network** attack surface for the first time and
  hands **capability sets to AI agents** — deliberate, planned expansions that each need
  their own trust-boundary RFC (`lantern-network`, `lantern-ai-runtime`, the identity
  component). The carried-forward confined-execution gap means Phase 3's very first
  effort is a platform port, not a feature.
- **Committed to:** the accepted architecture (RFC-0002/0003/0005/0006 and ADR-0001–0020)
  is unchanged. Normal RFC gates apply within Phase 3: DID method, wallet trust model, the
  anonymity transport, the AI runtime's agent/capability model, and any new cryptographic
  primitive each need their own RFC. Whether Phase 2/3 "v0" code carries a prototype or a
  higher bar remains explicitly undecided (RFC-0009 raised it; still open — worth a short
  standalone ADR before Phase 3 goes far). Phase 3's exit criterion — a capability-bounded
  local AI agent with a faithful audit log, and an unlinkable network identity — gates the
  next transition, which follows this same RFC + ADR pattern.
