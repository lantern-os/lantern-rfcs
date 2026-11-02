---
rfc: 0020
title: The launch binder v0 — grants minted from a package's manifest, by a confined binder
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "runtime", "capabilities"]
domains: ["shell", "sdk", "runtime", "capabilities", "crypto", "filesystem", "abi"]
created: 2026-11-02
updated: 2026-11-02
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-02.** Fixed by [ADR-0025](../adr/0025-launch-binder.md): a confined binder as
> the sole holder of grant-request authority, the `GRANT` and `BIND` register protocols, the
> six-part decision, and a v0 policy that is explicitly a stand-in for consent. Both parts are
> implemented. Implementing it corrected this document in one place, `GRANTER_BADGE`, whose first
> draft value of `u64::MAX` doesn't survive `CNodeInvoke::Mint`'s packing. It also settled
> unresolved questions 1 and 2. See `lantern-shell/STATUS.md`.

# RFC-0020: The launch binder v0 — grants minted from a package's manifest, by a confined binder

## Summary

This RFC designs the launch binder that [RFC-0015](./0015-capability-manifest-format.md) and
[RFC-0018](./0018-confined-execution-port.md) both left to `lantern-shell`. A new confined
program, the **binder**, is the only holder of authority to request grants from a service. It
verifies a guest's signed package, reads its manifest, looks up each requested role in a
**deny-by-default policy**, and asks the owning service to mint a badge scoped to the policy's
object and the manifest's `ops`. The runtime then holds no service endpoint at all: it holds an
endpoint to the binder, and gets each grant from there. This replaces today's "first caller
gets the grant" and makes a manifest's `ops` enforceable. It adds two small register-only IPC
protocols and nothing to the kernel or the TCB.

**The v0 policy is a compiled-in table, not user consent.** It has the shape of a recorded
consent decision and takes the place where consent will sit, but no person is asked. This RFC
does not meet the Phase 3 exit criterion's "visible, revocable capability set"; it builds the
path that criterion needs.

## Motivation

Two known gaps come from the missing binder:

1. **First caller wins.** `keystore-service` and `store-service` each grant their one object to
   whoever `Call`s first. Anything holding the endpoint can take that grant, so holding the
   endpoint is the authority.
2. **`ops` aren't enforced end to end.** The grant's `ops` are fixed by the service's launch
   profile, not minted from the manifest. Measured under QEMU: a manifest asking only for
   `decrypt` runs and signs (see the implementation note in
   [ADR-0020](../adr/0020-capability-manifest-format.md)).

Together these break [Principle 1](../../lantern-docs/wiki/Principles.md) on the confined path:
explicit permissions and least privilege are declared in a signed file that nothing yet acts
on. They also block the Phase 3 exit criterion, because a capability set cannot be visible or
revocable until something decides what it is and records it.

This needs an RFC because it fixes who may mint authority, and it adds IPC formats
(`GOVERNANCE.md`).

## Guide-level explanation

At boot today:

```
runtime ──Call (first caller wins)──▶ keystore-service ──badge──▶ runtime
```

With a binder:

```
                 ┌────────────── signed .lpkg (one Frame, read-only in both) ──────────┐
                 ▼                                                                     ▼
runtime ──"keystore, slot 0"──▶ binder ──GRANT(key 0, SIGN)──▶ keystore-service        │
   ▲                              │  ▲                              │                  │
   │                              │  └────── badged endpoint ───────┘                  │
   └────── badged endpoint ───────┘                                                    │
   │                                                                                   │
   └── verifies the same package itself before deserializing ◀─────────────────────────┘
```

For `confined-probe`, published under the demo key, whose manifest declares role
`probe-signing` with `ops = ["sign"]`:

1. The binder verifies the package against its trusted publishers and parses the manifest.
2. The runtime asks the binder for `keystore` slot 0.
3. The binder finds the matching role and looks up
   `(publisher, "confined-probe", keystore, "probe-signing")` in its policy. The entry names
   key 0 and approves `sign`.
4. `sign` is within what the entry approves, so the binder asks the keystore-service to mint
   SIGN on key 0.
5. The service mints the badge and replies with it. The binder passes it on to the runtime.

If any check fails, the runtime is told the slot was declined, and the guest's `open(0)` returns
`none`. The guest cannot tell a declined role from an unbound one, as RFC-0015 requires.

## Reference-level explanation

### Program and placement

