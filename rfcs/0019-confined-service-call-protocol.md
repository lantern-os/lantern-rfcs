---
rfc: 0019
title: The confined-service call protocol — shared-Frame framing, and the keystore and store wire formats
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["runtime", "crypto", "filesystem", "capabilities"]
domains: ["runtime", "crypto", "filesystem", "capabilities", "abi"]
created: 2026-09-06
updated: 2026-09-06
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0019: The confined-service call protocol — shared-`Frame` framing, and the keystore and store wire formats

## Summary

[ADR-0022](../adr/0022-confined-service-model-and-call-transport.md) fixed that a confined
component's host calls reach `Keystore` / `Store` over **a badged endpoint capability plus a
shared `Frame`**, and deliberately left "the exact `label` values and `mr` / shared-region
layout for `keystore` (SIGN/ENCRYPT/DECRYPT) vs `store` (READ/WRITE)" as a per-service
follow-up, "the way [RFC-0014](./0014-wit-handle-capability-mapping.md) fixed the two
mapping *shapes* and left each interface to be worked separately." This RFC is that
follow-up. It fixes:

1. **A common framing** over the shared `Frame` — a small fixed request/reply header, a
   payload region, the `label` and error conventions every confined service reuses. This
   is the concrete form of ADR-0022's transport, shared so `Keystore` and `Store` (and
   later `lantern-network`, an audit service, …) do not each reinvent it.
2. **The `keystore` wire format** — SIGN / ENCRYPT / DECRYPT request and reply layouts,
   worked end to end, the direct successor to the `KeystoreService` trait
   ([ADR-0018](../adr/0018-wit-handle-capability-mapping.md)) `lantern-runtime` calls today.
3. **The `store` wire format** — READ / WRITE, plus how a payload larger than one `Frame`
   chunks across calls.

It does **not** change the WIT interfaces the guest sees (`lantern:host/keystore`,
`lantern:host/filesystem` — RFC-0014/RFC-0016), the capability model, or anything in the
kernel. It fixes only what crosses the `(runtime, service)` boundary once both are confined
processes.

## Motivation

