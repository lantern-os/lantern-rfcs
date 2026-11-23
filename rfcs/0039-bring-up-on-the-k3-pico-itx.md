---
rfc: 0039
title: Bring-up on the Banana Pi K3 Pico-ITX
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["boot", "hal", "kernel"]
domains: ["boot", "hal", "kernel", "abi"]
created: 2026-11-22
updated: 2026-11-22
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0039: Bring-up on the Banana Pi K3 Pico-ITX

> **Draft. Direction agreed 2026-11-22:**
> - DMA buffers are mapped uncached (`Svpbmt`), not managed by cache maintenance.
> - Bring-up stays single-hart, booting on whichever hart the firmware chooses.
> - The board keeps its vendor firmware; building firmware from source belongs with secure boot (stream B).
> - IOMMU confinement (stream D) is its own RFC, straight after bring-up.

## Summary

This is [RFC-0036](./0036-phase-3-to-phase-4-transition.md)'s stream **A**, and the installable track's step 6:
LanternOS installed on, and running on, a real RISC-V board. The board is the Banana Pi K3 Pico-ITX (SpacemiT K3, 8 GB),
arriving around 18–20 December 2026.

Today LanternOS runs only on QEMU's `virt` machine, and some of that machine's details are written into the code: where
RAM starts, where the UART and interrupt controller are, which hart boots, how fast the timer runs, and that devices are
virtio on fixed addresses. None of those hold on the K3. This RFC:

1. **Takes each QEMU assumption out,** reading the machine from the device tree the firmware passes instead.
2. **Adds what the K3 needs:**
   - an AIA interrupt driver (the K3 has no PLIC);
   - an NVMe driver on the K3's PCIe host;
   - DMA on a bus that doesn't keep caches coherent.
3. **Does as much as possible in QEMU first,** while the board ships, so the board's arrival starts with testing rather than
   writing code.
4. **Sets the bring-up exit criterion:** the same disk-installed system that boots in QEMU boots on the board from
   power-on, and does what RFC-0032's proof does.

It doesn't confine drivers with the IOMMU (stream **D**), or use the board's root of trust (stream **B**). Each gets its own
RFC once the board is running. Until then the board is a "to run LanternOS" board in RFC-0036's terms, and says so at
boot.

## Motivation

The installable track's goal is an OS a person can install and run in a VM or on hardware. The VM half is done (RFC-0032).
Phase 4's exit criteria need real hardware: a root of trust, IOMMU-confined drivers, measured boot. All of that sits on
top of LanternOS running on the board. Bring-up comes first, and it has to be done properly: an OS that only runs on
hardware because of addresses copied from one board would need redoing for the next.

## The board

What the work depends on, from SpacemiT's K3 datasheet (V1.8), the board's schematic (`SCH-146-V10`), and SpacemiT's
Linux device tree for it (`spacemit-com/linux-6.18`, branch `k3-br-v1.0.y`, `k3-pico-itx.dts`):

| | QEMU `virt` today | K3 Pico-ITX |
|---|---|---|
| CPU | RV64GC, 1 hart | 8 × X100 (RVA23, Sv39, with `Svpbmt`, `Zicbom`, `Sstc`, `Smaia`/`Ssaia`); 8 A100 AI cores, not used |
| Boot hart | 0 | Whichever hart the firmware picks: not necessarily 0 |
| RAM | 256 MiB from `0x8000_0000` | 8 GiB from `0x1_0200_0000`, with firmware and real-time-core regions reserved (`reserved-memory`, `no-map`) |
| Firmware | OpenSBI, then EDK2 (RFC-0032) | Vendor U-Boot SPL, OpenSBI, then U-Boot's UEFI, giving a device tree |
| Console | ns16550a at `0x1000_0000` | 16550-compatible (`intel,xscale-uart`) at `0xd401_7000`, registers 4 bytes apart, 115,200 baud, set up by the firmware; a 3-pin 3.3 V header (`J_UART_DBG1`) |
| Interrupts | PLIC at `0x0c00_0000` | AIA only: an APLIC at `0xe080_4000` delivering MSIs to IMSICs at `0xe040_0000` |
| Timer | 10 MHz, through SBI | 24 MHz (`timebase-frequency`), through SBI |
| Storage | virtio-blk on virtio-mmio | NVMe in the M.2 M-key slot, on PCIe port A (a DesignWare host); also onboard UFS, not used |
| DMA | Cache-coherent | **Not coherent:** the SoC bus is `dma-noncoherent`, and the PCIe hosts aren't marked coherent |
| IOMMU | None | A standard RISC-V IOMMU at `0xc0f0_0000`, covering some PCIe ports (stream D) |
| Entropy | virtio-rng | A TRNG on the SoC (stream B) |
| Network | virtio-net | 10GbE (RTL8127) on PCIe port C; gigabit Ethernet on the SoC; Wi-Fi on PCIe port E |

