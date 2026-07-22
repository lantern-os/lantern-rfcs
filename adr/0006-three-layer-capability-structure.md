---
adr: 0006
title: Three-layer capability structure — kernel / service / sealed
status: Accepted
date: 2026-07-22
deciders: ["TSC"]
rfc: ../rfcs/0003-capability-model.md
supersedes: []
superseded_by: null
---

# ADR-0006: Three-layer capability structure — kernel / service / sealed

## Context

Having fixed object capabilities as the universal authority model
([ADR-0005](./0005-object-capabilities-as-universal-authority-model.md)), RFC-0003 also
specified how capabilities are structured across the stack, from the kernel up to
cross-machine delegation. This ADR fixes that layering.

## Decision

**LanternOS capabilities are structured in three cooperating layers:**

1. **Kernel capabilities** (`lantern-kernel`) — seL4-style capabilities to CNodes,
   endpoints, notifications, untyped memory, page tables, and IRQs, stored in a
   per-process CSpace. The kernel checks rights on every syscall. This layer is in the
   TCB.
2. **Runtime/service capabilities** (`lantern-capabilities`, `lantern-runtime`) —
   higher-level handles (a "file capability", a "socket capability") brokered by
   user-space services and implemented on top of kernel endpoint capabilities. This layer
   runs in confined user space and is **not** in the TCB.
3. **Sealed/cryptographic capabilities** — signed, attenuable tokens (macaroon-style) used
   when delegation must cross machines or persist, enabling revocation and attenuation in
   the decentralised setting.

Endpoints may be **badged** so a service can distinguish callers without trusting
self-asserted identity, enabling safe multiplexing of one service among mutually
distrusting clients.

## Consequences

- **Easier:** each layer can evolve independently — the kernel's capability mechanism can
  stay minimal and verifiable while service-layer brokering iterates freely in user space;
  distributed/offline delegation (sealed caps) is handled without forcing every in-machine
  call through cryptographic overhead.
- **Harder:** every service that brokers capabilities must implement its own minting,
  granting, and revocation correctly on top of the kernel primitives — there is no single
  shared implementation across all three layers; mapping between sealed and live kernel
  capabilities at trust boundaries needs careful design.
- **Committed to:** only the kernel layer is in the TCB; `lantern-capabilities` and
  `lantern-runtime` implement the service layer without expanding TCB scope; sealed
  capabilities are the only sanctioned mechanism for cross-machine/persistent delegation.
- **Still open:** the precise mapping between sealed and live kernel capabilities at trust
  boundaries remains an unresolved question from RFC-0003, tracked in
  `lantern-capabilities/STATUS.md`.
