---
rfc: 0031
title: Storage — a confined block driver, and packages and the encrypted store on a disk
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["boot", "filesystem", "kernel", "shell"]
domains: ["kernel", "hal", "abi", "boot", "filesystem", "shell", "docs"]
created: 2026-11-14
updated: 2026-11-14
supersedes: []
superseded_by: null
tracking_issue: null
--> **Accepted 2026-11-14, implemented and recorded in [ADR-0036](../adr/0036-storage-a-block-driver-and-the-store-on-disk.md).**
> Built as proposed, with these details settled along the way:
> - Boot finds the device by probing QEMU's virtio-mmio slots.
> - The volume serves three badges, each with its own channel: the launcher, the binder and the store.
> - The launcher and the binder read up to 4 packages each.
> - The store's image is sealed as an AEAD tag over its digest, through the keystore, since a keystore request is too
>   small for the whole image.
>
> **Found while implementing:** the HAL's 4 KiB trap stack, the kernel's stack during every system call, overflowed
> silently once a CNode grew to 160 slots: retyping one builds it on that stack. It's now 64 KiB, with a canary checked
> after every trap.
>
> Section 5's "saying so" when the store starts empty isn't built: the store has no console, and the audit log records
> nothing for it.
>
-

> **Direction agreed 2026-11-14; still a Draft until implemented.** The unresolved questions were settled as proposed:
> - the driver is trusted with memory, documented, and only DMA Frames reveal their address, until an IOMMU;
> - the store keeps the keystore's demo key, with RFC-0023's "Encrypted storage" row as the path to a device key;
> - the disk uses GPT now;
> - the store is a two-slot sealed image.

# RFC-0031: Storage — a confined block driver, and packages and the encrypted store on a disk

## Summary

Step 4 of the roadmap's "Toward an installable system" track. Today everything the system runs is compiled into its boot
image, and every file is lost at power-off: the store keeps its blocks in memory. This RFC adds a disk:

- **A confined virtio-blk driver**, the system's first program that drives a device by DMA.
- **A volume service**, which reads the disk's partition table and gives each client exactly its partition, no more.
- **Packages on the disk.** The launcher and the binder each read the packages partition, and each verifies every package
  itself, as they do the boot image's set today.
- **The store on the disk.** Its files outlive a reboot, sealed as before, with the whole store written as one
  authenticated, versioned image.
- **A host tool** that writes a disk image: a partition table, the packages, and an empty store.

The boot image keeps the programs that make up the system. The apps, and what they write, move to the disk.

## Motivation

- A person can't add an app without rebuilding the boot image, and nothing they make survives a reboot. Neither is an
  installable system.
