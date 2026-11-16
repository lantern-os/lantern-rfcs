---
rfc: 0033
title: Provenance for data at rest — who wrote each file, kept with the file
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["filesystem", "shell", "boot"]
domains: ["filesystem", "shell", "abi", "boot", "docs"]
created: 2026-11-15
updated: 2026-11-15
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-15, implemented and recorded in [ADR-0038](../adr/0038-provenance-for-data-at-rest.md).** One change
> from section 3: the boot number is the store's own count of its starts, kept in its image, rather than the policy
> store's `BOOT` count, which the store has no capability to read.
>
> **Direction agreed 2026-11-15.** Settled as proposed: each file keeps its last 4
> writers, and a `files` command shows them through a metadata-only store reader.

# RFC-0033: Provenance for data at rest — who wrote each file, kept with the file

## Summary

Phase 3's exit criteria include **"data at rest is encrypted and provenance-tracked."** The store's files are sealed at
rest (RFC-0031). This RFC records **who wrote each file**, keeps it with the file on the disk, and shows it to the person:

```
lantern> files
  file 0  71 bytes   written by "confined-fs-probe" (publisher 927308f5..00b7), write 12, boot 3
          before:    "confined-fs-probe" (927308f5..00b7), write 9, boot 2
```

A writer is a **principal**: the verified package's publisher key and app name, which only the binder knows. The binder
names it to the store with each grant, the store records it with every write that grant makes, and the sealed image keeps
it.

## Motivation

- It's a Phase 3 exit criterion.
- An agent's outputs must carry which agent produced them ([wiki/AI](../../lantern-docs/wiki/AI.md)'s "Provenance"). The
  next step, the local AI agent, writes files; this makes every such file say which agent wrote it, from the start.
- The audit log records each use of a grant, but its ring retires old records, and it says nothing about what a file holds
  now. Provenance belongs with the data.

Principles served: **transparency and accountability** (a person can see what wrote their data), and **security by
architecture** (the writer is named by the binder, from a verified package, never by the writer itself).

## Reference-level explanation

### 1. Principals

A principal is the first 8 bytes of BLAKE3 over a verified package's publisher key and app name. The binder computes it,
and nothing else can: the store only ever learns it from the binder. Two packages from the same publisher with the same
app name are the same principal, whatever their version.

### 2. The grant names its principal

`GRANT` gains the principal, in the object word's upper 48 bits, beside the object's own 16. A service that doesn't track
provenance ignores it, as it ignores the channel. The binder sends it for every grant it makes.

### 3. The store records it

- Each grant record holds its principal. A grant made without one (the demos' first-caller grants) records principal 0,
  shown as "unattributed".
- Each successful write records a **provenance entry** on the file: the principal, the store's write number, and the boot
  number (the policy store's `BOOT` count, which the store reads at start).
- A file keeps its **last 4 entries** (unresolved question 1). Only the current content is kept; earlier entries say who
  wrote what came before, not what it was.
- Every entry is in the sealed image, so it's as authentic as the file: only the store's key could have made it.

### 4. The person sees it

- A new `files` command at the console lists each file: its size, and its provenance, newest first.
- The shell reads it through a new **store reader** badge, metadata only. It can see sizes and provenance, never contents.
- The shell names each principal by asking the binder, which looks it up among the packages it verified. A principal no
  package on the disk matches shows as its publisher prefix alone.

### 5. The proof

- Host tests: entries are recorded per write, the oldest dropped after 4, saved and loaded with the image, and a grant's
  principal can't be changed by its holder.
- `console-matrix.sh`:
  - `files` after the filesystem app shows it as the writer;
  - after a reboot, it still does;
  - a second write adds an entry;
  - a file never written shows no writer.
- Mutation tests: a store that records the wrong principal, and one that drops entries on save.

## Threat model impact

- **Provenance can't be forged by an app**: the principal comes from the binder, which takes it from a verified package.
  The store records it; the writer never supplies it.
- **Provenance is as trustworthy as the store and the binder.** A compromised store can write anything into its own image.
- **Privacy:** provenance says which apps wrote which files, to whoever is at the console, as the audit log already does.
  It never leaves the device.
- **Rollback** of the store's image (RFC-0031) rolls provenance back with it.

## TCB impact

None: the binder, the store and the shell are all outside it.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** Unchanged.
2. **Can an agent use this without being unnecessarily trusted?** An agent can't name itself as anything else, nor write
   without being named.
3. **Same interface for humans and agents?** Yes: every app's writes are attributed the same way, and an agent is an app.

## Alternatives considered

- **Derive provenance from the audit log.** No new record, but the log retires old records, and a person asking "who wrote
  this?" would need the whole history.
- **The app names itself in each write.** Simple, and worthless: any app could claim to be any other.
- **Keep every version's content (full history).** The wiki's direction, but the store holds 32 small blocks; it would fill
  at once. Provenance now, history with a larger store.

## Unresolved questions

1. **How many entries per file:** this RFC proposes the last 4.
2. **What a write records:** principal, write number and boot number, as proposed; a wall-clock time needs a real-time
   clock and a trusted time source, which LanternOS has neither of.

## Future possibilities

- Versioned content (history) once the store grows.
- The model that produced an agent's output, recorded beside the agent, once an inference service can attest to it.
- Provenance carried across the network with sealed capabilities.

## Resulting ADRs

On acceptance, an ADR will record: principals, the grant's principal, the provenance entry and its depth, the store reader,
and the `files` command.
