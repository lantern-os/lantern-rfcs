---
adr: 0035
title: A confined launcher — apps a person starts, from signed packages, after boot
status: Accepted
date: 2026-11-14
deciders: ["TSC"]
rfc: ../rfcs/0029-a-confined-launcher.md
supersedes: []
superseded_by: null
---

# ADR-0035: A confined launcher — apps a person starts, from signed packages, after boot

## Context

`lantern-boot-system` ran one fixed app, chosen when the image was built, and only boot code could create a program.
[RFC-0029](../rfcs/0029-a-confined-launcher.md), step 3 of the roadmap's "Toward an installable system" track, proposed a
confined launcher that builds an app's process at a person's request. Its prerequisite,
[RFC-0030](../rfcs/0030-process-lifecycle-faults-and-reclamation.md) ([ADR-0034](./0034-process-lifecycle-faults-and-reclamation.md)),
made processes buildable, containable and reclaimable from user space. RFC-0029 has been accepted; this ADR is the durable
record.

## Decision

**A confined launcher, `lantern-shell/launcher`, builds each app from a signed package in the image's package set,
through system calls alone, when the shell asks, in one of 4 app slots, and reclaims it when it ends.**

- **Who can launch.** Only the shell holds the launcher's `COMMAND` bit, so a person starts every app: `apps` lists the
  packages the launcher verified, and `run NAME` starts one. They share a 4 KiB page and talk through notification bits,
  so the shell still waits on one notification.
- **What an app is.** A fresh `confined-runtime`, built from its slot's 18 MiB child `Untyped`:
  - the runtime's ELF, copied into 2 MiB Frames through a scratch mapping;
  - a stack, a heap and an arena;
  - a private copy of its package;
  - its slot's channel for the one interface it asks for;
  - `SetEntry` and a fault notification.
- **The binder serves every app** (`WITH_LAUNCHER`):
  - It verifies every package in the set itself, from its own read-only mapping, and trusts nothing the launcher says.
  - The launcher `REGISTER`s each slot's package under a badge only it holds. Each app binds under its slot's own badge,
    minted at boot, not by the launcher.
  - A badge with nothing registered is declined. Each slot has its own bound requests, ledger range and grant slots.
- **Services serve many apps.**
  - Boot makes each slot's channel Frames from system memory, and maps them into the keystore and store for good.
  - The launcher holds their capabilities, maps a slot's channel into its app, and clears it between apps.
  - `GRANT` carries the slot, and a service serves each badge only on its own slot's channel (`frame::ChannelMap`). An app
    can't name another app's channel, since the slot comes from the binder's registration, never from the app.
  - A client that meets another's multi-chunk exchange gets `BUSY` and retries.
- **An app's end.** The runtime's report or its fault reaches the launcher, which then:
  1. has the binder `UNREGISTER` the slot, which revokes its grants at their services and records `REVOKING` with
     `APP_ENDED`;
  2. `Reset`s the slot;
  3. clears its channels;
  4. tells the shell.

  Remembered decisions stay: they belong to the package.
- **Services forget revoked grants when they need the room.** A badge is never minted twice, and an unknown badge reads as
  revoked, so evicting a revoked record lets nothing back in. Without this, each service refused every grant after about
  30 runs.
- **Capacity.** Pools are sized for the system image plus 4 apps: 12 TCBs, CNodes, VSpaces and scheduling contexts,
  12 notifications, 80 Frames, 64 page tables, 128 slots per CNode. The image runs with `-m 256M`.
- **Test only:** boot's `launcher-hold-test` keeps ended apps resident, so a test can fill every slot.

## Consequences

- **Easier:** a person chooses what runs, as often as they like, and each run's grants end with it. Storage (step 4) only
  has to replace the package set with a directory.
- **Harder / committed to:**
  - The launcher is the most powerful program outside the TCB: it builds processes from memory it holds. It holds no
    service's VSpace, can mint no badge, and only the shell can ask it.
  - A compromised launcher could still give a runtime a different verified package than the one it registered. Every
    package it can reach is verified and in the set, and every request is still decided by the binder against its policy
    and a person.
  - Every `KernelState` is about 300 KB, so host tests keep one per function, or box it.
- **TCB:** the kernel's pools grew. Nothing else in the TCB changed.
- **Still not done:**
  - Packages from storage, and installing (steps 4 and 5).
  - An app starting another app.
  - Per-app memory quotas a person can see.
  - Stopping a running app from the console.
