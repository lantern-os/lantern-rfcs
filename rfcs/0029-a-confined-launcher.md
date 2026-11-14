---
rfc: 0029
title: A confined launcher — starting apps from signed packages after boot, at a person's request
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["boot", "kernel", "shell", "ai-runtime"]
domains: ["kernel", "boot", "shell", "abi", "runtime", "docs"]
created: 2026-11-13
updated: 2026-11-14
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-14, implemented and recorded in [ADR-0035](../adr/0035-a-confined-launcher.md).** Section 9 records
> the implementation's design. Section 7's "with reclamation" is what was built, on RFC-0030, so a slot runs any number of
> apps in turn. Two findings along the way:
>
> - Each service's grant table never freed a record, so it would have refused every grant after about 30 runs. Revoked
>   records are now reused.
> - The filesystem probe assumed its file was empty on every run. Files outlive a run, so it now accepts what it wrote
>   last time.
>
> **Direction agreed 2026-11-13.** Unresolved questions 1 to 3 were settled as proposed:
> **reclamation comes first**, in its own kernel RFC (RFC-0030), and this launcher is built on it; one binder serves every app
> and re-verifies each package; and the kernel's pools are sized for 4 apps at once. Question 4 defaults to
> `lantern-shell/launcher`.
>
> **Amended by [RFC-0030](./0030-process-lifecycle-faults-and-reclamation.md) while drafting it:** section 1's two kernel
> additions (the mapping template and `SetEntry`) move to RFC-0030, which builds and reclaims processes in its own proof.
> RFC-0030 also adds fault containment: today a fault in any program parks the whole hart. And **a service's shared Frames
> must never come from an app's memory**, since reclaiming an app unmaps them. The launcher allocates each slot's channel
> Frames from its own memory, maps them into the services for good, and clears them between apps. Services also need more
> than one client channel each, which this RFC's implementation must design.

# RFC-0029: A confined launcher — starting apps from signed packages after boot, at a person's request

## Summary

Today only boot code can create a program, so `lantern-boot-system` runs one fixed app, chosen when the image was built.
This RFC is step 3 of the roadmap's "Toward an installable system" track. It adds a **launcher**: a confined program that
builds a new process from a signed package, entirely through system calls, when a person asks for it at the console.

```
lantern> apps
  confined-probe        signs a fixed message
  confined-fs-probe     writes and reads back a file
lantern> run confined-fs-probe
lantern: "confined-fs-probe" asks for filesystem role "scratch": read, write ... allow? > y
lantern: the app finished.
```

It needs three small kernel additions, and one larger decision: **whether a finished app's memory and kernel objects can
be reclaimed** (unresolved question 1).

## Motivation

The track's goal is an OS a person can install and run. Running only the app the image was built with isn't that: a person
must be able to choose. Each later step also assumes a launcher. Storage (step 4) loads packages from a disk instead of the
boot image, and installing (step 5) means installing apps.

It also moves authority out of boot code. Today the loader builds every program with direct, unchecked access to kernel
state. A launcher that can only use system calls is checked by the kernel at every step, the way every other program is.

Principles served: **security by architecture** (process creation becomes a capability-checked operation), and the
installable-system goal. In tension: **TCB size**, since the kernel gains three small operations, and **the launcher's own
power**, since it holds memory to build processes from (section 4).

## Guide-level explanation

```
 person ── "run confined-fs-probe" ──▶ shell ── LAUNCH(name) ──▶ launcher
                                                                   │ verifies the package
                                                                   │ builds a VSpace, CNode, TCB from its own memory
                                                                   │ copies the runtime's ELF into fresh Frames
                                                                   │ maps the package read-only
                                                                   │ tells the binder: "badge B runs package P"
                                                                   └─ starts the runtime
 runtime ── BIND ──▶ binder (under badge B: package P's manifest) ──▶ consent, as today
```

The runtime is the same `confined-runtime` binary for every app; the package decides what it hosts (RFC-0018). The launcher
builds a fresh runtime process for each app it starts.

## Reference-level explanation

### 1. What the kernel gains

Each addition closes one thing the boot loader does today by writing kernel state directly.

