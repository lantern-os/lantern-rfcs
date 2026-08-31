---
rfc: 0018
title: The confined-execution port — services and the runtime as confined processes on the kernel
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["runtime", "kernel", "capabilities", "crypto", "filesystem", "hal"]
domains: ["runtime", "kernel", "capabilities", "crypto", "filesystem", "hal", "boot"]
created: 2026-09-01
updated: 2026-09-01
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-09-01.** The three parts are fixed by two ADRs:
> [ADR-0022](../adr/0022-confined-service-model-and-call-transport.md) (Parts 1–2 — services
> as confined U-mode programs, the `lantern-abi` substrate, and the badged-endpoint +
> shared-`Frame` call transport) and
> [ADR-0023](../adr/0023-wasmtime-no-std-pulley-hosting.md) (Part 3 — Wasmtime `no_std` +
> the Pulley interpreter and the custom-platform contract). Both record that this work adds
> nothing to the TCB. The per-service wire protocols, the launch binder / consent UX
> (`lantern-shell`, per RFC-0015), and native-AOT execution (Phase 4) remain out of scope,
> as stated below.

# RFC-0018: The confined-execution port — services and the runtime as confined processes on the kernel

## Summary

[ADR-0021](../adr/0021-phase-2-complete-phase-3-opened.md) named one large piece of work
as Phase 3's foundational prerequisite: **the Phase 2 services (`Broker`, `Keystore`,
`Store`) and `lantern-runtime` run as in-process stand-ins against a real `KernelState` on
a native host target, not as confined processes on the kernel.** This RFC fixes how that
changes. Three parts: (1) each service becomes a **confined U-mode program** with its own
CSpace/VSpace, listening on an endpoint, holding only its own retained authority;
(2) a confined component's host calls (`key.sign`, `file.read`, …) are forwarded from
`lantern-runtime` to the owning service over **a badged endpoint capability plus a shared
memory region** — no `&mut KernelState`, no `u64` badge passed as data, and (deliberately)
no in-memory IPC buffer; (3) `lantern-runtime` is hosted on `riscv64` by building Wasmtime
**`no_std` with the Pulley bytecode interpreter** and implementing its custom-platform
hooks over `Frame` capabilities and the single-stack execution model. It does **not** fix
the per-service wire protocols (each service's own follow-up, mirroring how RFC-0014 fixed
the mapping shapes and left each interface separate), the launch binder / consent UX
(`lantern-shell`, per RFC-0015), or native AOT execution (Phase 4).

## Motivation

Phase 2 proved the capability boundary holds for genuinely untrusted code — adversarially,
via `lantern-example-signer`. But the proof runs on a host: `Keystore::sign`,
`Store::read`, and `Broker::mint` all take `&mut lantern_kernel::state::KernelState`
directly (valid only for privileged, same-address-space code), `lantern-runtime` builds
only for a native `std` target, and the demo's host constructs the services in its own
address space so the "IPC round trip" between a confined component and the keystore is an
ordinary function call.

Every Phase 3 deliverable depends on closing this. "Capability-scoped agents with a
faithful audit log" ([Roadmap](../../lantern-docs/wiki/Roadmap.md)) cannot be faithful
while the runtime and the services it calls share an address space — a compromised runtime
would *be* the keystore. Encrypted-at-rest storage "tied to hardware-backed keys" needs the
keystore as a process an attacker who owns the app still cannot read. `lantern-network`'s
capability-scoped sockets need to reach a confined component the way `lantern:host/keystore`
already does. This is the work that turns Phase 2's *demonstration* of isolation into
*deployed* isolation — [Principle 1](../../lantern-docs/wiki/Principles.md) (security by
architecture): "if this component is fully compromised, what is the blast radius? The
answer must be exactly the capabilities it held."

Fixing the **model and the contracts** in an RFC — rather than discovering them inside the
first porting PR — matters because this touches the kernel/HAL trust boundary's *use* (not
its definition), spans six components, and sets the substrate every later confined service
is built on. `GOVERNANCE.md` reserves exactly this for the RFC process.

## Guide-level explanation

Today, on the host demo:

```
┌──────────────── one address space ────────────────┐
│  lantern-runtime  ──(fn call)──▶  Keystore  Store  │   ← "confined" is Wasm-only
└───────────────────────────────────────────────────┘
```

After this RFC, on `riscv64`:

