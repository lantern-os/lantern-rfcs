---
rfc: 0035
title: Pairwise network identities — an identity per context, over an encrypted channel, that no two contexts can link
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["network", "crypto", "boot", "shell"]
domains: ["network", "crypto", "abi", "boot", "shell", "runtime", "sdk", "docs"]
created: 2026-11-16
updated: 2026-11-16
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-16, implemented and recorded in [ADR-0040](../adr/0040-pairwise-network-identities.md).** Built as
> proposed, with these changes:
> - The pairwise secret is keyed BLAKE3 of the root, a pseudorandom function, not HKDF.
> - The wallet's Noise is its own allocation-free state machine over reviewed primitives, checked byte for byte against
>   `snow` with fixed keys and for interoperability in both roles; `snow` itself needs a heap that frees, which no
>   confined program has. The test peers use `snow`.
> - There is no separate session resource: `context.connect()` starts the handle's session, and `send` and `recv` use it.
> - The contexts' names and addresses are the system's table, `lantern_abi::identity::CONTEXTS`; the binder's policy names
>   them by id, as it names files.
> - The wallet's root is sealed under a keystore key of its own, on a 1 MiB wallet partition.
> - Without the network and random-number devices, the system still runs, and the wallet refuses every session.
>
> **Found while implementing**, committed separately: the kernel's `Yield` wrote its success into the next thread's tag
> register, which under load changed one bit of an app's `a4` now and then (an app started at once after boot failed its
> own package's check about one boot in four); and the HAL didn't save floating-point registers across a switch.
>
> **Direction agreed 2026-11-16.** The unresolved questions were settled as proposed:
> entropy from a virtio-rng driver; a context is per (app, peer); sessions are Noise `XX`; addresses come from the
> binder's policy.

# RFC-0035: Pairwise network identities

## Summary

Phase 3's last exit criterion: **"a network identity can be presented without cross-context linkage."**

This RFC adds networking and an OS wallet, and the smallest thing that meets the criterion end to end:

1. **`net-driver`**, a confined virtio-net driver, and **`net`**, a confined service with a minimal IPv4/UDP stack. It holds
   no key.
2. **`wallet`**, a confined service holding one secret root. For each **context** it derives a **pairwise identity**: an
   X25519 key pair no other context's can be linked to. It runs a Noise `XX` handshake as that identity, over `net`, and
   gives the app an encrypted session. The app never sees a key.
3. **An entropy source**, which the system doesn't have yet: the wallet's root and every handshake's ephemeral key need
   real randomness.
4. **A proof**: an app, `greeter`, says hello to two peers. A host-side test peer, using an independent Noise
   implementation, checks that each sees a stable identity across runs and reboots, and that the two identities are
   different and unrelated.

```
lantern> run greeter
"greeter" from publisher 927308f5..00b7 asks to use:
  an identity for "alice.test"  (connect)    allow? [y]es once, [a]lways ...
  an identity for "bob.test"    (connect)    allow? ...
lantern> identities
  "greeter" as itself to "alice.test": 3f2a..91c0, used 2 times
  "greeter" as itself to "bob.test":   b71e..04d8, used 2 times
```

## Motivation

- It's the last Phase 3 exit criterion.
- [Identity](../../lantern-docs/wiki/Identity.md): "partitionable by default ... linkage is opt-in, never the default", and
  "wallets are operating-system services ... an application never sees the keys".
- [Networking](../../lantern-docs/wiki/Networking.md): the network stack is confined and "does not hold the keystore";
  apps get endpoint capabilities, never ambient sockets.

## Reference-level explanation

### 1. Networking

- **`net-driver`**: virtio-net over virtio-mmio, as `blk-driver` is virtio-blk: its register page, its interrupt line, a
  DMA `Untyped`, and one client. Trusted with memory, like the block driver, until there's an IOMMU (RFC-0023's checklist
  already has this row).
- **`net`**: Ethernet, ARP, IPv4 and UDP. A static address in this RFC (QEMU's user network gives `10.0.2.15`, gateway
  `10.0.2.2`); DHCP later. No TCP. It serves UDP to one client, the wallet, under one badge: send a datagram to an
  address and port, receive datagrams on a port. It parses everything the network sends, so it's the most exposed
  program in the system, and it holds nothing worth stealing.
- In QEMU: `-netdev user,id=net -device virtio-net-device,netdev=net`. The test peer listens on the host, which the guest
  reaches at `10.0.2.2`.

### 2. Entropy

Unresolved question 1. Proposed: **a virtio-rng driver** (`-device virtio-rng-device`), confined, serving random bytes
to the wallet alone. A board with a hardware generator gets a driver for it behind the same protocol.

