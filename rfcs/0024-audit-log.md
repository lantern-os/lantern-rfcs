---
rfc: 0024
title: The audit log v0 — a hash-chained record of the launch binder's decisions
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "ai-runtime", "capabilities"]
domains: ["shell", "ai-runtime", "abi", "boot"]
created: 2026-11-03
updated: 2026-11-05
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-05.** Fixed by [ADR-0029](../adr/0029-audit-log.md). The unresolved questions were settled
> as proposed: `policy-store` owns both logs; v0 covers the binder's decisions only; a full log stops all grants; and the
> code stays in `lantern-shell`. Implemented as specified. `audit-dump --forge-drop` produces the matrix's consistent rewrite.

# RFC-0024: The audit log v0 — a hash-chained record of the launch binder's decisions

## Summary

The Phase 3 exit criterion asks for "a faithful audit log", and nothing records anything today. This RFC starts one. The
launch binder records every grant, decline and revocation in a hash-chained log on flash, **before** acting on a grant, so
an unrecorded grant can't happen. A read-only capability lets a holder read the log and verify the chain.

**v0 records what the binder decided, not what a guest then did with a grant.** Its tamper-evidence is partial: without a
key held in hardware, someone who can write the flash can rewrite the whole chain consistently, or erase it. The RFC says
exactly what it detects, and adds the rest to [RFC-0023](./0023-hardware-key-custody-in-phase-4.md)'s Phase 4 checklist.

## Motivation

Every grant and revocation now happens through one program, so there's one place to record them. System threat T4 lists a
"mandatory audit log", RFC-0003 says grants and revocations must be observable, and `lantern-shell`'s audit surface and any
future consent view need something to present.

`lantern-ai-runtime/ARCHITECTURE.md` asks for "which agent, under which capability, did what, to which object, when". This
RFC delivers that shape for one class of action, granting authority, and doesn't claim the rest.

This needs an RFC because it adds IPC formats and a new read capability, and changes what must happen before a grant
(`GOVERNANCE.md`).

## Guide-level explanation

```
 runtime ── BIND ──▶ binder ── IS_REVOKED ──▶ durable store
                       │ ───── APPEND(GRANTING) ──▶ durable store   recorded before acting
                       │ ───── GRANT ─────────────▶ service
                       │ ───── APPEND(GRANT_FAILED), only if refused
 runtime ◀── capability ┘

 reader ── READ(n), HEAD ──▶ durable store                          read-only, no other authority
```

Each record's hash covers the previous record's hash. A reader recomputes the chain from the start and compares it with
the current head. A reader that kept an earlier head can also tell whether the history before that point was rewritten.

## Reference-level explanation

### 1. Where it lives

The audit log goes in **`policy-store`**, alongside the revocation log and on the same flash Frame, in sectors 1 to 7 of the
mapped megapage. Flash has one command mode for the whole device, so two programs can't drive it independently, and one owner
needs one driver and no limit raises. The cost is that a bug in that program affects both logs. See unresolved question 1.

The revocation log's format, torn-write rules and fail-closed behaviour carry over unchanged.

### 2. Records

Fixed 64-byte slots, about 28,000 in seven sectors:

| Field | Bytes | Meaning |
|---|---|---|
| `event` | 1 | `BOOT`, `GRANTING`, `GRANT_FAILED`, `DECLINED`, `REVOKING` |
| `reason` | 1 | For `DECLINED`, see below. Otherwise 0 |
| `interface`, `slot` | 2 | What was asked for |
| `object` | 2 | The `KeyId` or `FileId`, raw |
| `ops` | 1 | `KeyOps` or `FileOps` bits |
| reserved | 1 | Zero |
| `key` | 16 | The role's `RevocationKey` (RFC-0022), or zero if the package never verified |
| `sequence` | 8 | Position in the chain |
| `chain` | 16 | First 128 bits of `BLAKE3(previous chain ‖ all fields above)` |
| padding | 16 | Zero, with its last word written last, as the torn-write marker |

