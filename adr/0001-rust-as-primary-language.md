---
adr: 0001
title: Rust is the primary implementation language
status: Accepted
date: 2026-06-09
deciders: ["TSC"]
rfc: null
supersedes: []
superseded_by: null
---

# ADR-0001: Rust is the primary implementation language

## Context

Memory-safety bugs (use-after-free, buffer overflow, data races) account for a large
majority of critical vulnerabilities in systems software. Our first-tier principle is
**memory safety**, and our security ceiling is set by the integrity of the TCB. We need a
language that gives memory and thread safety *without* a garbage collector or large runtime
(neither is acceptable in a kernel), supports `no_std` bare-metal targets, has a mature
RISC-V backend (via LLVM), and lets us isolate and audit the unavoidable `unsafe` code.

Candidates considered: Rust, Zig, C, C++, Ada/SPARK.

## Decision

**Rust is the primary implementation language for LanternOS** — kernel, services, runtimes,
and SDK. Specifically:

- `unsafe` is permitted only where required (MMIO, page tables, the lowest HAL), must be
  isolated behind safe abstractions, justified in an in-line comment, and reviewed.
- **Zig** is permitted where it is materially better — small freestanding boot/HAL pieces
  and C-interop — subject to an ADR per use.
- **C** is allowed only when unavoidable (vendored firmware blobs, hardware contracts) and
  must be justified and minimised.
- **TypeScript** is the language for tooling, the website, and SDK bindings for web.
- **WebAssembly** is a compilation target, not a source language (see ADR-0003).

We do not adopt additional implementation languages without an ADR.

## Consequences

- **Easier:** eliminating an entire class of vulnerabilities in the TCB; fearless
  concurrency; strong abstractions at zero cost; a single language across most of the stack.
- **Harder:** a steeper learning curve for contributors; some hardware bring-up needs
  careful `unsafe`; toolchain bring-up on new RISC-V targets must be tracked.
- **Committed to:** `no_std` discipline, an `unsafe` review policy, and treating `unsafe`
  density in the TCB as a tracked quality metric. Ada/SPARK techniques may still inform our
  later formal-verification work without changing the implementation language.
