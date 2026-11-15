---
adr: 0037
title: Installing — a disk a machine boots from, through UEFI
status: Accepted
date: 2026-11-14
deciders: ["TSC"]
rfc: ../rfcs/0032-installing-a-bootable-disk.md
supersedes: []
superseded_by: null
---

# ADR-0037: Installing — a disk a machine boots from, through UEFI

## Context

QEMU started LanternOS with `-kernel`, which no real machine can do, so nothing was installable.
[RFC-0032](../rfcs/0032-installing-a-bootable-disk.md), step 5 of the roadmap's "Toward an installable system" track,
proposed booting from the disk through UEFI. It has been accepted; this ADR is the durable record.

## Decision

**A machine's UEFI firmware runs the Lantern EFI loader from the disk's EFI System Partition. The loader places the boot
image where the kernel runs and starts it, with the contract OpenSBI gives. Installing is writing the disk image
`lantern-mkdisk --bootable` makes.**

- **The loader** (`lantern-boot/efi-loader`), run as `\EFI\BOOT\BOOTRISCV64.EFI`:
  1. It reads `\LANTERN\BOOT.ELF` from its own partition.
  2. It claims the kernel's 4 MiB from `0x8020_0000`, and places each segment there (`plan`, `place`, host-tested). It
     refuses a segment outside that range or the file, and an entry outside every segment.
  3. It finds the device tree in the configuration table, and the boot hart through `RISCV_EFI_BOOT_PROTOCOL`.
  4. It calls `ExitBootServices`, masks interrupts, turns paging off, runs `fence.i`, and jumps with `a0` = hart and `a1`
     = device tree.

  The kernel is unchanged. **The loader verifies nothing**: RFC-0023's checklist has the row.
- **Building it:**
  - Rust has no RISC-V UEFI target, so the loader is a position-independent riscv64 ELF (`-pie`, `-znorelro`), linked
    from `0x1000`.
  - `tools/elf2efi` makes it a PE32+ EFI application: one section per segment, with that segment's permissions, and each
    `R_RISCV_RELATIVE` becoming a `DIR64` base relocation.
  - `file` reads the result as a RISC-V PE32+ EFI application, and EDK2 runs it.
- **The disk** (`lantern-mkdisk --bootable`):
  - an EFI System Partition (32 MiB, FAT16, written by `tools/mkdisk`'s own FAT writer, with long names), which
    `fsck.fat` passes;
  - RFC-0031's packages and store partitions;
  - a **policy partition** (2 MiB of `0xFF`). Every disk now has one.
  - `--update` rewrites only the packages partition of an existing disk.
- **The policy store moves to the disk.** UEFI keeps its variables in QEMU's second flash bank. With its new `ON_DISK`
  flag, the policy store mirrors its partition in memory and writes each change through, a sector at a time, before it
  returns. A write the disk refused is undone in the mirror, so the log's read-back sees it fail. The log's format is
  unchanged, so `audit-dump` reads the partition as it read flash. The demos keep it on flash.
- **Running it:** `scripts/make-image.sh` builds `lantern.img`. QEMU boots it with EDK2, `-machine virt,acpi=off` (with
  ACPI on, EDK2 gives no device tree) and the disk alone, as `lantern-boot/README.md` shows.

## Consequences

- **Easier:** LanternOS is installable. A VM boots it from its disk, and the same image is what a machine's disk would
  hold.
- **Harder / committed to:**
  - The loader is trusted code before the kernel.
  - Booting takes EDK2's own start-up, and its 5-second prompt.
  - The policy store reads its whole partition at start.
- **Not mitigated:** whoever can write the disk can replace the loader or the boot image, and the firmware runs it. A
  verified boot chain is on RFC-0023's checklist.
- **TCB:** the loader joins it, as OpenSBI's jump firmware did. The kernel is unchanged.
- **Still not done:**
  - An installer inside LanternOS, and updating the boot image from inside it.
  - Secure boot.
  - EDK2's ACPI path.
  - A real board (step 6).