- **A new VSpace carries the kernel's own mappings.** The kernel image, the UART and the PLIC are mapped S-mode-only in
  every address space (`launch::map_kernel_shared`). Boot code writes those entries itself today, so a VSpace made by
  `UntypedRetype` would lack them, and its first trap would fault. Instead, boot registers the kernel-shared root-table
  entries with the kernel once, and the kernel copies them into every VSpace it retypes. The kernel learns no layout: it
  copies what it was given.
- **A thread's initial registers.** A new `TCB` invocation, `SetEntry(pc, sp, arg0)`, valid only while the thread is
  `Inactive`. `TCBConfigure` then admits it, as now.
- **Larger pools**, sized for the system image plus a stated number of apps (section 6).

No new object type. Building a process otherwise uses what exists: `UntypedRetype`, `FrameInvoke::Map`/`Unmap`, `CNodeInvoke`
(`CopyCross`, `Mint`), and `TCBConfigure`.

### 2. The launcher

A new confined program, `lantern-shell/launcher` (or `lantern-boot`'s; see the open questions), holding:

| Capability | Why |
|---|---|
| An `Untyped` for **app memory only**, sized by the loader | What every app's objects and pages are made from. Boot creates it as a separate region, so the launcher's authority over memory is bounded without a kernel change |
| Its own VSpace and CNode | To map a fresh Frame into itself, copy an ELF segment in, and zero the rest, then unmap it and map it into the app. Every Frame an app gets is cleared by the launcher first |
| The runtime's ELF, and the package set, read-only | What it builds from (section 5) |
| The binder's endpoint, with `Mint` rights, and a registration badge | To give each runtime a distinct badge, and tell the binder which package that badge runs |
| Badged copies of the service endpoints the runtime needs, and of the shell's notification | What each runtime is granted at launch, as the boot loader grants them today |
| Its own endpoint, for the shell's `LAUNCH` | |

It verifies each package's signature before building anything. A package that doesn't verify is refused, and the shell says
so.

### 3. The binder, for many apps

Today the binder reads one package, mapped at boot. With a launcher, it serves every app:

- The launcher mints each runtime its own badge on the binder's endpoint, and before starting the runtime, sends the binder
  `REGISTER(badge, package Frame)`. That message comes under a registration badge only the launcher holds.
- The binder maps the package itself, **verifies it again**, and keeps `badge → (publisher, plan)`. It doesn't trust the
  launcher's verification: the binder holds mint authority, and parses only input it has verified itself (RFC-0020).
- A `BIND` is decided against the package its badge was registered with. An unregistered badge is declined.

The ledger, consent, revocation and audit work as now, per app. `grants` lists every app's grants.

### 4. Who decides to launch, and with what authority

- Only the shell can ask the launcher, so **a person starts every app**. An app can't start another app: it holds no launch
  capability. Agents launching agents is future work, and will need its own consent.
- The launcher can build any process from its memory. It can't grant authority it doesn't hold: it gives each runtime only
  the endpoints and notifications listed in section 2. It's the most powerful program after the kernel, so it is small, takes
  only verified packages, and has no other role.

### 5. Where packages come from

Until storage exists (step 4), the system image carries a **package set**: signed `.lpkg` files, read-only, with a small
index. `apps` lists it, by the manifest's app name, with each app's first justification as its description, sanitized.
Storage will replace the set with a directory on disk, and nothing else changes.

### 6. Capacity

The system image uses 5 of 6 VSpaces today, and every pool is sized to it. Each app adds one VSpace, one CNode, one TCB, one
scheduling context, about 8 Frames, two shared `Frame`s and a handful of slots. This RFC proposes room for **4 apps at
once**: about `MAX_VSPACES 10`, `MAX_CNODES 11`, `MAX_TCBS 12`, `MAX_FRAMES 64`, measured and stated on implementation.

### 7. When an app finishes

This is unresolved question 1, and it decides how far "run an app" goes.

- **Without reclamation (v0):** an app that finishes parks, as `confined-runtime` already does. Its objects and memory stay
  allocated until reboot, and its grants are revoked at the service. The launcher refuses a launch it has no room for, and
  says so. A person can run each of the 4 slots' worth of apps once per boot.
- **With reclamation:** the kernel gains a capability-derivation tree and a real `Revoke`/`Delete`
  (`lantern-kernel/STATUS.md` lists both as missing). The launcher can then tear an app down completely and reuse its memory.
  That is its own kernel RFC, and a larger one: it touches every object type.

### 8. The proof

- **Host tests** for the kernel additions, and for the launcher's ELF-to-pages planning, as the boot loader's is tested.
- **`console-matrix.sh`** gains cases:
  - `apps` lists the set;
  - `run` of each app goes through consent and finishes;
  - two apps running at once, each with its own grants in `grants`;
  - a tampered package refused by the launcher, and by the binder even if the launcher were fooled;
  - an unknown name;
  - and running out of room, which is refused and says so.
- **Mutation tests:** a binder that trusts the launcher's verification, a launcher that skips clearing a Frame (an app must
  not see another's old memory), and a kernel VSpace without the kernel mappings must each fail a case.

### 9. Implementation design (2026-11-14)

Settled while building it. Two scope decisions were the person's: **both apps** (`confined-probe` and
`confined-fs-probe`, so the store joins the system image), and a **test-only hold feature** to prove several apps at
once and "no room".

