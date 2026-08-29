---
rfc: 0014
title: WIT-handle ⇄ capability mapping for lantern-runtime
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["runtime", "capabilities"]
domains: ["runtime", "capabilities", "crypto", "filesystem", "sdk"]
created: 2026-08-27
updated: 2026-08-28
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-08-28.** The two mapping shapes, the two worked interfaces, and the
> `wasi:filesystem` deferral are fixed by
> [ADR-0018](../adr/0018-wit-handle-capability-mapping.md). First implementation landed in
> `lantern-runtime` the same round — see `lantern-runtime/STATUS.md`.

# RFC-0014: WIT-handle ⇄ capability mapping for lantern-runtime

## Summary

This RFC fixes how a Wasm component's WIT-typed imports become LanternOS object
capabilities, the piece [RFC-0013](./0013-wasm-engine-selection-and-aot-strategy.md)
explicitly deferred. It distinguishes **two mapping shapes**, because WASI 0.2's own
interfaces aren't uniform: a **resource-scoped** shape for interfaces with a real
per-instance handle (a file descriptor, a key) — the host stores a badge alongside the
Wasmtime `Resource<T>` in a `ResourceTable`, and every method call forwards to the owning
service, live-checked, deny-by-default, exactly once per call — and a **link-scoped**
shape for interfaces WASI models as ambient free functions with no handle at all (the
clock, randomness) — access is granted or refused once, at instantiation, by whether the
host links real functions for that whole interface or refuses to link it. Neither handle
type is ever manufactured inside a running component; every one is populated once, at
instantiation, from capabilities already granted by the (separate, `lantern-sdk`-owned)
capability manifest — there is no in-guest operation that conjures a handle from a bare
name or path.

