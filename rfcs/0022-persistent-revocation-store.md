---
rfc: 0022
title: The persistent policy store v0 — revocations that outlive a reboot, on flash, in a confined service
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "boot", "capabilities"]
domains: ["shell", "abi", "boot", "kernel"]
created: 2026-11-03
updated: 2026-11-03
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-03.** Fixed by [ADR-0027](../adr/0027-persistent-revocation-store.md). Questions 1 and 2
> were settled on acceptance: `policy-store` is a separate program, and durable-but-not-tamper-proof is
> acceptable for v0. Implementing it refined section 4 in three ways: the service scans flash instead of
> holding an in-memory set; a torn record is one whose last word is still erased, so a flipped bit in a finished
> record is corrupt rather than torn; and a torn slot is zeroed into a tombstone, so later records stay readable.

# RFC-0022: The persistent policy store v0 — revocations that outlive a reboot, on flash, in a confined service

## Summary

[RFC-0021](./0021-grant-ledger-and-revocation.md) lets a grant be revoked, but only until the next
boot. This RFC makes a revocation last. A new confined service, `policy-store`, keeps an append-only
log of revoked roles on QEMU's second flash device, and the launch binder consults it before every
grant.

**The store can only narrow.** v0 holds revocations, not grants: anything it says is on top of the
compiled-in policy, so a compromised or corrupted store can refuse what that policy allows but can never
exceed it.

**It is durable, not tamper-proof.** This project has no hardware root of trust, so nothing here resists
someone who can write the flash from outside the system. Erasing the flash brings every revoked role back.
This RFC says so plainly rather than adding a MAC whose key would sit in the same image.

## Motivation

ADR-0026 settled that revocation must outlive a reboot before the Phase 3 exit criterion counts as met.
Today nothing in the project can do that:

1. **There is no persistent storage of any kind.** `lantern_filesystem::Store` is in memory, and no block
   driver exists.
2. **There is no device capability design.** No RFC or ADR covers a confined program reaching a device.
3. **There is no root of trust.** Key seeds are public demo constants (`lantern-crypto/STATUS.md`), and QEMU's
   `virt` machine offers no monotonic counter or sealed storage.

The first two have to be solved here, in the smallest form that serves this one need. The third can't be, and
this RFC must not hide that.

This needs an RFC because it introduces a program's first access to a device, adds an IPC format, and changes
what decides a grant (`GOVERNANCE.md`).

## Guide-level explanation

```
 first boot                                             next boot
 ──────────                                             ─────────
 grant-control ── REVOKE(0) ──▶ binder                  runtime ── BIND(keystore, 0) ──▶ binder
                                  │                                                        │
                                  ├─ RECORD(key) ─▶ policy-store ─▶ flash                  ├─ IS_REVOKED(key) ─▶ policy-store
                                  │◀─ OK ──────────                                        │◀─ yes ─────────────
                                  ├─ REVOKE(badge) ─▶ service                              │
                                  │◀─ OK ──────────                                        │
 grant-control ◀── OK ────────────┘                     runtime ◀── DECLINED ──────────────┘
```

A revocation is keyed by what it means, not by this boot's ledger index or badge, both of which change:
`(publisher, app, interface, role)`. The guest sees what it would see for any declined role: `none`.

## Reference-level explanation

### 1. The medium: QEMU's second flash device

`virt.flash1`, a 32 MiB CFI flash at `0x2200_0000`, attached with
`-drive if=pflash,format=raw,unit=1,file=<image>`. Measured on QEMU 10.2: 256 KiB sectors, width 4, readable
directly, and reading all zeros when no drive is attached.

Flash needs no DMA. That is the main reason to choose it over `virtio-blk`, see Alternatives. It is also a QEMU
target, not a storage design: real hardware will need its own driver behind the same protocol.

### 2. A device Frame, with no kernel change

The launcher seeds a second `Untyped` with `Untyped::with_memory` over one megapage of `virt.flash1`, and retypes
one `FrameMega` from it. `admin::untyped_retype` only bump-allocates and records the address, and it doesn't write
to the memory, so this works today. The launcher then maps that Frame read/write into `policy-store` alone, which
gets no other device.

