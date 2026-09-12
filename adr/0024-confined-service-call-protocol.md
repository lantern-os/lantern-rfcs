---
adr: 0024
title: The confined-service call protocol — shared-Frame framing, keystore and store wire formats
status: Accepted
date: 2026-09-12
deciders: ["TSC"]
rfc: ../rfcs/0019-confined-service-call-protocol.md
supersedes: []
superseded_by: null
---

# ADR-0024: The confined-service call protocol — shared-`Frame` framing, keystore and store wire formats

## Context

[ADR-0022](./0022-confined-service-model-and-call-transport.md) fixed that a confined
component's host calls reach `Keystore` / `Store` over a badged endpoint capability plus a
shared `Frame`, and explicitly deferred "the exact `label` values and `mr` / shared-region
layout for `keystore` (SIGN/ENCRYPT/DECRYPT) vs `store` (READ/WRITE)" as a per-service
follow-up — the same way [ADR-0018](./0018-wit-handle-capability-mapping.md) fixed the two
WIT mapping *shapes* and left each interface to be worked separately.

[RFC-0019](../rfcs/0019-confined-service-call-protocol.md) is that follow-up. It matters
before the first porting PR, not inside it, because a confined service's request parser is
a **trust boundary**: it runs on bytes a possibly-compromised runtime wrote into shared
memory, and a sloppy parser (trusting a length, reading past a buffer, acting before
`check_access`) reintroduces exactly the confused-deputy and TOCTOU classes the whole
confinement effort exists to close ([Principle 1](../../lantern-docs/wiki/Principles.md)).
It has been accepted. This ADR is the durable record.

## Decision

**One shared framing over the badged-endpoint + shared-`Frame` transport, reused by every
confined service — not a bespoke protocol per service.**

- **One 4 KiB `Frame` per `(runtime process, service)` pair**, mapped RW into both VSpaces
  at launch time (RFC-0018 Part 1), holding at most one outstanding request-then-reply at
  a time (confined programs are single-threaded, ADR-0010 — no sequence numbers, no
  locking). Ownership (runtime ↔ service) between `Call` and `Reply` is a discipline, not
  enforced — the service's mandatory copy-in (below) is what makes a violation harmless.
- **A fixed 16-byte header** at the start of the `Frame` (`FRAME_PAYLOAD = 4080` bytes
  follow): request = `{ op: u16, flags: u16 (bit0 = MORE), arg_len: u32, offset: u64 }`;
  reply = `{ status: u16, flags: u16 (bit0 = EOF), ret_len: u32, _reserved: u64 }`. The
  message tag's `mr1` redundantly carries `arg_len`/`ret_len` and the tag `label` carries
  `op`, as a cheap early sanity gate before the receiver touches shared memory; a mismatch
  against the `Frame`-resident header (authoritative) is `INVALID`.
- **A four-value error map** every service computes internally and the protocol just
  carries: `OK`, `ACCESS` (denied/revoked/wrong-object — `check_access`'s own verdict,
  never invented by the runtime, the same ADR-0018 rule now enforced across a real
  boundary), `INVALID` (malformed request), `FAILED` (the operation itself failed — AEAD
  tag mismatch, key destroyed).
- **Chunking via `offset`/`MORE`/`EOF`** for any payload larger than one `Frame` — fixed
  offsets (not length-prefixed streaming) so a service can pre-size its reassembly buffer
  and reject a bad total immediately rather than growing a buffer as chunks arrive.
- **`keystore`**: `op ∈ {SIGN=1, ENCRYPT=2, DECRYPT=3}`; the badge alone identifies
  `(KeyId, KeyOps)` — no key argument on the wire. `DECRYPT` returns an **empty** reply
  payload on tag mismatch (`FAILED`), closing a partial-plaintext oracle the in-process
  trait boundary never had to consider.
- **`store`**: `op ∈ {READ=1, WRITE=2}`; the badge identifies `(FileId, FileOps)`. `READ`
  is request-header-only with `offset`; `WRITE` chunks with `MORE`; v0 has no
  chunk-boundary durability guarantee (matches the in-process `Store` today).
- **The service copies the entire logical argument into a private buffer, across all
  chunks, before `check_access` and before any crypto/IO** — a runtime that shrinks
  `arg_len` on a later chunk gets `INVALID`, never a half-processed operation. This is the
  concrete TOCTOU mitigation the Context section calls out.
- **A new `lantern_abi::frame` module**: `Header` (the 16-byte codec, pure and
  host-testable) and a `Channel { endpoint, frame }` helper implementing "write
  header+payload, `Call`, read reply header+payload" with the chunk loop — non-TCB (it
  marshals; it enforces nothing), used by both `IpcKeystore`/`IpcFilesystem` on the
  runtime side and each service's own `Recv` loop, so the framing is reviewed once.

Full byte-level layouts, the worked keystore/store request/reply shapes, and the rejected
alternatives (a kernel in-memory IPC buffer, a bespoke protocol per service, a generated
RPC/IDL layer, length-prefixed streaming, an always-register-only fast path) are in the
RFC; this ADR does not restate them.

## Consequences

- **Easier:** `Keystore`/`Store` (and later `lantern-network`, an audit service) share one
  reviewed framing instead of each inventing its own; the runtime side is one
  `marshal → Call → unmarshal` helper parameterised by interface.
- **Harder / committed to:** every confined service's `Recv` loop must implement the
  copy-in-before-validate discipline correctly — this ADR fixes the *format*, not the
  service-side review burden, which is real and per-service.
- **TCB impact: none.** The framing structs and the `Channel` helper live in `lantern-abi`
  (non-TCB, ADR-0022). `Keystore`/`Store` request-loop code is confined user-space. No new
  kernel object, syscall, or `lantern-hal` surface — a convention layered on the existing
  `Call`/`Reply` plus a shared `Frame` mapped with `FrameInvoke::Map`. The kernel's
  designed-but-unbuilt in-memory IPC buffer remains unbuilt; ADR-0022 already chose shared
  memory over it and this ADR does not revisit that choice.
- **Open, carried into implementation (not blocking acceptance):** a distinct WIT
  `error-code` for `FAILED` vs. malformed-request (v0 collapses both to `invalid`); the
  `store` WRITE total-size check (v0 caps late, on the chunk that would exceed); whether
  4 KiB is the right `Frame` size once crypto-in-a-loop benchmarking exists; one badged
  endpoint per handle vs. per service (v0: per handle, falling out of ADR-0022's mint
  model); fault signalling when a service panics mid-call (folds into RFC-0018's open
  runtime-process fault-handling policy — the service-death case wants the launcher to
  notice and fail the runtime's outstanding call).
- **Unblocks:** `Keystore`/`Store` can now be written as real `Recv`-loop confined
  programs against a fixed wire format — the remaining ADR-0022 Part 1 work, tracked in
  `lantern-abi`/`lantern-crypto`/`lantern-filesystem`/`lantern-runtime`/`lantern-capabilities`
  `STATUS.md`.

## Privacy impact

Positive, marginally: `DECRYPT`'s empty-payload-on-tag-mismatch rule closes a
partial-plaintext oracle. Otherwise unchanged — the same bytes cross the same boundary,
now with a 16-byte header.