`lantern-shell/launch-binder/`: a standalone `no_std` riscv64 program on `lantern-abi`, linking
`lantern-sdk` (`manifest`) and `lantern-crypto`'s `KeyOps` and `lantern-filesystem`'s `FileOps`
types. It is `lantern-shell`'s first code. It runs as an ordinary U-mode program with no special
kernel privilege. Its power is only the capabilities the launcher hands it.

**What the launcher hands each party:**

| Program | Receives | No longer receives |
| --- | --- | --- |
| binder | The granter-badged endpoint of each service; its own binder endpoint; the package read-only; a heap | |
| runtime | A binder endpoint at `ENDPOINT_CPTR`; the same package read-only; arena; heap; proof notifications; the shared `Frame` | Any service endpoint |
| service | Its own endpoint, as today | |

The runtime and binder map the **same** `Frame`, which `MAX_FRAME_MAPPINGS == 2` allows. The
launcher drops its own capability after mapping it into both, so neither can write it and both
verify the same immutable bytes.

### Identifying the binder to a service

The launcher mints the binder's copy of each service endpoint with the reserved badge
**`GRANTER_BADGE = (1 << 56) - 1`** (`launch::mint`, which exists). `CNodeInvoke::Mint` packs a
badge above 8 bits of rights, so this is the largest representable badge. `Broker` allocates
client badges counting up from 1, so a collision is not reachable in practice. A service acts on a grant
request only when the kernel-delivered badge equals `GRANTER_BADGE`. The badge is set by the
kernel, so a caller cannot claim it.

### Register use

Both protocols below use `mr2` and `mr3` for payload in both directions. `mr0` carries routing,
and `mr1` is the kernel's when a capability slot is involved: the reply slot on a
`call_with_reply_slot`, the transfer slot on a `reply_with_cap`. Replies that transfer no
capability keep the same positions, with `mr1 = 0` (`sys::reply([0, status, 0])`).

### Protocol 1: binder → service, `GRANT`

Registers only, with no shared `Frame`: the binder holds no `Frame` to a service and never needs
one. Sent with `sys::call_with_reply_slot`.

| Register | Request | Reply |
| --- | --- | --- |
| `mr2` | Object id (`KeyId` or `FileId`, raw) | Status: RFC-0019's `OK`/`ACCESS`/`INVALID`/`FAILED` |
| `mr3` | Ops bits (`KeyOps` or `FileOps`) | 0 |
| Transferred capability | none | The minted badged endpoint, on `OK` only |

The service runs its existing checks: `request_key_access` / `request_file_access` refuse a
destroyed or missing object, and `ops` invalid for the object's purpose (`WrongPurpose`). Any
caller without `GRANTER_BADGE` gets `ACCESS`, and that request is never treated as an
RFC-0019 data request. The service's main loop becomes one `Recv` dispatching on badge. Phase 1
("the first `Recv` is the grant") goes away for binder-launched instances. The older demos that
prove other things keep it, selected by `arg0`, until they are migrated.

### Protocol 2: runtime → binder, `BIND`

Also registers only, with `sys::call_with_reply_slot`.

| Register | Request | Reply |
| --- | --- | --- |
| `mr2` | Interface: `1` keystore, `2` filesystem | `0` granted, `1` declined |
| `mr3` | Slot (`open(slot)` index) | 0 |
| Transferred capability | none | The badged endpoint, if granted |

There is deliberately one `declined` reply. The binder does not tell the runtime, and so the
guest, why: not approved, wrong publisher, service refused, and already bound are the same
answer. Each `(interface, slot)` is bound at most once per launch, so a caller can't accumulate
badges or fill the service's grant table.

### The decision

For a `BIND(interface, slot)`, the binder grants only if **all** of these hold:

1. The package verified under a key in the binder's trusted-publisher set. Otherwise it declines
   everything, for the whole launch.
2. The manifest parsed and validated, and declares `slot` for `interface`.
3. The policy has an entry for exactly `(publisher key, app name, interface, role)`.
4. The manifest role's `ops` are a subset of the entry's approved `ops`.
5. That slot hasn't already been bound.
6. The service returned `OK`.

Then it mints exactly the **manifest's** `ops`, not the entry's. A request beyond what is approved
is declined, not silently narrowed: an app that gets less than it declared should see `none`,
not a handle that fails later in surprising ways. The publisher key is part of the policy key so
that a different publisher can't claim an approved app's name.

### Policy, v0

A table compiled into the binder:

```rust
PolicyEntry { publisher: DEMO_KEY, app: "confined-probe", interface: Keystore,
              role: "probe-signing", object: 0, approved_ops: SIGN }
```

