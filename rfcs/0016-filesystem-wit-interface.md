---
rfc: 0016
title: A capability-scoped filesystem WIT interface for lantern-runtime
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["runtime", "filesystem"]
domains: ["runtime", "filesystem", "capabilities", "sdk"]
created: 2026-08-29
updated: 2026-08-29
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0016: A capability-scoped filesystem WIT interface for lantern-runtime

> **Accepted 2026-08-29.** Resolves [RFC-0014](./0014-wit-handle-capability-mapping.md)'s
> deferred filesystem choice in favour of option (a) — a custom `lantern:host/filesystem`
> interface. Fixed by [ADR-0019](../adr/0019-filesystem-wit-interface.md); implemented in
> `lantern-runtime` the same round (see `lantern-runtime/STATUS.md`).

## Summary

[RFC-0014](./0014-wit-handle-capability-mapping.md) fixed the WIT-handle ⇄ capability
mapping and deliberately **did not** map `wasi:filesystem`, because that interface's
path/directory model does not fit `lantern-filesystem`'s flat, content-addressed `Store`.
It named two options for a future RFC: **(a)** a small custom `lantern:filesystem`
interface shaped like `Store`'s actual object model, or **(b)** wait for `Store` to grow a
path layer worth adopting real `wasi:filesystem` for. This RFC picks **(a)** and designs
it, exactly mirroring RFC-0014's `lantern:crypto/keystore` treatment: a resource-scoped
`file` handle backed by a `HostCapability` (badge + `FileId`), obtained only via
`open(slot)` from the capability manifest, with `read`/`write` forwarding to
`lantern-filesystem`'s real `Store::read`/`Store::write` and relaying its deny-by-default
check as `error-code::access`. No paths, no directories, no `open-at` on a guest-supplied
string. It is the missing half of the Phase 2 exit criterion — "reads a file *only* via a
granted capability, demonstrated adversarially" ([Roadmap](../../lantern-docs/wiki/Roadmap.md)).

## Motivation

The Phase 2 exit criterion has two capability halves: a key (RFC-0014, done — the
`lantern-example-signer` demo signs via `lantern:crypto/keystore`) and a **file**. Nothing
yet lets a confined component read a file at all. `lantern-filesystem`'s `Store` is real
(content-addressed, `FileId`-keyed, AEAD-encrypted, badge-gated through a `Broker` — the
same shape `Keystore` established), and `lantern-runtime` has a proven resource-scoped
mapping pattern. The only missing piece is the WIT interface between them, and the
decision RFC-0014 explicitly deferred: which of its two options to take.

