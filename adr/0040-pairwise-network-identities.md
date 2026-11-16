---
adr: 0040
title: Pairwise network identities — an identity per context, over an encrypted channel, that no two contexts can link
status: Accepted
date: 2026-11-16
deciders: ["TSC"]
rfc: ../rfcs/0035-pairwise-network-identities.md
supersedes: []
superseded_by: null
---

# ADR-0040: Pairwise network identities

## Context

Phase 3's last exit criterion is "a network identity can be presented without cross-context linkage". LanternOS had no
network, no randomness and no wallet. [RFC-0035](../rfcs/0035-pairwise-network-identities.md) proposed all three, and
the smallest proof. It has been accepted; this ADR is the durable record.

## Decision

**A confined wallet holds one secret root and presents, for each (app, peer), a pairwise identity derived from it, over
a Noise `XX` session it runs itself. Apps get sessions, never keys. The network stack is a separate confined program
that holds no key.**

- **Randomness.** `rng-driver`, a virtio-rng driver (`-device virtio-rng-device`), serves 24 bytes a call to the wallet
  alone (`lantern_abi::net::random`).
- **The network.**
  - `net-driver`, a virtio-net driver, moves whole frames for one client and parses none. Like `blk-driver`, it's trusted
    with memory until there's an IOMMU. The card's interrupt goes to `net`'s notification, so the driver only ever waits
    for requests.
  - `net` is IPv4 and UDP (`lantern-network`: Ethernet, ARP, IPv4, UDP, no allocation, host-tested against hand-built and
    hostile frames). A static address, `10.0.2.15` behind `10.0.2.2`, as QEMU's user network gives. It keeps datagrams
    only for the wallet's ports, and sleeps on its notification and a deadline. It holds no key, no DMA, no store, no
    console.
- **The wallet.**
  - **One root**: 32 random bytes, made on the first start, sealed (XChaCha20-Poly1305) under the wallet's own keystore
    key, and kept on the disk's wallet partition (`mkdisk` writes 1 MiB, zeroed). A sector that doesn't unseal is never
    overwritten: the wallet then refuses every session.
  - **Pairwise keys**: `lantern_crypto::pairwise`, keyed BLAKE3 (a pseudorandom function) of the root over the app's
    principal (RFC-0033, named by the binder) and the peer's name. Without the root, two contexts' keys are independent.
  - **Sessions**: `Noise_XX_25519_ChaChaPoly_BLAKE2s` as the initiator, with a fresh ephemeral key from `rng-driver`
    (`lantern_crypto::noise`: the state machine over reviewed X25519, ChaCha20-Poly1305 and BLAKE2s crates, allocating
    nothing; checked byte for byte against `snow` with fixed keys, and for interoperability in both roles).
  - **Recorded first**: every `connect`, `send` and `recv` is in the audit log before it runs, with the context, the
    identity's public key prefix and the length, never the content.
- **The capability.** `lantern:host/identity`, resource-scoped: `[[identity.context]]` in a manifest, op `connect`;
  `open(slot) -> option<context>`, then `context.connect()`, `send`, `recv`. The session belongs to the handle. Peers
  are named by context id in the binder's policy, and the system's table of contexts (`lantern_abi::identity::CONTEXTS`)
  says where each is, until name resolution exists.
- **Seeing it.** `identities` at the console lists each app, peer, key prefix and session count, through a reader badge on
  the wallet that never yields a key.
- **The proof.** `scripts/net-matrix.sh`, against two host peers using `snow` (`tools/peer`): different identities to
  alice.test and bob.test; the same ones again and after a reboot; a declined peer never contacted; a forged handshake
  refused; no network devices, no sessions; a changed sealed root neither used nor overwritten.

## Consequences

- **Easier:** an app can talk to a peer as an identity a person allowed, with no way to learn its key, link it to another
  context, or reach a peer it wasn't granted.
- **Found and fixed on the way** (committed separately): `lantern-kernel`'s `Yield` wrote its success into the next
  thread's tag register, which under load changed one bit of an app's `a4` now and then; and `lantern-hal` didn't save
  the floating-point registers across a switch.
- **Harder / committed to:**
  - Four more programs: kernel limits are now 24 CNodes, threads, VSpaces, scheduling contexts and notifications; 16
    endpoints; 176 Frames; 128 page tables; 256 CNode slots; 8 interrupt handlers.
  - A disk is 128 MiB by default, with a wallet partition after the model's.
  - Without `-device virtio-rng-device` and `-netdev user,... -device virtio-net-device,...`, the drivers refuse every
    request and the wallet every session; nothing else changes.
- **Not done:**
  - Network-level unlinkability: both peers see QEMU's one address, and timing. An anonymity transport is future work.
  - DHCP, DNS, TCP, discovery.
  - The root in hardware (Phase 4, RFC-0023's checklist); its sealing key is a demo key, like the store's.
  - Recovery of a lost root, and DIDs and credentials over these identities.
