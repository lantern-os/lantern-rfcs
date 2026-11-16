---
adr: 0038
title: Provenance for data at rest — who wrote each file, kept with the file
status: Accepted
date: 2026-11-15
deciders: ["TSC"]
rfc: ../rfcs/0033-provenance-for-data-at-rest.md
supersedes: []
superseded_by: null
---

# ADR-0038: Provenance for data at rest — who wrote each file, kept with the file

## Context

Phase 3's exit criteria include data at rest that is "encrypted and provenance-tracked". The store's files were sealed at
rest (ADR-0036), but nothing recorded what wrote them. [RFC-0033](../rfcs/0033-provenance-for-data-at-rest.md) proposed
recording it with each file. It has been accepted; this ADR is the durable record.

## Decision

**Every write records its principal, the store's write number and its start number. Each file keeps its last 4, in the
sealed image, and `files` at the console shows them.**

- **Principals.**
  - A principal is 48 bits of BLAKE3 over a verified package's publisher key and app name
    (`lantern_launch_binder::principal`), never 0. Only the binder computes one.
  - `GRANT` carries it in the object word, above the object's 16 bits (`lantern_abi::provenance::grant_object`).
  - The store records it with the grant (`Store::request_file_access_for`). A grant the binder didn't make records 0,
    shown as "unattributed".
- **Entries.**
  - Each successful write records `{principal, write, boot}`. The write number is the store's own count of writes, and
    the boot number its count of starts (`Store::start`). No wall-clock time: there's no trusted clock.
  - Each file keeps its last 4 entries, newest first, and its current content only.
  - Entries are in the sealed image (format `LSTORE02`), so they're as authentic as the file. `LSTORE01` images still
    load, with no history.
- **Showing it.**
  - The shell holds the store under `STORE_READER_BADGE`, which answers `FILE_INFO` (size and entry count) and
    `FILE_ENTRY`, never contents.
  - The binder names a principal (`CONTROL_PRINCIPAL`) by finding the verified package that is it. A principal no package
    on the disk matches shows as itself.
  - `files` lists each file with its writers.

## Consequences

- **Easier:** a person can see what wrote their data. The AI agent's outputs will carry the agent's name from their first
  write.
- **Harder / committed to:**
  - Every grant carries a principal.
  - The store's image grows by about 1.3 KB.
  - Provenance lives and dies with the store's image: it can be rolled back with it (RFC-0031), and is only as
    trustworthy as the store and the binder.
- **Not done:**
  - Versioned content (history).
  - The model behind an agent's output.
  - Provenance across the network.