Deciding **(a)** now, rather than force-fitting `wasi:filesystem` in an implementation PR,
serves [Principle 1](../../lantern-docs/wiki/Principles.md) (security by architecture — "no
ambient authority, no global namespaces"): `wasi:filesystem`'s `open-at(path: string)`
mints a fresh grant keyed by a string the *guest* supplies, which is much closer to
ambient authority than a granted capability. A `FileId`-by-handle interface has no
namespace for a guest to walk — it can reach exactly the files whose handles the manifest
put in its table, and nothing else. This is the same trust-boundary reasoning
`GOVERNANCE.md` reserves for an RFC, and the same reasoning RFC-0014 used to defer rather
than guess.

Choosing **(a)** over **(b)**: option (b) waits on a path/directory layer for `Store` that
`lantern-filesystem/STATUS.md`'s own "Next" list does not contain and that the long-term
[Contexts-over-files](../../lantern-docs/wiki/Intent-Model.md) vision may make the wrong
thing to build at all. A small custom interface shaped like what `Store` *is* today
unblocks the exit criterion now, adds no surface that has to be walked back if `Store`'s
model changes, and follows the precedent `lantern:crypto/keystore` already set (a
LanternOS-owned interface where no WASI one fits).

## Guide-level explanation

A component that needs to read or write a file declares it in its capability manifest
(RFC-0015 shape) exactly as it declares a key:

```toml
[[filesystem.file]]
role          = "config"
ops           = ["read"]
justification = "Reads its configuration blob at startup."
```

At launch the user binds `"config"` to a concrete file (a `FileId` in the store), the
owning filesystem service mints a badge scoped to that file and `read`, and the runtime
puts one `HostCapability` in the component's resource table. The component obtains the
handle the only way there is:

```wit
let file = filesystem::open(0);   // Some(file) — slot 0 was granted
let bytes = file.read();          // Ok(list<u8>) — the file's current content
```

`open(1)` on a slot the manifest left empty returns `none`. `file.write(...)` on a handle
whose badge was granted only `read` returns `error-code::access`, forwarded verbatim from
`Store::check_access`. There is no `filesystem::open("some/path")` — the interface has no
string-keyed lookup, no directory listing, and no way to name a file the manifest did not
already grant.

## Reference-level explanation

### The interface (new, LanternOS-owned)

Added to `lantern-runtime`'s `wit/host.wit`, `package lantern:host@0.1.0`:

```wit
interface filesystem {
    /// Relayed verbatim from the owning store's own deny-by-default check.
    enum error-code {
        /// The badge behind this handle was denied by the store: unknown, revoked,
        /// scoped to a different file, or not granted this operation.
        access,
        /// The arguments were malformed (e.g. a write larger than the store's block
        /// size) or the store's own AEAD layer rejected them.
        invalid,
    }

    /// A capability to one file in the store, scoped to the operation subset the
    /// manifest granted. Methods forward to the store, which re-checks the badge on
    /// every call. A component cannot create, destroy, or enumerate files — only
    /// operate on the handles `open` returns.
    resource file {
        /// The file's entire current content. v0 `Store` holds one block per file, so
        /// this is bounded by the store's block size; a future chunked `Store` will add
        /// a `read-at(offset, len)` companion rather than change this.
        read: func() -> result<list<u8>, error-code>;

        /// Replaces the file's entire content. Bounded by the store's block size in v0
        /// (an oversized write is `error-code::invalid`, not a partial write).
        write: func(bytes: list<u8>) -> result<_, error-code>;
    }

    /// The ONLY way a component obtains a `file`. `slot` indexes the capability
    /// manifest's explicit grant list — not any path or namespace. `none` for a slot
    /// the manifest left empty.
    open: func(slot: u32) -> option<file>;
}
```

`world app` in `wit/host.wit` gains `import filesystem;` alongside the existing two.

### Host side

Identical structure to `lantern:crypto/keystore`:

- **`HostCapability`** gains a second constructor / variant. A `file` handle's record is
  `(badge, FileId, ServiceEndpoint::Filesystem)`, the same `HostCapability` type extended
  with a `FileId` alternative to its `KeyId` — or a small enum inside it. (Implementation
  detail; the RFC fixes only that a `file` handle carries a badge and a `FileId`, never a
  raw kernel `CPtr` and never a path.)
- **`FilesystemService`** trait, the store as the mapping sees it — `read(&self, badge,
  file, &mut buf) -> Result<usize, StoreError>` and `write(&mut self, badge, file, &[u8])
  -> Result<(), StoreError>`. Implemented for a wrapper holding the real
  `lantern_filesystem::Store` plus the `lantern_crypto::Keystore` its AEAD key lives in
  (`Store::read`/`write` both take a `&Keystore`), threaded internally so the trait method
  signatures stay clean. A test double implements the same trait. Note `write` needs
  `&mut` where every `keystore` method was `&self` — the generated `HostFile` host-trait
  methods are already `&mut self`, so this is free.
- **`RuntimeState`** gains `filesystem: Option<Box<dyn FilesystemService>>` and
  `files: Vec<HostCapability>` (or the manifest's file grants), exactly like `keystore` /
  `keys`.
- **`filesystem::open`** — `self.files.get(slot).map(|cap| self.table.push(*cap))`.
- **`file.read`** — look up the `HostCapability`, forward `badge`/`FileId` to
  `FilesystemService::read` into a `MAX_BLOCK_LEN`-sized buffer, return the bytes;
  translate `StoreError` → `error-code` (denied/revoked/wrong-file/unknown → `access`;
  crypto/oversize/wrong-purpose → `invalid`) — the same table as keystore's `to_error_code`,
  over `StoreError` instead of `KeystoreError`.
- **`file.write`** — reject `bytes.len() > MAX_BLOCK_LEN` as `invalid` before forwarding
  (mirrors keystore's nonce-length pre-check), then forward to `FilesystemService::write`.
- **`build_linker`** — link `filesystem` when the manifest grants at least one file, same
  rule as `keystore` today.

### What this RFC does not decide

- **`Store` growing chunking / a path layer / per-object keys** — all `lantern-filesystem`'s
  own "Next", untouched here. When chunking lands, `read`/`write` gain `-at` companions;
  the handle model does not change.
- **File creation and lifecycle from a confined guest.** A guest never creates or destroys
  a file through this interface — it only operates on granted handles, same as it never
  generates a key through `lantern:crypto/keystore`. Who creates files, and how the
  manifest binder resolves a `role` to a `FileId`, is the binder's job (`lantern-shell`,
  per RFC-0015).
- **Immutable version history / provenance** (`lantern-filesystem/ARCHITECTURE.md`'s
  fourth pillar) — `read` returns current content only. A `history` sub-interface is
  future work.
- **`wasi:filesystem` adoption** — still deferred. If `Store` ever grows a real directory
  model, adopting the standard interface then is a separate decision; this RFC does not
  foreclose it, it just does not wait for it.
- **Resource-scoped call latency** — same unbenchmarked question RFC-0014 already flagged;
  a `read` is now also a service round trip once the store is a confined process.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none moved. Identical to RFC-0014 — the mapping lives in
  `lantern-runtime`'s confined runtime-role process and forwards to an already-trust-
  boundaried service (`Store`), it does not move where that boundary sits.
- **New assets introduced and who can reach them:** `file`-handle `ResourceTable` entries,
  as sensitive as the badge they carry; reachable only by the one component instance
  Wasmtime scoped the table to, and the runtime-role host code. File *content* now flows
  through a confined guest — inherent to granting file access, and bounded to exactly the
  files whose handles the manifest granted.
- **New adversary capabilities, if any:** none. No operation manufactures a `file` handle
  from guest data; `open` indexes an explicit grant list; there is no path or directory
  namespace to traverse. A guest granted `read` on file X cannot name file Y (its handle's
  badge is scoped to X; `Store::check_access` rejects a mismatched `FileId` as `WrongFile`
  → `access`).
- **Mitigations:** deny-by-default forwarding relayed from `Store`'s own check; per-call
  re-check (the mapping caches no "allowed" decision); oversize-write pre-check before the
  service is consulted; `filesystem::file` is its own host resource type, not type-confusable
  with `keystore::key` (R5).
- **Net change to attacker surface:** this is surface ADR-0003 already accepted for
  capability-backed WASI; **reduces** relative to the alternative of force-fitting
  `wasi:filesystem`, whose string-keyed `open-at` is a namespace a compromised guest could
  probe. A `FileId`-by-handle interface has nothing to probe.

Cross-reference [`Threat-Model.md`](../../lantern-docs/wiki/Threat-Model.md) and
`lantern-filesystem/THREAT_MODEL.md` (F6 — plaintext-content linkability by address, an
existing acknowledged v0 cost, unchanged here).

## TCB impact  *(mandatory)*

- **Does this add code to the Trusted Computing Base?** No. The interface and its host
  binding live entirely in `lantern-runtime`'s confined runtime-role process (ADR-0004's
  TCB boundary — kernel + HAL + boot — is unchanged).
- **Does this add a dependency to the TCB?** No. `lantern-runtime` gains a normal
  (non-TCB) dependency on `lantern-filesystem`, already a sibling Phase 2 crate.
- **Effect on TCB size and auditability:** neutral. The `lantern:host/filesystem` WIT is
  new host-defined, reviewed surface, not guest-supplied; it does not touch the TCB.

## Privacy impact

File content is user data the component was explicitly granted access to; the interface
adds no telemetry and no metadata channel. The `FileId`-by-handle model is privacy-positive
versus `wasi:filesystem`: the package and the guest never see a path, a directory listing,
or any file they were not granted, so a shared package leaks nothing about the installing
user's file layout. `lantern-filesystem`'s existing F6 acknowledgement (identical plaintext
→ identical content address → linkable) is unchanged by exposing `read`/`write` — it was
already a property of the store.

## Alternatives considered

- **Force-fit `wasi:filesystem` now (RFC-0014's option, via a path-emulation shim).**
  Rejected again, same reasoning: inventing a path model `Store` does not have, and
  turning `open-at(string)` into a capability mint keyed by guest input — ambient-authority-
  shaped, the opposite of ADR-0003.
- **Wait for `Store` to grow a path layer (RFC-0014's option (b)).** Rejected: not on
  `lantern-filesystem`'s roadmap, possibly the wrong thing to build given the Contexts
  vision, and it blocks the Phase 2 exit criterion indefinitely on speculative work.
- **One `read-at(offset, len)` / `write-at` from the start** instead of whole-file
  `read`/`write`. Rejected for v0: `Store` is one block per file, so offsets have nothing
  to address yet; adding them now would be untested API shaped for a `Store` that does not
  exist. They are the natural first addition when chunking lands.
- **A `directory` resource** (even without real directories, as an organising handle).
  Rejected: `Store` has no grouping concept; a `directory` with one operation
  (`open-file`) is just `open(slot)` with extra steps and a namespace to get wrong.
- **Reuse `keystore::key`'s host type for files too** (one universal handle). Rejected for
  the same reason RFC-0014 rejected a universal `capability` resource: it collapses the
  type-level distinction the component model gives for free (R5).

## Prior art

- **RFC-0014 / `lantern:crypto/keystore`** — this RFC is that pattern applied a second
  time; the value of a second instance is confirming the resource-scoped shape generalises
  before RFC-0015's manifest format bakes in assumptions about it.
- **Fuchsia's `fuchsia.io`** — capability-routed file access where a component gets a
  `Directory`/`File` channel it was handed, not a path it opens. The `File`-you-were-given
  model, minus the directory tree Phase 2 does not have.
- **CloudABI / Capsicum** — `openat` only relative to a directory fd you already hold, no
  global root. The same "no ambient namespace" principle; this RFC goes further by having
  no path component at all in v0.
- **`wasi:filesystem@0.2` `descriptor`** — studied and rejected (RFC-0014), cited here for
  what a future path-capable `Store` might adopt.
- **IPFS / content-addressed stores** — `Store`'s own model; addressing by content hash,
  not location, is why there is no natural path namespace to expose.

## Unresolved questions

- **`read` return-size bound.** v0 files are ≤ `MAX_BLOCK_LEN` (256 B), so `read` returning
  the whole file is fine. The interface must not imply unbounded reads; whether to document
  a hard cap in the WIT (`list<u8>` with a comment) or add `read-at` immediately is open —
  this RFC leans "whole-file `read` now, `read-at` with chunking."
- **Write atomicity / durability semantics.** `Store::write` replaces content
  synchronously; whether the WIT should ever expose `flush`/`sync` (or whether the store's
  guarantee is always "returned = durable") is undecided and tied to `Store`'s own
  persistence story (not yet designed — it is in-memory today).
- **Interface-time revocation** — same open question as RFC-0014; a handle stays live for
  the component's lifetime once opened.
- **Manifest `role` → `FileId` binding for files that do not exist yet** (an app's first
  run creating its own config). Needs the binder + a "create on first bind" affordance;
  RFC-0015's concern, noted here as a real gap for write-capable apps.

## Future possibilities

- `read-at` / `write-at` / a `size` accessor when `Store` grows chunking.
- A `history` sub-interface exposing `lantern-filesystem`'s version pillar (read prior
  versions of a granted file).
- Streaming (`wasi:io/streams`-shaped) reads for large files, once there are large files.
- The manifest-binder "create a new file for this role" flow for write-capable apps.
- If `Store` ever grows a directory model: a separate RFC to adopt real `wasi:filesystem`,
  which this interface's existence does not block.

## Resulting ADRs

On acceptance, an ADR will record: RFC-0014's deferred filesystem choice resolved in
favour of **(a)** a custom `lantern:host/filesystem` interface; its resource-scoped shape
(`file` handle backed by badge + `FileId`, `open(slot)` the only acquisition path,
whole-file `read`/`write` in v0); the explicit exclusion of paths, directories, listing,
and guest-driven file creation; and `wasi:filesystem` remaining deferred, not foreclosed.
