---
rfc: 0003
title: The LanternOS capability model
status: Accepted
authors: ["LanternOS founding stewards"]
stewards: ["capabilities", "kernel"]
domains: ["capabilities", "kernel", "runtime", "sdk"]
created: 2026-06-09
updated: 2026-07-22
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0003: The LanternOS capability model

## Summary

Authority in LanternOS is expressed exclusively as **object capabilities**: unforgeable,
transferable tokens that designate an object *and* the rights to act on it. There is no
ambient authority, no global namespace, and no access-control list checked against an
identity. If you hold the capability, you can act; if you do not, the operation is not
even nameable.

## Motivation

Most security failures are failures of *authority*: ambient permissions (a process can
open any file), confused deputies (a privileged service is tricked into using its
authority on behalf of an attacker), and identity-based ACLs that drift out of sync with
intent. Object capabilities eliminate these classes structurally:

- **No ambient authority** → no "the process already had permission".
- **Designation = authority** → no confused deputy: you can only act on what you were
  explicitly handed.
- **Delegation is explicit and attenuable** → least privilege is the default, not an
  aspiration.

This RFC operationalises the **security by architecture** and **least privilege**
principles across the whole stack.

## Guide-level explanation

A capability is like a physical key that also encodes which doors it opens and what it may
do behind them (read-only, read-write, etc.). You can:

- **Use** it to invoke an object.
- **Delegate** a copy to another component (optionally over IPC).
- **Attenuate** it — hand over a strictly weaker key (fewer rights, narrower object).
- **Revoke** capabilities you granted, recursively.

You can never **amplify** a capability (turn a read key into a write key) or **forge**
one. A program with no capabilities can do nothing but compute and, when given an
endpoint, talk.

Example: a text editor agent is granted a write capability to *one* document object and a
read capability to a font directory. It physically cannot touch the network, other files,
or the user's keys, because it holds no capability to them — regardless of bugs in the
editor.

## Reference-level explanation

### Layers

LanternOS has capabilities at three cooperating layers:

1. **Kernel capabilities** (in [`lantern-kernel`](https://github.com/lantern-os/lantern-kernel)) — seL4-style:
   capabilities to CNodes, endpoints, notifications, untyped memory, page tables, IRQs.
   Stored in a per-process **CSpace**; the kernel checks rights on every syscall.
2. **Runtime/service capabilities** (in [`lantern-capabilities`](https://github.com/lantern-os/lantern-capabilities)
   and [`lantern-runtime`](https://github.com/lantern-os/lantern-runtime)) — higher-level handles brokered by
   user-space services (a "file capability", a "socket capability"), implemented *on top
   of* endpoint capabilities to those services.
3. **Sealed/cryptographic capabilities** — for delegation that must cross machines or
   persist: capabilities expressed as signed, attenuable tokens (macaroon-style), so that
   delegation and revocation work in the decentralised setting.

### Core operations

| Operation | Semantics |
| --- | --- |
| `invoke(cap, method, args)` | Perform a method if `rights ⊇ required`. |
| `mint(cap, subset_rights, badge)` | Create an attenuated copy; never adds rights. |
| `grant(cap, endpoint)` | Send a capability to another component via IPC. |
| `revoke(cap)` | Recursively invalidate the cap and everything minted from it. |
| `seal(cap)/unseal(token)` | Convert between live and cryptographic forms. |

### Properties we require

- **Unforgeability** — enforced by the kernel for live caps; by signatures for sealed caps.
- **Monotone attenuation** — rights only ever decrease across `mint`.
- **Transitive revocation** — revoking a parent revokes its derived children.
- **No identity check** — possession is authority; we do not consult a global ACL.
- **Auditability** — grants/revocations affecting user data or agents are observable by
  the user (ties into the AI runtime audit log).

### Badging and confinement

Endpoints can be **badged** so a service can distinguish callers without trusting their
self-asserted identity — the basis for safe multiplexing of a single service among mutually
distrusting clients.

## Threat model impact

- **Trust boundaries affected:** every boundary — this is the cross-cutting authority model.
- **New assets:** capabilities themselves; their integrity is now a top-tier asset.
- **New adversary capabilities:** none; the model's purpose is to *deny* authority by
  default.
- **Mitigations:** unforgeability, attenuation, revocation, no ambient authority.
- **Net change:** large **reduction**; eliminates ambient-authority and confused-deputy
  attack classes by construction.

## TCB impact

The kernel-level capability mechanism is in the TCB and must be minimal and verifiable.
Higher-layer capability brokering runs in confined user space and is **not** in the TCB.

## Privacy impact

Strongly positive: least privilege limits what any component (especially an AI agent) can
observe; sealed capabilities allow data sharing without surrendering ambient access or
linking identities.

## Alternatives considered

- **Identity + ACLs (POSIX/Unix model).** Rejected: ambient authority, confused deputy,
  and an explicit project non-goal.
- **Role-based access control.** Rejected as the base: still identity-centric and coarse.
- **Pure cryptographic capabilities only.** Rejected as the base: great for distribution,
  poor for fast in-machine mediation; we use them as the *sealed* layer instead.

## Prior art

seL4 and KeyKOS/EROS/Coyotos (object capabilities), Capsicum (capability mode on Unix),
CHERI (hardware-enforced capabilities), the object-capability model (Miller, "Robust
Composition"), macaroons (attenuable bearer tokens), Fuchsia handles.

## Unresolved questions

- Exact rights lattice per object type.
- Revocation cost model and whether to bound revocation depth.
- Mapping between sealed (cryptographic) caps and live kernel caps at trust boundaries.
- Hardware acceleration via CHERI when targeting capable RISC-V cores.

## Future possibilities

- CHERI-backed capabilities for intra-address-space safety.
- A capability-aware SDK so application developers manipulate caps ergonomically
  ([`lantern-sdk`](https://github.com/lantern-os/lantern-sdk)).
- User-facing "capability inspector" in the shell for consent and audit.

## Resulting ADRs

[ADR-0005](../adr/0005-object-capabilities-as-universal-authority-model.md) fixes object
capabilities (no ambient authority) as the universal authority model.
[ADR-0006](../adr/0006-three-layer-capability-structure.md) defines the three-layer
(kernel / service / sealed) structure.
