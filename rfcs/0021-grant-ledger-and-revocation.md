---
rfc: 0021
title: The grant ledger and revocation v0 — a confined binder that can list and withdraw what it granted
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "capabilities", "abi"]
domains: ["shell", "abi", "capabilities", "crypto", "filesystem", "runtime", "boot"]
created: 2026-11-02
updated: 2026-11-02
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-02.** Fixed by [ADR-0026](../adr/0026-grant-ledger-and-revocation.md). The
> unresolved questions were settled on acceptance: a scripted `grant-control` holds the control
> capability for v0, a revoked role stays unavailable until the next boot, and revocation must outlive
> a reboot before the exit criterion counts as met.

# RFC-0021: The grant ledger and revocation v0 — a confined binder that can list and withdraw what it granted

## Summary

[RFC-0020](./0020-launch-binder.md)'s binder decides each grant, then forgets everything except the
badged capability. This RFC makes it remember. The binder keeps a **ledger** of what it granted, and
a holder of a new **control capability** can list that ledger and revoke an entry. Revoking sends a
new `REVOKE` to the owning service, whose existing deny-by-default check then refuses that badge with
`ACCESS` on every later call.

It adds no kernel feature and nothing to the TCB. It also fixes a gap it would otherwise depend on:
`lantern_abi::frame::Channel` accepts a message from any caller in the middle of a chunked exchange.

**This is the mechanism for "visible, revocable", not the whole of it.** No person sees anything yet,
because there is no trusted-path surface, and a revocation lasts only until the next boot, because
there is no persistent policy store. The v0 control holder is a scripted test program.

## Motivation

The Phase 3 exit criterion asks for "a visible, revocable capability set". Today:

1. **Nothing can be listed.** The binder holds each badged capability in a slot, with no record of
   what it stands for.
2. **Nothing can be revoked.** `Keystore::revoke_access` and `Store::revoke_access` exist and are
   tested, and every operation already consults them through `check_access`. No confined path reaches
   them, and the binder couldn't name the target anyway: both services discard the badge
   (`.and_then(|_badge| ...)`) when they answer `GRANT`.
3. **A chunked exchange can swallow another caller's message.** `Channel::recv_request` and
   `Channel::reply` both block in an inner `sys::recv` between chunks, and neither looks at who
   answered. `reply`'s own comment says the message's "content is irrelevant, only its arrival
   matters". A `REVOKE` arriving at that moment would be consumed as a chunk: the binder would get an
   unrelated reply, and the client would still hold its authority. A client controls when it pauses
   between chunks, so it could arrange this deliberately.

This needs an RFC because it extends the IPC formats and decides who may withdraw authority
(`GOVERNANCE.md`).

## Guide-level explanation

```
 control holder                binder                        service            runtime
       │                         │                              │                  │
       │                         │◀───── BIND(keystore, 0) ─────┼──────────────────│
       │                         │── GRANT(object, ops) ───────▶│                  │
       │                         │◀─ (OK, badge) + capability ──│                  │
       │                         │   records ledger entry 0     │                  │
       │                         │── capability ────────────────┼─────────────────▶│
       │                         │                              │◀─── SIGN ────────│  OK
       │── LIST(0) ─────────────▶│                              │                  │
       │◀─ keystore, 0, ops, ... │                              │                  │
       │── REVOKE(0) ───────────▶│                              │                  │
       │                         │── REVOKE(badge) ────────────▶│                  │
       │                         │◀─ OK ────────────────────────│                  │
       │◀─ OK ───────────────────│                              │◀─── SIGN ────────│  ACCESS
```

The control holder names an entry by its ledger index and never sees a badge. To the guest, a revoked
handle behaves as a handle whose operation was refused: `keystore::error-code::access`.

Revocation looks forward only. It can't recall a signature already made or plaintext already read.

## Reference-level explanation

### 1. `GRANT` returns the badge

On `OK`, the `GRANT` reply carries the minted badge in `mr3` (today `0`). The binder keeps it and
never passes it on: `BIND`'s reply to the runtime is unchanged.

### 2. `REVOKE` (binder → service)

| | `mr2` | `mr3` |
|---|---|---|
| Request | badge | 0 |
| Reply | RFC-0019 status | 0 |