## Reference-level explanation

### 1. Read the machine from the device tree

Each fixed address becomes a value read from the device tree the firmware passes, with the QEMU value as no special case:

- **RAM:** every `memory` node, minus every `reserved-memory` region and the device tree itself, becomes untyped memory.
  Today `fdt::ram_region` takes one region and assumes nothing is reserved inside it.
- **Where the kernel runs:**
  - **Today:** the kernel is linked to run at `0x8020_0000`, and the EFI loader copies it there.
  - **Proposed:** the EFI loader picks a 2 MiB-aligned place in RAM that the firmware's memory map marks free, and the
    kernel runs there.
  - **The direct map** (RFC-0030), today the first GiB from `0x8000_0000`, maps RAM from wherever RAM starts.
  - **To be decided in the work:** whether the kernel becomes position-independent, or is linked high and mapped. Either
    choice must keep the formal-verification harnesses (RFC-0037) building.
- **The console:** the UART named by `/chosen/stdout-path`, honouring `reg-shift` and `reg-io-width`. The boot loader and
  the console service both use it. The firmware has already set the baud rate, and LanternOS doesn't change it.
- **The timer:** `timebase-frequency` from `/cpus`, instead of the 10 MHz constant in `lantern-hal`. Every deadline
  (RFC-0028) and the preemption quantum (RFC-0027) are in nanoseconds already; only the conversion changes.
- **Devices:** the launch tables that map each driver's MMIO and grant its interrupt come from device tree nodes, not
  constants. QEMU's virtio-mmio slots are found by scanning its `virtio,mmio` nodes.

A device tree LanternOS can't use is refused at boot, with the reason on the console, never guessed around.

### 2. Any boot hart

`entry.rs` parks every hart but hart 0. On the K3 the firmware may boot on another hart, and LanternOS would then park
itself and print nothing. The boot hart becomes whichever hart enters first (`a0`, from the EFI loader's
`RISCV_EFI_BOOT_PROTOCOL` call), and its ID is kept for the interrupt controller. Other harts aren't started: LanternOS
stays single-hart (ADR-0010). **Decided:** single-hart for bring-up; using more harts is its own RFC.

### 3. AIA interrupts

A driver for the APLIC in MSI mode and the boot hart's supervisor IMSIC, behind the same kernel interface the PLIC driver
serves today (RFC-0027): enable a line, claim, complete. The kernel chooses PLIC or AIA from the device tree. QEMU runs it
with `-machine virt,aia=aplic-imsic`, and every QEMU suite runs under both, so neither driver rots.

### 4. Storage: NVMe on PCIe

Two parts, both confined user-space programs like today's block driver (RFC-0031), serving the same block protocol so the
volume service doesn't change:

- **A PCIe host driver** for the K3's DesignWare root complex on port A: link up, configuration access through the
  address-translation unit, and bus numbering, enough to find one NVMe device. On QEMU, the generic ECAM host
  (`pci-host-ecam-generic`) stands in, so enumeration code is shared and only the host part differs.
- **An NVMe driver:** one admin queue and one I/O queue, polled or on one interrupt, reading and writing 512-byte or 4 KiB
  sectors. QEMU's `-device nvme` tests it.

`lantern-mkdisk`'s disk layout doesn't change. Installing means writing the image to the NVMe drive from another computer,
as RFC-0032 installs to a VM's disk.

### 5. DMA without coherent caches

On QEMU, a device sees memory as the CPU does. On the K3 it doesn't: a buffer the CPU just wrote may still be in its cache
when the NVMe drive reads it, and data the drive wrote may be hidden behind stale cache lines. Every DMA buffer needs one of:

