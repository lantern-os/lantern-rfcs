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