`grant::classify` gains a variant. Under `GRANTER_BADGE`, `tag.extra_caps == 2` remains `Grant` and
`tag.extra_caps == 0` is the new `Revoke { badge }`. Another caller can't produce it, since the kernel
delivers the badge. The service calls `revoke_access`: `OK` on success, including a repeat, and
`INVALID` for a badge it never minted.

### 3. The ledger

A fixed `[Option<LedgerEntry>; MAX_BINDS]` in the binder. `MAX_BINDS` stays 4, and ledger indexes are
the existing `BoundSlots` indexes. An entry is `{ interface, slot, object, ops, badge, revoked }`,
written only after the service returned `OK`. A declined request leaves no entry.

### 4. The control capability

`CONTROL_BADGE = (1 << 56) - 1`, on the binder's existing endpoint. The binder serves one endpoint in
one loop, as a service does, and classifies by delivered badge. The launcher mints the control holder's
copy with this badge, and the runtime's copy keeps badge `0`, so the runtime can't reach `LIST` or
`REVOKE`.

| Op | Request `mr2` | Request `mr3` | Reply `mr2` | Reply `mr3` |
|---|---|---|---|---|
| `LIST` | `1` | index | `status \| interface << 16 \| slot << 24 \| revoked << 32` | `object \| ops << 32` |
| `REVOKE` | `2` | index | `status` | 0 |

`status` is RFC-0019's: `OK`, or `INVALID` for an empty or out-of-range index. A caller walks `LIST`
from index 0 up to `MAX_BINDS`. `REVOKE` sets `revoked` only after the service answered `OK`, so the
ledger never claims a revocation that didn't happen. If the service's reply is anything else, the
binder reports that status and the entry stays live.

A revoked slot is not re-granted: `BoundSlots` already allows each `(interface, slot)` one bind.

### 5. Chunked exchanges only continue with their own client

`Channel`'s two inner `sys::recv` calls move to one helper, which takes the exchange's starting badge
and a handler supplied by the service:

```rust
pub fn recv_request<H: Interloper>(&mut self, received: Received, buf: &mut [u8], other: &mut H) -> ...
pub fn reply<H: Interloper>(&mut self, status: u16, payload: &[u8], badge: u64, other: &mut H) -> ...
```

- A message with the exchange's badge continues it, as today.
- A message with any other badge goes to the handler and the wait continues. `GRANT` and `REVOKE`
  are served immediately. Anything else gets a new RFC-0019 status, `BUSY`.
- **`reply` re-checks authority between chunks.** It takes a `still_allowed` check from the handler,
  and if the badge has been revoked, it ends the exchange with `ACCESS` and no further payload. Chunks
  already sent can't be recalled.

`recv_request` needs no such check, because `check_access` runs after reassembly, by which time a
`REVOKE` served mid-exchange has taken effect.

Serving `REVOKE` during the wait matters: if it had to wait for the exchange to finish, a client could
hold off its own revocation indefinitely by never sending its next chunk.

### 6. The v0 control holder and the proof

A new small confined program, `lantern-shell/grant-control/`, stands in for a future shell. It holds the
control capability and runs a fixed script. It isn't a user interface.

Both runtime demos gain this sequence, using one demo-only pair of notifications between the runtime and
`grant-control`, which is no part of the protocol:

1. The runtime calls the guest's `probe`, and it succeeds.
2. The runtime signals `grant-control`, then waits.
3. `grant-control` lists entry 0, checks its `interface`, `ops` and `revoked == 0`, revokes it, lists it
   again and checks `revoked == 1`, then signals back.
4. The runtime calls `probe` again on the same instance. The guest must report the refusal, and the
   host-side check for that handle must read `ACCESS`.

`binder-matrix.sh` gains cases for: the runtime attempting `LIST` and `REVOKE` under its own badge;
`REVOKE` of an empty index; revoking twice; and a client pausing mid-chunk while a `REVOKE` arrives.
A mutation test must show that a service ignoring `REVOKE` fails the sequence.

### 7. Capacity (estimated, not measured)

`runtime-fs-demo` uses 20 of 24 frames, 5 of 6 CNodes, 4 of 5 VSpaces and 42 of 56 root slots. A small
`grant-control` adds about 2 frames, 1 CNode, 1 VSpace and a few slots. That fits, leaving CNodes and
VSpaces exactly at their limits, so a sixth program would need them raised.

## Threat model impact

