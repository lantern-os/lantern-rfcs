---
adr: 0014
title: Phase 1 complete; Phase 2 (capability runtime & first services) opened
status: Accepted
date: 2026-08-18
deciders: ["TSC"]
rfc: ../rfcs/0009-phase-1-to-phase-2-transition.md
supersedes: []
superseded_by: null
---

# ADR-0014: Phase 1 complete; Phase 2 (capability runtime & first services) opened

## Context

[RFC-0009](../rfcs/0009-phase-1-to-phase-2-transition.md) proposed declaring Phase 1 —
Microkernel prototype complete, per the exit criteria in
[`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md), and opening Phase 2 — Capability
runtime & first services. `GOVERNANCE.md` requires an RFC for roadmap phase boundaries
specifically so this transition is a recorded decision, not a status edit. The RFC has
been accepted; this ADR is the durable record.

## Decision

**Phase 1 is complete. Phase 2 is open.**

Both halves of Phase 1's exit criterion are satisfied:

- A confined "hello service" reachable only via a granted capability — two
  independently-built programs exchange IPC under real QEMU, each granted *only* the one
  endpoint capability it needs ([ADR-0012](./0012-vspace-frame-capabilities-and-elf-loader.md)).
- IPC latency benchmarked and within target — a 2000-round-trip benchmark, ~26–28k
  cycles/round-trip average against a ~40,000 target in this environment
  ([ADR-0013](./0013-ipc-latency-benchmark.md)).

A real, reproducible, unresolved IPC round-trip-loss bug was found and documented
alongside the benchmark (ADR-0013). It does not block this transition — Phase 1's
criterion asked for a benchmarked, in-target number, not formal fault-freedom (explicitly
out of Phase 1's scope per [ADR-0009](./0009-phase1-scheduling-context-model.md)) — but it
is carried forward as a known risk Phase 2 inherits, tracked in `lantern-kernel/STATUS.md`
and `lantern-boot/STATUS.md`, not waived by this decision.

`lantern-capabilities`, `lantern-runtime`, `lantern-crypto`, and `lantern-filesystem` may
now accept prototype code toward Phase 2's goal: a capability-brokering service framework
(badged endpoints, mint/grant/revoke), a capability-backed WASM runtime, and the first two
real services (a content-addressed filesystem v0 and a crypto keystore).
`lantern-sdk` may begin its v0 so a developer can build and run a confined Wasm app
against them. No other component repository's phase status changes by this decision.

## Consequences

- **Easier:** the five Phase 2 components can now start implementation work without a
  per-PR debate over whether Phase 1 is "really" done — this ADR is the answer. Each was
  already blocked on exactly one thing (the kernel capability mechanism and/or IPC
  endpoints, per their own `STATUS.md`s), and that thing now exists and is proven.
- **Harder:** Phase 2 introduces the project's first genuinely untrusted running code (a
  third-party Wasm app, per Phase 2's own exit criterion) — a real increase in exercised
  attacker surface, deliberate and already justified by RFC-0002/RFC-0003, but the first
  time it's actually exercised rather than designed for. The carried-forward IPC
  round-trip-loss risk also means Phase 2 work that pushes IPC volume up (capability
  brokering, WASI calls) should stay alert to it, not assume Phase 1 already closed it out.
- **Committed to:** the accepted architecture (RFC-0002, RFC-0003, RFC-0005, RFC-0006, and
  ADR-0001–0013) is unchanged, so no existing design work needs to be redone. Whether
  Phase 2's "v0" services carry Phase 1's prototype/throwaway license or a higher bar is
  explicitly *not* decided here — RFC-0009 raised it as an open question, unresolved.
  Phase 2's exit criterion — a third-party Wasm app demonstrated confined, adversarially,
  reading a file only via a granted capability — gates the next transition (Phase 2 →
  Phase 3), which will follow this same RFC + ADR pattern.