- **(a) Uncached mappings:** the kernel maps DMA frames into a driver with `Svpbmt`'s non-cacheable type, so neither side
  sees a stale copy. It's correct by construction, and slower for large transfers.
- **(b) Cache maintenance:** drivers clean or invalidate each buffer (`Zicbom`) around every transfer. It's faster, but each
  missed instruction is a silent data-corruption bug that QEMU can't show.

**Decided: (a).** A frame a driver may give a device is mapped non-cacheable wherever it's mapped, and which frames are DMA
frames is a capability property the kernel enforces, not a driver convention. (b) can come later, for speed, by its own
RFC, once there's a way to test it.

### 6. Firmware

The board boots its vendor firmware (U-Boot SPL with SpacemiT's DDR setup, OpenSBI, then U-Boot's UEFI) from its own
storage. U-Boot runs `\EFI\BOOT\BOOTRISCV64.EFI` from the NVMe drive's EFI partition, exactly as EDK2 does in QEMU. **Decided:**
bring-up keeps the vendor firmware. Building it from SpacemiT's sources belongs with secure and measured boot, stream B,
and nothing here writes the board's eFuses.

### 7. What LanternOS says about the board

At boot, LanternOS states which RFC-0036 tier the machine meets, and lists the Phase 3 software-only limits that still
apply: no IOMMU confinement, no hardware entropy, no root of trust. The same check runs in QEMU, where it says the same.

### 8. Before the first boot of LanternOS on the board

Nothing of LanternOS's touches the board until these are done, in order:

1. **Boot the vendor Linux** from the onboard UFS, to show the board works.
2. **Keep a copy of the device tree the firmware passes** (`/sys/firmware/fdt`), the boot log, and the firmware versions.
   LanternOS is tested against that copy, under QEMU too, through `-dtb`, for the parts QEMU can run.
3. **Read the eFuse banks without writing them**, to learn whether secure boot is unprogrammed and the root-key banks are
   empty. That's input for stream B's RFC. Programming fuses is irreversible, and is never part of bring-up.
4. **Test the vendor firmware's recovery path** (the firmware-update button and USB download) before anything else is
   flashed, so a bad write can be undone.

### 9. Testing on the board

The console matrix and boot-disk proof drive QEMU over a pipe. On the board, the same cases drive the serial console
through a USB serial adapter, with a person power-cycling where a case needs a reboot. Each hardware run is recorded
(firmware version, device tree copy, results) under `lantern-boot/hardware/k3-pico-itx/`.

## Exit criterion

On the K3 Pico-ITX, from power-on:

- **It boots from the disk:** LanternOS installed by writing `lantern-mkdisk`'s image to the NVMe drive boots through the
  board's UEFI, with nothing else.
- **RFC-0032's proof passes:** a grant decided "always" at the console, an app run, a file kept, then the same disk
  rebooted with the decision and files still there.
- **It states its tier:** LanternOS says which RFC-0036 tier the board meets, and which software-only limits apply.
- **QEMU still passes:** every QEMU suite passes, under both PLIC and AIA.

Networking and the wallet aren't required. Without a network or an entropy source they refuse, as they already do in
QEMU, saying why.

## Sequencing

**Before the board arrives, in QEMU:**

1. The device tree work and any-boot-hart (1, 2).
2. AIA (3), running every suite under both interrupt controllers.
3. NVMe and generic PCIe enumeration (4), with the system image booting from an NVMe disk.
4. Uncached DMA frames (5).
5. The serial-console runner, pointed at QEMU's serial port, so the same runner drives the board later.

**When the board arrives:**

1. The pre-flight steps (8).
2. The UART and boot-hart path alone: the EFI loader and kernel printing to the board's console.
3. The DesignWare host driver, then NVMe on the board.
4. The exit criterion.

## Progress: the QEMU half

Everything in "Before the board arrives" is built and passes under QEMU, as of 2026-11-23. The exit criterion is the
board's, so this RFC stays a Draft until it passes there. What was decided in the work:

- **Reading the machine (§1).**
  - **The device tree reader:** `lantern-boot/src/fdt.rs` is a general but bounded reader: paths, phandles, cell
    sizes, at most 16 levels deep.
  - **The platform:** `src/platform.rs` builds the platform from it: RAM, reservations, timer, console UART and its
    interrupt, PLIC or AIA, virtio-mmio devices, an ECAM PCIe host, ISA extensions, DMA coherence.
  - **Fixtures:** host tests run it on QEMU's trees (PLIC and AIA) and on the K3's, built from SpacemiT's sources
    (`lantern-boot/testdata/`). Every truncation and a scatter of corrupted bytes is an error, never a panic.
- **Until the tree is read, boot prints through the SBI console**, so even a tree it can't use is reported.
- **The kernel became a static position-independent executable**, not one linked high and mapped.
  - **The build:** `lantern-boot/build.rs` links boot images `-pie` with `-z notext`. RISC-V's default `medany` code is
    already PC-relative, so the only relocations are 403 `R_RISCV_RELATIVE` data words.
  - **Applying them:** `src/entry.rs` applies them for the load offset before anything reads them, and refuses any
    other kind.
  - **The proofs:** RFC-0037's harnesses are untouched by this; the kernel's code is the same.
- **The EFI loader:**
  - **Placing the kernel:** it puts the kernel in the lowest free 2 MiB-aligned range of the firmware's map, keeping
    RAM above it in one run.
  - **The device tree:** it copies the tree into its own pages before `ExitBootServices`. Testing a kernel loaded away
    from its link address found that EDK2 reclaims its own copy: the kernel found it overwritten. Linux's EFI stub
    copies it for the same reason.
  - **Reserved memory:** it hands the kernel, in `a2`/`a3` behind a magic, every range the memory map it exits with
    says isn't free: firmware runtime, reserved, non-volatile, MMIO. Under EDK2 that's 4 ranges the kernel would
    otherwise have reused.
- **The direct map** starts at boot's general memory, wherever RAM is, and spans up to 16 GiB, one shared L1 table per
  GiB. A machine with more uses the first 16 GiB and says so at boot.
- **Any boot hart (§2):** the first hart to arrive wins an atomic flag. Under QEMU's `-smp 4`, OpenSBI picks harts 1, 2
  and 3 as often as 0; before this, LanternOS parked itself on those boots.
- **AIA (§3):** the APLIC in MSI mode, with sources sent as identity `n` to the boot hart's supervisor IMSIC; `sourcecfg`
  level-high; a still-asserted level source re-sent on completion. It picks the supervisor-level controllers from trees
  that also describe the machine level, as QEMU's does.
- **The console reaches the shell in `arg0`** (`lantern_abi::console`): register spacing, width, offset, and whether
  it's an XScale-style UART. The K3's is: its interrupt-enable register's unit-enable bit, which the shell used to clear,
  turns the UART off.
- **NVMe (§4).**
  - **Boot's part:** boot, not a separate host driver, finds an NVMe controller on an ECAM host's root bus. It sizes
    BAR0, keeps the firmware's assignment if it's in either memory window (EDK2 uses the 64-bit one) or assigns one, and
    enables decoding and bus mastering.
  - **Why boot does it:** the PCIe host is configured once, at boot, like the virtio slots it already scans. A
    DesignWare host (the K3's) needs its own set-up, which waits for the board.
  - **The driver:** `nvme-driver` is the virtio-blk driver's counterpart: same capabilities, same block protocol. It has
    one I/O queue pair, 512-byte blocks only, flushes through a volatile write cache, and uses INTx when the host routes
    one, polling otherwise.
- **Uncached DMA (§5).**
  - **Mapping:** where the boot hart has `Svpbmt`, every DMA frame is mapped with PBMT = NC wherever it's mapped.
  - **Zeroing:** where it also has `Zicbom`, the kernel flushes a DMA frame's lines after zeroing it through its
    cacheable direct map.
  - **Refusal:** a tree with a `dma-noncoherent` bus and a boot hart lacking either is refused. The K3's tree has such a
    bus, and its cores have both, with 64-byte blocks.
  - **Testing:** QEMU's default CPU lacks `Svpbmt`, so suites also run with `-cpu rv64,svpbmt=on`, and a host test
    checks a DMA frame's PTE. QEMU models no caches, so only the board can show the flushes matter.
- **The runner (§9):** `scripts/board-run.sh` runs RFC-0032's proof over a serial device. `QEMU=1` runs it against QEMU
  on a pseudo-terminal, power-cycling by restarting QEMU. `SERIAL=/dev/ttyUSB0` runs it on the board, asking a person to
  power-cycle, and records each run under `lantern-boot/hardware/`.
- **Every QEMU script takes the machine:**
  - **`MACHINE=virt,aia=aplic-imsic`:** AIA instead of the PLIC.
  - **`SMP`:** the hart count, so the boot hart varies.
  - **`CPU=rv64,svpbmt=on`:** uncached DMA.
  - **`DISK_BUS=nvme`:** an NVMe disk instead of virtio.
  - **`scripts/any-hart.sh`:** boots until the firmware picks a hart other than 0, under each controller, and checks
    interrupts reach it.

**Still to do on the board:**

- the pre-flight steps (§8);
- the DesignWare host's set-up;
- whether U-Boot's UEFI there gives the boot protocol and a usable memory map;
- the exit criterion.

**Found along the way, recorded for later.** Programs' fixed virtual addresses can collide with the kernel's identity
mappings, which sit at physical addresses. On the K3 they can't (its RAM and devices are far above programs' layouts),
and on QEMU the kernel lands at its link address. A collision makes a program's load fail, loudly. Moving the kernel's
mappings out of programs' half of the address space is future work.

