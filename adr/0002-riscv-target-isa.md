---
adr: 0002
title: RISC-V is the long-term target ISA
status: Accepted
date: 2026-06-09
deciders: ["TSC"]
rfc: null
supersedes: []
superseded_by: null
---

# ADR-0002: RISC-V is the long-term target ISA

## Context

Our **open hardware** and **user sovereignty** principles are incompatible with permanent
dependence on a proprietary, licence-gated ISA. We also want room to *innovate at the ISA
level* — custom extensions for capabilities (CHERI-style), cryptography, and AI — which
closed ISAs do not permit. At the same time, contributors today overwhelmingly develop on
x86-64, and the richest emulation/tooling is there.

## Decision

**RISC-V (RV64GC and beyond) is the long-term preferred ISA.** To make that real without
stalling progress:

- All architecture decisions must remain **compatible with a future RISC-V deployment**;
  ISA-specific logic is confined behind the HAL ([`lantern-hal`](https://github.com/lantern-os/lantern-hal)).
- **Initial development targets x86-64 and `riscv64` under QEMU** for practicality. x86-64
  is a *development convenience*, not a strategic target.
- We track relevant RISC-V extensions: hypervisor (H), vector (V), crypto scalar/vector,
  pointer masking, and CHERI/capability research (the "CHERI Alliance" / Codasip-style
  work), for later adoption.
- We avoid hard dependencies on proprietary platform features (e.g. closed boot ROMs,
  vendor-locked secure enclaves) where an open equivalent is plausible.

## Consequences

- **Easier:** sovereignty, auditability, and the option to co-design hardware/software
  (secure enclave, crypto accelerator, NPU, FPGA region — see
  [`lantern-hal`](https://github.com/lantern-os/lantern-hal) and the Hardware wiki page).
- **Harder:** RISC-V's silicon, firmware, and tooling maturity lag x86/ARM today; we carry
  a portability tax (two early targets) and must keep the HAL boundary disciplined.
- **Committed to:** a clean HAL seam, CI on both x86-64 and `riscv64` emulation, and
  periodic review of RISC-V extensions for capability and crypto acceleration.