Anything absent is declined. The table is deliberately the shape of a stored consent decision,
so a later prompt can write what this reads without changing the protocols. **It is a stand-in,
and must be replaced before anything in this path is described as user consent.**

### Runtime changes

`request_grant()` becomes `BIND`, against the binder at the same `ENDPOINT_CPTR`. A declined slot
stays `None` in `GrantManifest`. v0 still binds **slot 0 only**, because `IpcKeystore` and
`IpcFilesystem` each wrap one `(endpoint, Frame)`. Later declared roles stay holes until the
multi-grant work.

The runtime keeps its own package verification. It must not `deserialize` on someone else's
word, since Wasmtime documents `deserialize` as unsound on untrusted input.

### Proof, under QEMU

`lantern-boot-runtime-demo` and `lantern-boot-runtime-fs-demo`, each with a binder: the existing
success path, plus a checked-in negative matrix (`lantern-boot/scripts/binder-matrix.sh`), closing
the gap that the manifest reader's hostile runs were done with a throwaway tool. The runtime also
checks two refusals on every successful bind:

| Case | Expected |
| --- | --- |
| Manifest asks `sign`, policy approves `sign` | `SUCCESS` |
| Manifest asks `decrypt` only (today's gap) | Declined; guest sees `none`; fails at `Verdict` |
| Manifest asks `sign` + `encrypt`, policy approves `sign` | Declined, not narrowed |
| App or role not in policy | Declined |
| Same app name, different trusted publisher | Declined |
| Filesystem role asks `read`, approved `read` + `write`, guest writes | Mints READ; service refuses the write; fails at `Verdict` |
| Runtime sends `GRANT` on its own badge (every run) | Service replies `ACCESS` |
| Same slot bound twice (every run) | Second is declined |

### Capacity

Measured on the implementation, for `runtime-fs-demo`, the largest topology. The binder's ELF is
255 KiB.

| Limit | Was | Needs | Now |
| --- | --- | --- | --- |
| `MAX_FRAMES` | 16 | 20 | 24 |
| `MAX_CNODES` (root + programs) | 4 | 5 | 6 |
| `MAX_VSPACES` | 4 | 4 | 5 |
| Root `CNODE_SLOTS` | 32 | 42 | 56 |
| `MAX_KERNEL_PAGE_TABLES` | 32 | ≤ 16 | 32 |
| Boot image | 1 megapage | 2.1 MiB | 2 megapages |

The new values leave room for one more small service, and cost about 4 KiB of kernel `.bss`.
`runtime-demo` alone would have fit the old limits, with no spare `Frame` or `CNode`.

The second kernel megapage is at `0x8040_0000`, which was already unused below
`GENERAL_MEMORY_BASE`, and is mapped S-mode-only in every VSpace, as the first is.

`MAX_KERNEL_PAGE_TABLES` needed no raise because of a fix: `FrameInvoke::Map` allocated spare
branch tables before every walk and never returned the unused ones, so every map cost a page.
This also applied to the runtime's own repeated mapping. It now returns them.

## Threat model impact

- **Trust boundaries affected:** one is created. Authority to request grants moves from "anyone
  holding the endpoint" to one program, and the runtime's reach narrows to what that program
  hands it.
- **New assets:** the granter-badged endpoints and the policy table.
- **New adversary capabilities:**
  - **A compromised binder can mint any `ops` on any object in the services it holds.** It can't
    read key material, which stays in the keystore. It is the highest-value program in this
    topology, as the shell's threat SH3 anticipates.
  - **The binder parses a manifest while holding that authority.** Input is reachable only after
    signature verification, so this exposes it to a trusted publisher's mistakes or malice, not
    to anonymous bytes. It is still a worse place for a parser bug than the runtime
    (`lantern-runtime` threat R7).
- **Mitigations:** memory-safe Rust with the same tested parser as the SDK; verification before
  parsing; deny by default; no self-narrowing or retry path; one bind per slot; a single
  undifferentiated `declined`.
- **Not mitigated by v0:** a person's intent (no consent yet), and revocation (the binder records
  nothing a user can act on). A launcher bug that hands out `GRANTER_BADGE` wrongly defeats the
  scheme, though the launcher is already trusted with every capability.
- **Service-to-service grants are out of scope:** `store-service` still takes its AEAD grant from
  `keystore-service` by first call. That wiring is launcher-fixed infrastructure rather than an
  app's request, but it remains a residual first-caller grant.
