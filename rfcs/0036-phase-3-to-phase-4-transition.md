---
rfc: 0036
title: Closing Phase 3 and opening Phase 4 (hardware and assurance)
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["governance", "kernel", "hal", "boot", "crypto", "filesystem", "shell", "network", "ai-runtime"]
domains: ["governance", "kernel", "hal", "boot", "crypto", "filesystem", "shell", "network", "ai-runtime", "identity"]
created: 2026-11-16
updated: 2026-11-16
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0036: Closing Phase 3 and opening Phase 4 (hardware and assurance)

> **Accepted 2026-11-16.** Phase 3 is closed and Phase 4 is open; [ADR-0041](../adr/0041-phase-3-complete-phase-4-opened.md)
> is the durable record. The PRs carrying RFC-0033, RFC-0034 and RFC-0035 merged, and `main` was verified from fresh
> clones the same day.
>
> **Direction agreed 2026-11-16:**
> - Close Phase 3 now, with the narrow points stated.
> - Formal verification moves to the head of Phase 4.
> - Minimum hardware has two tiers: to run, and to exit Phase 4.
> - Phase 4 starts with formal verification while the board is chosen. Which board is still open.

## Summary

Phase 3's three exit criteria are met (evidence below). This RFC closes Phase 3 and opens **Phase 4 — Hardware and
assurance**, following [RFC-0004](./0004-phase-0-to-phase-1-transition.md), [RFC-0009](./0009-phase-1-to-phase-2-transition.md)
and [RFC-0017](./0017-phase-2-to-phase-3-transition.md).

It does three things beyond flipping the markers:

- **It says plainly where each criterion was met narrowly.** The agent's model is tiny. Identities can't be linked by
  their keys, but can by the network.
- **It carries Phase 3's unfinished deliverables forward by name.** Formal verification hasn't started, and Phase 4's own
  exit criterion depends on it. DIDs and credentials, peer discovery and an anonymity transport aren't built either.
