---
adr: 0025
title: The launch binder v0 — a confined binder as the sole holder of grant-request authority
status: Accepted
date: 2026-11-02
deciders: ["TSC"]
rfc: ../rfcs/0020-launch-binder.md
supersedes: []
superseded_by: null
---

# ADR-0025: The launch binder v0 — a confined binder as the sole holder of grant-request authority

## Context

[RFC-0015](../rfcs/0015-capability-manifest-format.md) and
[RFC-0018](../rfcs/0018-confined-execution-port.md) both left the launch binder to `lantern-shell`.
Without one, services granted their object to whichever caller came first, and a signed manifest's
`ops` narrowed nothing: a manifest asking only for `decrypt` could sign.
[RFC-0020](../rfcs/0020-launch-binder.md) proposed the binder and has been accepted; this ADR is the
durable record.

## Decision

**A confined program, the launch binder, is the only holder of authority to request grants from a
service.** The runtime holds no service endpoint until the binder hands it a badged one.

- **Identity:** the launcher mints the binder's copy of each service endpoint with
  `GRANTER_BADGE = (1 << 56) - 1`, the largest badge `CNodeInvoke::Mint` can pack. The kernel
  delivers badges, so a caller can't claim it. A service honours a grant request only under this
  badge, and answers any other caller asking for a capability back with `ACCESS`.
- **`GRANT` (binder → service) and `BIND` (runtime → binder):** register-only, with payload in `mr2`
  and `mr3` in both directions, and no shared `Frame`. `lantern_abi::grant` holds both.
- **The decision:** grant only if the package verifies under a trusted publisher; the manifest
  validates and declares the slot; the policy has an entry for exactly `(publisher, app, interface,
  role)`; the role's `ops` are within the entry's approved ops; the slot hasn't been bound; and the
  service returns `OK`.
- **Mint the manifest's `ops`, not the policy's.** A request beyond what's approved is declined, not
  narrowed.
- **One undifferentiated `declined`.** Neither the runtime nor its guest learns why.
- **Each `(interface, slot)` is bound at most once per launch**, granted or not.
- **The v0 policy is a compiled-in table and is not consent.** It has the shape of a recorded consent
  decision, in the place one will sit. Nothing on this path may be described as user consent until a
  real prompt writes what the table holds.

**Capacity, decided alongside:** `MAX_FRAMES` 24, `MAX_CNODES` 6, `MAX_VSPACES` 5, `CNODE_SLOTS` 56,
sized from the largest demo plus one small service, and a second S-mode kernel megapage. The earlier
values were placeholders from the first prototype, not chosen. `FrameInvoke::Map` now returns unused
spare page tables.

**Out of scope:** consent UX, revocation, multi-role grants, one runtime hosting both interfaces, and
service-to-service grants (`store-service` still takes its AEAD grant from `keystore-service` by
first call: launcher-fixed wiring rather than an app's request).

Rejected alternatives (full reasoning in the RFC): the runtime asking the service for what the
manifest says (self-authorizing); the binder in the S-mode launcher (a parser in privileged code);
fixed grants computed at build time; services reading manifests; silently narrowing `ops`.

## Consequences

- **Easier:** a manifest's `ops` now bound what a guest can do, end to end, and the path that consent,
  revocation and audit need exists.
- **Harder / committed to:** the runtime and binder each hold a trusted-publisher list for now, and
  every binder-launched service dispatches on `GRANTER_BADGE`.
- **Trust boundary:** one created. Authority to request grants moves from "anyone holding the
  endpoint" to one program. Nothing enters the ADR-0004 TCB; the kernel gains no mechanism.
- **Concentrated risk:** a compromised binder can mint any `ops` on any object in the services it
  holds, though it can't read key material, and it parses a manifest while holding that authority.
  Only signed input reaches the parser.
- **Net attacker surface: reduces.** A compromised runtime loses its route to any service's grant path.
- **Still not met:** the Phase 3 exit criterion's "visible, revocable capability set". This builds
  the path to it, not the criterion.