- **New authority:** the control capability can withdraw any grant the binder made. It can't create or
  widen one, can't obtain a badge, and can't reach key material. Its worst misuse is denial of service
  against the guest, which it could only do if the launcher gave it to the wrong program.
- **Mitigations:** the kernel-delivered badge; ledger indexes rather than badges; a ledger that records
  a revocation only after the service confirms it; and section 5's fix.
- **Not mitigated by v0:**
  - No person is involved, and revocation doesn't survive a reboot. The compiled-in policy grants the
    same thing again, so this must not be described to anyone as a durable user decision.
  - The revoked client keeps its kernel capability and can still send to the service, which refuses each
    call. It can consume service time, as any client can.
  - The shared `Frame` stays mapped, so the client can still read whatever the last exchange left in it.
    This is data it was already entitled to when it was written.
  - Anything already signed or read stays with the guest.
  - Service-to-service grants remain out of scope: revoking a filesystem grant doesn't touch
    `store-service`'s own access to `keystore-service`.
- **Net change to attacker surface: reduces.** A grant can now end, and a client loses the ability to
  disturb another caller's message in a chunked exchange.

## TCB impact

None. The binder, services and `grant-control` are all confined, and the kernel is unchanged.

## Privacy impact

The ledger records which roles a package was given, in the binder's own memory, and only the control
holder can read it. It isn't persisted and contains no user data. It is the raw material for a future
audit log and consent view, which will need their own privacy review.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** The kernel doesn't change,
   and its stubbed `Revoke` and the derivation tree it would need stay unbuilt.
2. **Can an agent use this capability without being unnecessarily trusted?** Yes. An agent's grants are
   listed and withdrawn the same way, and holding the control capability conveys no access to what the
   grants protect.
3. **Does this support humans and agents through the same interface?** Yes. A future consent screen and
   a future agent supervisor would each hold the same control capability, with no separate AI path.

## Alternatives considered

- **Kernel `Revoke` with a capability derivation tree.** This would delete the client's capability itself,
  which is stronger: the client couldn't even send. It's substantial new kernel state for something the
  service check already enforces. Service-side revocation is what `Broker::revoke` was designed around. A
  future RFC can add the kernel route if a case needs the capability actually gone.
- **Replying `BUSY` to `REVOKE` during a chunked exchange, with the binder retrying.** This is simpler, but
  it lets the client being revoked block its own revocation forever.
- **A separate endpoint for grants and revocation.** A service would have to wait on two endpoints, and the
  kernel has no such primitive.
- **Naming ledger entries by badge in `LIST`.** Simpler to encode, but it hands badge values to the control
  holder for no benefit.
- **Moving the ledger into each service.** The services already know their badges, but only the binder knows
  what each grant was for, and it would split the view across services.

## Prior art

- seL4's badged endpoints and `Revoke`, and the choice between revoking a capability and refusing at the server.
- Macaroon-style revocation at the verifier, already used by `Keystore::revoke_seal` (RFC-0011).
- Android and iOS permission settings, as a later model for the view this ledger would feed. Neither is a
  guide to the protocol.

## Unresolved questions

1. **Who holds the control capability in v0?** This RFC proposes a scripted `grant-control` program.
   The alternative is proving it from the binder's host unit tests alone, which wouldn't show the path
   working under the kernel.
2. **Should a revoked entry be re-grantable?** This RFC says no until a consent flow exists to authorise
   it, leaving a revoked role unavailable until the next boot.
3. **How does `LIST` identify the app?** v0 has one package per binder, so the entry doesn't need it. With
   several packages it will, and an app id is too long for registers, so that step will probably put
   `LIST` on a shared `Frame`.
4. **Must revocation outlive a reboot before we call the exit criterion met?** This RFC assumes yes, which
   makes a persistent policy store required follow-on work.

## Future possibilities

- A trusted-path consent surface as the real control holder, and a persistent policy store.
- Audit records of each grant and revocation, for `lantern-ai-runtime`'s log.
- Revoking `store-service`'s keystore grant, once service-to-service grants go through a binder.
- Zeroing the shared `Frame` when its client is revoked.

## Resulting ADRs

On acceptance, an ADR will record: the binder's ledger and control capability; `REVOKE` and the badge in
`GRANT`'s reply; service-side rather than kernel revocation; that chunked exchanges continue only with their
own client, with `GRANT` and `REVOKE` served during the wait; and that v0 revocation is neither user-driven
nor durable.
