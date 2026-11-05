---
adr: 0028
title: Hardware-backed key custody belongs in Phase 4, with a checklist so it isn't lost
status: Accepted
date: 2026-11-03
deciders: ["TSC"]
rfc: ../rfcs/0023-hardware-key-custody-in-phase-4.md
supersedes: []
superseded_by: null
---

# ADR-0028: Hardware-backed key custody belongs in Phase 4, with a checklist so it isn't lost

## Context

The Roadmap placed hardware-backed storage keys in Phase 3, `lantern-crypto/STATUS.md` placed custody in Phase 4, and
Phase 3 has no hardware to anchor a key in. [RFC-0023](../rfcs/0023-hardware-key-custody-in-phase-4.md) settled this.

## Decision

**Hardware-backed key custody is Phase 4 work, and moving it does not lower the bar.**

- **Phase 3** storage is encrypted under software-held keys, which doesn't protect against device theft or offline access.
  Nothing in Phase 3 may describe it otherwise.
- **Phase 4's exit criteria** now require: every key anchored in a hardware root of trust; device theft yielding only
  ciphertext; and revocation and audit logs that resist tampering and rollback.
- **The checklist in RFC-0023 is binding.** The RFC opening Phase 4 must carry it forward and show each item done or
  explicitly moved by another RFC. A later RFC taking on a software-only limit of this kind must add a row.

## Consequences

- **Easier:** Phase 3 can exit without real hardware, and its storage claims are honest.
- **Harder / committed to:** Phase 4 can't exit without this, and the checklist has to be maintained.
- **Trust boundary:** none changed. System threat T6 stays unmitigated through Phase 3, now stated rather than implied.
