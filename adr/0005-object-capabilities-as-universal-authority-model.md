---
adr: 0005
title: Object capabilities as the universal authority model
status: Accepted
date: 2026-07-22
deciders: ["TSC"]
rfc: ../rfcs/0003-capability-model.md
supersedes: []
superseded_by: null
---

# ADR-0005: Object capabilities as the universal authority model

## Context

[RFC-0003](../rfcs/0003-capability-model.md) proposed replacing identity-and-ACL-style
access control with object capabilities everywhere in LanternOS, to structurally eliminate
ambient authority and confused-deputy failures. The RFC has been accepted; this ADR fixes
object capabilities as the system's sole authority model.

## Decision

**Authority in LanternOS is expressed exclusively as object capabilities**: unforgeable,
transferable tokens that designate an object and the rights to act on it.

- There is no ambient authority, no global namespace, and no identity-based ACL anywhere
  in the system. Possession of a capability is authority; holding no capability to
  something means the operation on it is not even nameable.
- The core operations are fixed as `invoke`, `mint` (attenuate, never amplify), `grant`
  (delegate over IPC), `revoke` (recursive, transitive), and `seal`/`unseal` (convert
  between live and cryptographic forms).
- Required properties: unforgeability, monotone attenuation, transitive revocation, no
  identity check, and auditability of grants/revocations touching user data or agents.
- Identity/ACL models (POSIX-style) and role-based access control are rejected as the
  base model, project-wide, not just for the kernel.

## Consequences

- **Easier:** least privilege is the default rather than an aspiration; a component's
  blast radius on compromise is bounded by exactly what it was handed; no component needs
  to be trusted not to reach for authority it wasn't given.
- **Harder:** every cross-component interaction must be designed around explicit capability
  passing up front; there is no fallback "just check a global permission" escape hatch, so
  services (filesystem, network, AI runtime, shell) must design their APIs capability-first
  from the start.
- **Committed to:** no future service or subsystem may introduce an identity-checked ACL or
  ambient-permission path without superseding this ADR via a new RFC; the AI runtime's
  audit log and the shell's consent UX are built on capability grants/revocations, not on
  identity.
- **Still open:** the exact rights lattice per object type, the revocation cost model, and
  CHERI-based hardware acceleration remain unresolved questions from RFC-0003 and are
  tracked in `lantern-capabilities/STATUS.md`.