`BOOT` is written once on start, and gives each record a place in a numbered boot. **There is no wall clock:** "when" means
order, not a time. The confined programs have no time source today.

### 3. The decline reason

The runtime still gets one undifferentiated `DECLINED` (RFC-0020), but the log records why: `UNVERIFIED_PACKAGE`,
`NOT_DECLARED`, `NOT_IN_POLICY`, `OPS_EXCEED_POLICY`, `REVOKED`, `STORE_UNAVAILABLE`. `decide` changes from
returning an `Option` to a `Result` with this reason.

To stop a hostile runtime filling the log, only a slot's first attempt is recorded. `BoundSlots` already limits those to
`MAX_BINDS` per boot. Repeat attempts aren't recorded at all, so `ALREADY_BOUND` is dropped from the list, and this is a
known gap in what the log shows.

### 4. The protocol

On `policy-store`'s endpoint, distinguished by the kernel-delivered badge.

| Op | Caller | Request | Reply |
|---|---|---|---|
| `APPEND` | Binder (badge 0) | `label`: op, event, reason. `mr1`: slot, object, ops, interface. `mr2`, `mr3`: key | Status |
| `HEAD` | Reader | None | `mr1`: count. `mr2`, `mr3`: chain head |
| `READ` | Reader | `mr2`: index | `label`: event, reason. `mr1`: slot, object, ops, interface. `mr2`, `mr3`: key |

`READ` returns every hashed field except `sequence`, which the index already gives, so a reader can recompute the chain itself.
Under `READER_BADGE = (1 << 56) - 1`, only `HEAD` and `READ` are served. Under any other badge, they're refused `ACCESS`.

v0 gives the reader capability to `grant-control`, separately from its binder control capability, and to nothing else.

### 5. What must be recorded, and when

- **A grant is recorded first.** After `decide` and `IS_REVOKED`, the binder appends `GRANTING`. Only `OK` lets `GRANT` proceed,
  so a full, corrupt or missing log declines every grant. If the service refuses, `GRANT_FAILED` follows. A crash in between
  leaves a `GRANTING` with nothing after it, which reads as "may have been granted".
- **A revocation is never blocked by the log.** `REVOKING` is appended first, after the policy store's `RECORD`, but a failure
  doesn't stop the revocation, because the holder asked for authority to end. The reply is `FAILED`, as for a failed `RECORD`.
- **A decline** is appended, and failing to append doesn't change the answer.

### 6. The proof

`grant-control` reads the log and recomputes the chain in both demos, and checks the expected events. A new host tool,
`policy-store`'s `examples/audit-dump.rs`, reads a flash image and prints and verifies the log, which also lets a person see
what happened. `binder-matrix.sh` gains:

| Case | Expected |
|---|---|
| One boot | `BOOT`, `GRANTING`, `REVOKING`, and a verifying chain |
| Second boot | The log continues: a new `BOOT` and `DECLINED(REVOKED)` |
| Hostile manifest | `DECLINED` with the right reason |
| A byte edited in a record | Corrupt: every grant declined |
| A torn last record | Recovers, and the chain continues |
| No flash, or a full log | Every grant declined |
| **Whole log rewritten with a consistent chain** | **Accepted: the known limit, tested so it stays visible** |
| **The same rewrite, against a kept earlier head** | **Detected by `audit-dump`** |
| **Log erased** | **Accepted: the known limit** |

Mutation tests must show that a binder granting without `APPEND`, and a store accepting `APPEND` from the reader badge, each
fail a case.

### 7. Capacity

No new program and no new limits, if section 1 stands. One more root slot for the reader capability, out of the 8 spare, and a
few KiB of image.

## Threat model impact