### 3. The wallet

- **One root**, 32 random bytes, made on first start and kept sealed in the store, under its own key in the keystore.
  Software-held until Phase 4 (RFC-0023's checklist gains a row).
- **A context** names who the identity is presented to: a peer's name, as the app's manifest declares it. The pairwise key
  for a context is `HKDF(root, "lantern-pairwise-v1" ‖ scope ‖ context)`. Whether `scope` includes the app is unresolved
  question 2.
- **Unlinkable**: two contexts' keys are independent outputs of a keyed function. Without the root, nobody, including the
  two peers together, can tell they came from one wallet.
- **Sessions**: `connect(context)` runs a Noise `XX` handshake (`Noise_XX_25519_ChaChaPoly_BLAKE2s`) to the context's
  address as that identity, with a fresh ephemeral key, and returns a session. `send` and `recv` move encrypted messages.
  The wallet holds the keys and the Noise state; the app holds a badge.
- The wallet's Noise is its own, about 300 lines over the primitives `lantern-crypto` already uses, tested against
  `snow`, an independent implementation, on the host.

### 4. The capability

- A new interface, `lantern:host/identity`, resource-scoped: `[[identity.context]]` in a manifest names a context
  (`peer = "alice.test"`) and the op `connect`. The binder's policy lists which contexts an app may ask for, as a ceiling,
  and the person is asked for each.
- `open(slot) -> option<context>`, `context.connect() -> result<session>`, `session.send(bytes)`,
  `session.recv() -> bytes`.
- A context's address comes from the binder's policy, not the app, in this RFC: no name resolution yet.
- The audit log records each `connect`, `send` and `recv` before it runs: the grant, the context, the identity's public key
  prefix, the length. Never the content.

### 5. The person sees it

- `identities` at the console: each app, context and public key prefix, and how many sessions.

### 6. The proof

- Host tests: pairwise derivation (stable per context, different across, derived only from the root); Noise against
  `snow` in both roles; the UDP stack against hand-built and hostile packets.
- `scripts/net-matrix.sh`: a host peer (Rust, with `snow`) on two ports, standing in for `alice.test` and `bob.test`.
  - `greeter` connects to each; each peer checks the handshake and an encrypted round trip.
  - The peers compare: different static keys, and nothing else shared but the IP address QEMU gives every guest.
  - Run again, and after a reboot: each peer sees the same key it saw before.
  - Declined at the console: no handshake reaches that peer.
  - A peer that answers with garbage, or a forged handshake: the session fails, and nothing reaches the app.
- Mutation tests: the same key for every context; the root not persisted; the app given the key; the stack accepting a
  datagram for another port.

## Threat model impact

- **Linkage by keys** is prevented: pairwise keys are unlinkable without the root.
- **Linkage by the network** isn't: both peers see the same IP address, and timing. An anonymity transport is future work.
  The RFC's proof says so, and checks only what a key can link.
- **The network stack** is exposed and holds no key: compromising it reveals encrypted traffic, and can drop or forge
  datagrams, which the Noise session detects.
- **The wallet** holds every identity: it's in no app's address space, and holds no network or console capability beyond
  its badge on `net`.
- **The root** is software-held until Phase 4: device theft yields every identity.
- **The driver** has DMA, as `blk-driver` does, until there's an IOMMU.

## TCB impact

None: every new program is confined.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** Unchanged, unless the entropy source needs a kernel change.
2. **Can an agent use this without being unnecessarily trusted?** An agent gets one context, never a key, and the person
   decides each.
3. **Same interface for humans and agents?** Yes: any app, through the same binder and console.

## Alternatives considered

- **One identity, many signatures.** Simple, and linkable by construction.
- **TLS.** Certificates and a PKI; Noise is far smaller, and needs no authority.
- **The app holds the pairwise key.** Then any app could present any identity.
- **TCP.** Much more stack, for nothing this proof needs.

## Unresolved questions

1. **Entropy**: virtio-rng, as proposed, or something else.
2. **Scope**: is a context per (app, peer), so two apps talking to the same peer can't be linked either, or per peer, so a
   person has one identity with a peer whichever app they use?
3. **The channel**: Noise `XX` as proposed, or a simpler signed challenge with no encrypted session.
4. **Addresses**: from the binder's policy, as proposed, or DNS now.

## Future possibilities

- DHCP, DNS, TCP; P2P discovery.
- An anonymity transport, so the network can't link contexts either.
- DIDs and verifiable credentials over these identities.
- The root in a hardware enclave (Phase 4); recovery.

## Resulting ADRs

On acceptance, an ADR will record: the drivers and the network service, the entropy source, the wallet and its
derivation, the Noise channel, the identity interface, and the proof.