This is deliberately not a general device-capability design. It relies on the launcher being trusted to name
physical ranges, as it already is. That design needs its own RFC when a second device appears.

### 3. The `policy-store` protocol (binder → `policy-store`)

Register-only, on an endpoint only the binder holds. The op is in the tag's `label`.

| Op | `label` | Request `mr2`, `mr3` | Reply `mr2` |
|---|---|---|---|
| `IS_REVOKED` | 1 | key | `OK` not revoked, `ACCESS` revoked, `FAILED` unknown |
| `RECORD` | 2 | key | `OK` durable, including a repeat, `FAILED` otherwise |

The key is the first 128 bits of `BLAKE3(publisher ‖ len(app) ‖ app ‖ interface ‖ len(role) ‖ role)`. A collision
needs about 2^64 work and a trusted publisher's signature, and could only cause a refusal.

### 4. The log

Sector 0 of the mapped megapage holds the log, append-only, because flash programming can only clear bits.

- **Header (32 bytes):** magic, format version, reserved.
- **Records (32 bytes each):** the 16-byte key, then the first 16 bytes of `BLAKE3` over it. About 8,190 fit. v0
  has no compaction, since revocations are rare and a role is recorded once.
- **On start:** the service reads the log into a fixed in-memory set.
  - All `0xFF`: empty, and it writes the header.
  - A record whose check fails, as the last record before erased space: a torn write. Ignored, and its slot is
    skipped. It can't have been acknowledged, because `OK` is sent only after reading the write back.
  - Anything else: **corrupt**. Every `IS_REVOKED` answers `FAILED`, so everything is declined.
- **`BLAKE3` here detects accidents, not attackers:** it has no key.

A full log answers `RECORD` with `FAILED`.

### 5. The binder

**Before each grant:** once `decide` would grant, the binder asks `IS_REVOKED`. Only `OK` proceeds. Anything else
gets the usual single `DECLINED`.

**On `REVOKE`:** the order is `RECORD`, then the service's `REVOKE`, then the ledger.

- A crash after `RECORD` means the role is declined next boot, the safe direction.
- If `RECORD` fails, the service's `REVOKE` still goes ahead, because the holder asked for the authority to end now.
  The reply is `FAILED`, and the entry lists as revoked but not durable.
- A repeated `REVOKE` retries `RECORD`, which is how a holder recovers from that.

`ListedGrant` gains a `durable` bit (`mr2` bit 33). `grant-control`'s script checks it.

### 6. The proof

`binder-matrix.sh` gains a flash image for each case, and these:

| Case | Expected |
|---|---|
| Erased image, first boot | `SUCCESS`, including the revocation |
| Same image, second boot | `FAILURE:8`: the guest sees `none` |
| A different role in the same image | Unaffected |
| Image with a corrupted earlier record | `FAILURE:8`: everything declined |
| No flash attached | `FAILURE:8`: reads as corrupt |
| Torn last record | `SUCCESS` |
| **Second boot after erasing the image** | **`SUCCESS`: the known limit, tested so that it stays visible** |

Mutation tests must show that a binder skipping `IS_REVOKED` fails the second-boot case, and that a store which
ignores `RECORD` fails it too.

Other demos are unaffected. Both runtime demos now need the flash drive, so the demo documentation and test
scripts change.

### 7. Capacity

`policy-store` adds an image (2 Frames), a stack (1) and the flash (1), one CNode, one VSpace, one more `Untyped`,
and about eight root slots. `runtime-fs-demo` is at 23/24, 6/6, 5/5 and 53/56, so this RFC raises `MAX_FRAMES` to 28,
`MAX_CNODES` to 7, `MAX_VSPACES` to 6 and `CNODE_SLOTS` to 64. These numbers are estimated and will be measured
when implemented. The `.bss` cost is on the order of a few KiB.

## Threat model impact

- **New authority:** `policy-store` holds a flash mapping and can read and write it. It holds no service endpoint
  and can't mint anything.
- **New input:** flash contents are untrusted bytes. They're parsed in `policy-store`, not in the binder, which holds
  mint authority.
- **Bounded by design:** the store can only narrow. Its worst lie, "not revoked", restores what the compiled-in
  policy already allows. Its other lie, "revoked", is denial of service.
