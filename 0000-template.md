---
rfc: 0000            # assigned on acceptance; keep 0000 in drafts
title: <concise, descriptive title>
status: Draft        # Draft | Proposed | FCP | Accepted | Active | Rejected | Withdrawn | Postponed | Implemented | Superseded
authors: ["<name or handle>"]
stewards: []         # domain steward(s) shepherding this RFC
domains: []          # e.g. kernel, capabilities, crypto, network, ai-runtime, filesystem, hal, sdk
created: YYYY-MM-DD
updated: YYYY-MM-DD
supersedes: []       # RFC numbers this replaces
superseded_by: null
tracking_issue: null
---

# RFC-0000: <title>

## Summary

One paragraph. What is being proposed, in plain language.

## Motivation

What problem does this solve? Why now? What goes wrong if we do nothing? Tie this back to
the [Core Principles](../../lantern-docs/wiki/Principles.md) — which principle does this
serve, and is there any principle it is in tension with?

## Guide-level explanation

Explain the proposal as if teaching it to a contributor. Use examples. Describe new
concepts, syscalls, capabilities, or APIs from the user's/developer's point of view.

## Reference-level explanation

The precise design. Data structures, interfaces, state machines, wire formats, error
behaviour, and edge cases. Enough detail that two independent implementers would build
compatible things.

## Threat model impact  *(mandatory)*

- **Trust boundaries affected:**
- **New assets introduced and who can reach them:**
- **New adversary capabilities, if any:**
- **Mitigations:**
- **Net change to attacker surface:** (reduces / neutral / increases — justify)

Cross-reference [`Threat-Model.md`](../../lantern-docs/wiki/Threat-Model.md).

## TCB impact  *(mandatory)*

- **Does this add code to the Trusted Computing Base?** (yes/no — if yes, how much, in
  what language, and why it cannot live outside the TCB)
- **Does this add a dependency to the TCB?**
- **Effect on TCB size and auditability:**

## Privacy impact

Telemetry, metadata, linkability, fingerprinting. Default-off? User-visible? Auditable?

## Alternatives considered

What else was evaluated and why it was not chosen. "Do nothing" is always an alternative.

## Prior art

Relevant systems and research: seL4, CHERI, Fuchsia/Zircon, Barrelfish, Tock/Hubris,
Redox, Tor/I2P/Nym, libp2p/IPFS, DID/VC, etc. What did they get right or wrong?

## Unresolved questions

Known open issues to be resolved before/within implementation.

## Future possibilities

Natural follow-on work this enables. Out of scope for this RFC but worth recording.

## Resulting ADRs

On acceptance, list the ADR(s) this RFC produces.
