# Architecture Decision Records (ADRs)

An **ADR** captures a single architectural decision, its context, and its consequences.
ADRs are the project's long-term memory: they let a contributor years from now understand
*why* the system is the way it is, without archaeology through chat logs.

## Rules

1. **Append-only.** Once an ADR is `Accepted`, its substance is not edited. To change a
   decision, write a new ADR that **supersedes** the old one and update both
   `superseded_by`/`supersedes` links.
2. **One decision per ADR.** Keep them small and focused.
3. **Numbered sequentially.** `NNNN-short-title.md`.
4. **Linked to its origin.** If an ADR came from an RFC, link it. Standalone ADRs are
   allowed for decisions too small to need an RFC.

## Status values

`Proposed` → `Accepted` → (`Superseded` | `Deprecated`). `Rejected` is also recorded.

## Format

```
---
adr: NNNN
title: <decision>
status: Accepted
date: YYYY-MM-DD
deciders: [...]
rfc: <link or null>
supersedes: []
superseded_by: null
---

# ADR-NNNN: <decision>

## Context
The forces at play: technical, principle-driven, and practical.

## Decision
The position we are taking, stated plainly.

## Consequences
What becomes easier, what becomes harder, what we are now committed to.
```

## Index

| # | Decision | Status |
| --- | --- | --- |
| [0001](./0001-rust-as-primary-language.md) | Rust is the primary implementation language | Accepted |
| [0002](./0002-riscv-target-isa.md) | RISC-V is the long-term target ISA | Accepted |
| [0003](./0003-wasm-as-portable-app-abi.md) | WebAssembly is the portable application ABI | Accepted |
| [0004](./0004-kernel-responsibilities-and-tcb-boundary.md) | Kernel responsibility list and the TCB boundary | Accepted |
| [0005](./0005-object-capabilities-as-universal-authority-model.md) | Object capabilities as the universal authority model | Accepted |
| [0006](./0006-three-layer-capability-structure.md) | Three-layer capability structure — kernel / service / sealed | Accepted |
| [0007](./0007-phase-0-complete-phase-1-opened.md) | Phase 0 complete; Phase 1 (microkernel prototype) opened | Accepted |
| [0008](./0008-kernel-syscall-ipc-abi.md) | Kernel syscall/IPC ABI — capability invocation, message registers, IPC buffer | Accepted |
| [0009](./0009-phase1-scheduling-context-model.md) | Phase 1 scheduling-context model — MCS-shaped object, simplified semantics | Accepted |
| [0010](./0010-kernel-concurrency-model.md) | Kernel concurrency model — single-stack, run-to-completion (Phase 1) | Accepted |
| [0011](./0011-cryptographic-primitive-set.md) | Cryptographic primitive set (Phase 1) | Accepted |
| [0012](./0012-vspace-frame-capabilities-and-elf-loader.md) | VSpace/Frame capabilities and a minimal ELF loader | Accepted |
| [0013](./0013-ipc-latency-benchmark.md) | IPC latency benchmark — methodology, Phase 1 target, and a known QEMU-only bug | Accepted |
| [0014](./0014-phase-1-complete-phase-2-opened.md) | Phase 1 complete; Phase 2 (capability runtime & first services) opened | Accepted |
| [0015](./0015-sealed-capability-token-format.md) | Sealed capabilities — macaroon-style BLAKE3-keyed-MAC token format | Accepted |
| [0016](./0016-monotonic-clock-primitive.md) | A monotonic clock read primitive for lantern-hal | Accepted |
| [0017](./0017-wasm-engine-selection-and-aot-strategy.md) | Wasm engine selection and AOT execution strategy for lantern-runtime | Accepted |
| [0018](./0018-wit-handle-capability-mapping.md) | WIT-handle ⇄ capability mapping — resource-scoped vs link-scoped, first two interfaces | Accepted |
| [0019](./0019-filesystem-wit-interface.md) | Capability-scoped filesystem WIT interface — custom `lantern:host/filesystem`, no paths | Accepted |
| [0020](./0020-capability-manifest-format.md) | The lantern-sdk capability manifest format — TOML, abstract roles, combined-digest signing | Accepted |
| [0021](./0021-phase-2-complete-phase-3-opened.md) | Phase 2 complete; Phase 3 (privacy, identity, networking, and AI) opened | Accepted |
| [0022](./0022-confined-service-model-and-call-transport.md) | Confined-service model and the badged-endpoint + shared-memory call transport | Accepted |
| [0023](./0023-wasmtime-no-std-pulley-hosting.md) | Hosting Wasmtime on riscv64 — `no_std` + the Pulley interpreter and the custom-platform contract | Accepted |
