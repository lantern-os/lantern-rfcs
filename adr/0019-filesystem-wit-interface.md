---
adr: 0019
title: A capability-scoped filesystem WIT interface for lantern-runtime
status: Accepted
date: 2026-08-29
deciders: ["TSC"]
rfc: ../rfcs/0016-filesystem-wit-interface.md
supersedes: []
superseded_by: null
---

# ADR-0019: A capability-scoped filesystem WIT interface for lantern-runtime

## Context

[ADR-0018](./0018-wit-handle-capability-mapping.md) fixed the WIT-handle ⇄ capability
mapping and its two worked interfaces, and **deferred `wasi:filesystem`**: that interface
is built around a POSIX directory hierarchy (`open-at(path: string)`, `readdir`) that
`lantern-filesystem`'s flat, content-addressed `Store` has no concept of. It named two
options for a future RFC — **(a)** a small custom interface shaped like `Store`, or **(b)**
wait for `Store` to grow a path layer — and refused to guess.

The filesystem capability is the missing half of the Phase 2 exit criterion ("reads a file
*only* via a granted capability, demonstrated adversarially"). `Store` is real and
badge-gated; `lantern-runtime` has a proven resource-scoped mapping
([`lantern:crypto/keystore`](./0018-wit-handle-capability-mapping.md)). Only the interface
and the (a)/(b) decision were missing. [RFC-0016](../rfcs/0016-filesystem-wit-interface.md)
proposed (a) and has been accepted; this ADR is the durable record.

## Decision

**Adopt option (a): a new, LanternOS-owned `lantern:host/filesystem` WIT interface, shaped
like `Store`'s actual object model — no paths, no directories, no listing.**

- **`resource file`** with two methods: `read: func() -> result<list<u8>, error-code>`
  (the file's entire current content — v0 `Store` is one block per file) and
  `write: func(bytes: list<u8>) -> result<_, error-code>` (replaces the content; an
  oversized write is `error-code::invalid`, never partial).
- **`open: func(slot: u32) -> option<file>`** is the only way a component obtains a `file`
  handle. `slot` indexes the capability manifest's explicit grant list — not a path, not a
  namespace. `none` for an ungranted slot.
- Each `file` handle is backed host-side by a `HostCapability` carrying a badge and a
  `FileId` (never a raw kernel `CPtr`, never a path). `read`/`write` forward to
  `lantern-filesystem`'s real `Store::read`/`Store::write` (via a `FilesystemService`
  trait, in-process today — the store is not yet a confined process), which re-checks the
  badge on every call. Denied / revoked / wrong-`FileId` / unknown-badge all relay as
  `error-code::access`, verbatim from `Store::check_access` — the mapping adds no check of
  its own. AEAD failures and oversized writes are `error-code::invalid`.
- **A confined guest cannot create, destroy, or enumerate files** through this interface —
  only operate on the handles `open` returns, exactly as it cannot generate a key through
  `lantern:crypto/keystore`. File creation and `role` → `FileId` binding are the launch
  binder's job (`lantern-shell`, per RFC-0015).
- `wasi:filesystem` **remains deferred, not foreclosed.** If `Store` ever grows a real
  directory model, adopting the standard interface then is a separate decision.

Rejected alternatives (full reasoning in the RFC): force-fitting `wasi:filesystem` via a
path-emulation shim (ambient-authority-shaped `open-at`); waiting for a `Store` path layer
that is not on its roadmap; `read-at`/`write-at` from the start (nothing to offset into in
a one-block store); a `directory` resource (`Store` has no grouping concept); reusing
`keystore::key`'s host type for files (collapses the R5 type distinction).

## Consequences

- **Easier:** the Phase 2 exit criterion's filesystem half is now buildable. A confined
  component can read a file it was granted and be denied one it was not, demonstrably.
  RFC-0015's manifest gains a second resource-scoped entry type (`[[filesystem.file]]`)
  with no format change — confirming the resource-scoped shape generalises before the
  manifest format is fixed.
- **Harder / committed to:** `lantern:host/filesystem` is now LanternOS-owned surface to
  version and maintain. Every `read`/`write` is committed to a service round trip once
  `Store` is a confined process (unbenchmarked — the same latency risk RFC-0014 flagged).
  Whole-file `read`/`write` is a v0 shape that must gain `-at` companions when `Store`
  grows chunking rather than change.
- **Trust boundary:** none moved (ADR-0004 TCB unchanged). `lantern-runtime` gains a
  normal dependency on `lantern-filesystem`; no TCB dependency, no new adversary
  capability — a `FileId`-by-handle interface has no namespace to probe.
- **Still open (tracked in the RFC / `lantern-filesystem/STATUS.md`):** `read` return-size
  bound and `read-at`; write durability / `flush` semantics (tied to `Store`'s unbuilt
  persistence story); interface-time revocation; the binder's "create a file for this
  role" flow for write-capable apps; version-history exposure.
