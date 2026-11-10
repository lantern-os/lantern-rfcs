---
rfc: 0023
title: Hardware-backed key custody belongs in Phase 4, with a checklist so it isn't lost
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["governance", "crypto", "boot", "filesystem"]
domains: ["governance", "crypto", "boot", "filesystem", "hal", "shell"]
created: 2026-11-03
updated: 2026-11-03
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-03.** Fixed by [ADR-0028](../adr/0028-hardware-key-custody-in-phase-4.md).

# RFC-0023: Hardware-backed key custody belongs in Phase 4, with a checklist so it isn't lost

## Summary

The Roadmap lists "encrypted-by-default storage tied to hardware-backed keys" under Phase 3, while
`lantern-crypto/STATUS.md` calls hardware custody a Phase 4 concern, and Phase 4 is where the Roadmap puts real
hardware and a secure enclave. This RFC settles it: **hardware-backed key custody is Phase 4 work.** Phase 3's storage
uses software-held keys, and says what those do and don't protect.

Moving it must not lower the bar. Protection against device theft becomes an explicit Phase 4 exit criterion, and the
RFC that opens Phase 4 must account for every item in the checklist below.

## Motivation

Phase 3 can't deliver this. It needs hardware the project doesn't run on yet, and QEMU's `virt` machine offers nothing
to anchor a key in. Leaving it in Phase 3 means one of two bad outcomes: Phase 3 can't exit, or it exits with "encrypted"
meaning something weaker than it appears to.

The Roadmap also says "we would rather move a date than lower a bar". So the risk in moving this is that it gets forgotten,
with temporary software-only limits quietly becoming permanent. Phase boundaries need an RFC (`lantern-rfcs/README.md`).

## What changes

**Phase 3:** storage is encrypted under software-held keys. This protects data from other confined programs and proves
the path, but **it does not protect against theft of the device or offline access**, because the key is available to
anyone holding the image. Nothing in Phase 3 may describe it otherwise.

**Phase 4 gains explicit exit criteria:** every key is anchored in a hardware root of trust; device theft yields only
ciphertext; and the revocation and audit logs resist tampering and rollback.

## The Phase 4 checklist

Everything known to be waiting on this. The RFC opening Phase 4 must carry this table forward and show each item
done, or explicitly moved by another RFC. Any later RFC that takes on a software-only limit like these must add a row.

| Item | Today | Needs | Source |
|---|---|---|---|
| Key seeds | Public demo constants | A per-device root key | `lantern-crypto/STATUS.md` |
| Randomness | Caller-supplied bytes | A hardware-seeded CSPRNG | ADR-0011 |
| Key custody | In the confined keystore's memory | Keys that resist even a compromised service | `lantern-crypto` threat X2 |
| Encrypted storage | Software-held key, once persistent | A key bound to the device | System threat T6 |
| Policy store | Unkeyed log; erasing restores revocations | A keyed log and a monotonic counter | ADR-0027 |
| Audit log | Tamper-evident only against partial edits (RFC-0024 proposal) | Hardware-held signing and a monotonic counter | RFC-0024 |
| Audit retention | Retired history is erased on the device; sector headers and `RETIRED` records are unsigned, so a forged retirement verifies | Signed headers and retirements, and retired history exported to device-bound encrypted storage before it's erased | RFC-0025 |
| Trusted publishers | Demo keys compiled in | A root of trust for what the system trusts | RFC-0020 |
| Boot | No measured boot | Measured boot and attestation | System threat T7 |
| Driver isolation | Flash only, because there's no IOMMU | IOMMU-confined drivers, so DMA devices can be confined | ADR-0027 |
| Minimum hardware | Open question | A decision on what's required | `lantern-boot/STATUS.md` |

## Threat model impact

System threat T6, device theft, stays unmitigated through Phase 3. That's not new, since no persistent key exists today, but
it is now stated. T7 is unchanged.

## TCB impact

None.

## Privacy impact

None directly. Until Phase 4, a stolen device's stored data must be assumed readable.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** The kernel is unaffected.
2. **Can an agent use this capability without being unnecessarily trusted?** Unaffected.
3. **Does this support humans and agents through the same interface?** Unaffected.

## Alternatives considered

- **Keep it in Phase 3 and extend Phase 3 until real hardware is ready.** This would hold identity, networking and AI behind
  a hardware bring-up that Phase 4 is already scoped around.
- **Encrypt under a TPM emulated by QEMU.** This could exercise an interface, but it proves nothing about real custody, and
  `virt` would need a device model and a driver with no real-hardware counterpart yet.
- **Move it without new exit criteria.** Rejected: that lowers the bar, which the Roadmap says we won't do.

## Unresolved questions

None. What hardware is the minimum remains open for Phase 4 to decide, and is on the checklist.

## Resulting ADRs

On acceptance, an ADR will record the move, Phase 3's stated limit, the Phase 4 exit criteria, and the checklist rule.
