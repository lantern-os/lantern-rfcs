---
adr: 0017
title: Wasm engine selection and AOT execution strategy for lantern-runtime
status: Accepted
date: 2026-08-27
deciders: ["TSC"]
rfc: ../rfcs/0013-wasm-engine-selection-and-aot-strategy.md
supersedes: []
superseded_by: null
---

# ADR-0017: Wasm engine selection and AOT execution strategy for lantern-runtime

## Context

`lantern-runtime/STATUS.md` named "select a Wasm engine and AOT strategy" as its first
blocked "Next" item, and it was the last zero-code Phase 2 component with an otherwise
unblocked path: `lantern-capabilities`'s `Broker`, `lantern-crypto`'s `Keystore`, and
`lantern-filesystem`'s `Store` each already document that turning their prototype code
into deployable confined-service code needs `lantern-runtime`'s not-yet-built execution
environment. [RFC-0013](../rfcs/0013-wasm-engine-selection-and-aot-strategy.md) proposed a
concrete engine and AOT strategy and has been accepted; this ADR is the durable record.

## Decision

**`lantern-runtime` embeds Wasmtime as its Wasm engine, split into two roles that are
never linked into the same build.**

- **Runtime role** (the confined per-component host process): links only Wasmtime's
  execution crates (`runtime` + `component-model` cargo features) — no `cranelift`, no
  `winch`. Because `Component::new`/`Engine::precompile_component` (Wasmtime's
  compile-from-source API) are themselves `#[cfg]`-gated on those features, they are
  genuinely absent from this build's symbol table, not merely unused by convention. This
  role only ever calls `Component::deserialize` on a `.cwasm` artifact whose Ed25519
  signature ([RFC-0007](../rfcs/0007-cryptographic-primitive-set.md)'s ratified primitive)
  it has already verified — required ordering, since Wasmtime documents `deserialize` as
  unsound on untrusted input.
- **Compiler role** (offline, packaging/install time — not the running per-component
  process): links Wasmtime's `cranelift` feature, runs `Engine::precompile_component`,
  and signs the resulting artifact. Where exactly it runs (SDK build tooling vs. an
  on-device install service) is left to `lantern-sdk`/packaging design.
- WASI (Component Model / WASI 0.2) host bindings are **custom and capability-gated**,
  not the ambient `wasmtime-wasi` crate — following the Fermyon Spin / Fastly Compute
  precedent of replacing that crate's ambient implementation with a scoped one. Not
  wired up yet: today's runtime role can load and run a component with no host imports.
  Which WIT-typed resource handle maps to which LanternOS capability is deliberately
  **not** decided by this ADR — a separate future RFC, already tracked in
  `lantern-runtime/STATUS.md`.

Rejected alternatives (full reasoning in the RFC): WAMR and WasmEdge (C/C++, weaker
Component Model support than Wasmtime's reference implementation), wasm3 (no AOT path at
all), and rolling a custom engine (Wasm validation is large, security-critical surface not
worth reinventing, mirroring RFC-0007's rejection of rolling custom AEAD).

## Consequences

- **Easier:** `lantern-capabilities`/`lantern-crypto`/`lantern-filesystem` now have a
  concrete target to become deployable confined services against, instead of an open
  question. A first prototype can prove the compile → sign → verify → deserialize → run
  mechanism against a real Wasmtime `Engine` today, on the host target, the same
  "prototype against something real, not a simulation" discipline those three crates'
  own test suites already follow.
- **Harder:** Wasmtime assumes a POSIX-ish host (mmap, threads, signal-based traps) that
  LanternOS does not provide. Getting the runtime role actually hosted in a confined
  `riscv64` process needs Wasmtime's custom-platform embedding hooks implemented against
  `lantern-hal`/VSpace-Frame capabilities directly — real, unstarted porting work this
  decision requires but does not perform. Until that lands, `lantern-runtime` only builds
  and runs against a native `std` host target.
- **Committed to:** Wasmtime as the engine (a toolchain/dependency choice `GOVERNANCE.md`
  treats as RFC-governed by analogy, given the universal blast-radius surface this
  engine has across every third-party app and agent); the compiler-role/runtime-role
  split as a build-time (not merely runtime-checked) property; mandatory signature
  verification before any `.cwasm` deserialization; and capability-gated custom WASI
  bindings rather than `wasmtime-wasi`'s ambient ones, once WASI is wired up at all.
- **Still open:** the WIT-handle ⇄ capability mapping, resource-accounting/fuel-metering
  wiring to scheduling contexts, the `.cwasm` artifact signature wire format, and the
  custom-platform porting work itself — all tracked in `lantern-runtime/STATUS.md`, none
  resolved here.
