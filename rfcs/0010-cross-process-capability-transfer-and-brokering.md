---
rfc: 0010
title: Cross-process capability transfer and the service-layer capability-brokering API
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["kernel", "capabilities"]
domains: ["kernel", "capabilities", "runtime"]
created: 2026-08-19
updated: 2026-08-19
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0010: Cross-process capability transfer and the service-layer capability-brokering API

## Summary

This RFC proposes two tightly coupled pieces of work that together unblock Phase 2's
capability-brokering goal (RFC-0009): (1) a minimal kernel IPC extension letting one
capability be attached to a `Send`/`Call` and land in the receiver's own CSpace —
implementing the `extra_caps` field [ADR-0008](../adr/0008-kernel-syscall-ipc-abi.md)'s
wire format already reserves but Phase 1 left entirely unimplemented, gated on the
`Rights::GRANT` bit [RFC-0003](./0003-capability-model.md) already specifies but nothing
currently enforces; and (2) the service-layer `mint`/`grant`/`revoke` brokering API that
[`lantern-capabilities`](../../lantern-capabilities) builds on top of it. Both
`lantern-capabilities/STATUS.md` and `lantern-runtime/STATUS.md` name this pairing as
their blocking Phase 2 "Next" item.

## Motivation

Phase 2 opened (RFC-0009/ADR-0014) explicitly so `lantern-capabilities` and
`lantern-runtime` could move from design-only to prototype capability-brokering code. But
`grant` is not actually real across processes today:

