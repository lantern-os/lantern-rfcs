---
adr: 0003
title: WebAssembly is the portable application ABI
status: Accepted
date: 2026-06-09
deciders: ["TSC"]
rfc: null
supersedes: []
superseded_by: null
---

# ADR-0003: WebAssembly is the portable application ABI

## Context

We need an application execution model that is **portable** across x86-64 and RISC-V,
**sandboxed by default** (no ambient authority), **capability-friendly**, and **language
agnostic** so the ecosystem is not limited to Rust. We also want applications and AI agents
to run under explicit, fine-grained authority rather than full native access.

## Decision

**WebAssembly (Wasm), with the Component Model and WASI Preview 2, is the primary portable
application ABI for LanternOS.**

- Applications and many agents are distributed as **Wasm components**; the
  [`lantern-runtime`](https://github.com/lantern-os/lantern-runtime) hosts them.
- The host does **not** expose POSIX/WASI ambient capabilities by default. Instead, WASI
  interfaces are backed by **LanternOS object capabilities** (RFC-0003): a component
  receives a handle only if it was explicitly granted one. This makes Wasm's "deny by
  default" sandbox and our capability model the same mechanism viewed from two sides.
- Native code is still used for the TCB and performance-critical services; Wasm is for the
  *application and agent* layer, not the kernel.
- The Component Model's typed interfaces (WIT) become the contract surface the SDK
  ([`lantern-sdk`](https://github.com/lantern-os/lantern-sdk)) generates bindings for.

## Consequences

- **Easier:** write-once-run-on-any-ISA apps; strong, cheap sandboxing aligned with our
  capability model; a language-agnostic ecosystem; safe hosting of untrusted AI agents.
- **Harder:** Wasm performance overhead vs native (mitigated by AOT compilation and
  pre-validation); the Component Model/WASI surface is still maturing and must be pinned and
  tracked; capability-backed WASI requires custom host bindings rather than off-the-shelf
  WASI.
- **Committed to:** an AOT-capable Wasm engine, capability-backed (not ambient) WASI host
  functions, and treating the WIT interface set as a versioned public ABI governed by RFCs.
