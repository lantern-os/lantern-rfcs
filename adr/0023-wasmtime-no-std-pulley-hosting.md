---
adr: 0023
title: Hosting Wasmtime on riscv64 — no_std + the Pulley interpreter and the custom-platform contract
status: Accepted
date: 2026-09-01
deciders: ["TSC"]
rfc: ../rfcs/0018-confined-execution-port.md
supersedes: []
superseded_by: null
---

# ADR-0023: Hosting Wasmtime on riscv64 — `no_std` + the Pulley interpreter and the custom-platform contract

## Context

[ADR-0017](./0017-wasm-engine-selection-and-aot-strategy.md) fixed Wasmtime as the engine,
split `lantern-runtime` into a runtime role (no Cranelift/Winch linked) and an offline
compiler role, and committed to "AOT-compiled, no JIT in sensitive contexts" — but it
explicitly left *how the runtime role is hosted on `riscv64`* to later work, naming
"Wasmtime's custom-platform embedding hooks against `lantern-hal` / VSpace-Frame
capabilities" as the deferred piece. [ADR-0021](./0021-phase-2-complete-phase-3-opened.md)
then carried that forward: `lantern-runtime` builds only for a native `std` host target and
does not build for `riscv64gc-unknown-none-elf` the way `lantern-hal` / `lantern-kernel` /
`lantern-capabilities` / `lantern-crypto` / `lantern-filesystem` do.

[RFC-0018](../rfcs/0018-confined-execution-port.md) proposed closing this and has been
accepted. This ADR is the durable record of its Part 3;
[ADR-0022](./0022-confined-service-model-and-call-transport.md) records Parts 1–2.

## Decision

**`lantern-runtime`'s runtime role is hosted on `riscv64` by building Wasmtime `no_std`
with the Pulley bytecode interpreter and implementing its custom-platform C hooks over
`Frame` capabilities and the single-stack execution model.**

### Build configuration

The runtime role builds Wasmtime with `default-features = false` and features
`["runtime", "component-model", "pulley", "custom-virtual-memory",
"custom-native-signals", "custom-sync-primitives"]` — **no `std`**, no `threads`, no
`component-model-async`. Cranelift is already absent (ADR-0017).

### Execution — the Pulley portable bytecode interpreter

No runtime code generation, no executable memory, no signal-based traps: Wasm traps are
interpreter-detected and surface as `Result::Err`. This is consistent with ADR-0017's
"no JIT in sensitive contexts" — the artifact is still ahead-of-time compiled, just to
portable bytecode rather than native machine code.

### The compiler role emits Pulley

`lantern-sdk build` / `precompile_and_sign` target `pulley64` instead of the host ISA. The
`.cwasm` inside a `.lpkg` becomes a **portable** artifact — one package runs on any
LanternOS regardless of the machine's ISA, which the native-code `.cwasm` was not. The
compiler role still runs off-device, on a dev machine or build server.

### The custom-platform C contract

The symbols Wasmtime's `sys/custom` layer (`src/runtime/vm/sys/custom/capi.rs`) needs,
implemented by `lantern-runtime` against `lantern-abi`:

| Symbol(s) | LanternOS implementation |
| --- | --- |
| `wasmtime_page_size` | fixed 4 KiB (`RISCV64_PAGE_SIZE`) |
| `wasmtime_mmap_new` / `wasmtime_mmap_remap` / `wasmtime_munmap` / `wasmtime_mprotect` | retype `Untyped` → `Frame`, `FrameInvoke::Map` / `Unmap` into the runtime's VSpace at a reserved virtual range; `mprotect` re-maps with new `PteFlags`. Wasm linear memories become mapped `Frame` regions the runtime owns. |
| `wasmtime_tls_get` / `wasmtime_tls_set` | one static pointer — the runtime is single-threaded per ADR-0010's single-stack model |
| `wasmtime_sync_lock_*` / `wasmtime_sync_rwlock_*` | uncontended: acquire/release are a compare-swap that never spins |
| `wasmtime_init_traps` / the trap handler | a no-op registration — Pulley raises no host traps; a genuine runtime-process fault (a Wasmtime or Pulley bug) faults the process, blast-radius = its capabilities |
| `wasmtime_fiber_*` | not linked (`component-model-async` off) |
| `wasmtime_memory_image_*` (CoW fast instantiate) | v0: return "unsupported" so Wasmtime falls back to zero-fill; a later optimisation maps a CoW `Frame` |

### Interruption

Preempting a runaway confined component uses Wasmtime **fuel** (deterministic, no timer
needed) for v0. Epoch-based interruption driven by a real timer interrupt is a
`lantern-hal` "Next" item and a later refinement.

### Not decided here

- **Native-AOT execution** (Cranelift → `riscv64` machine code, executable-mapped,
  guard-page bounds checks, CPU-fault delivery to a U-mode handler) — a Phase 4
  performance/assurance item with its own RFC, which carries the TCB impact of CPU-fault
  delivery. Pulley is the v0 and the "sensitive contexts" default regardless.
- **`wasmtime_memory_image_*` over a CoW `Frame`** — a later fast-instantiation
  optimisation.
- **Pulley performance** for the crypto-in-a-loop and file-scan cases RFC-0014 flagged as a
  latency risk — needs benchmarking on the methodology ADR-0013 established.
- **Fault handling for the runtime process itself** — whether "the process dies,
  blast-radius = its caps" is acceptable for v0 or the launcher needs a restart/report path.
- **`lantern-abi`'s crate layout and allocator** — see
  [ADR-0022](./0022-confined-service-model-and-call-transport.md).

## Consequences

- **Easier:** `lantern-runtime` finally builds for `riscv64gc-unknown-none-elf`; a confined
  component can run on the kernel; `.lpkg` artifacts become ISA-portable — and *more*
  privacy-neutral than native ones, identical across machines, leaking nothing about the
  build host's ISA or the deploying user's hardware.
- **Harder / committed to:** Wasmtime's custom-platform C contract is now surface
  `lantern-runtime` tracks across Wasmtime upgrades; Pulley interpretation is slower than
  native execution (unbenchmarked, flagged as a real risk for hot-loop crypto); a
  Wasmtime/Pulley bug faults the runtime process (blast-radius = its capabilities —
  acceptable for v0, a launcher restart/report path left open).
- **TCB impact: none.** Dropping `std` and adding `pulley` *shrinks* Wasmtime's surface
  rather than growing it. All the custom-platform shims are confined user-space code in
  `lantern-runtime`, built on `lantern-abi` (itself outside the TCB, ADR-0022). Wasmtime
  remains a `lantern-runtime` (non-TCB) dependency. No new kernel object, syscall, or
  `lantern-hal` surface. ADR-0004's TCB boundary (kernel + HAL + boot) is unchanged.
- **Relationship to ADR-0017:** this *refines*, it does not supersede. ADR-0017's engine
  choice and runtime/compiler split stand; this fixes the `no_std` hosting strategy and the
  custom-platform contract ADR-0017 left open. If a later RFC pursues native-AOT execution,
  *that* ADR carries the TCB impact of delivering CPU faults to a U-mode handler — this one
  does not.
- **Open (tracked in `lantern-runtime` `STATUS.md`):** Pulley benchmarking; the
  runtime-process fault-handling policy; `wasmtime_memory_image_*` CoW; epoch preemption
  once `lantern-hal` has a timer interrupt.