- `CNodeInvoke`'s `Copy`/`Move` only operate on slots *within a single CNode*
  (`lantern-kernel/STATUS.md`'s "Known Phase 1 gaps"). There is no capability-checked way
  for one process to hand a capability to another.
- The one place Phase 1 needed cross-process capability placement —
  `lantern-boot/loader.rs` giving each loaded program the one shared endpoint it needs —
  works around this with a direct, privileged pool write, not a real invocation. That is
  acceptable for a root task in a throwaway Phase 1 prototype, but it is not a mechanism
  any confined, unprivileged Phase 2 service can use — least of all the capability broker
  itself, whose entire job is granting capabilities to other confined services.
- RFC-0003 already specifies `grant(cap, endpoint)` as a first-class operation, and
  `Rights::GRANT` already exists in `lantern-kernel/src/cap.rs` as a rights bit ("permission
  to grant this capability... to another component over IPC") — but grepping the tree
  shows nothing anywhere checks it. It has been a dormant, unenforced bit since RFC-0003.

Without a real kernel-enforced transfer primitive, "service-layer capability brokering"
has no honest way to hand a client a capability the client can actually invoke directly at
kernel speed. The broker would be reduced to a permanent invocation proxy for every single
operation on every object it mediates — a materially different, slower, less
capability-native architecture than RFC-0003 envisions, and a structural single point of
failure for everything it mediates.

This serves **security by architecture** (closes a rights bit that looks enforced but
isn't — exactly the kind of latent footgun the project's own principles ask to be caught
before it matters) and **least privilege** (a real, kernel-checked `grant` is what lets a
broker hand out narrowly attenuated capabilities instead of ambient access to itself).

## Guide-level explanation

Consider a Phase 2 service — say the eventual filesystem-v0 broker — that holds an
internal capability to some object and wants to hand a client read-only access to it. With
this RFC:

1. The broker **mints** an attenuated copy of its own capability into a scratch CSpace
   slot: `rights = READ`, a fresh badge identifying this grant. `CNodeInvoke::Mint` already
   does this for real today — nothing new here.
2. The broker replies to the client's request (or sends, if unsolicited) with
   `tag.extra_caps = 1`, naming that scratch slot as the capability to transfer.
3. The kernel, as an atomic part of message delivery, copies that capability into a
   destination slot the client pre-registered when it issued `Recv`.
4. The client now holds a real, kernel-backed capability. For a raw kernel object this is
   already everything it needs. For a broker-defined object (the common Phase 2 case), what
   the client actually receives is a **badged Endpoint capability to the broker** — per
   [ADR-0006](../adr/0006-three-layer-capability-structure.md)'s existing "service
   capability = badged endpoint" pattern — so future invocations dispatch through the
   broker keyed by that badge, but the *grant itself* was a single, real, capability-gated
   kernel operation rather than a standing, ambient trust relationship.

Revocation stays broker-local: since kernel-level `Revoke` remains refused pending a
capability-derivation tree (unchanged Phase 1 gap, not addressed here), the broker tracks
a `badge → revoked` flag itself and checks it on every dispatch. The client's Endpoint
capability keeps working at the kernel level — the kernel does not retract it — but every
call through it fails once the broker marks that badge revoked. This is sufficient for
RFC-0003's `revoke` operation for every object Phase 2 actually introduces, because every
one of them (filesystem v0, the crypto keystore) is broker-mediated, not a bare kernel
object handed out directly.

## Reference-level explanation

### Kernel: single-capability transfer

- `MessageTag.extra_caps` becomes meaningful for the value `1` only. `>1` continues to be
  rejected with `TruncatedMessage`, exactly as all nonzero values are today — multi-capability
  transfer is deferred (see "Future possibilities"), the same "narrowest slice that closes
  the concrete gap" precedent [RFC-0008](./0008-vspace-frame-capabilities-and-elf-loader.md)
  set by shipping `Mega`-only frames.
- Wire convention, extending `abi.rs`'s existing `mr0`-is-CPtr / packed-argument precedent:
  when `tag.extra_caps == 1`, `mr1` is the sender's CPtr naming the capability to transfer,
  in the sender's own CSpace. The message payload shrinks to `mr2`/`mr3` for that send —
  the same kind of word-count trade `cnode::mint`'s packed `(badge << 8) | rights` argument
  already makes elsewhere in this crate.
- Transfer is a real **copy** of whatever capability sits in the sender's named slot — same
  rights, same badge — never an implicit mint. A sender that wants to attenuate first
  mints into a scratch slot, then transfers that slot. This composes the two already-real
  primitives instead of re-implementing attenuation logic a second time.
- **`Rights::GRANT` is now enforced.** The transfer fails with `IllegalOperation` unless the
  source capability's rights include `GRANT`. This is the first real consumer anywhere in
  the tree of a rights bit that has existed, unchecked, since RFC-0003.
- Receive side: `Recv` gains an optional destination-CPtr argument — a slot in the
  receiver's own CSpace, carried in a currently-unused `Recv` argument register. If the
  delivered message declares `extra_caps == 1` and the receiver registered no destination
  slot, or the slot is already occupied, the rendezvous fails cleanly with
  `IllegalOperation` rather than silently dropping the capability or completing a partial
  transfer — the same "validate everything before doing anything" discipline
  `admin::untyped_retype` already follows. `Call`'s reply path gets symmetric treatment,
  since a broker's grant is naturally a `Call`/`Reply` (client calls the broker, the broker
  replies with the minted capability attached), not a bare `Send`.
- No new kernel object type and no new syscall number: this extends `Send`/`Call`/`Recv`'s
  existing argument handling, in keeping with RFC-0002's minimal-kernel mandate.

### Service layer: the `lantern-capabilities` brokering API

- A broker is an ordinary confined user-space service — not a kernel concept — reachable
  via one or more badged Endpoint capabilities, per ADR-0006's existing layer-2 definition.
- `mint` + `grant` compose exactly as walked through above: `CNodeInvoke::Mint` into a
  scratch slot, then `Send`/`Call` with `extra_caps = 1` naming that slot.
- `revoke` is broker-local: a `badge → { object, rights, revoked: bool }` table the broker
  consults on every dispatch. This RFC fixes that as the sanctioned Phase 2 revoke
  mechanism for broker-mediated objects — the actual answer to
  `lantern-capabilities/STATUS.md`'s "Phase 2 prototype: brokering + mint/grant/revoke over
  kernel endpoints," not a stopgap.
- Badging convention: each `grant` mints a fresh, broker-chosen badge (a counter scoped per
  broker instance) rather than reusing any identity the client asserts about itself —
  preserves "designation = authority, no self-asserted identity" (RFC-0003) for the badge
  itself, the same way the kernel's own Endpoint badging already works.

## Threat model impact

- **Trust boundaries affected:** none newly defined — this operates entirely within
  RFC-0003's already-accepted capability model. It is, however, the first time `grant`
  becomes a real, capability-gated, cross-process *kernel* operation, rather than either
  same-CNode-only `Copy`/`Move` or a root task's privileged direct pool write.
- **New assets introduced and who can reach them:** the in-flight transferred capability
  during rendezvous (reachable only by the two parties to that specific `Send`/`Recv` or
  `Call`/`Reply`); a broker's `badge → object/rights/revoked` table, which becomes a
  genuinely security-critical asset — corrupting it is equivalent to corrupting the
  capability system for everything that broker mediates.
- **New adversary capabilities, if any:** none intended. Transfer requires `Rights::GRANT`
  on the source capability, kernel-enforced, so a component still cannot grant what it
  doesn't hold, and — new as of this RFC — cannot grant even what it holds unless the
  granting right was itself explicitly granted. Because no transfer primitive existed
  before this RFC, nothing previously depended on `Rights::GRANT` being checked, so nothing
  regresses; leaving that bit unenforced through the start of real cross-process granting
  is exactly the latent risk this RFC closes.
- **Mitigations:** kernel-enforced `Rights::GRANT` check; explicit destination-slot
  pre-registration, so no capability is ever delivered to a slot the receiver did not
  designate; copy-not-move-not-implicit-mint transfer semantics, keeping attenuation an
  explicit, separate, auditable step.
- **Net change to attacker surface:** increases, deliberately — real cross-process
  authority delegation becomes possible for the first time, which is precisely Phase 2's
  purpose (RFC-0009). The increase is bounded by `GRANT`-gating and mandatory
  destination-slot registration, not left ambient.

## TCB impact

- **Does this add code to the Trusted Computing Base?** Yes, in `lantern-kernel`: the
  `extra_caps == 1` transfer path in `ipc.rs`, the new `Rights::GRANT` check, and a
  destination-slot argument to `Recv`/`Call`'s reply path. It has to live there — it is a
  kernel-mediated, capability-checked authority transfer, exactly like every other
  capability operation the kernel already performs.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** modest, bounded growth in the same style as the
  existing `Send`/`Recv`/`Call` fast path — no new syscall number, no new object type, no
  external dependency. `lantern-capabilities`' broker logic itself is confined user space,
  not in the TCB, per RFC-0002/ADR-0004's already-fixed boundary; this RFC does not touch
  that boundary.

## Privacy impact

Minimal, directly. Broker badge tables are internal bookkeeping, not user-facing telemetry
or an identity surface. Making `grant` a real, single, kernel-mediated event (rather than
an ambient or privileged shortcut) makes future provenance/audit work — the AI-runtime
audit log named in Phase 3's exit criteria — simpler to build correctly later; this RFC
does not build that log itself.

## Alternatives considered

- **Multi-capability transfer now, using `extra_caps`' full 4-bit range.** Rejected for the
  same reason RFC-0008 shipped `Mega`-only frames: the concrete Phase 2 need (a broker
  hands out one capability per grant call) doesn't require it, and supporting more than one
  CPtr per message without further shrinking the payload needs a real in-memory IPC buffer
  — separable, bigger work, deferred.
- **Broker as a permanent invocation proxy** (the client never receives a direct kernel
  capability; every operation is re-dispatched through the broker forever). Rejected: this
  defeats the point of a capability-native design — RFC-0003 wants direct, unmediated
  invocation once a capability is held — and turns the broker into a standing bottleneck
  and single point of failure for every object it mediates.
- **Implement kernel-level `Revoke`'s derivation tree now**, instead of broker-local revoke
  tracking. Rejected: derivation-tree revocation is real, cross-cutting kernel work
  independent of this RFC's scope (already tracked as an open Phase 1 gap in
  `lantern-kernel/STATUS.md`); broker-local revoke is sufficient for every object Phase 2
  actually introduces (all broker-mediated) and does not need to wait on it.
