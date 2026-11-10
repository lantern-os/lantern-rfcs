---
rfc: 0025
title: The audit log v1 — every use of a grant, recorded by the service before it acts
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "ai-runtime", "crypto", "filesystem"]
domains: ["shell", "ai-runtime", "abi", "crypto", "filesystem", "boot"]
created: 2026-11-10
updated: 2026-11-10
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-10.** Fixed by [ADR-0030](../adr/0030-audit-log-service-operations.md). The unresolved questions
> were settled as proposed: retire the oldest history; refuse operations past the quota; accept repeated relaunch
> for v1; and make `policy::audited` a review rule. Implementation changed four details, all folded in below:
> retirement writes `RETIRED` as the first record of the reopened sector, after the erase, not before it; a reader
> gets the oldest surviving position from a new `OLDEST` op; `QUOTA_REACHED` keeps the refused record's event and
> reason; and a service that can't record its own first-caller grant withdraws it and keeps serving.

# RFC-0025: The audit log v1 — every use of a grant, recorded by the service before it acts

## Summary

[RFC-0024](./0024-audit-log.md) records what the launch binder decides. It doesn't record what a guest then does with a
grant, so the Phase 3 exit criterion's "faithful audit log" isn't met. This RFC closes that gap for the services that exist.
**The service that enforces a grant records each use of it, in the same chain, before acting.** If the log can't take the
record, the operation doesn't happen.

Recording every operation means the log will fill up in ordinary use, so this RFC also gives the log a **retention rule**. The
oldest sector is retired, and the retirement is recorded in the chain. A **per-grant quota** stops one guest from pushing
everyone else's history out.

## Motivation

The exit criterion asks for a faithful audit log of what a local AI agent does. `lantern-ai-runtime/ARCHITECTURE.md` asks for
"which agent, under which capability, did what, to which object, when". Today the log answers "which capability was handed
out", but not "what was done with it". An agent could sign a thousand documents under one recorded grant, and the log would
show one line.

RFC-0024 deferred this, and gave two reasons: a flash write on every operation's path, and no design for a log that fills up.
Both have to be answered now. With every operation recorded, a log that stops all grants when it's full stops being a
thousands-of-boots-away problem and becomes a this-week problem.

Principles served: **least authority** (the program recording an operation is the one that holds the authority, not the one
asking), and **verifiability** (one total order a reader can recompute). The tension is with **performance**: each operation
gains an IPC round trip and a flash write. This RFC accepts that cost rather than weaken record-before-act.

This needs an RFC because it gives two services a new capability, changes the log's format and protocol, and makes the log
erase its own history (`GOVERNANCE.md`).

## Guide-level explanation

```
 runtime ── SIGN ──▶ keystore-service ── APPEND(USING) ──▶ policy-store     recorded before acting
                          │ only on OK: sign
                          │ ── APPEND(USE_FAILED), only if the op didn't succeed
 runtime ◀── signature ───┘

 binder ── GRANT ──▶ service ──▶ badge
        ── APPEND(GRANTED, badge) ──▶ policy-store                         links the badge to the role
```