- **Net change to attacker surface: reduces.** A compromised runtime loses its route to a service's
  grant path entirely. The cost is concentrating mint authority in one small program.

Cross-reference [`Threat-Model.md`](../../lantern-docs/wiki/Threat-Model.md) (T4, T9).

## TCB impact

- **Does this add code to the TCB?** No. The binder is U-mode, outside the ADR-0004 TCB. It is
  trusted by policy for what it grants, the same posture the shell's architecture states.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** The raised capacities grow kernel `.bss` by about 4 KiB
  and add no logic. The `FrameInvoke::Map` fix is a small kernel change that removes a leak.

## Privacy impact

No telemetry. The policy table and the binder hold no user data in v0. The binder knows which app
was bound to which object, so a future stored policy becomes sensitive metadata, and should be
treated as such when it stops being compiled in. An undifferentiated `declined` avoids telling an
app which of the user's objects exist.

## The three vision questions

1. **Smaller, safer kernel?** The kernel gains no mechanism. Mint policy stays out of it.
2. **Can an agent use this without being unnecessarily trusted?** Yes. An agent is a guest with a
   manifest. It gets only what a role binds to, and a runtime hosting it can't reach a grant path.
3. **Same deterministic interface for humans and agents?** Yes. There is one binder and one policy
   lookup, with no AI path. Consent, when added, sits in one place for both.

## Alternatives considered

- **The runtime asks the service for what the manifest says.** Rejected: that makes the manifest
  self-authorizing, because the app's own signed file would decide its authority.
- **The binder inside the S-mode launcher.** Smallest change, and it avoids the fourth program, but
  it puts a TOML parser and policy into privileged code. Rejected on Principle 1 and ADR-0004.
- **Launcher computes grants at build time from a fixed table.** Removes the parser, but it can't
  read a package it wasn't built with, and leaves nothing for consent to attach to.
- **Service reads the manifest and decides.** Spreads policy across every service, and each would
  need the parser and trust store.
- **Silently narrow to the approved `ops`.** Rejected above: it yields handles that fail later.
- **The binder tells the runtime the layout, so the runtime drops its parser.** Attractive: one
  parser in the system, and about 168 KiB back. Not chosen for Part A, since the runtime's reader is
  merged, works, and doesn't depend on trusting the binder for layout. See Unresolved question 2.
- **Do nothing.** Leaves both gaps, and `ops` in a signed manifest continue to mean nothing.

## Prior art

- **Fuchsia's Component Manager**: capability routing resolved by a privileged-by-position,
  unprivileged-by-kernel component from declarations. This is the closest shape.
- **seL4's root task and CAmkES**: authority distributed once at startup by a component that holds
  it all; this RFC's launcher-to-binder split is the same idea with a request-time step.
- **Android's `PackageManager` and iOS's permission prompts**: a separate trusted party decides
  against a declaration, and the app learns only the result.

## Unresolved questions

1. ~~**Part B's capacity.**~~ Resolved at acceptance: the limits were raised, as above. They had
   been placeholders from the first prototype, not chosen values.
2. ~~**The image limit, within Part B.**~~ Resolved: a second kernel megapage. The runtime keeps
   its parser. Dropping it is still available as a later saving of about 168 KiB.
3. **The runtime's trust store.** The runtime keeps its own compiled-in list of trusted publishers
   for v0, which duplicates the binder's set. The binder is where it belongs (RFC-0015), but the
   runtime must not `deserialize` on the binder's word without a design for that.
4. **Binder identity for more than one app.** v0 has one runtime and one package per launch. A
   per-instance badge on the binder endpoint is the likely route.
5. **Where the binder's mint authority should end up**: whether parsing should move to a separate
   unprivileged process, as it would for a real shell.

## Future possibilities

- A real consent prompt writing the policy store, once a trusted-path surface exists.
- Revocation: the binder records each badge it obtained, and `Keystore::revoke_access` exists. This
  is the natural route to "visible, revocable".
- Multi-role grants and one runtime hosting both interfaces.
- Audit records of each decision, feeding `lantern-ai-runtime`'s log.
- `monotonic-clock` as a binder-decided link-scoped grant.

## Resulting ADRs

On acceptance, an ADR will record: a confined binder as the sole holder of grant-request authority,
via `GRANTER_BADGE`; the `GRANT` and `BIND` register protocols; the six-part decision, including
minting the manifest's `ops` and declining rather than narrowing; the undifferentiated `declined`;
and that the v0 policy is a stand-in, not consent.