## Threat model impact  *(mandatory)*

- **No new trust is given to hardware.** Drivers stay confined to their capabilities, but DMA-capable devices can still
  reach all memory, as on QEMU: RFC-0023's "DMA" and "Driver isolation" rows stay open, for stream D.
- **A device tree is parsed from firmware** that LanternOS doesn't verify. It's already the case in QEMU (RFC-0032), and
  "Boot chain" stays open for stream B. The parser treats the tree as untrusted input: bounded, and refusing what it can't
  parse rather than guessing. It gets host fuzz tests.
- **DMA without coherence** would be a new way for a driver to read stale data. DMA frames are mapped uncached, and which
  frames are DMA frames is enforced by the kernel.
- **The vendor firmware is trusted,** as EDK2 and OpenSBI are in QEMU, and is recorded as such.

## TCB impact  *(mandatory)*

- **The kernel gains the AIA driver,** about the PLIC driver's size, chosen at boot.
- **The kernel's memory set-up reads more of the device tree:** reserved regions, and RAM anywhere.
- **The NVMe driver is user space,** confined like the block driver, and not in the TCB. As built, boot configures the
  PCIe host itself, as it already scans the virtio slots: it reads configuration space and writes one function's BAR0 and
  command register, a few dozen lines, in the TCB. A DesignWare host's link and translation-unit set-up, the K3's, is
  larger; where it goes is decided on the board.
- **The kernel changes are covered:** RFC-0037's Kani harnesses still build, and any kernel path whose shape changes gets
  its harness updated.

## Privacy impact

None. Hardware runs record the board's firmware versions and device tree, not personal data.

## The three vision questions

1. **Smaller, safer, easier to reason about:** yes. Constants copied from one machine are replaced by stated inputs,
   checked once at boot.
2. **An agent using a capability without being unnecessarily trusted:** unchanged. Drivers stay confined programs serving
   the same protocols.
3. **One interface for humans and agents:** unchanged.

## Alternatives considered

- **A board-specific build, with the K3's addresses as constants.** It's quick, but it repeats the QEMU problem: each new
  board would be a fork.
- **Using the onboard UFS instead of NVMe.** The UFS controller isn't behind the IOMMU. NVMe on port A is, which is what
  stream D needs.
- **Waiting for the board before writing code.** Most of the work runs in QEMU. Waiting would put four weeks of coding after
  the board's arrival instead of before it.

## Unresolved questions

- **Whether U-Boot's UEFI on this board implements `RISCV_EFI_BOOT_PROTOCOL`.** If not, the boot hart comes from
  `/chosen/boot-hartid`.
- **Whether the firmware brings up the PCIe link before LanternOS runs.** It affects how much the host driver does.
- **Which K3 PCIe ports the IOMMU really covers.** The datasheet says A, B and E; the device tree maps A, B and C. Stream
  D's RFC tests it on the board.

## Resulting ADRs

To be written when this RFC is accepted.
