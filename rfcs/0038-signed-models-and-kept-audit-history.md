---
rfc: 0038
title: Signed model packages, and audit history kept before it's retired
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "boot", "ai-runtime", "filesystem", "sdk"]
domains: ["shell", "boot", "ai-runtime", "filesystem", "sdk", "abi", "crypto"]
created: 2026-11-18
updated: 2026-11-18
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0038: Signed model packages, and audit history kept before it's retired

> **Accepted 2026-11-19, implemented and recorded in [ADR-0043](../adr/0043-signed-models-and-kept-audit-history.md).** Built as
> proposed, with the changes in "As implemented".
>
> **Direction agreed 2026-11-18:**
> - Build signed model packages, and export audit history before it's retired.
> - Logs get a format with room for a MAC and a counter, which stays unused until the hardware root of trust exists.
> - A model's publisher is trusted the way an app's is.
> - If the ring fills before export, the sector is retired anyway and a `LOST` record names what wasn't kept: history can
>   be lost, never silently, and audited operations keep working.
> - Exported history is capped, and the oldest export is deleted first, with the deletion recorded.

## Summary

This is [RFC-0036](./0036-phase-3-to-phase-4-transition.md)'s second Phase 4 item: the parts of rollback resistance
(stream C) and signed models (stream F) that don't need hardware. It does three things:

1. **Models become signed packages**, checked by the inference service against the same publisher keys the binder uses for
   apps. The model's digest is no longer compiled into the boot image, so a new model is installed the way an app is,
   without rebuilding the system.
2. **Audit history is exported before it's retired.** A new confined program, the archiver, copies each full audit
   sector, sealed, to an archive partition on the disk. Only then may the policy store erase it. Today the oldest
   history is simply erased when the ring wraps.
3. **The logs get a versioned format** with room for a MAC and a monotonic counter value. Neither is written until the
   hardware root of trust exists.

## Motivation: what software alone can and can't do

RFC-0023's checklist asks for logs that resist tampering and rollback. That needs a key the attacker can't read and a
counter they can't wind back. Without hardware, every key the software holds is either a demo constant in the boot
image or something stored on the same disk. Whoever can rewrite the log can read both, and re-key the log. A keyed log
today would *look* tamper-resistant without being so, so this RFC doesn't key the logs.

What software can do now is real, and two things are worth building:

- **Signed models.** Today a model is trusted because its digest is compiled into the boot image. Replacing the model
  means rebuilding the image, and nothing records who published it. As signed packages, models are delivered and
  checked like apps: a publisher signs, the system verifies against its trusted publishers, and the audit log records
  whose model ran. Against someone who can rewrite the disk, this becomes protection only with secure boot (RFC-0023's
  "Boot chain" row), since they could replace the publisher keys in the image too. That limitation is stated, not hidden.
- **Kept history.** The audit ring holds six sectors. When it wraps, the oldest is erased, and that history is gone.
  Exporting it first, sealed, to an archive on the disk keeps it. This is the export half of RFC-0023's "Audit retention"
  row.

There's also a design constraint. The keystore and the store each record their own operations in the policy store's
audit log, with a blocking call. So the policy store must never call either of them, or the two could wait on each other
forever. A future log key therefore has to come from boot or the hardware root, not from the keystore. And export has to
be pulled by a third program, not pushed by the policy store.

## Signed model packages

- **The package.** A model package (`.lmod`) is a small manifest signed by its publisher, followed by the model file in
  `LMODEL01` format (RFC-0034). The manifest holds the model's name, its format, its size and its BLAKE3 digest.
  The signature is Ed25519 over the manifest, made the way `lantern-sdk build` signs app packages (RFC-0020's combined
  digest).
  - `lantern-sdk build-model` (or `tools/mkmodel`) makes one.
  - `lantern-mkdisk --model` writes one to the models partition.
- **Checking it.** The inference service checks the signature against the system's trusted publisher keys, the same list
  the binder uses for apps, and then the digest against the file. A package that fails either check is refused, exactly
  as a wrong digest is today. `assets/model.digest` and the compiled-in `MODEL_DIGEST` go away.
- **Recording it.** The audit log's model-call records name the model by publisher and digest prefix, so `audit` shows
  whose model answered. Today they show the digest only.
- **One model at a time,** as now. Several models, and choosing among them, are future work.

## Kept audit history

- **Where it goes.** The archive is a partition of its own, written through the volume service. The store can't hold
  it: the store is capped at 16 files and 32 blocks of 256 bytes, about 8 KB, while one audit sector is 256 KB (4,095
  records of 64 bytes).
- **The archiver** is a new confined program in the system image. It holds:
  - a reader capability on the policy store;
  - the archive partition, through a volume badge of its own;
  - an encryption key the keystore grants it, once, as it grants the wallet one (RFC-0035);
  - nothing else: no console, no network.

  It calls the keystore, but nothing calls the archiver, so no call can deadlock.
- **The protocol.** The policy store gains two operations:
  - `OLDEST_UNEXPORTED`: the oldest full sector not yet exported, as its records;
  - `EXPORTED(seq)`: that history up to `seq` is safely kept.

  The archiver asks for the oldest unexported sector, verifies its chain, seals it in chunks under its key, and writes it
  to the next slot of the archive. Once the write is read back, it reports `EXPORTED`. The policy store retires only
  history it was told is exported.
- **Waking it.** The policy store signals the archiver's notification when a sector fills, so the archiver doesn't poll.
- **If the archiver falls behind,** the ring can fill before export. The policy store then retires the sector anyway,
  and appends a `LOST` record naming the sequence range that wasn't kept (decided 2026-11-18). History can be lost, but
  never silently, and audited operations keep working.
