---
rfc: 0001
title: The RFC and ADR process
status: Active
authors: ["LanternOS founding stewards"]
stewards: ["TSC"]
domains: ["governance"]
created: 2026-06-09
updated: 2026-06-09
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0001: The RFC and ADR process

## Summary

LanternOS adopts a written, public, two-instrument decision process: **RFCs** for
deliberation and **ADRs** for durable decisions. Substantial changes must be designed in
the open before implementation.

## Motivation

LanternOS is a multi-year, multi-contributor systems project where decisions made early
constrain everything built later — especially decisions about the kernel, capabilities,
and cryptography. Verbal or chat-only decisions do not survive contributor turnover and
cannot be audited. A lightweight but mandatory written process serves three of our
principles directly:

- **Security by architecture:** trust-boundary changes get explicit, reviewed scrutiny.
- **Auditability:** the *why* of every major decision is recoverable forever.
- **No ambient authority (applied to governance):** no one changes the architecture by
  fiat; authority to decide is explicit and scoped to stewards.

## Guide-level explanation

When you want to make a substantial change you write an **RFC** from the template, open a
PR, and discuss it. A **domain steward** shepherds it. When discussion converges, the
steward opens a **Final Comment Period (FCP)**. If consensus holds, the RFC is
**Accepted** and produces one or more **ADRs** — short, append-only records of the actual
decision. Code may then be written against the accepted design.

Small decisions that do not warrant a full proposal can be recorded directly as an ADR.

## Reference-level explanation

- **Storage:** RFCs live in `lantern-rfcs/rfcs/NNNN-title.md`; ADRs in
  `lantern-rfcs/adr/NNNN-title.md`. Both carry YAML front-matter for tooling.
- **Numbering:** sequential, assigned at acceptance/merge. Drafts keep `0000`.
- **Lifecycle:** as defined in [`README.md`](../README.md). FCP is ≥ 1 week, ≥ 2 weeks
  for governance/security RFCs.
- **Mandatory sections:** every RFC includes a **Threat model impact** and **TCB impact**
  section. Reviewers must not waive these.
- **Mutability:** RFCs freeze on terminal status; ADRs are append-only and superseded by
  new ADRs rather than edited.
- **Authority:** domain stewards approve RFCs in their domain; cross-domain or contested
  RFCs escalate to the TSC (see [`GOVERNANCE.md`](https://github.com/lantern-os/.github/blob/main/GOVERNANCE.md)).

## Threat model impact

Process-only. Net effect is a **reduction** in attacker surface over time, because every
TCB and trust-boundary change is now gated on explicit review and a recorded threat-model
delta.

## TCB impact

None (no code).

## Privacy impact

Positive: privacy-affecting changes acquire a mandatory, reviewable record.

## Alternatives considered

- **Chat/issue-only decisions.** Rejected: not durable, not auditable, not reviewable.
- **Heavyweight standards-body process.** Rejected for Phase 0: too slow for a small team.
  The lazy-consensus + FCP model scales down now and up later.
- **ADRs only (no RFCs).** Rejected: ADRs record outcomes well but are a poor medium for
  open-ended deliberation on large designs.

## Prior art

Rust RFCs, Python PEPs, IETF RFCs, the "Documenting Architecture Decisions" ADR pattern
(Michael Nygard), and the Fuchsia RFC process.

## Unresolved questions

- Tooling to auto-generate the RFC/ADR indices from front-matter (deferred).
- Whether to mirror accepted RFCs into the wiki automatically.

## Future possibilities

A bot enforcing the mandatory sections and FCP timers; cross-linking ADRs into the
per-repo `ARCHITECTURE.md` files.

## Resulting ADRs

This RFC is self-describing process and is recorded as **Active** rather than producing a
separate ADR.