- **It accounts for every item in [RFC-0023's checklist](./0023-hardware-key-custody-in-phase-4.md#the-phase-4-checklist)**,
  which RFC-0023 made a condition of opening Phase 4. Each item gets a Phase 4 work stream and a note on whether it needs
  the hardware.

## Motivation

`GOVERNANCE.md` requires an RFC at a roadmap phase boundary. The Roadmap's gate is quality, not the calendar: "we would
rather move a date than lower a bar". RFC-0023 moved hardware key custody out of Phase 3 on the condition that the RFC
opening Phase 4 could not quietly drop any of it. This is that RFC.

There is also a practical reason. A RISC-V board is on order (the installable-system track's step 6). Phase 4 work that
needs no hardware should be sequenced to start now, with bring-up following once the board arrives.

## Phase 3 exit criteria: the evidence

| Criterion | Met by | How it was shown |
|---|---|---|
| **Data at rest is encrypted** (under software-held keys, until Phase 4) **and provenance-tracked** | [RFC-0031](./0031-storage-a-block-driver-and-the-store-on-disk.md)/ADR-0036 (the store sealed on the disk), [RFC-0033](./0033-provenance-for-data-at-rest.md)/ADR-0038 (each file keeps which apps wrote it, when, in which boot) | `console-matrix.sh`: files outlive a reboot sealed; `files` shows each writer; a writer the store didn't record can't be claimed |
| **A user can run a capable local AI agent with a visible, revocable capability set and a faithful audit log** | [RFC-0034](./0034-local-inference-and-a-file-assistant.md)/ADR-0039 | The file assistant, an ordinary app, reads a granted file, calls a confined inference service, and writes its continuation. Each capability is asked for at the console, listed (`grants`), revocable (`revoke`), and recorded in the audit log before it's used, model calls included. The output is attributed to the app. The confined output matches the host reference bit for bit |
| **A network identity can be presented without cross-context linkage** | [RFC-0035](./0035-pairwise-network-identities.md)/ADR-0040 | `net-matrix.sh`, against two peers using an independent Noise implementation (`snow`): each sees a different identity; each sees the same one again, and after a reboot; a declined peer is never contacted; a forged handshake is refused; a tampered or unrecognisable sealed root is neither used nor overwritten |

Each was verified with host tests, every QEMU suite, and deliberate breakage ("mutation") runs, and was verified again
from fresh clones of `main` after merging.

### Where each was met narrowly

These were all stated in the RFCs that met the criteria. They're repeated here because a phase gate is where they would
otherwise be forgotten.

- **"Capable" agent.** The model is stories15M: 15 million parameters, int8, CPU-only, and it continues text rather than
  following instructions. What Phase 3 proved is the **boundary**: a model behind a capability, reached only by consent,
  every call audited, its output attributed. A larger model, or an accelerator, sits behind the same interface. Running
  one is hardware-dependent, so it belongs with the board.
- **"Without cross-context linkage."** No two contexts' keys can be linked: each comes from a separate pseudorandom
  derivation of a root only the wallet holds. The **network** still links them: both peers see one IP address, and the
  timing. Closing that needs an anonymity transport.
- **"Encrypted."** Under software-held keys derived from demo seeds, which protect data from other confined programs but
  not from someone who has the disk. RFC-0023 moved device-theft protection to Phase 4, and made it an exit criterion
  there.

## What Phase 3 leaves unfinished

| Deliverable (Roadmap, Phase 3) | State | Proposed home |
|---|---|---|
| Begin **formal verification** of the IPC and capability paths | **Not started** | **Phase 4, first.** Phase 4's exit criterion requires machine-checked proofs of the most critical kernel paths, and none of this needs hardware. Its own RFC chooses the tool and the first properties |
| DIDs, verifiable credentials | Not built; pairwise identities and the wallet are their foundation | Continues as ordinary Phase 3-scope work, alongside Phase 4 |
| P2P discovery; a first anonymity-capable transport | Not built; authenticated, encrypted channels are | Continues alongside Phase 4. The anonymity transport is what closes the network-level linkage above |
| Encrypted storage "tied to hardware-backed keys" | Moved to Phase 4 by RFC-0023 | Phase 4: the checklist below |

RFC-0004, RFC-0009 and RFC-0017 set the precedent: deliverables beyond the exit criteria continue as ordinary work and
don't hold a phase open. Formal verification is the exception. It moves onto Phase 4's critical path, because Phase 4
can't exit without it.

## The RFC-0023 checklist, accounted for

RFC-0023 requires this RFC to account for every item. Each gets a Phase 4 work stream:

- **B**, a hardware root of trust: a per-device key, hardware entropy, custody, secure and measured boot, attestation.
- **C**, rollback resistance: a monotonic counter, and keyed logs.
- **D**, IOMMU-confined drivers.
- **F**, signed models and a root of trust for publishers.

"Board?" says whether the work needs the real hardware. Board bring-up itself is stream **A**, and formal verification is
**E**.

| Item | Today | Stream | Board? |
|---|---|---|---|
| Key seeds | Public demo constants | B: a per-device root key | Yes |
| Randomness | Caller-supplied bytes (the wallet alone uses virtio-rng) | B: a hardware-seeded generator for every service | Yes |
| Key custody | In the confined keystore's memory | B: keys that resist even a compromised service | Yes |
| Encrypted storage | Software-held key | B: a key bound to the device | Yes |
| Policy store | Unkeyed log; erasing it restores revocations | C: a keyed log and a monotonic counter | Counter: yes. Keyed log format: no |
| Audit log | Tamper-evident only against partial edits | C: hardware-held signing and a monotonic counter | Counter: yes. Format: no |
| Remembered decisions | Unkeyed: someone who can write the log can plant an "always" | C: keyed, rollback-resistant | As the policy store |
| DMA | No IOMMU on QEMU `virt`'s MMIO devices: the block and network drivers can reach all memory | D | Yes, unless QEMU's RISC-V IOMMU is usable first (an open question) |
| Store on disk | An older store image that still verifies can be put back | C: a monotonic counter the image is bound to | Yes |
| Boot chain | Firmware runs any loader; the loader runs any image | B: secure boot, with keys a person controls | Yes |
| Audit retention | Retired history erased; headers and retirements unsigned | C: signed headers and retirements, and export before erasure | Signing: yes. Export format: no |
| Trusted publishers | Demo keys compiled in | B/F: a root of trust for what the system trusts | Partly |
| Boot | No measured boot | B: measured boot and attestation | Yes |
| Driver isolation | Flash only, because there's no IOMMU | D | Yes |
| Minimum hardware | Open question | **Decided by this RFC (below)** | — |
| Wallet root | Sealed under a demo keystore key on the disk | B: in hardware, never leaving it; recovery that isn't a backdoor | Yes |
| Entropy | virtio-rng, which the host controls | B: a trusted hardware generator behind the same protocol | Yes |
| Model integrity | Digest compiled into an unverified boot image | F: models as signed packages, checked under the root of trust, in a verified boot chain | Packages: no. Trust: yes |

Nothing is dropped. Every row stays open until its stream closes it, and Phase 4's exit criteria ("a hardware root of
trust holding every key; device theft yields only ciphertext; the revocation and audit logs resist tampering and
rollback") can't be met while any row is open.

### Minimum hardware

The checklist's open question needs an answer before bring-up chooses a direction. Decided, two tiers:

- **To run LanternOS** (the bring-up target): RV64GC with Sv39, a PLIC or AIA interrupt controller, an SBI timer, a
  device tree, and storage the system can drive. A board meeting only this runs Phase 3's system, with Phase 3's
  software-only limits, stated at boot.
- **To meet Phase 4's exit criteria:** additionally an IOMMU covering every DMA-capable device; an entropy source (the
  `Zkr` extension's `seed` CSR, or a hardware generator); and a root of trust that holds keys and a monotonic counter
  without releasing them (an on-chip enclave, or a discrete secure element).

So a board can be useful for bring-up without being enough to exit Phase 4. LanternOS should say which kind of board it is
running on, not assume.

## Phase 4 scope and sequencing

The Roadmap's Phase 4 scope is unchanged: real RISC-V hardware and IOMMU-confined drivers; hardware-backed custody,
measured boot, attestation; exploring CHERI; extending formal verification. Proposed order, by what each needs:

1. **Now, without hardware:**
   - **E**, formal verification: the RFC choosing a tool and the first properties. Candidates are capability derivation
     and revocation, and the IPC fast path.
   - The hardware-independent parts of **C** (keyed log formats, signed audit headers) and **F** (models as signed
     packages).
2. **When the board arrives:** **A**, bring-up, which is the installable track's step 6. Then **B** and **D**, as far as
   the board's hardware allows.
3. **Throughout:** each stream's design gets its own RFC. This RFC pre-approves none of them.

## What this RFC does not do

- It doesn't design any Phase 4 work stream.
- It doesn't change Phase 4's exit criteria, or any earlier RFC or ADR.
- It doesn't close any component's own "Next" items.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:** none newly defined. Phase 4 brings hardware into the trusted base for the first time,
  as secure elements and an IOMMU. Each needs its own RFC and threat-model review.
- **New assets:** none from this RFC. The device root key, attestation keys and the counter come in Phase 4's RFCs.
- **New adversary capabilities:** none from this RFC. Phase 4 takes on a **physical** adversary, one who holds the device
  and its disk, and that is what the checklist's rows defend against.
- **Net change to attacker surface:** none from this RFC.

## TCB impact  *(mandatory)*

- **Does this add code to the TCB?** No.
- **Does this add a dependency to the TCB?** Not by itself. Phase 4 will, deliberately: hardware roots of trust and
  IOMMUs become trusted dependencies, each by its own RFC.

## Privacy impact

None directly. Two stated limits carry forward: network-level linkage, and identities readable by whoever has the disk.

## Alternatives considered

- **Hold Phase 3 open until the network can't link contexts either** (an anonymity transport). Rejected. The exit
  criterion is met as RFC-0035 scoped it, with the remaining linkage stated. A transport is substantial work that doesn't
  block anything in Phase 4.
- **Hold Phase 3 open until a larger model runs.** Rejected. That's a hardware and performance question, and it belongs
  with the board. The criterion is about the capability set and the audit log, both of which are met.
- **Hold Phase 3 open until formal verification has begun.** Possible: it is a Phase 3 deliverable. Rejected in favour of
  moving it, explicitly, to the head of Phase 4. Phase 4's exit criterion already depends on it, so it gets the same
  priority without blocking the hardware work.
- **Open Phase 4 without the checklist accounting.** Rejected: RFC-0023 forbids it.

## Unresolved questions

- Which board: not yet chosen. Bring-up's details, and how much of streams B and D it supports, depend on it. The
  two-tier minimum above is what to check a candidate against.
- The formal-verification tool (Kani, Verus, a proof assistant, or a seL4-style refinement) and the first properties to
  prove: stream E's RFC.
- Whether QEMU's RISC-V IOMMU can confine the virtio drivers before the board arrives. It attaches to PCI, and today's
  drivers are virtio-mmio.

## Resulting ADRs

On acceptance, ADR-0041 records the transition, the evidence, the narrow points, the carried-forward deliverables, and the
checklist accounting.