**Slots.** The launcher has 4 app slots. Boot carves it one `Untyped` of 4 × 18 MiB, and the launcher carves each slot
a child `Untyped` of 18 MiB. An app is a fresh `confined-runtime` built from that slot:

- the runtime's ELF, copied into 2 MiB Frames;
- a stack, a 2 MiB heap and a 3-megapage arena;
- **a private copy of its package**;
- a VSpace, a CNode, a scheduling context and a TCB.

When it ends, the slot is `Reset` (RFC-0030).

**The package set.** The image carries a read-only set: each signed `.lpkg` in its own 2 MiB Frame, with a small index.
Boot maps the set into the binder and into the launcher, nowhere else. A Frame has at most two mappings, so an app gets
a copy in a Frame from its own slot, never the set's Frame. The runtime verifies its copy, as it does today.

**The shell and the launcher** share a 4 KiB Frame and talk through notification bits, so the shell still waits on its
one notification:

- **`apps`:** the shell writes the command and signals `LIST`. The launcher writes the names and justifications of the
  packages it verified, and signals `DONE`. The shell sanitizes the text before printing it, as it does for consent.
- **`run <name>`:** the shell writes the name and signals `RUN`. The launcher answers `STARTED(slot)` or `REFUSED(reason)`:
  unknown, unverified, no room, or out of memory.
- **When an app ends**, the launcher signals the shell's per-slot `ENDED` or `FAILED` bit.

**The binder, registered.** The binder runs `WITH_LAUNCHER`.

- It reads and verifies every package in the set itself at start, from its own mapping, and trusts nothing the launcher
  says about them.
- The launcher mints each slot's runtime its own badge on the binder's endpoint (`1 + slot`). Before starting that runtime,
  it sends `REGISTER(slot, package index)` under a registration badge only it holds.
- A `BIND` under an unregistered badge is declined. Each slot has its own bound requests, ledger entries, and grant
  capability slots.
- **`UNREGISTER(slot)`**, sent when the app ends, revokes each of that slot's live grants at its service and records
  `REVOKING`. It deletes the binder's copies of the slot's capabilities and forgets the registration. Remembered decisions
  stay: they belong to the package, not the run.

**Services with many clients.**

- Boot creates each service's per-slot channel Frames and maps them into the service for good: the keystore's at
  `FRAME_VADDR + slot × 4 KiB`, and the store's likewise. Boot gives the launcher the Frame capabilities and nothing else
  of the services'. The launcher maps a slot's channel into its app, and clears it between apps.
- `GRANT` carries the slot in `ops`' second byte. A service ties the badge it mints to that slot's channel, and serves a
  badge only on its own channel. So **an app can never name another app's channel**: the slot comes from the binder, which
  learned it from the registration, never from the app.
- The store's own channel to the keystore is a fifth, fixed keystore channel. The keystore runs a **system profile**: a
  signing key the binder grants to apps, and a sealing key granted once to its first caller, the store, as before.
- A client that arrives while a service is midway through another client's multi-chunk exchange is answered `BUSY`.
  `Channel::call` now yields and retries, a bounded number of times, instead of failing. `BUSY` is only ever sent before a
  request is processed.

