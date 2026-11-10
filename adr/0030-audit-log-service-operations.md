---
adr: 0030
title: The audit log v1 — every use of a grant, recorded by the service before it acts
status: Accepted
date: 2026-11-10
deciders: ["TSC"]
rfc: ../rfcs/0025-audit-log-service-operations.md
supersedes: []
superseded_by: null
---

# ADR-0030: The audit log v1 — every use of a grant, recorded by the service before it acts

## Context

[ADR-0029](./0029-audit-log.md) records the launch binder's decisions, but not what a guest does with a grant, so the
Phase 3 exit criterion's "faithful audit log" wasn't met. Recording every operation means the log fills up in ordinary use,
which ADR-0029 deferred. [RFC-0025](../rfcs/0025-audit-log-service-operations.md) proposed both, and has been accepted;
this ADR is the durable record.

## Decision

**The service that enforces a grant records each use of it, in the same chain, before acting, and the log retires its oldest
history, visibly, when it's full.**

- **Who records:** the service, not the runtime, since the runtime is the party being audited. `keystore-service` and
  `store-service` each hold the policy store's endpoint under their own writer badge, and record through one function,
  `lantern_abi::policy::audited`: `USING` first, the operation only on `OK`, and `USE_FAILED` with its status if it didn't
  succeed. An operation the log refuses doesn't run, and the client gets `FAILED`.
- **The store decides what each badge may append**, from the kernel-delivered badge (`policy::Source`), and stamps a service's
  records with its own interface. The binder appends decisions and `GRANTED`; a service only `USING`, `USE_FAILED` and
  `SERVICE_GRANTED`; the reader nothing. `BOOT`, `QUOTA_REACHED` and `RETIRED` are the store's own.
- **Linking a use to a role:** the binder appends `GRANTED` with the service's badge after `OK`. A service records its own
  first-caller grant as `SERVICE_GRANTED`, after minting and before delivering, and if it can't, withdraws the badge, answers
  `FAILED`, and keeps serving. Within a boot, a badge names one grant. The record format is unchanged; RFC-0024's `key` field
  is now `subject`.
- **What is recorded:** every request that reaches a service's dispatch under a badge it minted, including one for an op
  outside the grant, or under a revoked badge, each followed by its `USE_FAILED`. Not recorded: malformed framing, `BUSY`,
  badges never minted, and refusals the runtime makes before calling a service. None of them uses authority.
- **A per-grant quota:** `USE_QUOTA` (1,024) `USING` and `USE_FAILED` records per grant per boot. Past it, the store writes
  `QUOTA_REACHED` once, keeping the refused record's event and reason, and refuses the rest, so the operations don't run.
  The launcher can lower it, through the store's `arg0`.
- **Retention:** each audit sector carries a header with its first sequence number and the chain head before it. When the
  ring is full, the oldest sector is erased, given a new header, and its first record is `RETIRED`. Every interruption
  recovers on the next open, by a rule read from flash: opening a sector retires history exactly when the sector after it
  holds a header starting above 0. Sector 0, the revocation log, is never touched. A full log no longer stops grants; a
  sector the store can't erase or write makes it corrupt, which stops every grant and every operation.
- **Reading:** a new `OLDEST` op gives the first surviving record and the chain head before it, where a reader starts.
  A kept head is `Matches`, `Differs`, or `Retired` when its history is gone.
- **Launching:** services record only under the `grant::AUDITED` flag. The two pre-binder demos run unaudited.

**Settled on acceptance:** retire the oldest history rather than stop; refuse past the quota rather than count; accept
repeated relaunch for v1; and `policy::audited` is a review rule for every future service.

## Consequences

- **Easier:** the log answers "under which grant, did what, to which object, in what order" for both services, and shows an
  attempt to exceed a grant, or use a revoked one, and that it failed. With the binder's records, that is what the exit
  criterion's "faithful audit log" asks for, for the services that exist.
- **Harder / committed to:** every operation now depends on the policy store, and costs an IPC round trip and a flash write.
  QEMU can't measure the real cost; that's a Phase 4 measurement. A sector erase can stall the store for about a second on
  real flash. Every future service that serves granted authority must record through `policy::audited`.
- **Trust boundary:** two writer capabilities created, each limited by the store. Nothing enters the ADR-0004 TCB, and the
  kernel is unchanged.
- **Detected:** everything ADR-0029 detects, across sectors, and any retirement, since it's in the chain.
- **Not detected, and on [RFC-0023](../rfcs/0023-hardware-key-custody-in-phase-4.md)'s checklist:** a rewrite or erase from
  outside, and a forged retirement, since headers and `RETIRED` are unsigned. Retired history is gone, and nobody can verify
  it.
- **Known gaps:** a compromised service can omit its own records; a guest relaunched repeatedly can retire history, a quota at
  a time, visibly; "when" is still order only.
- **Still not met for Phase 3:** a person seeing and deciding any of this. The consent surface and trusted path are next.
