---
rfc: 0013
title: Wasm engine selection and AOT execution strategy for lantern-runtime
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["runtime"]
domains: ["runtime", "capabilities", "sdk"]
created: 2026-08-25
updated: 2026-08-27
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0013: Wasm engine selection and AOT execution strategy for lantern-runtime

## Summary

This RFC picks **Wasmtime** (Bytecode Alliance) as the Wasm engine `lantern-runtime` embeds
to host Component-Model/WASI-0.2 applications and agents, per
[ADR-0003](../adr/0003-wasm-as-portable-app-abi.md). It further fixes the **AOT execution
strategy**: Wasmtime's Cranelift compiler runs only in an offline, non-runtime **compiler
role** (packaging/install time), producing a serialized, **signed** `.cwasm` artifact; the
per-component confined **runtime role** that `lantern-runtime` actually spawns never links
Cranelift, never JIT-compiles, and only ever `deserialize`s an artifact whose signature it
has verified first. WASI host functions are **not** the off-the-shelf `wasmtime-wasi` crate
(which grants ambient preopens/sockets) — they are custom bindings, gated per-call on a held
LanternOS capability, matching ADR-0003's "capability-backed WASI" commitment. This RFC does
not decide the WIT-handle ⇄ capability mapping (`lantern-runtime/STATUS.md`'s separate "Next"
item) or resource-accounting wiring — both remain open, future work.

## Motivation

`lantern-runtime/STATUS.md` names "select a Wasm engine and AOT strategy" as its first
blocked "Next" item, and it is now the critical-path gap for Phase 2's exit criterion — *"a
third-party Wasm app runs confined, reads a file only via a granted capability, and cannot
touch anything it wasn't granted, demonstrated adversarially"*
([`Roadmap.md`](../../lantern-docs/wiki/Roadmap.md)). `lantern-capabilities`,
`lantern-crypto`, and `lantern-filesystem` have each already built real prototype code
(`Broker`, `Keystore`, `Store`) and each says the same thing in its own `STATUS.md`: turning
that code into deployable confined-service code needs `lantern-runtime`'s not-yet-built
execution environment, not more work in their own crate. This is the one remaining
zero-code Phase 2 component with an unblocked path forward.