- **New authority:** read access to the log for `grant-control`. A compromised reader learns which roles were granted, as hashes.
- **Faithfulness:** the log says what the binder reported. A compromised binder can omit or misreport, though it can't remove
  what's already written, since it can only append. A compromised `policy-store` can do anything to both logs.
- **Detected:** accidental corruption, and any edit that doesn't recompute the rest of the chain.
- **Not mitigated by v0:**
  - **Rewriting or erasing from outside.** Without a secret key, the chain can be recomputed. Without a monotonic counter, a truncated
    log looks complete. Only a head kept elsewhere reveals this, and nothing keeps one by default.
  - **What a guest does with a grant.** Every `SIGN` and `READ` goes unrecorded, so this doesn't by itself meet the exit criterion.
  - **Grants outside the binder.** `store-service`'s first-caller grant from `keystore-service` isn't recorded.
  - **"When".** Order only.
- **Net change to attacker surface: slightly larger**, one new parser path and one read capability, in exchange for a required
  property.

## TCB impact

None. Everything new is confined, and the kernel is unchanged.

## Privacy impact

The log is a durable history of what ran, on unencrypted flash. Roles are hashed, but someone with the image who knows an app's name
can confirm it, and the record of decisions and order is readable. A real audit surface will need retention rules and access control
for people, not only capabilities. Encrypting it under a public demo seed would protect nothing, so that waits for Phase 4.

No log entry leaves the device, and no telemetry path is added.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** It adds nothing to the kernel, and answers an open question
   ("how much of the log lives in the TCB?") with "none, for now".
2. **Can an agent use this capability without being unnecessarily trusted?** An agent's grants are recorded by something other than
   the agent, and reading the log conveys no other authority.
3. **Does this support humans and agents through the same interface?** Yes: any reader uses the same capability and sees the same records.

## Alternatives considered

- **A separate `audit-log` program.** This is stronger isolation, but it needs its own storage. Two programs can't share one flash
  device's command mode, so one would have to serve the other anyway, and it needs another round of limit raises.
- **Recording every service operation now.** This is what the exit criterion finally needs, but each `SIGN` would write flash, on a
  hot path nobody has measured. It should follow, once there's a design for batching.
- **A keyed MAC or signature.** The key would be a public constant in the same image, so it would look like protection without being any.
  It moves to Phase 4.
- **Recording after acting.** This is simpler, but a crash could leave a grant with no record.
- **Blocking a revocation when the log fails.** It would keep the log complete, but it would let a full log stop authority from ending.
  Ending authority matters more.

## Prior art

- Certificate Transparency and Git, for hash chains and checking against a kept head.
- Linux IMA and TPM PCRs, for what hardware anchoring adds.
- Secure logging with forward-secure keys (Schneier and Kelsey), as a possible Phase 4 design.

## Unresolved questions

1. **`policy-store` as the single owner of the flash?** This RFC proposes yes, and would describe it as a durable-log service.
2. **Binder decisions only for v0?** This RFC proposes yes, with service operations as the next RFC. The exit criterion isn't met until
   then.
3. **Should a full log stop all grants?** This RFC proposes yes for v0, since an unrecorded grant is what it exists to prevent. At about
   five records per boot, that's thousands of boots away. Retention needs a design before real use.
4. **Where should the code live?** `lantern-ai-runtime` is named as the audit log's owner but has no code. This RFC keeps it with
   `policy-store` in `lantern-shell` for now.

## Future possibilities

- Service operations in the log, starting with the keystore and filesystem services.
- Signing the chain with a hardware-held key and a monotonic counter (RFC-0023's checklist).
- Keeping heads elsewhere, as a sibling device or trusted peer might, to detect rewriting.
- A person-facing audit view in the shell, with names rather than hashes.
- A time source, so "when" means more than order.

## Resulting ADRs

On acceptance, an ADR will record: the chain and record format; `policy-store` as the flash's single owner; record-before-grant, and
never blocking a revocation; the read capability; that v0 covers the binder's decisions only; and what the chain does and doesn't detect.