```
┌─ keystore proc ─┐   ┌─ store proc ─┐   ┌──────── runtime proc (one per app) ────────┐
│ own VSpace      │   │ own VSpace   │   │ own VSpace; holds ONLY the service         │
│ key material    │   │ blocks       │   │ endpoints + shared regions the app's       │
│ Recv(endpoint)  │   │ Recv(endpt)  │   │ manifest granted                          │
└────────▲────────┘   └──────▲───────┘   │  ┌──── Wasm component (sandboxed) ────┐    │
         │ Call(SIGN)        │ Call(READ)│  │  key.sign(msg) ───────────────────┼────┼─▶ (badged endpoint + shared mem)
         └───────────────────┴───────────┼──┤  file.read()  ────────────────────┼────┼─▶
                                         │  └───────────────────────────────────┘    │
                                         └───────────────────────────────────────────┘
```

A **launcher** (extending `lantern-boot`'s narrowing-waterfall root task) loads the
keystore program, the store program (granting it a keystore endpoint for its one AEAD
key), and — per app — one runtime program, granting the runtime exactly the keystore/store
endpoints and shared regions that app's `.lpkg` manifest declares. The runtime deserializes
the app's Pulley `.cwasm` and instantiates it *inside its own address space*; Wasm's memory
model plus the runtime's own confinement are the two nested boundaries (ADR-0004: "an
engine bug is bounded by *its* capabilities").

When the guest calls `key.sign(msg)`, the runtime copies `msg` into the shared region it
holds for that key handle, invokes the **badged endpoint capability** for that handle with
`Call(op = SIGN, len)`, and blocks. The kernel delivers the badge to the keystore, which
re-checks it against its own grant table (the same `check_access` deny-by-default it does
today), signs from key material only it can see, writes the signature back into the shared
region, and `Reply`s. The runtime reads the result and returns it to the guest. A
compromised runtime can scribble the shared region, lie about `len`, or invoke the wrong
badge — and gets exactly `error-code::access` or a `CryptoFailure`, never another app's
key.

## Reference-level explanation

### Part 1 — Services as confined programs

A service (keystore, store) is a standalone `#![no_std]` `riscv64` program, loaded like
`lantern-boot/hello-service` / `broker-service` already are (RFC-0008/ADR-0012), with:

- its own **CSpace** (a flat CNode) and **VSpace** (Sv39, built from `Frame` retypes);
- one **request endpoint** it `Recv`s on in a run-to-completion loop (RFC-0006);
- its **retained authority**: for the keystore, the key material lives in its own BSS/heap
  and a GRANT-able source capability its composed `Broker` attenuates from; for the store,
  the block array in its own memory plus a keystore endpoint (badged for its single v0
  AEAD key) it was handed at launch;
- **no** capability to any other component's objects.

`Broker`, `Keystore`, and `Store` keep their exact Phase 2 *logic* — the badge/grant
tables, `check_access`, the mint→grant sequence, the AEAD/signing/refcounting — but their
Rust API changes from `&mut KernelState` to **issuing syscalls**. That substrate is a new
small component (working name **`lantern-abi`**): typed wrappers for the RFC-0005/ADR-0008
syscall table (`CNodeInvoke`, `FrameInvoke`, `Send`/`Recv`/`Call`/`Reply`, `Signal`/`Wait`),
plus a minimal program runtime (entry point, a bump/`talc` allocator over a granted heap
`Frame`, a `#[panic_handler]` that `Signal`s a fault notification and halts). Every confined
program — services and the runtime — links `lantern-abi`. It is **not** in the TCB (it
issues syscalls; it does not implement them).

The launcher is the narrowing-waterfall, extended: `lantern-boot`'s loader gains the
ability to load several programs and, for each, place exactly the capabilities named by a
launch description into its CSpace (via `CNodeInvoke::CopyCross`, RFC-0010). For Phase 3
v0 the launch description is hard-wired in the launcher, exactly as
`lantern-example-signer`'s runner hard-wires its `GrantManifest` today; resolving a
manifest `role` to a concrete key/file with user consent is `lantern-shell`'s job
(RFC-0015) and layers on top later.

### Part 2 — The service-call transport

**Badged endpoint, not a `u64`.** In the Phase 2 prototype a `HostCapability` /`HostFile`
carries `badge: u64` and `KeystoreService::sign(badge, key, msg)` takes it as an argument.
After this RFC each resource-scoped handle is backed by a **badged capability to the owning
service's endpoint** — minted by that service's `Broker` (`CNodeInvoke::Mint`, scoped to
one `(KeyId, KeyOps)` / `(FileId, FileOps)`) and transferred into the runtime's CSpace at
grant time (the RFC-0010 `extra_caps == 1` live transfer). The runtime never sees or
handles the badge value; it invokes the capability, and the kernel delivers the badge to
the service on `Recv` (`mr0` ← sender's endpoint badge, ADR-0008). A guest cannot forge a
handle for a key it was not granted because the corresponding badged capability simply is
not in the runtime's CSpace — the same property RFC-0014 already relies on one layer up.

**Shared memory for payloads, not the IPC buffer.** Signing a message, encrypting a blob,
and reading a file all move more than the three payload registers (`mr1..mr3`) hold, and
the kernel's in-memory IPC buffer is designed (RFC-0005/ADR-0008) but not implemented
(`length > 0` is rejected). Rather than implement it, this RFC uses **a shared `Frame`**
per `(runtime, service)` pair, mapped read-write into both VSpaces at launch:

```
runtime:  write args into shared[..n]
          Call(endpoint_badged_for_this_handle, tag{ label = SIGN, mr1 = n })
service:  Recv → badge tells it (KeyId, KeyOps); copy-in shared[..n] to a private buffer
          FIRST, then check_access(badge, op), then operate; write result to shared[..m]
          Reply(tag{ label = OK | ACCESS | INVALID, mr1 = m })
runtime:  read shared[..m], return to the guest
```

The service always copies-in before validating (TOCTOU). The shared `Frame` is reachable
only by that one runtime and that one service, so it is no worse than the arguments
already crossing the boundary. Payloads larger than one `Frame` (large file reads) chunk
across multiple `Call`s — the same shape `lantern-filesystem`'s eventual multi-block
chunking will need anyway.

**The trait impls.** `lantern-runtime`'s `KeystoreService` / `FilesystemService` traits
(RFC-0014/RFC-0016) are unchanged as *interfaces*; the Phase 2 `InProcessFilesystem`
stand-in is joined by an `IpcKeystore` / `IpcFilesystem` that hold `(endpoint CPtr, shared
Frame view)` and implement each method as the marshal → `Call` → unmarshal sequence above.
`RuntimeState::with_keystore` / `with_filesystem` take whichever. The host-target demo
keeps working with the in-process stand-in; the `riscv64` build uses the IPC ones.

### Part 3 — Hosting Wasmtime on `riscv64`

`lantern-runtime`'s runtime role builds Wasmtime **`no_std` with the Pulley interpreter**:
`default-features = false`, features `["runtime", "component-model", "pulley",
"custom-virtual-memory", "custom-native-signals", "custom-sync-primitives"]` — **no `std`**,
no `threads`, no `component-model-async`. Cranelift is already absent (RFC-0013). Execution
is the **Pulley portable bytecode interpreter**: no runtime code generation, no executable
memory, no signal-based traps — Wasm traps are interpreter-detected and surface as
`Result::Err`. This is consistent with ADR-0017's "AOT-compiled, no JIT in sensitive
contexts": the artifact is still ahead-of-time compiled, just to portable bytecode rather
than native machine code.

**The compiler role changes to emit Pulley.** `lantern-sdk build` / `precompile_and_sign`
target `pulley64` instead of the host ISA. The `.cwasm` in a `.lpkg` becomes a
**portable** artifact — one package runs on any LanternOS regardless of the machine's ISA,
which the native-code `.cwasm` was not. The compiler role still runs off-device on a dev
machine or build server.

**The custom-platform C symbols** Wasmtime's `sys/custom` layer needs
(`src/runtime/vm/sys/custom/capi.rs`), implemented by `lantern-runtime` against
`lantern-abi`:

| Symbol(s) | LanternOS implementation |
| --- | --- |
| `wasmtime_page_size` | fixed 4 KiB (`RISCV64_PAGE_SIZE`) |
| `wasmtime_mmap_new` / `wasmtime_mmap_remap` / `wasmtime_munmap` / `wasmtime_mprotect` | retype `Untyped` → `Frame`, `FrameInvoke::Map`/`Unmap` into the runtime's VSpace at a reserved virtual range; `mprotect` re-maps with new `PteFlags`. Wasm linear memories become mapped `Frame` regions the runtime owns. |
| `wasmtime_tls_get` / `wasmtime_tls_set` | one static pointer — the runtime is single-threaded per RFC-0006's single-stack model |
| `wasmtime_sync_lock_*` / `wasmtime_sync_rwlock_*` | uncontended: acquire/release are a compare-swap that never spins (single thread) |
| `wasmtime_init_traps` / the trap handler | a no-op registration — Pulley does not raise host traps; a genuine runtime-process fault (a bug in Wasmtime or Pulley) faults the process, blast-radius = its capabilities |
| `wasmtime_fiber_*` | not linked (`component-model-async` off) |
| `wasmtime_memory_image_*` (CoW fast instantiate) | v0: return "unsupported" so Wasmtime falls back to zero-fill; a later optimisation maps a CoW `Frame` |

**Epoch/interruption.** Preempting a runaway confined component uses Wasmtime **fuel**
(deterministic, no timer needed) for v0. Epoch-based interruption driven by a real timer
interrupt is a `lantern-hal` "Next" item and a later refinement.

### Part 4 — Prerequisites, and what becomes more pressing

**Prerequisites (ordinary engineering, no new RFC):**

- **DTB-based physical memory discovery** — `lantern-boot/pmm.rs` seeds one hardcoded
  range today; the launcher spawning several programs, each with its own VSpace and heap
  and Wasm memories, needs a real memory map. This is already `lantern-boot`'s "Next".
- A `talc`-style allocator (or bump + free-list) for `lantern-abi`'s program heap.
- `Frame`-region reservation discipline in the runtime's VSpace so Wasm memories,
  the `.cwasm` image, and `lantern-abi`'s heap do not collide.

**Becomes more pressing (each a `lantern-kernel` "Next"), not blocking v0:**

- **`Revoke` + a capability-derivation tree.** Tearing down a running app today means
  killing its runtime process; *revoking one granted capability* from a still-running app
  (the Phase 3 exit criterion's "revocable capability set") needs real `Revoke`.
- **An idle thread.** A blocking `Recv` with no ready thread currently errors. A
  synchronous request/reply service mesh with no external I/O never reaches that state
  (a `Call` always makes its `Recv`er ready), so v0 is fine — but `lantern-network`'s
  first blocking socket read will need it.
- **Delivering CPU faults to a U-mode handler.** Not needed for Pulley (no guard-page
  traps); required before native-AOT execution (Part 3's Phase 4 note).

### What this RFC does not decide

- **The per-service wire protocols** — the exact `label` values and `mr`/shared-region
  layout for `keystore` (SIGN/ENCRYPT/DECRYPT) vs `store` (READ/WRITE). Each is its own
  small follow-up, the way RFC-0014 fixed the two mapping *shapes* and left each interface
  to be worked separately.
- **The launch binder and consent UX** — `lantern-shell`'s, per RFC-0015. This RFC fixes
  the launcher *mechanism* (spawn programs, wire capabilities from a launch description);
  the policy that produces the launch description is separate.
- **Native AOT execution** (Cranelift → `riscv64` machine code, executable-mapped, guard-page
  bounds checks, CPU-fault delivery) — a Phase 4 performance/assurance item; Pulley is the
  v0 and the "sensitive contexts" default regardless.
- **Multi-hart / SMP** — RFC-0006 fixed single-stack, run-to-completion for now.
- **`lantern-abi`'s exact name, crate layout, and whether it is one crate or two**
  (syscall wrappers vs. program runtime) — an implementation decision.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none *newly defined*, but a designed boundary is
  **enforced for the first time**. RFC-0002/ADR-0004 fixed the TCB boundary and
  RFC-0003 the capability model; Phase 2 ran the services *inside* the runtime's address
  space, so a runtime compromise was a service compromise. After this RFC the keystore's
  key material, the store's blocks, and each app's capability set sit behind real kernel
  address-space and IPC boundaries. The blast-radius test finally has its intended answer.
- **New assets introduced and who can reach them:** the per-`(runtime, service)` **shared
  `Frame`** — reachable only by that one runtime and that one service, holding request/reply
  payloads that already crossed the boundary. The badged **service endpoint capabilities**
  in a runtime's CSpace — each scoped to one object and one op subset, mints by the owning
  service. `lantern-abi` code in every confined program — not privileged, issues syscalls
  only.
- **New adversary capabilities, if any:** a compromised runtime process can now (a)
  corrupt a shared `Frame` mid-request — mitigated by services copying-in before validating
  (TOCTOU); (b) invoke any badged service capability it holds with arbitrary arguments —
  mitigated by each service's own deny-by-default `check_access` on the kernel-delivered
  badge, unchanged from Phase 2; (c) spin, exhaust its heap, or wedge — bounded by its
  scheduling context and fuel, blast-radius = its own capabilities. It **cannot** read
  another app's key, name a file it was not granted, forge a badge, or reach a service it
  holds no endpoint to.
- **Mitigations:** kernel-enforced address-space isolation between services and runtimes;
  badged endpoints (no forgeable `u64`); copy-in-before-validate on every service; fuel
  metering; `lantern-abi` outside the TCB. The still-open `Revoke` gap means a *misbehaving
  but not compromised* app's capabilities can only be withdrawn by tearing the process
  down until the derivation tree lands — recorded, not waived.
- **Net change to attacker surface:** **reduces sharply.** This is the deployment of the
  isolation Phase 2 only demonstrated; a runtime compromise stops being a service
  compromise. The one genuinely-new surface (the shared `Frame`) is the minimal
  request/reply channel and is defended by copy-in.

Cross-reference [`Threat-Model.md`](../../lantern-docs/wiki/Threat-Model.md), and the
`THREAT_MODEL.md` of `lantern-runtime`, `lantern-crypto`, `lantern-filesystem`,
`lantern-capabilities` — none of whose defined boundaries move.

## TCB impact  *(mandatory)*

- **Does this add code to the Trusted Computing Base?** **No.** ADR-0004's TCB is kernel +
  HAL + boot. This RFC's Pulley + shared-memory + badged-endpoint design needs **no new
  kernel object, no new syscall, and no new `lantern-hal` surface**: it composes the
  existing IPC fast path, `CNodeInvoke` (incl. `Mint`/`CopyCross`, RFC-0010), `FrameInvoke`
  (`Map`/`Unmap`, RFC-0008), and badged endpoints (ADR-0008). The in-memory IPC buffer —
  the one kernel addition this could have needed — is deliberately **not** built; shared
  memory replaces it. `lantern-abi`, `IpcKeystore`/`IpcFilesystem`, the Wasmtime
  custom-platform shims, and the extended launcher are all confined user-space code.
- **Does this add a dependency to the TCB?** No. Wasmtime remains a `lantern-runtime`
  (non-TCB) dependency; dropping `std` and adding `pulley` shrinks its surface rather than
  growing it.
- **Effect on TCB size and auditability:** **neutral-to-positive.** No TCB growth. The
  design *increases* auditability of the whole system: key material and user data now sit
  behind kernel-checked boundaries a reviewer can point at, rather than a Rust module
  boundary in a shared address space.

If a later RFC pursues native-AOT execution, *that* RFC carries the TCB impact of
delivering CPU faults to a U-mode handler — this one does not.

## Privacy impact

None directly beyond the security improvement. Moving the keystore and store into their own
processes means a compromised app or runtime can no longer read key material or another
app's file blocks — a privacy gain. The audit log (a separate Phase 3 deliverable,
`lantern-ai-runtime`) is out of scope. Pulley bytecode `.cwasm` artifacts are *more*
privacy-neutral than native ones: identical across machines, leaking nothing about the
build host's ISA or the deploying user's hardware.

## Alternatives considered

- **Native AOT (Cranelift → `riscv64`) instead of Pulley.** Faster execution, but needs
  executable memory mapping, guard-page memory traps, and — critically — a mechanism to
  deliver CPU faults from the kernel to a U-mode trap handler (new kernel/HAL surface, TCB
  impact). Rejected for v0: Pulley needs none of that, ADR-0017 already committed to
  "no JIT in sensitive contexts," and the artifact becomes portable. Native AOT is
  recorded as Phase 4 performance/assurance work with its own RFC.
- **Implement the in-memory IPC buffer (RFC-0005/ADR-0008) and pass payloads through it.**
  Rejected: it is kernel code (TCB growth), file reads/writes want more than any fixed
  buffer size anyway, and shared memory is zero-copy for the large cases. The IPC buffer
  may still land later for other reasons; this RFC does not need it.
- **One merged "services" process** holding the keystore and the store together. Rejected:
  it re-creates the Phase 2 problem one level up (a store bug reaches key material), and
  the store↔keystore call (`Store` uses the keystore's AEAD key) is exactly the kind of
  boundary this RFC exists to make real.
- **A shared runtime process hosting many apps' components.** Rejected for v0: a runtime
  bug then crosses every hosted app. One runtime process per app (or per explicitly-grouped
  app set) keeps ADR-0004's "bounded by *its* capabilities" literal. A pooled runtime is a
  later density optimisation.
- **Keep `&mut KernelState` and just add a "kernel proxy" that marshals it over IPC.**
  Rejected: `&mut KernelState` is a same-address-space privileged handle by construction;
  there is nothing to proxy. The services genuinely have to be rewritten onto syscalls.
- **Wait for multi-hart / a preemptive timer before doing any of this.** Rejected: the
  synchronous request/reply mesh works under RFC-0006's single-stack model with fuel-based
  interruption; SMP and timer preemption are refinements, not blockers.

## Prior art

- **seL4** — badged endpoints delivering the badge to the receiver, the IPC-buffer /
  shared-memory split for payloads, and services as confined threads with their own CSpace
  are all seL4's model; this project already follows it for the kernel (RFC-0002/0005/0006)
  and `Broker` (RFC-0010).
- **The `lantern-boot` broker demo** — `broker-service` / `broker-client` already prove a
  confined service `Recv`ing a request, minting a badged capability, and `Reply`ing with it
  under real U-mode `ecall`s on QEMU. This RFC generalises that from a demo to the keystore
  and store.
- **Wasmtime Pulley + `min-platform` example** (`docs/examples-minimal.md`,
  `examples/min-platform`) — Wasmtime's own supported path for `no_std` embedding behind a
  minimal C API, which this RFC's Part 3 targets directly.
- **Fuchsia component instances** — one component per sandbox, capabilities routed in, the
  runner (ELF or a language runtime) hosting the code; the runtime-process-per-app model
  here is the same shape.
- **CloudABI / Capsicum, WASI-on-seL4 experiments** — capability-oriented POSIX-ish
  runtimes where every resource is a handed capability; the service-endpoint-per-grant
  design matches.

## Unresolved questions

- **Shared-`Frame` sizing and lifecycle** — one fixed-size `Frame` per `(runtime,
  service)` pair, or a small pool, or a per-call transient mapping? Chunking large
  file I/O across `Call`s needs a decided answer.
- **How the launcher's launch description is expressed** before `lantern-shell` exists —
  a static table in the launcher, a tiny on-image config, or the app's own manifest read
  by the launcher directly.
- **`lantern-abi` allocator choice** and whether the program runtime is one crate or
  split from the syscall wrappers.
- **Pulley performance** for the crypto-in-a-loop and file-scan cases RFC-0014 already
  flagged as a latency risk — needs the benchmark ADR-0013's IPC work established the
  methodology for.
- **Fault handling for the runtime process itself** — a Wasmtime/Pulley bug faults the
  process; is "the process dies, blast-radius = its caps" acceptable for v0, or does the
  launcher need a restart/report path?
- Whether Part 1 (services) and Part 3 (Wasmtime port) proceed as one work item or two,
  and in which order — Part 3 has no hard dependency on Part 1 (a `riscv64` runtime can
  first run a component with *no* host imports), so they can overlap.

## Future possibilities

- Native-AOT execution (its own RFC) once CPU-fault delivery exists.
- A pooled/shared runtime process for density once one-per-app is proven.
- `wasmtime_memory_image_*` over a CoW `Frame` for fast instantiation.
- The in-memory IPC buffer, if a future service wants small fixed-size messages without a
  shared region.
- Epoch-based preemption once `lantern-hal` has a timer interrupt.
- The audit log tapping the service-call boundary (every `Call` a runtime makes is a
  natural audit event) — `lantern-ai-runtime`'s concern, enabled here.

## Resulting ADRs

On acceptance, expected to produce **two ADRs**: one fixing the confined-service model and
the badged-endpoint + shared-memory call transport (Parts 1–2), and one fixing the Wasmtime
`no_std` + Pulley hosting strategy and the custom-platform contract (Part 3). Both will
record that this RFC adds nothing to the TCB.