This decision also fits `GOVERNANCE.md`'s "new languages, toolchains, or external
dependencies in the TCB" trigger by analogy: although the engine itself sits outside the
kernel TCB (it is confined user space, per `THREAT_MODEL.md` R2 — "engine bug ≠ kernel
compromise"), it is the single dependency every third-party app and AI agent on LanternOS
will run inside, and `lantern-ai-runtime`'s own capability-scoped-agent story
([Principle 6](../../lantern-docs/wiki/Principles.md), AI-native) builds directly on top of
it. That combination — universal blast-radius surface, hard to swap once apps/SDK target it
— is exactly what an RFC (not an implementation PR) should settle, mirroring RFC-0007's
reasoning for ratifying crypto primitives rather than letting `lantern-crypto` pick them ad
hoc.

## Guide-level explanation

A LanternOS app or agent ships as a Wasm **component**. Before it ever runs on a device, a
build/packaging step compiles that component ahead-of-time into a `.cwasm` file (Wasmtime's
serialized, machine-code-bearing module format) and signs it. `lantern-runtime` spawns a
confined per-component host process; that process's only job is to check the signature,
load the already-compiled `.cwasm`, and run it — it never invokes a compiler, so a component
never gets to influence what machine code ends up executing beyond what the offline compile
step already fixed and signed off on.

Inside that host process, whenever the component calls a WASI interface — open a file, read
the clock, send on a socket — the call lands on a LanternOS-specific host function, not a
generic OS shim. That function's first move is always: "does this component hold a
capability that covers this exact call?" If not, the call fails, the same deny-by-default
posture `lantern-capabilities`' `Broker` already established one layer down.

## Reference-level explanation

### Engine: Wasmtime

`lantern-runtime` embeds [Wasmtime](https://wasmtime.dev), the Bytecode Alliance's
reference-quality Wasm engine, for:

- **Component Model + WASI 0.2 support** — Wasmtime *is* the reference implementation the
  Component Model and WASI 0.2 proposals are prototyped and tested against, the exact
  surface ADR-0003 already committed to. Any alternative engine's support for that surface
  trails Wasmtime's by definition.
- **Memory safety** — Wasmtime is written in Rust ([ADR-0001](../adr/0001-rust-as-primary-language.md)'s
  primary-language commitment), unlike its closest embedded-oriented competitors. Even
  though the engine is confined and out of the kernel TCB, Principle 4 (memory safety) still
  favors not introducing a second (C/C++) memory-unsafe toolchain into the one process every
  third-party app and agent runs inside, when a safe alternative already meets every other
  requirement.
- **A real AOT pipeline** — Cranelift, with a documented, stable `Engine::precompile_component`
  / serialized-module (`.cwasm`) path designed exactly for "compile once, load-and-run many
  times without the compiler present" deployments.
- **A path off assumed-POSIX hosting** — Wasmtime has an explicit "custom platform" embedding
  mode (its execution crates can run against caller-supplied hooks for virtual memory,
  traps, and threading rather than assuming a POSIX host). LanternOS has no POSIX layer;
  `lantern-runtime` will need to implement those hooks against `lantern-hal`/the kernel's
  VSpace/Frame capabilities ([ADR-0012](../adr/0012-vspace-frame-capabilities-and-elf-loader.md))
  directly. This RFC fixes the engine choice on the strength of that mode existing; it does
  **not** resolve the porting work itself (see "Unresolved questions").
- **A resource-metering hook already built in** — Wasmtime's fuel/epoch-interruption
  mechanism gives a natural future attachment point for R4's per-component CPU/memory
  budgets, tied to scheduling contexts. Not wired up by this RFC.

### AOT strategy: compiler role and runtime role are different code

Two distinct roles, deliberately never colocated in the same running confined-app process:

1. **Compiler role.** Takes a `.wasm` component (from the SDK build output or an app
   package), runs it through Wasmtime's Cranelift-backed compiler
   (`Engine::precompile_component`), and emits a `.cwasm` artifact, which is then signed
   with the packager's Ed25519 key ([RFC-0007](./0007-cryptographic-primitive-set.md)'s
   ratified primitive, via `lantern-crypto`'s `Keystore`) — the same signature-over-a-trusted-
   artifact shape `lantern-boot` already uses for kernel-image verification
   ([RFC-0002](./0002-microkernel-architecture.md)). This role links Cranelift and the full
   Wasmtime compilation stack. Exactly where it executes (SDK build tooling on a developer
   machine, vs. a dedicated on-device "package install" service) is left to `lantern-sdk`
   and packaging design, out of scope here — this RFC fixes only that it is **not** part of
   the per-component runtime process below.
2. **Runtime role.** The confined host process `lantern-runtime` spawns per running
   component. It links only Wasmtime's execution crates (no Cranelift, no in-process
   compilation of any kind — a Wasm binary is never accepted directly, only a `.cwasm`).
   Before touching a `.cwasm` artifact it **verifies its signature**
   (`lantern-crypto`-mediated Ed25519 check). Only then does it call Wasmtime's
   `deserialize` entry point to load it. This ordering is load-bearing, not incidental:
   Wasmtime documents `deserialize` as requiring the caller to already trust the input,
   since deserializing attacker-controlled bytes is not memory-safe. Verifying the signature
   first is what makes calling `deserialize` sound at all.

This gives the "AOT-compiled and validated before running; no JIT in sensitive contexts"
line in `lantern-runtime/ARCHITECTURE.md` a concrete mechanism, not just a stated intent.

### WASI: custom, capability-gated host bindings — not `wasmtime-wasi`

The off-the-shelf `wasmtime-wasi` crate implements WASI 0.2 against ambient host resources
(preopened directories, real sockets) — the opposite of ADR-0003's "no preopened
directories, no ambient sockets" stance. `lantern-runtime` instead implements the WASI 0.2
Component-Model interfaces itself, as host functions that take a WIT-typed resource handle
and, before doing anything else, check that the calling component actually holds a
LanternOS capability covering that handle and operation — the same
`Broker`/`Keystore`/`Store` deny-by-default badge check already proven one layer down.
**Which** capability a given resource handle maps to is explicitly not decided here; it is
`lantern-runtime/STATUS.md`'s separate, already-tracked "specify the WIT-handle ⇄
capability mapping" item, and belongs in its own RFC.

### What this RFC does not decide

- The WIT-handle ⇄ LanternOS-capability mapping (a separate, future RFC).
- Where the compiler role physically runs (`lantern-sdk`/packaging design, not this RFC).
- The exact `.cwasm` artifact signature wire format (left to implementation, following
  RFC-0007's precedent of fixing primitives without fixing every wire format up front).
- Resource accounting / fuel-metering wiring to scheduling contexts (R4) — a future RFC.
- The custom-platform porting work itself (replacing Wasmtime's assumed mmap/signal/thread
  hooks with LanternOS-native ones) — real engineering work this RFC's decision requires,
  not work this RFC performs.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none moved. The engine remains confined user space per
  ADR-0003/`THREAT_MODEL.md` R2 ("engine bug ≠ kernel compromise"); this RFC fixes *which*
  engine, not where it sits relative to the kernel.
- **New assets introduced and who can reach them:** the compiled `.cwasm` artifact and its
  signature become a new asset — anything that can substitute an unsigned or corrupted
  `.cwasm` for a legitimate one, ahead of the runtime role's `deserialize` call, could reach
  memory-unsafe behavior inside the runtime process (Wasmtime's own documented contract:
  `deserialize` requires trusted input). This is the one genuinely new sharp edge this
  decision introduces.
- **New adversary capabilities, if any:** none beyond the artifact-substitution concern
  above, which this RFC treats as a first-class requirement rather than a deferred
  hardening step.
- **Mitigations:** mandatory Ed25519 signature verification (RFC-0007 primitive) before any
  `.cwasm` is deserialized, mirroring `lantern-boot`'s kernel-image verification pattern;
  capability-gated custom WASI host bindings (R1); confinement per component (R2); no
  in-process compiler in the runtime role, shrinking what a compromised component can
  possibly influence.
- **Net change to attacker surface:** ADR-0003 already accepted hosting untrusted Wasm as
  the app/agent model; this RFC's job is choosing the specific engine and making the
  compiled-artifact trust chain explicit and mitigated, rather than leaving it as an
  implicit assumption an implementation PR might get wrong silently.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No. `lantern-runtime` is confined
  user space under ADR-0004's TCB boundary (kernel + HAL + boot only); Wasmtime's execution
  crates run inside that confined service, not the kernel.
- **Does this add a dependency to the TCB?** No, per above. Additionally, the compiler
  role's Cranelift/compilation dependencies are excluded even from the confined runtime
  service's own shipped binary — only the smaller execution-only Wasmtime crates link there.
- **Effect on TCB size and auditability:** neutral for the kernel TCB (unchanged); positive
  for the runtime service's own audit surface, since splitting the compiler role out means
  the code actually running per-component at execution time is smaller than "the whole
  engine including its compiler."

## Privacy impact

WASI clock/RNG/env interfaces stay mediated and minimized exactly as `THREAT_MODEL.md` R3
and `lantern-docs/wiki/Runtime.md` already require; this RFC does not change that stance —
it only fixes which engine's primitive calls the custom host bindings dispatch to
underneath.

## Alternatives considered

- **WAMR (WASM Micro Runtime).** A real, embedded-oriented alternative with its own AOT
  path. Rejected: it is written in C, which reintroduces exactly the memory-unsafe-language
  exposure Principle 4 steers away from, inside the one process every third-party app and
  agent runs in; and its Component Model / WASI 0.2 support trails Wasmtime's, since
  Wasmtime is the surface's reference implementation rather than a downstream adopter of it.
- **WasmEdge.** Similar reasoning to WAMR: C++, and secondary (not reference-level) Component
  Model support.
- **wasm3.** Interpreter-only, no AOT compilation path at all — fails the "AOT-compiled...
  no JIT in sensitive contexts" requirement outright, since there is no compiled artifact to
  validate ahead of execution in the first place.
- **Roll a custom minimal Wasm engine.** Rejected outright, mirroring RFC-0007's rejection of
  rolling custom AEAD: Wasm validation and spec compliance is large, security-critical
  surface; reinventing it is far riskier than adopting the actively-maintained reference
  implementation, for no compensating benefit, and would divert effort from kernel/capability
  work that has no substitute.
- **Defer engine selection until `lantern-sdk`/an app actually needs to run.** Rejected:
  `lantern-capabilities`, `lantern-crypto`, and `lantern-filesystem` are each already
  blocked on exactly this (their `STATUS.md`s each name it), and `GOVERNANCE.md`'s
  toolchain-selection trigger applies regardless of when implementation catches up.

## Prior art

**Fermyon Spin** and **Fastly Compute** — both build confined, multi-tenant platforms on
Wasmtime by replacing `wasmtime-wasi`'s ambient implementation with their own scoped host
bindings, the direct precedent for this RFC's "custom capability-gated WASI, not
off-the-shelf `wasmtime-wasi`" decision. **Bytecode Alliance**'s own Component Model / WASI
0.2 specification work, which Wasmtime tracks as reference implementation. **seL4**-family
microkernels hosting confined userland execution, the general precedent this project's
engine-outside-the-TCB stance already follows. **WAMR** and **WasmEdge** as the rejected
embedded-oriented alternatives.

## Unresolved questions

- The WIT-handle ⇄ LanternOS-capability mapping — separate future RFC, already tracked in
  `lantern-runtime/STATUS.md`.
- Where the compiler role runs in practice (developer-side SDK tooling vs. an on-device
  install-time service) — a `lantern-sdk`/packaging design question, not resolved here.
- The custom-platform porting work (Wasmtime's mmap/signal/thread hooks against
  `lantern-hal`/VSpace-Frame capabilities directly) — real, sequenced engineering work this
  decision requires but does not perform.
- The exact `.cwasm` artifact signature wire format and key-management story (reusing
  RFC-0011's sealed-token infrastructure vs. a simpler direct signature) — left to
  implementation.
- Resource-accounting/fuel-metering wiring to MCS scheduling contexts (R4) — deferred to a
  future RFC.

## Future possibilities

- The WIT-handle ⇄ capability-mapping RFC, building directly on this one.
- A resource-accounting RFC tying Wasmtime's fuel/epoch-interruption mechanism to scheduling
  contexts.
- `lantern-ai-runtime`'s capability-scoped agent work, which is blocked on exactly this
  execution environment existing (its own `STATUS.md`).
- Renewed interest from Phase 3's formal-verification push in the Component Model
  call boundary, once that work begins.

## Resulting ADRs

[ADR-0017](../adr/0017-wasm-engine-selection-and-aot-strategy.md) fixes Wasmtime as
`lantern-runtime`'s Wasm engine, the compiler-role/runtime-role split with mandatory
signature verification before `deserialize`, and custom capability-gated WASI host
bindings in place of `wasmtime-wasi`.