- **Do nothing — keep capability placement root-task-only and privileged.** Rejected: this
  makes it structurally impossible for `lantern-capabilities` to be the thing that grants
  capabilities, since it is not the (privileged) root task. Directly blocks Phase 2's
  stated goal.

## Prior art

seL4's IPC capability transfer via the IPC buffer's `extraCaps` mechanism — this RFC is a
deliberately narrower, register-only, single-capability slice of the same idea. Fuchsia's
handle transfer over channels (handles move without implicit attenuation — the same
"mint explicitly, then transfer" discipline this RFC adopts). EROS/KeyKOS
capability-passing IPC.

## Unresolved questions

- Exact register layout for `Call`'s reply-path transfer and `Recv`'s destination-slot
  argument — left to implementation, to be documented in `abi.rs` alongside the existing
  `mr0` conventions, the same way RFC-0005/RFC-0008's ABI details were.
- Whether a badge counter scoped per broker *instance* is sufficient, or whether it needs
  to persist across a broker restart. Out of scope while Phase 2 has no service-supervision
  story yet.
- Multi-capability transfer and a real per-thread IPC buffer — deferred, see "Alternatives
  considered."

## Future possibilities

- Multi-capability transfer, once a grant pattern needs to hand over several capabilities
  atomically (e.g., "here are three file capabilities at once").
- A real per-thread IPC buffer, generalising message delivery beyond register-only —
  mentioned as future work when ADR-0008 first fixed the tag format.
- Sealed-capability integration (RFC-0003's third layer) once `lantern-crypto`'s keystore
  exists: sealing a broker-granted capability for cross-machine delegation.

## Resulting ADRs

On acceptance, an ADR will fix (a) the `extra_caps = 1` kernel transfer mechanism and
`Rights::GRANT` enforcement as a durable decision, and (b) the broker `mint`/`grant`/
`revoke` pattern as `lantern-capabilities`' canonical Phase 2 design.
