---
adr: 0029
title: The audit log v0 — a hash-chained record of the launch binder's decisions
status: Accepted
date: 2026-11-05
deciders: ["TSC"]
rfc: ../rfcs/0024-audit-log.md
supersedes: []
superseded_by: null
---

# ADR-0029: The audit log v0 — a hash-chained record of the launch binder's decisions

## Context

The Phase 3 exit criterion asks for "a faithful audit log", and nothing recorded anything. Every grant and
revocation now passes through the launch binder. [RFC-0024](../rfcs/0024-audit-log.md) proposed recording
there first, and has been accepted; this ADR is the durable record.

## Decision

**The launch binder records every grant, decline and revocation in a hash-chained, append-only log on flash,
and records a grant before attempting it.**

- **One owner of the flash:** `policy-store` keeps the audit log in sectors 1 to 7, beside the revocation log in
  sector 0, because two programs can't drive one flash device's command mode independently. It is now a durable-log
  service.
- **Records:** 64 bytes each: event, decline reason, interface, slot, object, ops, revocation key, sequence, and the
  first 128 bits of `BLAKE3(previous chain ‖ those fields)`. The store writes `BOOT` itself, once per start. There's
  no clock, so "when" means order.
- **Record before grant:** `GRANTING` is appended after `decide` and `IS_REVOKED`, and only `OK` lets the grant proceed.
  A full, corrupt or missing log declines every grant. A refused grant is followed by `GRANT_FAILED`.
- **A revocation is never blocked by the log.** `REVOKING` is appended after the policy store's `RECORD`, and a failure
  is reported `FAILED` without stopping the revocation.
- **The decline reason is logged, not told:** the runtime still gets one `DECLINED`. Only a slot's first attempt is
  decided and recorded, so a caller can't fill the log by repeating one.
- **A read-only capability**, `READER_BADGE`, serves only `HEAD` and `READ`, with every hashed field, so a holder can
  recompute the chain. `APPEND` from it is refused. v0 gives it only to `grant-control`.
- **`audit-dump`**, a host tool, prints and verifies a flash image's log, and checks it against a kept head.

**Settled on acceptance:** `policy-store` owns both logs; v0 covers the binder's decisions only; a full log stops all grants;
and the code stays in `lantern-shell` until `lantern-ai-runtime` has code.

## Consequences

- **Easier:** every grant has a record made before it happened, and declines say why, for a future audit surface and
  consent view.
- **Harder / committed to:** grants depend on the log being writable, and the log needs a retention design before real use.
- **Trust boundary:** one created, the reader capability. Nothing enters the ADR-0004 TCB, and the kernel is unchanged.
- **Detected:** accidental corruption, and any edit that doesn't recompute the rest of the chain.
- **Not detected, and on [RFC-0023](../rfcs/0023-hardware-key-custody-in-phase-4.md)'s checklist:** a consistent rewrite of
  the whole chain, or erasing it, by someone who can write the flash. Only a head kept elsewhere reveals a rewrite.
- **Still not met:** the exit criterion's "faithful audit log". What a guest does with a grant, every `SIGN` and `READ`, isn't
  recorded yet, and neither is `store-service`'s first-caller grant from `keystore-service`. Recording service operations is
  the next RFC.