- **Not mitigated by v0:**
  - **Offline tampering and rollback.** Anyone able to write the flash image can erase or replay it, restoring every
    revoked role. Stopping this needs a hardware root of trust and a monotonic counter.
  - **Still no person involved.** `grant-control` is a test script, so this is durable machinery, not a durable user
    decision. The other half of the exit criterion, consent through a trusted path, remains.
  - **A corrupted store declines every grant**, and v0's only recovery is replacing the image.
- **Net change to attacker surface: slightly larger, in exchange for a required property.** One new confined program,
  one new parser, and one device mapping.

## TCB impact

None if section 2 holds as measured: no kernel code changes beyond raised limits, and every new component is confined.
The launcher, already trusted, gains one more physical range to name.

## Privacy impact

Records hold hashes of `(publisher, app, interface, role)`, unencrypted. A party with the image who knows or guesses an
app's name can confirm it was revoked. `lantern-filesystem/ARCHITECTURE.md` says "no plaintext-at-rest mode is offered"
for the filesystem. This store isn't that filesystem, but it is data at rest. Encrypting it today would use a public
demo seed, which protects nothing. It should move onto real storage when a root of trust exists.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** It adds no kernel mechanism, though it does
   raise fixed limits.
2. **Can an agent use this capability without being unnecessarily trusted?** A revoked agent role stays revoked across
   boots, and the store can't widen what an agent gets.
3. **Does this support humans and agents through the same interface?** Yes: any control holder's `REVOKE` takes the same
   path.

## Alternatives considered

- **`virtio-blk`.** This is the more realistic device, but it uses DMA, and QEMU's `virt` machine has no IOMMU in use here.
  A driver able to program DMA could read or write all of memory, including the kernel's, so it wouldn't really be
  confined. The wiki names IOMMU-confined drivers as later work. A driver would also need physical addresses, and polling
  without interrupt support.
- **Making `lantern_filesystem::Store` persistent first.** This is the right long-term home, and the exit criterion also
  needs it, for "data at rest is encrypted". But it needs a block layer, chunking and real key custody, and revocation
  shouldn't wait for all of that. The protocol in section 3 hides the medium, so the log can move later.
- **The binder owning the flash directly.** This saves a program and the limit raises, but it puts a device driver and a
  parser of untrusted bytes in the process with mint authority.
- **A keyed MAC.** The key would have to live in the same image, as a public demo constant, so it would look like protection
  without being any.
- **Storing the whole policy rather than revocations.** That's what consent will eventually need, but it makes the store
  authoritative: a compromised one could grant. Nothing writes grants yet, so v0 shouldn't accept that risk.
- **Failing open on a corrupt store.** It would be more available, but corruption would silently restore revoked roles.

## Prior art

- Append-only logs on NOR flash, such as those used by embedded key-value stores, for the torn-write handling.
- Android's rollback-protected storage and TPM monotonic counters, as what a real root of trust would add.
- seL4's device untyped, as the eventual shape of a general device capability.

## Unresolved questions

1. **A separate `policy-store` program, with the limit raises?** This RFC proposes yes.
2. **Is durable-but-not-tamper-proof acceptable for v0?** This RFC proposes yes, with the limit tested and stated. If the
   exit criterion must resist offline tampering, this RFC can't deliver that, and a root-of-trust RFC has to come first.
3. **Does "the store only narrows" hold once consent exists?** No, and that later RFC must take on a store that can grant,
   with whatever integrity it requires.
4. **What is the recovery path for a corrupt store**, beyond replacing the image? Probably a consent-surface action, later.

## Future possibilities

- A root-of-trust RFC: a sealed key and a monotonic counter, with a keyed log that resists rollback.
- A general device-capability design, once a second device exists.
- Moving the log onto a persistent, encrypted `Store`.
- Grants in the store, written by a real consent prompt.

## Resulting ADRs

On acceptance, an ADR will record: `policy-store` as a confined service holding the flash; a launcher-named device Frame
with no general device design; the `IS_REVOKED`/`RECORD` protocol; the revocation key; the record-first order; failing
closed on a corrupt or missing store; that v0's store can only narrow; and that it doesn't resist offline tampering or
rollback.
