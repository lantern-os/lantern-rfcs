---
adr: 0026
title: The grant ledger and revocation v0 — service-side revocation through a confined binder
status: Accepted
date: 2026-11-02
deciders: ["TSC"]
rfc: ../rfcs/0021-grant-ledger-and-revocation.md
supersedes: []
superseded_by: null
---

# ADR-0026: The grant ledger and revocation v0 — service-side revocation through a confined binder

## Context

[RFC-0020](../rfcs/0020-launch-binder.md)'s binder decides each grant but kept no record of it, and
nothing on a confined path could reach `revoke_access`. While designing the fix, we also found that
`lantern_abi::frame::Channel` took a message from any caller in the middle of a chunked exchange, so a
`REVOKE` could be swallowed as a chunk. [RFC-0021](../rfcs/0021-grant-ledger-and-revocation.md) proposed
the change and has been accepted; this ADR is the durable record.

## Decision

**The binder keeps a ledger of what it granted, and the holder of a control capability can list it and
revoke an entry. Revocation is enforced by the owning service, not the kernel.**

- **`GRANT`'s reply carries the minted badge** in `mr3`. The binder keeps it and never passes it on.
- **`REVOKE` (binder → service):** under `GRANTER_BADGE`, with no capability slot, `mr2` = badge. The
  service answers `OK`, including for a repeat, or `INVALID`.
- **The ledger:** one entry per `BoundSlots` index, written only after the service's `OK`. An entry is
  marked revoked only after the service's `OK` too, so it never claims a revocation that didn't happen.
- **The control capability:** `CONTROL_BADGE = (1 << 56) - 1` on the binder's existing endpoint.
  `LIST(index)` and `REVOKE(index)` name entries by index, so the holder never sees a badge.
- **A chunked exchange continues only with its own client.** Anything else arriving during the wait
  goes to the service's `Interloper`: `GRANT` and `REVOKE` are served at once, and other data gets the new
  status `BUSY`. Serving `REVOKE` there is what stops a client holding off its own revocation by never
  sending its next chunk. `Channel::reply` re-checks the badge between chunks.
- **A revoked role is not re-granted** until the next boot, since no consent flow exists to authorise it.

**v0's control holder is `grant-control`, a scripted test program.** It isn't a user interface.

**Settled on acceptance:** revocation must outlive a reboot before the exit criterion counts as met, which
makes a persistent policy store required follow-on work.

Rejected alternatives (full reasoning in the RFC): kernel `Revoke` with a derivation tree (substantial
kernel state for what the service already enforces); answering `REVOKE` with `BUSY` and retrying (a client
could block its own revocation forever); a separate endpoint (the kernel can't wait on two); badges in
`LIST`; a ledger in each service.

## Consequences

- **Easier:** a grant can now end, and there is a list of what was granted for a future consent view and
  audit log to read.
- **Harder / committed to:** every service that a binder launches serves `GRANT` and `REVOKE` from both its
  main loop and its mid-exchange wait, and `Channel`'s callee methods all take an `Interloper`.
- **Trust boundary:** one created, the control capability. It can withdraw any grant the binder made, and
  can't create or widen one or reach key material. Nothing enters the ADR-0004 TCB, and the kernel is
  unchanged.
- **Net attacker surface: reduces.**
- **Still not met:** the Phase 3 exit criterion's "visible, revocable capability set". No person sees or
  decides anything yet, and the compiled-in policy grants the same thing again at the next boot, so this
  must not be described as a durable user decision. A revoked client also keeps its kernel capability and
  its mapped `Frame`, and anything already signed or read stays with the guest.