Two concrete interfaces are worked through end to end: `wasi:clocks/monotonic-clock`
(link-scoped, wired directly to `lantern-hal`'s real `monotonic_time_ns()`) and a new,
LanternOS-specific `lantern:crypto/keystore` interface (resource-scoped, wrapping
`lantern-crypto`'s `Keystore` badges) — chosen over the standard `wasi:filesystem`
specifically because that interface's path/directory shape does not fit
`lantern-filesystem`'s actual content-addressed object model; that mismatch is surfaced
as an explicit open question for a future RFC rather than papered over.

## Motivation

`lantern-runtime/STATUS.md` names this as the first item blocking any real WASI host
binding: today's runtime role can load and run a component with no host imports at all.
It is also the item every downstream Phase 2 service is waiting on —
`lantern-capabilities`/`lantern-crypto`/`lantern-filesystem` each document, in their own
`STATUS.md`, that turning their prototype code (`Broker`/`Keystore`/`Store`) into
something a real confined app can actually reach needs exactly this mapping to exist.
`lantern-ai-runtime/STATUS.md` names the same gap as its own blocker for capability-scoped
agents ([Principle 6](../../lantern-docs/wiki/Principles.md), AI-native — "an agent never
receives unrestricted access; it receives explicit capabilities").

This is also where [Principle 1](../../lantern-docs/wiki/Principles.md) (security by
architecture) meets Wasm's own sandbox model most directly, and where it is easiest to
get wrong quietly: WASI 0.2 was designed to be backed by an ambient POSIX host, and most
of its interfaces assume that shape. Deciding *now*, in an RFC, which parts of that shape
LanternOS reuses as-is and which parts it deliberately does not (rather than discovering
the mismatch mid-implementation) is exactly the kind of trust-boundary decision
`GOVERNANCE.md` reserves for the RFC process, not an implementation PR.

## Guide-level explanation

A LanternOS app or agent is a Wasm component with a WIT `world` describing its imports.
Before it runs, its capability manifest (a `lantern-sdk` artifact this RFC does not
design — see "What this RFC does not decide") has already declared, for each imported
interface, either a specific object it's been granted access to (a particular key, a
particular file) or a coarse yes/no for the whole facility (clock access, at all).

At instantiation, `lantern-runtime` walks that manifest and does one of two things per
imported interface:

- **Resource-scoped** (`lantern:crypto/keystore`, and — once the shape question below is
  resolved — a future filesystem interface): for each granted object, it puts one entry
  in Wasmtime's `ResourceTable` holding the LanternOS badge for that object, and hands the
  component a handle to it. The component can call methods on that handle; every method
  call forwards to the real owning service (`Keystore`, eventually `Store`) over IPC,
  which checks the badge itself, the same deny-by-default check `Keystore::check_access`
  already performs for a native caller. A handle for a key the manifest didn't grant
  simply never exists in the component's table — there is nothing to guess or forge.
- **Link-scoped** (`wasi:clocks/monotonic-clock`): there is no per-call object to scope,
  because the WIT interface itself has none (`now()`/`resolution()` take no arguments).
  So the grant decision happens once, at instantiation: if the manifest grants clock
  access, the host links real functions that read `lantern_hal::monotonic_time_ns()`; if
  it doesn't, the host refuses to link the interface at all, and a component that
  imports it anyway fails to instantiate — the same "no ambient host access" ADR-0003
  already committed to, just enforced at the only point this shape of interface actually
  admits enforcement.

## Reference-level explanation

### The host-side capability record

Every resource-scoped WIT resource type LanternOS exposes is backed, on the host side, by
one Rust type stored in a Wasmtime `wasmtime::component::ResourceTable` (the same generic
table `wasmtime-wasi`'s own implementation uses internally — reused here as a mechanism,
not by depending on that crate, per RFC-0013's "custom, not `wasmtime-wasi`" decision):

```rust
struct HostCapability {
    /// The badge this handle is scoped to — minted by whichever service's `Broker`
    /// owns the underlying object (RFC-0010). Never a raw kernel `CPtr`: a component
    /// only ever holds what a service already narrowed for it, one layer removed from
    /// the kernel capability itself, same layering `Keystore`/`Store` already use.
    badge: u64,
    /// Which service endpoint forwards calls on this handle — implicit from the WIT
    /// interface/resource type in practice (a `keystore::key` handle always forwards
    /// to the crypto service), kept explicit here for clarity.
    service: ServiceEndpoint,
}
```

A guest never sees `badge`/`service` — Wasmtime's own component-model ABI already
represents the handle as an opaque, per-instance, type-checked `i32` the guest cannot
forge into pointing at a different table entry (`ResourceTable::get`/`get_mut` reject a
mismatched type at the Rust level; the component-model ABI itself rejects a
resource handle from the wrong resource type or the wrong instance). This RFC adds
nothing to that guarantee; it only fixes what the host stores behind it.

Every host function implementing a resource-scoped method — registered via
`LinkerInstance::resource`/`func_wrap` on the `wasmtime::component::Linker` — looks up the
`HostCapability` for its handle argument, forwards `badge` plus the call's arguments to
`service` over IPC, and translates that service's own `KeystoreError`/equivalent into the
WIT interface's own error type (`error-code::access` for a denied/revoked/wrong-object
badge, matching `wasi:filesystem`'s and any future LanternOS interface's own POSIX-`EACCES`-
shaped variant) — the mapping never invents its own notion of "denied," it only relays
the owning service's.

### Worked example 1 (link-scoped): `wasi:clocks/monotonic-clock`

The real WASI 0.2 interface (`wasi:clocks@0.2.12`) is two free functions with no resource
argument at all:

```wit
interface monotonic-clock {
  type instant = u64;
  type duration = u64;
  now: func() -> instant;
  resolution: func() -> duration;
}
```

If the manifest grants this component coarse "monotonic-clock" access, `lantern-runtime`
links `now` directly to `lantern_hal::monotonic_time_ns()` (RFC-0012/ADR-0016 — already
real) and a fixed `resolution` matching that primitive's own tick rate; if not, the
import is left unlinked and instantiation fails for any component that declared it. No
badge, no per-call check, no service round trip — this is the entire enforcement
mechanism this shape of interface gets, which is also why it is the right *first*
interface to wire up: the mechanism is trivial to get right precisely because there is
nothing granular to get wrong.

### Worked example 2 (resource-scoped): `lantern:crypto/keystore` (new, not WASI)

No interface in the stable WASI 0.2 snapshot models a keystore (`wasi-crypto` is a
separate, non-0.2, still-experimental proposal) — so this is a small, new, LanternOS-owned
WIT interface, not a reused standard one:

```wit
package lantern:crypto@0.1.0;

interface keystore {
  resource key {
    encrypt: func(nonce: list<u8>, aad: list<u8>, plaintext: list<u8>) -> result<tuple<list<u8>, list<u8>>, error-code>;
    decrypt: func(nonce: list<u8>, aad: list<u8>, ciphertext: list<u8>, tag: list<u8>) -> result<list<u8>, error-code>;
    sign: func(message: list<u8>) -> result<list<u8>, error-code>;
  }
  enum error-code { access, invalid }
}
```

Each `key` resource's `HostCapability` names one `(badge, KeyId)` pair the manifest
granted. `encrypt`/`decrypt`/`sign` forward directly to the real
`Keystore::encrypt(badge, key, ..)`/`decrypt(badge, key, ..)`/`sign(badge, key, ..)`
(`lantern-crypto/src/lib.rs`) — no new semantics invented here, only a WIT-shaped face on
an already-proven mechanism. A component granted `ENCRYPT` but not `DECRYPT` for a given
key gets a `key` resource whose `decrypt` method exists (WIT resources aren't
per-operation-attenuable at the type level) but whose every call fails with
`error-code::access`, forwarded from `Keystore::check_access`'s own real rejection — the
mapping does not add a capability check of its own to enforce this narrower case; it
relies entirely on the service's.

### Deferred: `wasi:filesystem`

The standard interface's central resource, `descriptor`, is built around a POSIX
directory hierarchy — `open-at` takes a `path: string` and returns a *new* `descriptor`
scoped to that path, `readdir` lists directory entries, and rights are POSIX-flag-shaped
(`descriptor-flags`). `lantern-filesystem`'s `Store` v0 has none of this: it is a flat,
content-addressed object store keyed by `FileId`, with no path or directory concept at
all (`lantern-filesystem/STATUS.md`'s own "Next" list — chunking, per-object keys, version
history — never mentions one either). Wiring the real `wasi:filesystem` interface onto
`Store` today would mean either quietly inventing a path-to-`FileId` translation layer
that doesn't reflect any real LanternOS concept yet, or exposing `open-at` as a
capability-checked *mint of a brand-new grant* keyed by a string the component supplies —
which is much closer to ambient authority (any string names an equally-reachable object)
than a granted capability, exactly what ADR-0003 exists to rule out.

This RFC does not resolve that tension, and deliberately does not force a decision by
picking one side under the cover of a WIT-mapping RFC. It surfaces two real options for a
future, dedicated RFC: (a) a small custom `lantern:filesystem` interface shaped directly
like `Store`'s actual object model (open/read/write by `FileId`, no paths), mirroring
this RFC's `keystore` treatment, or (b) waiting to adopt real `wasi:filesystem` until
`Store` itself grows a path/directory layer worth exposing faithfully. Either is
defensible; this RFC just refuses to guess.

### Handle population: only at instantiation, only from the manifest

`lantern-runtime` never creates a resource-table entry or links an interface in response
to anything a running component does — every entry exists because the (not-yet-designed)
`lantern-sdk` capability manifest named it before the component started, mirroring the
narrowing-waterfall root task ([`Architecture`](../../lantern-docs/wiki/Architecture.md))
one layer up. This RFC fixes the runtime-side contract the manifest must satisfy (one
badge per resource-scoped grant; one yes/no per link-scoped facility) — not the manifest's
own file format, syntax, or how a developer authors one, which remains `lantern-sdk`'s job
(`lantern-sdk/STATUS.md`'s own "design the capability manifest format").

### What this RFC does not decide

- The `lantern-sdk` capability manifest's file format.
- `wasi:filesystem`'s fate (see "Deferred" above) — a future RFC.
- How the *owning services themselves* (`Keystore`, eventually `Store`) become deployable
  confined processes reachable over IPC at all — a separate, larger, already-tracked gap
  (`lantern-capabilities`/`lantern-crypto`/`lantern-filesystem` `STATUS.md`'s own "needs a
  genuine confined runtime capable of hosting real Rust service code," distinct from
  hosting *Wasm* components). This RFC assumes that IPC endpoint exists and fixes only
  what a Wasm guest's handle to it looks like.
- Resource-scoped call latency / whether every crypto or (eventual) filesystem operation
  paying a cross-process IPC round trip is acceptable — a real question, not benchmarked
  here (see "Unresolved questions").
- Sockets, environment variables, and any coarsening of the clock's resolution as an
  anti-fingerprinting mitigation (R3) — all future work.
- Wiring RFC-0011's sealed-capability tokens through this mapping for cross-machine
  sharing — `lantern-crypto/STATUS.md` already names a real consumer as the trigger for
  that, not this RFC.

## Threat model impact *(mandatory)*

- **Trust boundaries affected:** none moved. The runtime role remains confined user space
  (ADR-0004); this RFC fixes how it forwards to already-trust-boundaried services, not
  where those boundaries sit.
- **New assets introduced and who can reach them:** the `ResourceTable` entries
  themselves — a `HostCapability` is as sensitive as the badge it carries. It is reachable
  only by the one component instance Wasmtime scoped that table to (component-model ABI
  guarantee, not new to this RFC) and by the runtime-role host code itself.
- **New adversary capabilities, if any:** none intended. A component can only ever reach
  a `HostCapability` the manifest put there before it started; there is no operation that
  manufactures one from guest-controlled data (R1, `THREAT_MODEL.md`).
- **Mitigations:** deny-by-default forwarding (a resource-scoped call that fails the
  owning service's own check surfaces as `error-code::access`, never partial success);
  link-scoped interfaces are refused entirely rather than linked-but-gated, so there is no
  "linked but secretly inert" state to get wrong; per-resource-type host records (not one
  universal "capability" blob) so a `keystore::key` handle can never be type-confused
  with a future, differently-scoped resource (R5).
- **Net change to attacker surface:** this is the surface ADR-0003 already accepted when
  it committed to capability-backed WASI; this RFC's job is making the mapping concrete
  and reviewed rather than leaving it to be decided silently inside a host-binding
  implementation PR.

## TCB impact *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No — the mapping lives entirely
  inside `lantern-runtime`'s confined runtime-role process (ADR-0004's TCB boundary:
  kernel + HAL + boot only), unchanged by this RFC.
- **Does this add a dependency to the TCB?** No new dependency at all — `ResourceTable`
  is already part of the `wasmtime`/`component-model` feature set RFC-0013 fixed.
- **Effect on TCB size and auditability:** neutral. The `lantern:crypto/keystore` WIT
  interface is new surface, but it is host-defined and reviewed here, not guest-supplied;
  it does not touch the TCB in either direction.

## Privacy impact

The clock worked example is exactly the surface `THREAT_MODEL.md` R3 already flags
(fingerprinting via precise timing). This RFC wires `monotonic_time_ns()` through
unmodified — it does not add resolution-coarsening or any other mitigation, leaving that
an explicit open item rather than an implied "handled." No other privacy-relevant surface
is touched (`lantern:crypto/keystore` operates on caller-supplied key material and
messages only, nothing about the component itself).

## Alternatives considered

- **Reuse `wasmtime-wasi`'s ambient implementation, then wrap it in a post-hoc
  restriction layer.** Rejected: RFC-0013 already rejected this shape outright (ambient
  preopens/sockets, the opposite of ADR-0003); a wrapper around an ambient
  implementation is strictly more attack surface than host functions that never had
  ambient access to begin with.
- **Force-fit `wasi:filesystem` onto `Store` now, via a path-emulation shim.** Rejected
  for now: inventing a path model `Store` itself doesn't have yet is exactly the kind of
  ahead-of-phase design `CLAUDE.md` warns against, and risks conflating this mapping's
  scope with the long-term Contexts-over-files vision prematurely. Deferred to its own
  RFC instead of guessed here.
- **One universal `capability` resource type for every facility**, instead of a distinct
  host resource type per interface (`keystore::key` vs. some future `filesystem::file`).
  Rejected: this collapses exactly the type-level distinction the component model gives
  for free (R5) into a single blob a host function would have to runtime-check the
  "real" type of — strictly worse than what WIT already offers.
- **Wait for `lantern-sdk`'s manifest format before fixing this mapping.** Rejected, same
  reasoning RFC-0013 used for not waiting on `lantern-sdk`: the runtime-side contract
  (badge-per-resource, yes/no-per-interface) is independent of the manifest's eventual
  syntax, and `lantern-sdk` itself is blocked on stable WIT interfaces existing first
  (`lantern-sdk/STATUS.md`).

## Prior art

**`wasmtime-wasi`**'s own implementation, for the `ResourceTable`-per-handle *mechanism*
(not its ambient host bindings, which RFC-0013 already rejected reusing). **Fermyon
Spin** and **Fastly Compute**, again, for building confined multi-tenant capability-scoped
WASI hosts on top of Wasmtime by replacing its ambient bindings with scoped ones — the
same precedent RFC-0013 cited, now applied to the actual handle-mapping mechanics.
**seL4**'s badge model, already this project's own precedent via `Broker`, for the
deny-by-default-per-call discipline every resource-scoped host function here follows.

## Unresolved questions

- Interface-scoped (link-time) revocation has no live mechanism at all — once a
  component is instantiated with clock access linked, there is no way to revoke it short
  of tearing the component down. Left open, not solved here.
- `wasi:filesystem`'s fate (custom interface vs. wait for `Store`'s own path layer) —
  explicitly deferred to a future RFC.
- Whether per-call IPC forwarding for resource-scoped operations is fast enough in
  practice — unbenchmarked, and a real risk for anything latency-sensitive (crypto
  operations in a hot loop, say). A future benchmark, mirroring ADR-0013's IPC latency
  work, is the natural way to answer this.
- Clock resolution coarsening (or any other anti-fingerprinting mitigation for R3) —
  named, not designed.
- Whether resource-scoped handles ever need `ResourceTable::push_child`-style parent/child
  relationships (e.g. a derived, more-attenuated key handle) — no concrete need yet;
  `Keystore`'s own attenuation story today is sealed capabilities (RFC-0011), not a
  live-handle derivation, so this may never be needed at this layer.

## Future possibilities

- The deferred `wasi:filesystem`-or-custom-interface RFC.
- A `lantern:crypto/keystore` implementation RFC/PR, once this mapping RFC is accepted —
  the natural next "accept + implement" step, mirroring RFC-0013's own two-step history.
- Socket/network capability mapping, once `lantern-network` has a real service to map
  onto (currently Phase 0, design-only).
- A resource-accounting RFC tying per-resource-scoped-call IPC cost into Wasmtime's
  fuel/epoch mechanism (RFC-0013's identified attachment point).
- Sealed-capability (RFC-0011) unsealing feeding a resource-scoped grant, once a real
  cross-machine-sharing consumer exists.

## Resulting ADRs

On acceptance, an ADR will fix the two mapping shapes (resource-scoped via
`HostCapability`-in-`ResourceTable`, link-scoped via link-or-refuse-at-instantiation), the
`wasi:clocks/monotonic-clock` and `lantern:crypto/keystore` worked examples as the first
two interfaces `lantern-runtime` implements, and the explicit deferral of
`wasi:filesystem` to its own future RFC.
