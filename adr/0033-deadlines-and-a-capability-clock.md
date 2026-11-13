---
adr: 0033
title: Deadlines and a capability-gated clock
status: Accepted
date: 2026-11-12
deciders: ["TSC"]
rfc: ../rfcs/0028-deadlines-and-a-capability-clock.md
supersedes: []
superseded_by: null
---

# ADR-0033: Deadlines and a capability-gated clock

## Context

No confined program could read the time or wait with a limit. A consent request nobody answered held the launch binder, and
so every grant, forever, which [ADR-0032](./0032-trusted-path-and-consent.md) recorded as unmitigated.
[ADR-0016](./0016-monotonic-clock-primitive.md) gave the kernel a monotonic clock and deliberately exposed it to nobody.
[RFC-0028](../rfcs/0028-deadlines-and-a-capability-clock.md) proposed deadlines on [ADR-0031](./0031-interrupt-delivery-timer-and-idle.md)'s
timer, and has been accepted; this ADR is the durable record.

## Decision

**A `Deadline` signals a bound notification once at an absolute monotonic time, and is the only way a program can read the
clock.**

- **The object:** `SetNotification` (a nonzero badge), `Arm(at)` (absolute ns, replacing any earlier arming; a time already
  past fires on the way out of that call), `Cancel`, all needing `WRITE`; and `Now`, needing `READ`. One-shot. Syscall 15,
  `DeadlineInvoke`. `MAX_DEADLINES = 8`. Not retypable: only boot code creates one.
- **Time is a capability.** `scounteren` stays clear, so U-mode can't read `time` directly. A program without a `Deadline`
  has no clock.
- **The timer** is armed lazily for the earliest of every armed deadline and the current slice's end. On a timer interrupt,
  due deadlines fire, earliest first, and the slice preempts only if it has really ended. In idle, the timer is armed for the
  earliest deadline, and a pending timer is cleared explicitly. The kernel reads the clock once per trap.
- **Fired is final.** A `Cancel` can't take back a deadline that has already signalled, so a program treats its bit as "the
  time came".
- **Consent has a limit:** 60 seconds, armed by the binder, not the shell. Nobody answering is `NO_CONSENT`. The binder
  confirms each request's outcome to the shell (`ASK_DONE` or `ASK_CANCELLED`), so the console never claims a late answer
  counted. It drains stale bits before each request, and declines without asking if it can't arm its deadline. The launcher
  can lower the limit, never raise it. The shell holds no clock.

## Consequences

- **Easier:** a program can sleep, retry later, or give up waiting, by binding a deadline to the notification it already
  waits on. A hung or hostile shell can delay grants by at most the limit.
- **Harder / committed to:**
  - Every trap scans the deadline pool. The IPC benchmark is unchanged within its spread, at 30,200–32,100 instructions per
    round trip.
  - A deadline fires late by at most a system call's length plus a tick, then waits its turn: a slice per ready thread. That
    was measured at about 3 ms beside a spinner under QEMU.
- **TCB:** grows by the `Deadline` object, its invocations and firing, in `lantern-kernel`. `lantern-hal` is unchanged.
- **Timing channels:** a holder of `Now` can time things; that's why the clock isn't ambient.
- **Still not done:** a user-space time service that would multiplex and coarsen the clock, wall-clock time, retyping
  `Deadline`s from memory, and a person-set consent limit.
