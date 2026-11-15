---
rfc: 0032
title: Installing — a disk a machine boots from, through UEFI
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["boot", "shell", "docs"]
domains: ["boot", "shell", "abi", "docs"]
created: 2026-11-14
updated: 2026-11-14
supersedes: []
superseded_by: null
tracking_issue: null
--> **Accepted 2026-11-14, implemented and recorded in [ADR-0037](../adr/0037-installing-a-bootable-disk.md).** Built as
> proposed. Settled along the way:
> - QEMU must run `-machine virt,acpi=off`: with ACPI on, EDK2 gives no device tree.
> - Every disk has the policy partition, not only a bootable one, so the `-kernel` suites use it too.
> - `lantern-mkdisk --update` rewrites only the packages.
> - The loader is linked without RELRO, so its relocations sit in a read-only page of their own.
>
-

> **Direction agreed 2026-11-14; still a Draft until implemented.** The unresolved questions were settled as proposed:
> - UEFI with a Lantern loader;
> - the policy store on a disk partition;
> - the bootable image as this step's install flow;
> - a host tool that converts the loader's ELF to PE/COFF.

# RFC-0032: Installing — a disk a machine boots from, through UEFI

## Summary

Step 5 of the roadmap's "Toward an installable system" track. Today QEMU starts LanternOS with `-kernel`: the emulator
puts the boot image in memory itself, so no real machine could start it. After this RFC, **a machine boots LanternOS from
its disk**:

1. The machine's UEFI firmware (EDK2 on QEMU `virt`; U-Boot's UEFI on most RISC-V boards) finds an EFI System Partition
   on the disk.
2. It runs a small **Lantern EFI loader**, which reads the boot image from that partition, places it where the kernel
   runs, leaves UEFI behind, and jumps to it, exactly as OpenSBI does today.
3. The system then finds its packages and its store on the same disk (RFC-0031).

**Installing** is writing that disk: `lantern-mkdisk` produces a bootable image, which a VM uses as its disk, and which is
written whole to a real machine's disk.

## Motivation

The track's destination is an OS a person installs and runs, in a VM or on hardware. Without a bootable disk there's
nothing to install. Every other step (an interactive system, launching, storage) is already in place for it.

Principles served: the installable-system goal, and **verifiability** (the boot chain becomes explicit: firmware,
loader, kernel, each with one job). In tension: **TCB size**, since the loader runs before the kernel and is trusted.

## Guide-level explanation

```
$ lantern-mkdisk --bootable --out lantern.img --package probe.lpkg --package fs-probe.lpkg
$ qemu-system-riscv64 -machine virt -m 256M \
    -drive if=pflash,unit=0,format=raw,readonly=on,file=RISCV_VIRT_CODE.fd \
    -drive if=pflash,unit=1,format=raw,file=vars.fd \
    -drive if=none,format=raw,file=lantern.img,id=disk -device virtio-blk-device,drive=disk
   ... UEFI ... Lantern EFI loader ... LanternOS. This console is the shell's alone ...
lantern> run confined-fs-probe
```

On hardware, `lantern.img` is written to the boot disk, as an image for a single-board computer is.

## Reference-level explanation

### 1. The disk

`lantern-mkdisk --bootable` adds a third GPT partition, the **EFI System Partition** (standard type GUID), formatted FAT,
holding `\EFI\BOOT\BOOTRISCV64.EFI` (the loader) and `\LANTERN\BOOT.ELF` (the boot image, as `lantern-boot-system` is
today). The packages and store partitions stay as RFC-0031 made them.

### 2. The loader

A new program, `lantern-boot/efi-loader`, a UEFI application:

1. It finds `\LANTERN\BOOT.ELF` on the partition it was loaded from (`EFI_LOADED_IMAGE_PROTOCOL`, then
   `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`), and reads it into memory UEFI allocates.
2. It allocates the boot image's physical range, from 2 MiB above the start of RAM, as OpenSBI's jump address is today,
   and copies each `PT_LOAD` segment there. It refuses an ELF that doesn't fit, or isn't RISC-V.