**An app's end.** The runtime's `SUCCESS` and `FAILURE`, and its fault notification, all go to the launcher's
notification, under per-slot badges. The launcher then:

1. sends `UNREGISTER`;
2. tells the shell;
3. `Reset`s the slot;
4. clears its channels.

**The hold feature.** With boot feature `launcher-hold-test`, the launcher keeps a finished app resident until all 4
slots are full. Then console-matrix can see 4 apps' grants in `grants`, and a fifth `run` is refused for room. The normal
image reclaims each app as it ends.

**Capacity.** Measured on implementation, and stated in `lantern-kernel/src/limits.rs`. The system image runs with
`-m 256M`.

## Threat model impact

- **Trust boundaries affected:** process creation moves from boot code to a confined program, so it is capability-checked.
  The binder gains a registration channel.
- **New assets:** the launcher's app memory, the package set, and the registration badge.
- **New adversary capabilities:**
  - A compromised launcher can build any process from its memory, and register any badge with any package. But the binder
    re-verifies every package, so it can't make the binder trust an unsigned manifest. It can run a verified app the person
    didn't ask for, which is the launcher's main risk, and why only the shell may ask it.
  - A hostile package is verified before anything is built.
- **Mitigations:** bounded app memory, double verification, shell-only launching, Frames cleared before use.
- **Net change to attacker surface: smaller in one way** (boot code no longer builds apps with unchecked kernel access) and
  larger in another (a powerful confined program).

## TCB impact

Small: the kernel-mapping template, `SetEntry`, and larger pools. The launcher itself is outside the TCB, but it is the most
trusted program outside it. Reclamation, if chosen, is a larger TCB change in its own RFC.

## Privacy impact

`apps` shows what's installed to anyone at the console. The binder and the audit log now record which apps a person ran.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** Safer: process creation becomes
   capability-checked instead of a boot-time privilege, for three small additions.
2. **Can an agent use this capability without being unnecessarily trusted?** No agent can launch anything. The launch
   capability is the shell's alone.
3. **Same interface for humans and agents?** A person launches through the shell. An agent that one day launches an app would
   go through the same launcher, by a grant a person consented to, with no side door.

## Alternatives considered

- **The kernel loads ELFs.** Fewer moving parts, but it puts a parser for package-supplied bytes in the TCB, which ADR-0004
  rules out.
- **A binder per app.** Simpler bookkeeping, but it doubles the processes per app, and every binder holds mint authority.
- **The launcher verifies, and the binder trusts it.** It saves one verification, but it moves the binder's trust onto a
  program that parses packages.
- **Untyped for every program**, so programs build their own children. It's general, but every program would hold unbounded
  memory authority (`launch.rs`'s note on why the runtime gets Frames, not Untyped).

## Prior art

- seL4's root task and CAmkES/Microkit's static system builders, and seL4's user-level process servers.
- Fuchsia's component manager and `fuchsia.process.Launcher`.
- Genode's init and its resource-trading model, for memory bounded per child.

## Unresolved questions

1. **Reclamation now or later?** This RFC proposes a separate kernel RFC for the capability-derivation tree, `Revoke` and
   reclaiming `Delete`, **before** the launcher, so that "run an app" can run it again, and a finished app returns its memory.
   The alternative is the launcher first, without reclamation, so each app slot runs once per boot.
2. **One binder for every app, re-verifying each package?** This RFC proposes yes.
3. **How many apps at once?** This RFC proposes 4, sizing the kernel's pools to it.
4. **Where does the launcher live?** It builds processes, like `lantern-boot`'s loader, but runs as a confined program, like
   `lantern-shell`'s services. This RFC proposes `lantern-shell/launcher`, beside the binder and the shell.

## Future possibilities

- Packages from storage (step 4), and installing and removing apps (step 5).
- An app asking for another app to start, with its own consent.
- Per-app memory quotas, which a person can see and change.
- Bounded retype in the kernel, so the launcher could hand an app a capped allowance of memory.

## Resulting ADRs

On acceptance, an ADR will record: the launcher and its authority; the kernel-mapping template and `SetEntry`; the binder's
registration and re-verification; shell-only launching; the package set; capacity; and the decision on reclamation.