- Step 5 (installing) writes a disk. That needs a disk the system already knows how to read.
- Every later device (network, a real board's storage) needs the same thing first: a confined driver that does DMA, and
  a clear statement of what it's trusted with.

Principles served: **security by architecture** (the driver and the partition logic are confined and separate; packages
stay signed and the store stays sealed, so the disk itself is untrusted), and the installable-system goal. In tension:
**the driver can reach all memory**, since QEMU `virt` has no IOMMU (unresolved question 1).

## Guide-level explanation

```
 virtio-blk (QEMU) ◀─DMA─▶ blk-driver ──(raw sectors, one client)──▶ volume
                                                                       │ reads the partition table
                         ┌───────────────────────┬─────────────────────┼─────────────────────┐
                    launcher (packages, read)  binder (packages, read)  store (store partition, read/write)
```

```
$ lantern-mkdisk --out disk.img --package probe.lpkg --package fs-probe.lpkg
$ qemu-system-riscv64 ... -drive if=none,format=raw,file=disk.img,id=disk -device virtio-blk-device,drive=disk
lantern> apps          (now read from the disk)
lantern> run confined-fs-probe
   ... reboot ...
lantern> run confined-fs-probe   (reads back what it wrote before the reboot)
```

## Reference-level explanation

### 1. The kernel: DMA addresses

A device addresses memory physically, so a driver must know where its buffers are. A new `FrameInvoke::Address` returns
a Frame's physical address, **only for a Frame retyped from a DMA `Untyped`**: one boot marks as such and gives only to a
driver. Any other Frame refuses it, so no other program learns physical addresses. Nothing else in the kernel changes:
the driver uses the existing `IrqHandler` (RFC-0027) and a 4 KiB device page, as the shell's UART does.

### 2. `blk-driver`

A new confined program in `lantern-boot` (a driver for the boot image's own device, as `uart-driver` was). It holds:

- the virtio-mmio device's 4 KiB register page, and its `IrqHandler`;
- a DMA `Untyped`, for the virtqueue and its bounce buffers;
- one endpoint, under one badge: the volume's.

It speaks virtio 1.x over MMIO with one queue, polls nothing, sleeps on its interrupt, and serves `READ(sector, count)`
and `WRITE(sector, count)` over an RFC-0019 channel to its one client. It parses nothing on the disk.

### 3. `volume`

A new confined program. It reads the disk's partition table through the driver (unresolved question 3), finds the
LanternOS partitions by type, and serves each client its own extent:

| Badge | Partition | Access |
|---|---|---|
| packages reader (launcher, binder) | packages | read |
| store | store | read and write |

Sector numbers are relative to the partition, and checked against its length, so a client can't name a sector outside
it. The partition table is untrusted input. It's parsed here, in a program with no DMA and no mint authority, with every
field bounds-checked, and a malformed one gives clients nothing.

### 4. Packages on the disk

The packages partition holds an index (a magic, a count, and each package's sector and length) followed by the signed
`.lpkg` files, sector-aligned. The launcher and the binder each read it through the volume, into their own memory, and
verify each package against their trusted publishers before parsing anything. Neither trusts the other's reading, and
neither trusts the disk. The boot image's set goes away, but the format each program keeps in memory, and everything
after it, stays as RFC-0029 built it.

### 5. The store on the disk

The store keeps its files in memory, as now, and makes them durable as **one sealed image**: every file's record and
every block, sealed under the store's key through the keystore, with a sequence number and a hash over the whole. There
are two image slots in the partition, and each write goes to the older one (unresolved question 4). At start, the store
reads both, keeps the newest that verifies, and starts empty if neither does. A write the app was told succeeded has
reached the disk.

The store's key is the keystore's AEAD key, as now (unresolved question 2).

### 6. The disk image

`lantern-mkdisk`, a host tool in `lantern-boot/tools`, writes a raw image: the partition table, the packages partition
with its index, and an erased store partition. `console-matrix.sh` and every suite that boots the system make one.

### 7. The proof

- Host tests for the volume's partition parsing (including hostile tables), the packages index, and the store image's
  sealing, rollover and recovery from a torn or corrupt slot.
- `console-matrix.sh`:
  - `apps` lists the disk's packages;
  - the filesystem app's file survives a reboot;
  - a tampered package on the disk is neither listed nor run;
  - a corrupt store image is ignored, and the store starts empty, saying so;
  - a write cut off mid-way keeps the previous image.
- Mutation tests: a volume that doesn't check a sector against its partition; a store that accepts an image that doesn't
  verify; a kernel that gives any Frame's address.

## Threat model impact

- **The disk is untrusted.** Packages are signed and verified by every reader; the store's image is sealed and
  authenticated; the partition table is parsed with no DMA authority nearby.
- **New: a program with DMA.** Without an IOMMU the driver can program the device to read or write any memory, so it is
  trusted with all memory, like the kernel, though it holds no capability beyond its device. It is small, parses nothing
  from the disk, and serves one client. An IOMMU confines it on hardware that has one (unresolved question 1).
- **Rollback:** someone who can write the disk can put back an older store image that still verifies. Not mitigated
  without a hardware monotonic counter: a row for RFC-0023's checklist.
- **Net change:** larger attack surface (a device, a parser, persistent data), each confined and each separately
  replaceable.

## TCB impact

The kernel gains one Frame invocation and the DMA mark on an `Untyped`. In effect, the driver joins the TCB for memory
until an IOMMU exists, which is why it's kept small and separate from everything that parses.

## Privacy impact

Files now persist. Their contents are sealed, but sizes, write times and how many files exist are visible to someone
holding the disk. With a demo key (unresolved question 2), anyone with the source can unseal them.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** One invocation, gated by a mark only boot can set. The device
   logic, partitions and file format all live in confined programs.
2. **Can an agent use this without being unnecessarily trusted?** An agent reaches storage only through a file grant, as
   now; it never sees a disk, a partition or a sector.
3. **Same interface for humans and agents?** Yes: both get files through the store, by consent.

## Alternatives considered

- **The kernel does block I/O.** Puts a device driver in the TCB; ruled out by ADR-0004.
- **The driver also parses partitions.** One fewer program, but the code holding DMA authority would parse untrusted disk
  bytes.
- **The flash device instead of a disk.** QEMU's second flash bank needs no DMA and is already used by the policy store,
  but no real machine installs onto it, and step 5 needs a disk.
- **Each file as its own on-disk blocks, updated in place.** Scales further, but torn writes and integrity of the file
  table need a journal; the whole-image design is enough for the store's 16 small files today.

## Prior art

- seL4's and Genode's user-level block drivers, and Genode's partition multiplexer (`part_block`).
- Fuchsia's block device stack (driver, partition, then filesystem).
- A/B slots: Android's and ChromeOS's update partitions.

## Unresolved questions

1. **DMA without an IOMMU.** This RFC proposes accepting the driver as trusted with memory, documented, with
   `FrameInvoke::Address` limited to DMA Frames, until hardware with an IOMMU (Phase 4).
2. **The store's key.** This RFC proposes the keystore's demo key, as today, with RFC-0023's existing "Encrypted storage"
   row as the path to a device key. The alternative is a passphrase typed at the console at boot, which would make the
   data really confidential now but adds a login-like step the person decided against for v0.
3. **The partition table.** This RFC proposes GPT now, with LanternOS partition type GUIDs, since step 5's bootable disk
   needs one a firmware can read. The alternative is a minimal LanternOS header now, and GPT at step 5.
4. **The store's format.** This RFC proposes the two-slot whole image, since the store is small. The alternative is a
   journaled per-block layout, which scales, but is much more to get right.

## Future possibilities

- An IOMMU-confined driver on hardware that has one.
- A device-bound key and a monotonic counter (RFC-0023).
- Installing and removing apps on the disk (step 5), and a store that grows beyond 16 small files.

## Resulting ADRs

On acceptance, an ADR will record: DMA Frames and their address, the driver's trust, the volume and its partition
checks, packages on the disk, the store's image format, and the disk tool.