3. It finds the device tree (`EFI_DTB_TABLE_GUID` in the configuration table) and the boot hart (the RISC-V boot
   protocol, or the device tree's `/chosen/boot-hartid`).
4. It calls `ExitBootServices`, turns paging off, and jumps to the entry with `a0` = hart and `a1` = device tree: the
   contract OpenSBI's jump firmware already gives the kernel. Nothing in the kernel changes.

It is built as a position-independent riscv64 ELF and converted to the PE/COFF format UEFI loads (with its base
relocations) by a small host tool, since Rust has no RISC-V UEFI target (unresolved question 4).

**It doesn't verify the boot image.** Secure boot (the firmware verifying the loader, and the loader the image) needs keys
in firmware a person controls, which is Phase 4's hardware work. RFC-0023's checklist gains the row.

### 3. The policy store moves to the disk

UEFI on QEMU `virt` keeps its variables in the second flash bank, where the policy store lives today, and a real board has
no such flash. The policy store moves to a fourth partition (unresolved question 2), as the volume's fourth client. It
keeps its log format, but on sectors instead of flash words:

- Erased means `0xFF`, as on flash, since `lantern-mkdisk` fills the partition with it.
- A record is written whole, sector by sector, with a read before each write. The existing torn-write rules stand: a
  record is valid only once complete.

The `-kernel` images that test the policy store on flash (`binder-matrix.sh`) keep doing so: those are demos, not the
system.

### 4. Installing

**This step's install flow is the image** (unresolved question 3): `lantern-mkdisk --bootable` writes the disk, a VM uses
it, and on hardware it's written to the machine's disk. An installer that runs inside LanternOS and writes another disk is
future work: it needs a second disk, and a decision about what a person may erase.

### 5. The proof

- Host tests for the FAT writer, the PE conversion, and the loader's ELF placement and checks (on a fake address space).
- `scripts/boot-disk.sh`: QEMU with EDK2 and only the disk boots to the shell, then `run confined-fs-probe` and a reboot
  show the file survived, and remembered decisions survived too, now on the disk.
- `console-matrix.sh` runs on the bootable disk.
- Mutation tests: a loader that copies a segment to the wrong place (the system mustn't come up), and a policy store that
  ignores a torn record (the matrix's torn-record cases must fail).

## Threat model impact

- **New trusted code before the kernel:** the loader. It's small, parses one ELF it reads from its own partition, and
  hands over to the kernel with UEFI gone.
- **Not mitigated:** someone who can write the disk can replace the loader or the boot image. Without secure boot, the
  firmware runs whatever is there. RFC-0023's checklist: a verified boot chain.
- **The policy store on the disk** is as unprotected offline as it was on flash: it was never keyed.

## TCB impact

The loader joins the TCB, as OpenSBI's jump firmware is. The kernel is unchanged.

## Privacy impact

None beyond RFC-0031's: the boot partition holds code, not data.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** Unchanged: the loader ends before the kernel starts, and gives it the
   same contract OpenSBI does.
2. **Can an agent use this without being unnecessarily trusted?** No agent touches booting.
3. **Same interface for humans and agents?** Not applicable.

## Alternatives considered

- **U-Boot without UEFI** (`booti` or `go` on a raw image). Simple, but it isn't installed here, and UEFI is what both EDK2
  and U-Boot offer.
- **Turning the kernel itself into an EFI program.** One fewer stage, but the kernel would carry UEFI's calling convention
  and memory map handling, and a different entry on every firmware.
- **Keep `-kernel` for VMs.** Then nothing is installable, which is this step's point.

## Prior art

- Linux's EFI stub, and `systemd-boot`.
- `elf2efi` and Limine's EFI build, for producing PE/COFF from an ELF.
- Single-board computer images, written whole to a card or disk.

## Unresolved questions

1. **The boot route.** This RFC proposes UEFI, with a Lantern loader. The alternative is to keep `-kernel` until hardware.
2. **Where the policy store goes.** This RFC proposes a fourth disk partition, served by the volume. The alternative is to
   keep it on flash, and boot UEFI without persistent variables, which QEMU allows but no board does.
3. **The install flow.** This RFC proposes the image is the install flow for this step, with an in-system installer as
   future work. The alternative is an installer inside LanternOS now, writing a second disk.
4. **Building the loader.** This RFC proposes a small host tool that turns a position-independent ELF into PE/COFF. The
   alternative is a hand-written PE header in the loader's own assembly, as Linux's riscv `Image` does.

## Future possibilities

- An in-system installer, and updating the boot image from inside LanternOS.
- A verified boot chain: secure boot for the loader, the loader verifying the image.
- Booting on a RISC-V board (step 6).

## Resulting ADRs

On acceptance, an ADR will record: the boot chain, the loader's contract, the bootable disk's layout, the policy store's
move, and the install flow.
