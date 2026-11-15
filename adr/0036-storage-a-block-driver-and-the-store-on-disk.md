---
adr: 0036
title: Storage — a confined block driver, and packages and the encrypted store on a disk
status: Accepted
date: 2026-11-14
deciders: ["TSC"]
rfc: ../rfcs/0031-storage-a-block-driver-and-the-store-on-disk.md
supersedes: []
superseded_by: null
---

# ADR-0036: Storage — a confined block driver, and packages and the encrypted store on a disk

## Context

Everything `lantern-boot-system` could run was compiled into its boot image, and every file was lost at power-off.
[RFC-0031](../rfcs/0031-storage-a-block-driver-and-the-store-on-disk.md), step 4 of the roadmap's "Toward an installable
system" track, proposed a disk. It has been accepted; this ADR is the durable record.

## Decision

**A confined virtio-blk driver serves raw sectors to one client, a volume service that reads the disk's GPT and serves
each client exactly its partition. The packages, and the store's files, live on the disk.**

- **The kernel: DMA addresses.**
  - Boot can mark an `Untyped` for DMA, and its children inherit the mark.
  - `FrameInvoke::Address` (label 3) returns a Frame's physical address, but only for a Frame retyped from a DMA
    `Untyped`. Any other is refused, so no other program learns where memory is.
- **`blk-driver`** (in `lantern-boot`):
  - virtio 1.x over MMIO (version 2; QEMU needs `-global virtio-mmio.force-legacy=false`), one split queue of 8
    descriptors, one request at a time, sleeping on its interrupt.
  - It holds the device's 4 KiB register page, its line, and a 2 MiB DMA `Untyped` for its queue and data pages.
  - A `WRITE` is followed by a `FLUSH` when the device offers one, and is answered `OK` only once both complete.
  - It parses nothing from the disk, and serves only the volume's badge.
  - Boot finds the device by reading each virtio-mmio slot's magic, version and device ID.
  - **It is trusted with all memory** until an IOMMU: RFC-0023's checklist has the row.
- **`volume`** (in `lantern-boot`):
  - It parses the GPT (`lantern_abi::gpt`), checking the header's signature, size and CRC, the entries' CRC, and each
    LanternOS partition's extent against the disk and against the other. If any check fails, every client is refused.
  - It serves the packages partition read-only to the launcher and the binder, and the store partition read-write to the
    store, each on its own channel, with every sector checked against the partition.
  - It has no DMA and no mint authority.
- **Packages:**
  - The packages partition holds an index (`lantern_abi::block::packages`) and the signed `.lpkg` files.
  - The launcher and the binder each read it into 4 Frames of their own, and verify every package themselves.
  - The boot image no longer carries packages.
- **The store:**
  - Its files outlive a reboot as one sealed image (`lantern_filesystem::image`) in two slots. The image holds every
    file's record and every block, already sealed, with a BLAKE3 digest over the whole.
  - The digest is sealed through the keystore, as an AEAD tag over no plaintext. Its nonce is the digest's own prefix,
    so two different images never share one.
  - At start, the store loads the newest image that verifies, or starts empty. After each successful write, it saves the
    next image to the older slot, and answers `OK` only once it's on the disk.
  - The key is still the keystore's demo key.
- **`lantern-mkdisk`** (in `lantern-boot/tools`) writes a disk:
  - a GPT;
  - the packages partition (16 MiB), with its index;
  - the store partition (1 MiB), zeroed.
- **The kernel's trap stack is 64 KiB, with a canary.** Found while building this: at 4 KiB, a 160-slot CNode built on it
  during a retype overflowed it silently into kernel state. `lantern-hal` now paints the stack, checks a canary after
  every trap, and panics on an overflow. The lifecycle demo uses 4,232 bytes of it at most.
- **Capacity:**
  - 16 TCBs, CNodes, VSpaces, scheduling contexts and notifications, 112 Frames, 96 page tables, 160 slots per CNode.
  - Before any app, the system image takes 9 TCBs, 9 CNodes, 8 VSpaces, 53 Frames, 38 page tables and 136 root slots.
  - Host tests that build a `KernelState`, now about 470 KB, run with an 8 MiB stack (`RUST_MIN_STACK`).

## Consequences

- **Easier:**
  - A person's files survive a reboot.
  - Packages can change without rebuilding the boot image.
  - Step 5's installer needs only to write a disk this system already reads.
- **Harder / committed to:**
  - A program with DMA exists, and is trusted with memory.
  - The disk is untrusted input: every reader verifies or authenticates what it takes from it.
  - Each write costs a whole-image save, which suits the store's 16 small files and nothing much larger.
- **Not mitigated:**
  - Someone who can write the disk can put back an older image that still verifies (rollback).
  - The store's key is a public demo key, so the files' contents are only as confidential as the source code is secret.
  - Sizes and write times are visible.
  - All three are on RFC-0023's checklist.
- **TCB:** the kernel gains the DMA mark and one Frame invocation; the driver joins it in effect, for memory, until an
  IOMMU.
- **Still not done:**
  - Installing (step 5): a bootable disk image.
  - A store that grows past 16 small files.
  - The backup GPT used for recovery.
  - More than 4 packages read at once.