`Broker` already runs confined ([ADR-0022](../adr/0022-confined-service-model-and-call-transport.md),
`lantern-boot`'s `broker-service`). `Keystore` and `Store` are next, and each needs a
request/reply protocol before it can be written as a `Recv`-loop program. Deciding that in
the open — rather than inside the first porting PR — matters because a confined service's
**request parser is a trust boundary**: it runs on bytes a possibly-compromised runtime
wrote into shared memory, and a sloppy parser (trusting a length, reading past a buffer,
acting before `check_access`) reintroduces exactly the confused-deputy and TOCTOU classes
the whole confinement effort exists to close ([`Principles.md`](../../lantern-docs/wiki/Principles.md)
#1). ADR-0022 also flagged **shared-`Frame` sizing and lifecycle** as unresolved; this RFC
resolves it.

One protocol, not three ad-hoc ones: `Keystore` and `Store` differ only in their operation
set and payload shapes. A shared header and shared error mapping means the runtime side is
one `marshal → Call → unmarshal` helper parameterised by interface, and a reviewer audits
the framing once.

## Guide-level explanation

Today, in-process, `lantern-runtime` calls a trait:

```rust
keystore.sign(badge, key, message) -> Result<Vec<u8>, KeystoreError>
```

After this RFC, with the keystore a separate process, the same call is:

```
runtime:                                          keystore (Recv loop):
  frame.hdr = { op: SIGN, arg_len: message.len }
  frame.payload[..arg_len] = message
  Call(sign_badged_endpoint, tag{ mr1 = arg_len })
                                                    Recv -> badge = (KeyId, KeyOps)
                                                    copy frame.payload[..mr1] to a private buf   (before any check)
                                                    check_access(badge, SIGN)                     (deny by default)
                                                    sig = ed25519_sign(key_material, buf)
                                                    frame.payload[..sig.len] = sig
                                                    Reply(tag{ label = OK, mr1 = sig.len })
  read frame.payload[..mr1] -> the signature
```

The badge — kernel-delivered on `Recv`, never a number the runtime typed — is the only
thing that says *which key* and *which operations*; the runtime cannot lie about it. The
shared `Frame` is mapped RW into exactly these two address spaces and nothing else, so its
contents are no more sensitive than the arguments already crossing the call. The service
**copies the payload out before it validates anything**, so a runtime that rewrites the
`Frame` mid-call changes nothing the service has already read.

## Reference-level explanation

### The shared `Frame` and its lifecycle

- **One shared `Frame` per `(runtime process, service)` pair**, 4 KiB (one `RISCV64_PAGE_SIZE`),
  mapped RW into both VSpaces by the launcher at spawn time (RFC-0018 Part 1). Not per-call,
  not a pool: a runtime process talks to at most a handful of services, and a 4 KiB region
  each is cheap and bounds the blast radius (a bug in the keystore path cannot scribble the
  store's channel).
- **Single outstanding call.** A confined program is single-threaded (ADR-0010) and every
  service call is a synchronous `Call`/`Reply`, so the `Frame` holds exactly one
  request-then-reply at a time. No sequence numbers, no locking.
- **Ownership convention.** Between the runtime's `Call` and the matching `Reply`, the
  `Frame` belongs to the service; otherwise to the runtime. Neither is *enforced* (both
  hold a RW mapping) — it is a discipline, and the service's copy-in is what makes a
  violation harmless rather than a vulnerability.
- **Payloads larger than 4 KiB** (a large `store.read`, a bulk `encrypt`) **chunk**: the
  request header carries an `offset` and the reply header an `eof` flag, and the runtime
  loops `Call`s until `eof`. Same shape `lantern-filesystem`'s eventual multi-block
  chunking needs anyway.

### The request/reply header

The first bytes of the shared `Frame` are a fixed header; the rest is payload. All
multi-byte fields little-endian (the `riscv64` native order — no network byte order here,
both ends are the same machine).

```
request header (16 bytes):
  u16  op          operation code (per-interface, below)
  u16  flags       bit 0 = MORE (this is a chunk, more follow)
  u32  arg_len     bytes of payload that are meaningful this call (<= FRAME_PAYLOAD)
  u64  offset      byte offset into the logical argument, for chunked calls (0 otherwise)

reply header (16 bytes):
  u16  status      0 = OK, 1 = ACCESS (denied/revoked/wrong-object), 2 = INVALID
                   (malformed request / bad length / unsupported op), 3 = FAILED
                   (the operation itself failed — e.g. AEAD tag mismatch, key destroyed)
  u16  flags       bit 0 = EOF (no more reply chunks)
  u32  ret_len     bytes of payload that are meaningful in this reply
  u64  _reserved
```

`FRAME_PAYLOAD = 4096 - 16 = 4080` bytes.

The **message tag** carries only a redundant copy of `arg_len` / `ret_len` in `mr1` (so the
receiver can reject a wildly-out-of-range length before it even reads the header) and the
`op` in the tag `label` (so a service can dispatch without touching shared memory for a
no-argument call). The header in the `Frame` is authoritative; `mr1` is a cheap early
sanity gate. A mismatch between them is `INVALID`.

### Error mapping

Each service already computes one of these outcomes internally; the protocol just carries
it, and each side maps to/from its own error type:

| wire `status` | `Keystore` / `Store` internal | runtime → WIT (`error-code`) |
| --- | --- | --- |
| `OK` | success | — |
| `ACCESS` | `check_access` denied; revoked badge; wrong `KeyId`/`FileId` | `error-code::access` |
| `INVALID` | bad length, unknown op, arg out of range | `error-code::invalid` |
| `FAILED` | AEAD tag mismatch, key destroyed, store I/O error | `error-code::invalid` (v0 — no distinct WIT code yet; see Unresolved) |

The runtime **never invents `ACCESS`** — exactly the ADR-0018 rule, now across a real
boundary: it relays the service's own verdict.

### `keystore` wire format

`op` codes: `SIGN = 1`, `ENCRYPT = 2`, `DECRYPT = 3`. The badge identifies `(KeyId,
KeyOps)`; there is no key argument on the wire.

- **SIGN** — request payload is the message to sign (chunked if > 4080 B, `offset` tracks
  position, final chunk clears `MORE`). Reply payload is the 64-byte Ed25519 signature,
  `status = OK`. `FAILED` if the key was destroyed.
- **ENCRYPT** — request payload is `[u32 nonce_len][nonce][u32 aad_len][aad][plaintext…]`;
  `plaintext` chunks. Reply payload is `[u32 tag_len][tag][ciphertext…]`, `ciphertext`
  chunks, matching the plaintext length.
- **DECRYPT** — request payload is `[u32 nonce_len][nonce][u32 aad_len][aad][u32
  tag_len][tag][ciphertext…]`. Reply is the plaintext, chunked; `status = FAILED` on tag
  mismatch (constant-time compare, service side — X6 in `lantern-crypto/THREAT_MODEL.md`),
  and the reply payload MUST be empty in that case (no partial plaintext leak).

The service copies the *entire* logical argument into a private buffer across chunks
**before** `check_access` and before any crypto — a runtime that shrinks `arg_len` on a
later chunk gets `INVALID`, never a half-processed operation.

### `store` wire format

`op` codes: `READ = 1`, `WRITE = 2`. The badge identifies `(FileId, FileOps)`.

- **READ** — request header only (`arg_len = 0`); `offset` is the byte offset into the
  file. Reply payload is up to 4080 file bytes; `EOF` set when the last byte of the file is
  in this chunk. An unwritten file replies `OK` with `ret_len = 0, EOF`.
- **WRITE** — request payload is file bytes at `offset`; `MORE` while more chunks follow.
  Reply is header-only, `status = OK`. A write whose total length exceeds the store's
  per-file cap is `INVALID` on the *first* chunk (the runtime sends the total via `offset`
  of a zero-length probe, or the service caps and returns `INVALID` when a later chunk
  would exceed — v0 uses the latter, simpler rule; see Unresolved).

`store` v0 has no chunk-boundary durability guarantee — a `WRITE` that stops partway leaves
the file partially written, same as the in-process `Store` today.

### `lantern-abi` additions

A new `lantern_abi::frame` module: `Header` (the 16-byte struct + `to_bytes`/`from_bytes`,
pure and host-testable), `FRAME_PAYLOAD`, and a `Channel { endpoint: CPtr, frame: *mut u8 }`
helper wrapping "write header + payload, `Call`, read reply header + payload" with the
chunk loop. This is non-TCB (it marshals; it enforces nothing) and both the runtime's
`IpcKeystore`/`IpcFilesystem` and the services' `Recv` loops use it — the framing lives in
one reviewed place.

### What this RFC does not decide

- **The WIT interfaces.** `lantern:host/keystore` / `lantern:host/filesystem` and their
  `error-code` enums are unchanged (RFC-0014/RFC-0016).
- **The launcher / how the shared `Frame` and badged endpoints get into place.** RFC-0018
  Part 1; this RFC assumes they are mapped and granted.
- **`Broker`'s role.** The badged endpoint per handle is still minted by the service's
  composed `Broker` (ADR-0022); this RFC is about what travels *over* that endpoint.
- **A generic RPC/IDL.** This is a deliberately small, hand-written framing for two known
  services. If a third or fourth service makes the pattern painful, a generated layer is a
  later RFC — not this one.
- **The `monotonic-clock` (link-scoped) interface** — it has no per-call object and no
  payload; it stays a direct `lantern-hal` read in the runtime (RFC-0014), no service
  call.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none newly defined. This fixes the *format* of traffic
  across the `(runtime, service)` boundary ADR-0022 already established. The kernel, the
  capability model, and ADR-0004's TCB boundary are untouched.
- **New assets:** the request/reply header bytes in the shared `Frame` — same reachability
  as the payload (that one runtime, that one service), and the service's copy-in makes
  their integrity a non-issue for the service.
- **New adversary capabilities:** a compromised runtime can now write an arbitrary header
  — a lying `op`, `arg_len`, `offset`, `MORE`. Mitigations, all service-side: the `mr1`
  early length gate; copy-in of the *whole* logical argument before `check_access` and
  before operating; a fixed maximum reassembled argument size per op (reject `INVALID`
  past it); `offset`/`MORE` validated for monotonic non-overlapping progress; unknown `op`
  → `INVALID`. The service never allocates unbounded memory from a wire length.
- **Mitigations:** copy-in-before-validate (TOCTOU); deny-by-default `check_access`
  unchanged; constant-time AEAD tag compare with empty payload on failure (no plaintext
  oracle); per-op size caps; the framing helper in `lantern-abi` audited once.
- **Net change to attacker surface:** small and bounded. The confinement win is ADR-0022's;
  this RFC's job is to not squander it in the request parser. The one genuinely new thing
  a reviewer checks is each service's chunk-reassembly loop.

Cross-reference `lantern-crypto`, `lantern-filesystem`, `lantern-runtime`
`THREAT_MODEL.md`; none of their defined boundaries move.

## TCB impact  *(mandatory)*

**None.** The framing structs and the `Channel` helper live in `lantern-abi` (non-TCB,
ADR-0022). `Keystore` / `Store` request-loop code is confined user-space. No new kernel
object, syscall, or `lantern-hal` surface — this is a convention layered on the existing
`Call`/`Reply` + a shared `Frame` mapped with `FrameInvoke::Map`. The shared `Frame` is
*not* the kernel's designed-but-unbuilt in-memory IPC buffer; ADR-0022 already chose shared
memory over building that.

## Privacy impact

Positive, marginally: DECRYPT returns an empty payload on tag-mismatch, closing a partial-
plaintext oracle the in-process trait boundary never had to think about. Otherwise
unchanged — the same bytes cross the same boundary, just with a header.

## Alternatives considered

- **Implement the kernel in-memory IPC buffer and send requests through it.** Rejected for
  the same reasons ADR-0022 rejected it: kernel code (TCB growth), and file/blob payloads
  exceed any fixed buffer anyway. This RFC's chunking handles arbitrary sizes.
- **A separate bespoke protocol per service, no shared header.** Rejected: three services
  soon (keystore, store, network), each with a near-identical request/reply/chunk shape;
  one reviewed framing beats three.
- **A generated RPC layer (Cap'n Proto / a custom IDL / `wit`-over-IPC).** Rejected for
  v0: two services with <6 operations between them do not justify the dependency or the
  codegen, and a hand-written 16-byte header is trivially auditable. Revisit if the
  service count grows.
- **Length-prefixed streaming instead of a fixed header + chunk offsets.** Rejected: fixed
  offsets let the service pre-size its reassembly buffer and reject a bad total immediately,
  rather than growing a buffer as chunks arrive (a memory-exhaustion lever).
- **Put the whole request in the message registers when it fits (≤ 3 words).** A possible
  later optimisation (a SIGN of a 24-byte hash never touches shared memory) — deferred;
  v0 always uses the `Frame` so there is one path to review.

## Prior art

- **seL4 / the `lantern-boot` broker demo** — badged endpoint delivering identity on
  `Recv`, shared memory for bulk payload: this project's existing model, now given a
  concrete framing.
- **9P / virtio-9p** — a fixed small header (`size[4] type[1] tag[2]`) then a
  type-specific body, chunked reads/writes with an offset: the same shape as this RFC's
  header + per-op payload, minus the multiplexing this RFC's single-outstanding-call model
  does not need.
- **Cap'n Proto RPC over a shared segment**, **Fuchsia FIDL over a channel + VMO** — the
  "small control message references a shared memory region for bulk data" pattern; this
  RFC is the hand-rolled, two-service subset.
- **WASI Preview 2 `streams`** — the chunked read/write with an eof signal that the
  `store` format mirrors, so the runtime's WIT `read`/`write` map onto it directly.

## Unresolved questions

- **A distinct WIT `error-code` for `FAILED`** (crypto op failed vs. request malformed) —
  v0 collapses both to `error-code::invalid`. `lantern:host/keystore` may want a
  `crypto-failure` variant; a small RFC-0014-style amendment.
- **The WRITE total-size check** — v0 caps on the chunk that would exceed and returns
  `INVALID` late. A zero-length "reserve N bytes" probe as the first call is cleaner but
  adds a round trip; decide with the `Store` chunking work.
- **Shared-`Frame` size** — 4 KiB is a guess tuned to "one page, no allocator." If crypto-
  in-a-loop benchmarking (RFC-0014's flagged risk, ADR-0013's methodology) shows the chunk
  count dominates, a larger per-channel `Frame` (16–64 KiB) is a one-line change.
- **Whether `store` and `keystore` share one endpoint with an interface selector, or one
  endpoint each.** v0: one badged endpoint per *handle* (so per key / per file), which
  falls out of ADR-0022's mint model; revisit only if endpoint-object pressure shows up.
- **Fault signalling** — if a service panics mid-call the runtime's `Call` never returns.
  RFC-0018's open "runtime-process fault-handling policy" covers the mirror case; the
  service-death case wants the launcher to notice and fail the runtime's outstanding call.

## Future possibilities

- Register-only fast path for tiny requests (the SIGN-a-hash case).
- A third consumer: `lantern-network`'s socket interface over the same framing.
- The audit log (`lantern-ai-runtime`) tapping every `Channel::call` as an event.
- A shared-`Frame` pool if one runtime ends up talking to many services.
- Moving the framing into a generated layer if the hand-written version stops scaling.

## Resulting ADRs

On acceptance, expected to produce **one ADR** fixing the shared-`Frame` framing (header,
error mapping, chunking, lifecycle) and the `keystore` + `store` operation formats, and
recording that it adds nothing to the TCB.
