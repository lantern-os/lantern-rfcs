---
adr: 0018
title: WIT-handle ⇄ capability mapping for lantern-runtime
status: Accepted
date: 2026-08-28
deciders: ["TSC"]
rfc: ../rfcs/0014-wit-handle-capability-mapping.md
supersedes: []
superseded_by: null
---

# ADR-0018: WIT-handle ⇄ capability mapping for lantern-runtime

## Context

[ADR-0017](./0017-wasm-engine-selection-and-aot-strategy.md) fixed Wasmtime as the engine
and committed to "custom, capability-gated" WASI host bindings rather than the ambient
`wasmtime-wasi` crate — but explicitly left *which WIT-typed handle maps to which
LanternOS capability* to a later RFC. That was the last design question between
`lantern-runtime` and a real host-function surface: today's runtime role can load and run
a component only if it imports nothing at all.

Every downstream Phase 2 service is waiting on the answer.
`lantern-capabilities`/`lantern-crypto`/`lantern-filesystem` each document, in their own
`STATUS.md`, that a confined app can only reach `Broker`/`Keystore`/`Store` once this
mapping exists. `lantern-ai-runtime` names the same gap for capability-scoped agents.

The trap this decision avoids: WASI 0.2 was designed against an ambient POSIX host, and
most of its interfaces assume that shape. Deciding in the open which parts LanternOS
reuses and which it deliberately does not — rather than discovering the mismatch inside an
implementation PR — is the kind of trust-boundary call `GOVERNANCE.md` reserves for the
RFC process. [RFC-0014](../rfcs/0014-wit-handle-capability-mapping.md) proposed the mapping
and has been accepted; this ADR is the durable record.

## Decision

**A Wasm component's WIT-typed imports become LanternOS object capabilities via one of two
mapping shapes, chosen per imported interface by its handle structure.**

- **Resource-scoped** — for interfaces with a real per-instance handle (a key, and later a
  file). The host stores a `HostCapability { badge, service }` in a Wasmtime
  `wasmtime::component::ResourceTable` (the generic mechanism, not a dependency on
  `wasmtime-wasi`), and hands the guest an opaque, type-checked, per-instance handle to
  it. Every method call looks up the `HostCapability`, forwards the badge plus arguments
  to the owning service, and relays *that service's own* deny-by-default check —
  translating its rejection into the interface's own `error-code::access`. The mapping
  never invents its own notion of "denied", and never adds a capability check the owning
  service doesn't already perform.
- **Link-scoped** — for interfaces WASI models as ambient free functions with no handle
  (`monotonic-clock`'s `now`/`resolution` take no arguments). There is no per-call object
  to scope, so the grant decision happens once, at instantiation: the host either links
  real functions for the whole interface or refuses to link it, and a component that
  imports a refused interface fails to instantiate. Link-or-refuse is the entire
  enforcement mechanism this shape admits.

**Neither handle type is ever manufactured inside a running component.** Every
`ResourceTable` entry and every linked interface exists because the (separate,
`lantern-sdk`-owned) capability manifest named it before the component started. There is
no in-guest operation that conjures a handle from a bare name or path.

**The first two interfaces `lantern-runtime` implements are:**

- `wasi:clocks/monotonic-clock` (link-scoped) — `now` reads `lantern-hal`'s real
  `monotonic_time_ns()` ([ADR-0016](./0016-monotonic-clock-primitive.md)); `resolution`
  is fixed to that primitive's tick rate. Granted or refused whole.
- `lantern:crypto/keystore` (resource-scoped) — a new, LanternOS-owned WIT interface (no
  keystore exists in the stable WASI 0.2 snapshot). Each `key` handle's `HostCapability`
  names one `(badge, KeyId)` pair the manifest granted; `encrypt`/`decrypt`/`sign`
  forward to `lantern-crypto`'s real `Keystore::encrypt`/`decrypt`/`sign`. A component
  granted `ENCRYPT` but not `DECRYPT` gets a `key` whose `decrypt` method exists (WIT
  resources aren't per-operation-attenuable at the type level) but whose every call
  returns `error-code::access`, forwarded from `Keystore::check_access`.

**`wasi:filesystem` is deliberately not mapped.** Its `descriptor` resource is built
around a POSIX directory hierarchy (`open-at` takes a `path: string`, `readdir` lists
entries) that `lantern-filesystem`'s flat, content-addressed `Store` v0 has no concept of.
Forcing a fit would mean either inventing a path-to-`FileId` layer that reflects no real
LanternOS concept, or exposing `open-at` as a mint-a-new-grant-from-a-guest-string
operation — much closer to ambient authority than a granted capability. The choice
between (a) a small custom `lantern:filesystem` interface shaped like `Store`'s object
model and (b) waiting for `Store` to grow a path layer worth exposing is left to a future,
dedicated RFC.

Rejected alternatives (full reasoning in the RFC): wrapping `wasmtime-wasi`'s ambient
implementation in a post-hoc restriction layer (strictly more surface than bindings that
never had ambient access); force-fitting `wasi:filesystem` now via a path-emulation shim
(ahead-of-phase design); one universal `capability` resource type instead of a distinct
host resource type per interface (collapses the type-level distinction the component model
gives for free); waiting for `lantern-sdk`'s manifest format first (the runtime-side
contract — one badge per resource-scoped grant, one yes/no per link-scoped facility — is
independent of the manifest's eventual syntax).

## Consequences

- **Easier:** `lantern-runtime` now has a concrete host-function surface to build.
  `lantern-crypto`'s `Keystore` becomes reachable from a confined component through a
  reviewed mapping rather than an open question; `lantern-sdk` has a fixed runtime-side
  contract to design its manifest format against.
- **Harder / committed to:** the `lantern:crypto/keystore` WIT interface is now
  LanternOS-owned surface that has to be versioned and maintained (host-defined and
  reviewed here, not guest-supplied — no TCB impact). Every resource-scoped call is
  committed to paying a service round trip once the owning services are real confined
  processes; whether that is fast enough for hot-loop crypto is unbenchmarked and flagged
  as a real risk.
- **Trust boundary:** none moved. The mapping lives entirely inside `lantern-runtime`'s
  confined runtime-role process (ADR-0004's TCB boundary — kernel + HAL + boot — is
  unchanged). `ResourceTable` is already part of the feature set ADR-0017 fixed; no new
  dependency.
- **Still open (tracked in `lantern-runtime/STATUS.md`, none resolved here):**
  `wasi:filesystem`'s fate (its own future RFC); the `lantern-sdk` manifest file format;
  how the owning services themselves become IPC-reachable confined processes;
  link-scoped (interface-time) revocation, which has no live mechanism; clock-resolution
  coarsening as an R3 anti-fingerprinting mitigation; and resource-scoped call latency
  benchmarking.