A reader joins the two halves. `GRANTING` gives the role (RFC-0024), `GRANTED` gives the badge the service minted for it,
and each `USING` names that badge. Within one boot, a badge identifies one grant: each service counts badges up from 1 and never
reuses one (`lantern-capabilities`' `Broker`).

**Who records matters.** The runtime is the program being audited, and it can lie or stay silent. The service already decides
whether each request is allowed (RFC-0021's service-side revocation), so it records at the same point. A compromised service
can omit records, but it also holds the keys or files, so it's already trusted with what it would hide.

## Reference-level explanation

### 1. Who can append what

`policy-store`'s endpoint gains two more badged copies, one for each service. **The store, not the caller, decides which
events a badge may append**, and stamps the record's `interface` from the badge:

| Badge | Holder | May append | `interface` |
|---|---|---|---|
| `0` | Launch binder | `GRANTING`, `GRANT_FAILED`, `DECLINED`, `REVOKING`, `GRANTED` | As sent |
| `KEYSTORE_WRITER_BADGE` | `keystore-service` | `USING`, `USE_FAILED`, `SERVICE_GRANTED` | Keystore, always |
| `STORE_WRITER_BADGE` | `store-service` | `USING`, `USE_FAILED`, `SERVICE_GRANTED` | Filesystem, always |
| `READER_BADGE` | `grant-control` | Nothing; `HEAD` and `READ` only | — |

Anything else is refused `ACCESS`. A service can't write a decision record, can't pose as the other service, and the binder
can't write an operation record. The writer badges are `READER_BADGE - 1` and `READER_BADGE - 2`, in `lantern-abi::policy`.

### 2. New events

| Event | From | Meaning | Fields |
|---|---|---|---|
| `GRANTED` | Binder | The service's `GRANT` returned `OK` | `interface`, `slot`, `object`, `ops` as `GRANTING`; `subject` = `[badge, 0]` |
| `SERVICE_GRANTED` | Service | A first-caller grant outside the binder (RFC-0024's gap) | `object`, `ops`; `subject` = `[badge, 0]` |
| `USING` | Service | Recorded before an operation runs | `object` = the badge's key or file; `ops` = the one op bit used; `subject` = `[badge, request bytes]` |
| `USE_FAILED` | Service | The operation didn't return `OK` | As `USING`; `reason` = the RFC-0019 status |
| `QUOTA_REACHED` | Store | Written in place of the record that reached this badge's quota this boot | As the refused record; `slot` = its event, `reason` = its reason |
| `RETIRED` | Store | The first record of a sector reopened by erasing the oldest history | `slot` = the sector; `subject` = `[last retired sequence, 0]` |

`ops` keeps its RFC-0024 meaning: `KeyOps` or `FileOps` bits. `SIGN` is recorded as `KeyOps::SIGN`, and a file `READ` as `FileOps::READ`.

**The record format doesn't change.** The 16-byte field RFC-0024 calls `key` becomes `subject`. It holds the role's
`RevocationKey` for the binder's decisions, and the badge and its companion word for the new events. The chain hashes it the
same way.

A `GRANTING` with no `GRANTED` or `GRANT_FAILED` after it still reads as "may have been granted". A `USING` with no
`USE_FAILED` after it reads as "may have run".

### 3. What must be recorded, and when

- **Before every operation.** On a data request whose badge passes the service's own checks, the service appends `USING`.
  Only `OK` lets the operation run. Anything else answers the client `FAILED` and doesn't run it.
- **After a failed operation.** If the operation returns anything but `OK`, the service appends `USE_FAILED` with the status.
  This includes a request under a revoked badge that reaches dispatch, so "tried to use a revoked grant" is visible.
- **After a successful `GRANT`.** The binder appends `GRANTED` with the badge. Failing to append doesn't undo the grant, since
  `GRANTING` already covers it, but the reader loses the badge-to-role link for that grant. That's a known gap, made visible
  by a `GRANTING` with neither `GRANTED` nor `GRANT_FAILED` after it.
- **A first-caller grant.** `store-service`'s grant from `keystore-service` is launcher-fixed infrastructure, not an app's
  request, and it stays first-caller. `keystore-service` appends `SERVICE_GRANTED` after minting, since the badge is the
  record, and before delivering. If the append fails, the badge is withdrawn, the caller is answered `FAILED`, and the
  service keeps serving, refusing everything. It mustn't stop: with no preemption, a service that spins stops every
  program.
- **A use outside the grant is recorded, and refused.** A request for an op the badge doesn't carry, such as the runtime's
  own attenuation check of an ENCRYPT on a SIGN-only grant, reads as `USING` then `USE_FAILED(ACCESS)`. So the log shows
  an attempt to exceed a grant, and that it didn't succeed.
- **Requests that never reach dispatch aren't recorded.** These are malformed framing, a client that isn't the current data
  client (`BUSY`), and requests under a badge the service has never minted. None of them uses authority. Neither is a
  refusal the runtime makes itself before calling a service, such as an oversized write.

### 4. The quota

The store counts `USING` records per `(source badge, subject badge)` since boot, in a fixed table of `MAX_GRANTS` entries per
service. At `USE_QUOTA` (proposed 1,024), it appends `QUOTA_REACHED` once and refuses every later `USING` for that badge with
`ACCESS`. That badge's operations then fail until the next boot. Decision records have no quota: `MAX_BINDS` already bounds
them. `USE_FAILED` counts towards the quota too, so failing on purpose can't fill the log.

Without a quota, one guest could push every other record out through the retention rule. With it, the worst case one boot can
retire is `grants × USE_QUOTA` records. See unresolved question 2.

### 5. Retention

The audit log keeps sectors 1 to 7. **Each sector now starts with its own header**: magic, format version 2, the sequence
number of its first record, and the chain head before that record. Records follow as before, 4,095 to a sector.

When the active sector is full and the next sector is the oldest:

1. Erase that sector (Intel block erase).
2. Write its new header, continuing the chain.
3. Append `RETIRED` as its first record, naming the sector and the last sequence number it held.

`RETIRED` can't go first. It would need the last slot of the full sector, and a power cut while writing it would leave a
torn slot, no room, and a log stuck failing closed for good.

A power cut at any point recovers, and the rule is stated on flash, not remembered:

- **Opening a sector retires history exactly when the sector after it holds a header whose first sequence is above 0.** Before
  the ring first wraps, the sector after the one being opened is erased, or is sector 1, which starts at 0.
- On open, a full newest sector's neighbour that is neither valid nor erased, or is erased while opening it would retire
  history, was interrupted mid-erase or mid-header. It is erased again and opened.
- A newest sector that retired history but has no records yet gets its `RETIRED`.
- Anything else that's neither valid nor erased is corrupt, as now.

A reader verifies the chain from the oldest surviving sector's header, which the new `OLDEST` op returns, and sees in each
`RETIRED` exactly how much history is gone. **Retired history can't be
verified by anyone, and a head kept from before a retirement can no longer be checked.** Both are stated, and the matrix tests them.

The revocation log in sector 0 is never touched by retention. A revocation's enforcement doesn't depend on the audit log's
history.

The store refuses the binder nothing for space. The log no longer fills, so a full log no longer stops grants. A sector it can't
erase makes the log corrupt, and that does stop every grant and every operation, failing closed as before.

### 6. The protocol

`APPEND` keeps its registers: `label` = op, event, reason; `mr1` = slot, object, ops, interface; `mr2`, `mr3` = subject. A
writer's `interface` is overwritten by the store. New statuses: none. A quota refusal is `ACCESS`, and the service treats every
non-`OK` the same way. A badge that may not send an op gets `ACCESS` in both the `label` and `mr2`, whichever its client reads.

`OLDEST`, under `READER_BADGE` only, answers in `HEAD`'s shape: the sequence number of the first record still held, and the
chain head before it.

A service records only when launched with the `grant::AUDITED` `arg0` flag, holding its writer capability. The two demos from
before the binder (`keystore_demo`, `store_demo`) have no policy store, and run unaudited, as they already run without the
binder. The policy store's `arg0` can lower the quota below `USE_QUOTA`, never raise it, which is how the matrix tests it.

`lantern-abi` gains `policy::audited(store, entry, op)`. It appends `USING`, runs `op` only on `OK`, and appends `USE_FAILED`
for a non-`OK` result. Both services use it, so that "record, then act" is written once and tested once, and a future service
gets it by calling one function.

### 7. The proof

- `grant-control` checks, in both runtime demos, that each `USING` follows a `GRANTED` with the same interface and badge in the
  same boot, between the grant and the revocation, that a use outside the grant is followed at once by its `USE_FAILED`, and
  that every other service's use is under a badge its own `SERVICE_GRANTED` named. It runs between the two `probe` calls, so
  the uses after the revocation are checked from outside, by the matrix.
- `audit-dump` prints the join, as role, then badge, then each use, and verifies across sectors and retirements. A new
  `--fill <n>` writes `n` filler records into an image, so the matrix can force a retirement without running for hours.
- `binder-matrix.sh` gains:

| Case | Expected |
|---|---|
| Keystore demo, one boot | `GRANTED`, then `USING(SIGN)` per signature, with badges joining |
| Filesystem demo, one boot | `USING(WRITE)`, `USING(READ)` from `store-service`, and `USING(ENCRYPT/DECRYPT)` from `keystore-service` under its `SERVICE_GRANTED` badge |
| After revocation | The probe's `USE_FAILED(ACCESS)` |
| A service's append refused | The operation doesn't run, and the guest sees an error. The quota case produces it: the refused write has no `ENCRYPT` after it |
| Quota (lowered to 1 by the `audit-quota-test` feature) | Exactly the quota recorded, one `QUOTA_REACHED`, then `FAILED`, in both demos |
| A boot across a full ring | `RETIRED` recorded, the oldest sector erased, the chain verifying from the new oldest header, inside `grant-control` and outside |
| A power cut mid-erase: sector half erased, or erased with no header | The next boot finishes the retirement and continues |
| A head kept from before a retirement | Reported as `RETIRED`, not as a match; a head kept after it still matches |
| A corrupt log, with a first-caller grant to record | Every grant declined, and no service stops |

As implemented, the matrix has 62 cases, and `policy-store` has 36 host tests, including a power cut at every word of a
retirement's header, `RETIRED` and next record, and across the erase.

Mutation tests must show that a service that acts without `USING`, a store that accepts `USING` from the binder's badge, and
a store that trusts a writer's `interface` each fail a case. They did: the first fails `grant-control`'s step 9 in the keystore demo, the
second the `a_source_appends_only_its_own_events` test, and the third step 10 in the keystore demo and step 9 in the
filesystem demo.

### 8. Cost

Per operation: one more IPC round trip and a 16-word flash program, and two of each on failure. QEMU's flash is instant, so
QEMU can't measure the real cost. Datasheet NOR program times suggest a few hundred microseconds per record on hardware, and
a sector erase takes on the order of a second, during which `policy-store` answers nothing. Both are recorded as Phase 4
measurements. Batching records was rejected because it breaks record-before-act (Alternatives).

Capabilities: two more root slots (67 of 72 in the largest demo) and one slot in each service's CSpace. No new program and no
new Frames.

A store opening a full ring scans and hashes every record, about 28,000, before serving anything. Under QEMU that takes about
a second.

## Threat model impact

- **Trust boundaries affected:** each service now depends on `policy-store` to serve granted authority. A dead or corrupt store
  stops every operation, not only every grant. That's the fail-closed choice RFC-0024 made for grants, applied to use.
- **New assets:** two append-only writer capabilities, each limited by the store to operation events under its own interface.
- **New adversary capabilities:**
  - A guest can make its service write flash, at most `USE_QUOTA` times per grant per boot.
  - Through the retention rule, it can push the oldest history out. The quota bounds this per boot, but not across boots for
    a guest that's relaunched repeatedly. See unresolved question 3.
- **Mitigations:** the store decides what each badge may write, and stamps the interface. There's a per-grant quota, and
  retirement is in the chain.
- **Still not mitigated:** a rewrite or erase from outside (Phase 4, RFC-0023), a compromised service omitting its own
  records, and "when", which is order only.
- **Net change to attacker surface: slightly larger.** There are two more writers and an erase path. In exchange, the
  property the exit criterion asks for now holds for every granted operation.

Cross-reference [Threat-Model.md](../../lantern-docs/wiki/Threat-Model.md) T4 (mandatory audit log). `lantern-shell`'s SH6
gains the guest half, and `lantern-crypto` and `lantern-filesystem` each gain a threat row for "the store is unavailable".

## TCB impact

None. Everything changed is confined, and the kernel is unchanged. The erase command is driven by `policy-store`, which is
already the flash's only owner.

## Privacy impact

The log becomes a fine-grained record of what each app did: how many signatures, how many reads and writes, of which object,
in what order. It still holds no payloads, only sizes. It's on unencrypted flash, readable by anyone with the device, until
Phase 4. Retention is a privacy feature as well as a capacity one, but its horizon is set by volume, not by a time a person
chose. A person-facing retention setting needs the consent surface. No record leaves the device.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** It adds nothing to the kernel.
2. **Can an agent use a capability without being unnecessarily trusted?** Yes, and this is the point: what an agent does is
   recorded by the program holding the authority, not reported by the agent.
3. **Same interface for humans and agents?** Yes. A human-launched app and an agent are recorded identically, by the same services.

## Alternatives considered

- **The runtime records each host call.** It's simpler, and it sees the guest's intent directly. But it's the audited party:
  a compromised runtime omits whatever it likes.
- **Batching records, flushed every N operations or on idle.** It's much cheaper, but an operation would happen before its
  record exists, and a crash loses it. That is the property RFC-0024 was built to prevent.
- **Counting instead of recording.** Keep per-badge counters and write a summary periodically. It's cheaper, and it still
  shows how much was done, but not in what order, and a crash loses the unwritten count. It's a candidate for operations past the
  quota, instead of refusing them (unresolved question 2).
- **A separate operations log, with its own chain and sectors.** It isolates decision history from operation floods. But it
  loses the single total order between "granted" and "used", and doubles the format and the reader. The quota gives most of
  the isolation.
- **A full log fails closed, as v0.** With every operation recorded, the device would stop working after normal use.
- **Exporting old history before retiring it.** This is the right long-term answer, but there's nowhere to export to yet, no
  encrypted store reachable by `policy-store`, and no trusted peer (Future possibilities).

## Prior art

- Linux `auditd`: records at the enforcement point (LSM hooks), not in the calling process. Its `disk_full_action` /
  `space_left_action` settings are the same choice as unresolved question 1.
- Certificate Transparency: checkpoints that let a reader verify from a known point.
- Log-structured flash file systems (JFFS2, LittleFS): sector headers carrying sequence numbers, and erase-and-continue recovery.

## Unresolved questions

1. **Retire the oldest history, or stop when full?** This RFC proposes retiring, with each retirement recorded. Stopping when
   full is stricter about history but turns the audit log into an off switch for the whole system.
2. **What happens past the quota?** This RFC proposes refusing the operation, which keeps every recorded operation individually
   recorded but caps legitimate heavy use at `USE_QUOTA` per grant per boot. Alternatives: count without recording past the
   quota, or a larger quota. A quota a person sets belongs with consent.
3. **Repeated relaunch.** A guest relaunched many times can still retire history, `USE_QUOTA` at a time. This RFC proposes
   accepting that for v1, with the retirements visible. A real answer needs either export, or a per-role quota across boots,
   which would need the store to keep per-role counters on flash.
4. **Should future services be required to use `policy::audited`?** This RFC proposes yes, as a review rule in
   `lantern-shell/THREAT_MODEL.md`. Code can't enforce it.

## Future possibilities

- Export retired history to the encrypted filesystem, or to a trusted peer, before erasing it.
- A per-role quota that persists across boots.
- A person-facing audit view, with names instead of badges and hashes, as part of the consent surface.
- A time source, so "when" means more than order.
- Signing sector headers and `RETIRED` records with a hardware-held key (RFC-0023's checklist), so a retirement can't be faked.

## Resulting ADRs

On acceptance, an ADR will record: services record use before acting, through `policy::audited`; the store's per-badge
event rules and interface stamping; the new events and the `subject` field; per-sector headers and retirement; the per-grant
quota; and what's still not detected.