- **Reading it.** The archiver also answers a reader badge, which the shell holds. `audit` at the console can then read
  exported history and verify it with the surviving ring, so the chain is checked across the boundary.
- **Capacity.** The archive partition holds a fixed number of exported sectors, for example 64 (16 MiB). Beyond that the
  oldest export is overwritten first, and the overwrite is recorded (decided 2026-11-18).

## Log formats ready for the hardware

The policy, decision and audit logs' headers gain a format version, a reserved MAC field, and a reserved counter field.
Version 1 logs still load and are migrated on first write. In this RFC the new fields are always zero and are never
checked. The hardware RFC fills them, and must state that nothing before it did.

## As implemented

Built as proposed, with these changes:

- **The loss is in `RETIRED`, not a separate `LOST` record.** The policy store already writes `RETIRED` as it erases the
  oldest sector. That record now carries how far history had been kept (`subject[1]`), so "retired" and "what wasn't kept"
  are one write a power cut can't separate. A `kept` past the last retired record means nothing was lost. Otherwise
  `kept` to the last retired were lost. If a power cut left it unknowable, it says so (`KEPT_UNKNOWN`), and never claims
  more was kept than was. Logs from before this RFC wrote 0 there, which is right: nothing was exported then. The console
  shows it in words.
- **The archive is its own partition, not the store.** The store is capped at 16 files and about 8 KB, while one audit
  sector is 256 KB. The archive is 64 slots of 576 disk sectors (18 MiB), each a plaintext header (range, chain head,
  generation, how far the archive has dropped history) and the sector sealed in 1,024 pieces of 256 bytes.
- **`EXPORTED` also carries what the archive dropped** to make room, since the archiver writes no records of its own.
- **The archiver is woken by a non-blocking send**, on every policy-store request while a full sector waits, not by a
  notification. A single-threaded program can't wait on a notification and on the shell's calls at once. A dropped wake-up
  costs nothing, because the archiver asks for work after everything it does.
- **The archiver's own sealing isn't recorded in the audit log.** Each export is 1,024 keystore calls, and recording them
  would add a quarter of a sector of records about the archive to every sector it keeps, and use up its grant's quota
  (1,024 uses) at once. The grant of its key is recorded, once a boot.
- **The policy store marks a sector kept before it writes `EXPORTED`.** With the ring full, writing `EXPORTED` retires the
  oldest sector, and that `RETIRED` must already count the sector just kept. A host test found the opposite order
  recording kept history as lost.
- **No log format changed for the hardware.** Every log already has a format version that its reader checks, refusing an
  unknown one, and reserved words that must be zero: seven in the audit log's sector header (room for a 64-bit counter and
  a 128-bit MAC), five in the revocation and decision logs'. The hardware RFC defines a new version with the fields it
  needs, and keeps reading this one. Writing unused fields now would be code nothing can test.
- **Models are signed with `lantern-mkmodel`,** which converts and signs, and can re-sign an existing package. The
  conversion's digest stays pinned in `scripts/get-model.sh`, as a check that the conversion is exact.

**Tested by:**

- `lantern-sdk`'s model tests: a package round-trips, any changed byte is refused, and app and model packages can't pass
  for each other.
- The policy store's tests:
  - exporting every sector loses nothing, through two turns of the ring;
  - not exporting records the exact loss;
  - the archiver can read and report only the sector offered;
  - a changed bit in an exported sector is caught;
  - a power cut mid-retirement never claims more was kept.
- The archiver's tests:
  - a sealed export reads back exactly, and holds no record in the clear;
  - any changed byte, a moved piece, a lying header or another key fails the whole sector;
  - a full archive drops its oldest export and says how far;
  - a power cut mid-write leaves an empty slot, not a false one.
- The console matrix:
  - a changed, an untrusted and an unsigned model are each refused;
  - with small audit sectors (the test-only feature `audit-ring-test`), history is kept, reads back after a reboot, is
    recorded as lost on a disk without an archive, and a changed byte in the archive shows nothing.

## Threat model impact  *(mandatory)*

- **Signed models:** a model is accepted only if a trusted publisher signed it. Someone who can rewrite the disk can still
  replace the publisher keys in the image, until there's secure boot (RFC-0023, "Boot chain").
- **Kept history:** exported history is sealed under the archiver's keystore key, a demo key until Phase 4's hardware
  (RFC-0023, "Encrypted storage"). It survives the ring wrapping. It doesn't survive someone who can rewrite the disk,
  any more than the ring does.
- **The archiver** is a new confined program. It can read the audit log, use one keystore key, and read and write the
  archive partition, and nothing else. Its keystore calls aren't in the audit log; its grant is.
- **No new protection is claimed for the logs.** Nothing here keys them.

## TCB impact  *(mandatory)*

None. Everything here is confined user space.

## Privacy impact

The audit log's model records now name a publisher. Exported history holds what the audit log already held: no content,
only the who, what and when of each grant use.

## Alternatives considered

- **Key the logs now with a per-install key sealed on the disk.** Rejected (decided 2026-11-18): it adds no protection
  against someone with the disk, and would look as if it did.
- **The policy store pushes exports.** Rejected: the keystore's and the store's own audited calls into the policy store
  would deadlock against it.
- **Exporting into the store.** Rejected: its capacity is about 8 KB, and one audit sector is 256 KB.
- **Separate publisher keys for models.** Rejected (decided 2026-11-18): models are trusted the way apps are.

## Unresolved questions

- The cap's value, sized against a default disk while implementing.

## Resulting ADRs

On acceptance, ADR-0043.
