---
adr: 0043
title: Signed model packages, and audit history kept in a sealed archive before it's retired
status: Accepted
date: 2026-11-19
deciders: ["TSC"]
rfc: ../rfcs/0038-signed-models-and-kept-audit-history.md
supersedes: []
superseded_by: null
---

# ADR-0043: Signed model packages, and audit history kept before it's retired

## Context

[RFC-0036](../rfcs/0036-phase-3-to-phase-4-transition.md) put the hardware-independent parts of rollback resistance and
signed models second in Phase 4. Without a hardware secret, keying the logs adds no protection against someone with the
disk, so [RFC-0038](../rfcs/0038-signed-models-and-kept-audit-history.md) built what has real value now. It has been
accepted, and this ADR is the durable record.

## Decision

- **Models are signed packages** (`lantern_sdk::model`, `LMD1`):
  - **Format:** a manifest naming the model, padded so the model starts on a 512-byte boundary, then the model,
    signed by its publisher with the app package's scheme. Neither kind of package can pass for the other.
  - **Checked against the app list:** the inference service uses a model only if a publisher in
    `lantern_abi::publishers::TRUSTED` signed it, the list the binder uses for apps.
  - **Recorded:** the audit log names the model's publisher.
  - **Made by** `lantern-mkmodel`. The image no longer holds a model digest.
- **Audit history is kept before it's retired:**
  - **The archiver,** a new confined program, takes each full audit sector the policy store offers, seals it in 256-byte
    pieces under its own keystore key, and writes it to a slot of the archive partition: 64 slots, 18 MiB. It then reports
    `EXPORTED`, with how far the archive has dropped history to make room. Each write clears the slot's header first and
    writes it last.
  - **`RETIRED` records what wasn't kept,** or "unknown" if a power cut lost track, and never claims more was kept than
    was.
  - **Waking:** the policy store wakes the archiver by a non-blocking send, and never calls it.
  - **Not self-audited:** the archiver's own sealing calls aren't recorded in the log they seal. Its grant is.
  - **Reading it:** `archive` at the console lists the exports and reads records from them, each sector checked whole
    first.
- **No log format changed for the hardware.** Every log's reader checks its format version and reserved words. The
  hardware RFC defines a new version with a MAC and a counter.

## Consequences

- **Easier:**
  - A model can be replaced without rebuilding the image, and the log says whose model ran.
  - History outlives the ring; when it can't, the loss is recorded exactly.
- **Harder:**
  - Fifteen programs in the system image.
  - A disk has an archive partition: one made without it (`--no-archive`, or an older disk) loses retired history, as
    before, and says so.
  - Exports cost about 1,024 keystore calls per 4,095 audit records.
- **Not protected, still:**
  - Someone who can rewrite the disk can replace the publisher keys in the image (secure boot: RFC-0023's "Boot chain"),
    and can rewrite or erase the logs and the archive (a key and a counter in hardware: RFC-0023's "Audit log" and
    "Policy store").
  - The archive's key is a demo key, like the store's.
