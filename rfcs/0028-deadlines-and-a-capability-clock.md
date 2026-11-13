---
rfc: 0028
title: Deadlines and a capability-gated clock — user-space timeouts on RFC-0027's timer
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel", "hal", "shell"]
domains: ["kernel", "hal", "abi", "shell", "boot"]
created: 2026-11-12
updated: 2026-11-12
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-12.** Fixed by [ADR-0033](../adr/0033-deadlines-and-a-capability-clock.md). The unresolved questions
> were settled as proposed: the clock is capability-gated, only boot code creates `Deadline`s, the consent limit is 60 seconds
> (the launcher may lower it, never raise it, through the binder's `arg0`), and deadlines are one-shot. Implementation changed
> four things:
>
> - **The shell gets no `Deadline`, so no clock.** It learns of a timeout from the binder. Section 3's table gave it one.
> - **The binder confirms each request's outcome** with a new shell bit, `ASK_DONE`, or `ASK_CANCELLED` on a timeout. The
>   shell prints what was decided only then. An answer and the timeout can arrive close together, and the console mustn't
>   claim an answer counted when the binder had already given up. Before each new request, the binder drains its answer
>   notification, so a late answer or a stale timeout can't count for the next one. If it can't arm its deadline, it declines
>   without asking rather than wait without a limit.
> - **The kernel reads the clock once per trap** into its state, so `Now`, `Arm` and firing stay host-testable logic.
> - **Idle clears a pending timer explicitly.** Without that, a pending timer with no deadline left would make every `wfi`
>   return at once. A host test caught it, and fails when the fix is removed.
>
> The proof: 84 kernel tests; `lantern-boot-deadline-demo` and `scripts/deadline-demo.sh` (worst lateness about 3 ms beside a
> spinner, 0–1 CPU ticks over 3 s of idle sleeps); and `console-matrix.sh`'s two new cases. Mutations caught: deadlines that
> never fire, an idle that forgets them, a `Cancel` that doesn't disarm, and a binder that ignores its timeout.

# RFC-0028: Deadlines and a capability-gated clock — user-space timeouts on RFC-0027's timer

## Summary

A confined program can't tell the time, and can't wait for anything with a limit. So a consent prompt nobody answers holds
the launch binder forever, and nothing can sleep. This RFC adds one small kernel object, a **`Deadline`**, built on the timer
[RFC-0027](./0027-interrupt-delivery-timer-and-idle.md) already programs:

- **`Arm(at)`** signals a bound notification when the monotonic clock reaches `at`. **`Cancel`** stops it. Each deadline is
  one-shot.
- **`Now`** reads the monotonic clock. Only a holder of a `Deadline` can read it: **time is a capability**, not ambient.

Waiting with a limit then needs nothing new. Bind the deadline to the notification you already wait on, with its own badge
bit, which is the pattern the shell already uses for everything it waits for. The first user is the binder: a consent request
nobody answers within a limit is declined as `NO_CONSENT`, so a hung or hostile shell can't hold grants forever.

## Motivation

- **Consent can stall forever.** RFC-0026's binder waits on the shell's answer with no limit, and serves nothing else
  meanwhile. A person who walks away, or a shell that hangs, stops every grant. ADR-0032 records this as unmitigated.
- **Nothing can sleep or retry later.** Any driver that waits for a device "or give up", any protocol with a retry, and
  `lantern-crypto`'s sealed-token expiry (RFC-0012), once the keystore runs confined, all need a deadline or a clock.
- **No program can read the time.** RFC-0012 gave the kernel a monotonic clock, and deliberately exposed it to nobody.

Principles served: **security by architecture** (time is granted, like any other authority, and only those who need it can
measure), and the small-kernel thesis (one object, using the timer the kernel already owns). In tension: **simplicity**. An
ambient clock, like most operating systems', is less to explain. Unresolved question 1 is that choice.

## Guide-level explanation

```
 program ── Deadline::Arm(now + 30 s) ──▶ kernel        records it; the hardware timer is armed for whichever is
                                                         earliest: this, another deadline, or the time slice's end
 program ── Wait(notification) ─────────▶ blocks
   ... the answer arrives first:  Wait returns the answer's bit; the program Cancels the deadline
   ... or 30 s pass:              the kernel signals the deadline's bit; Wait returns it
```

A program that wants to sleep arms a deadline and waits. A program that wants a timeout binds a deadline to the notification
it already waits on, with a bit nothing else uses. The shell already waits on one notification for everything (RFC-0026), so
a prompt timeout is one more bit.

## Reference-level explanation

### 1. The object

```rust
pub struct Deadline {
    notification: Option<(NotificationId, u64 /* badge bits */)>,
    at: Option<u64>,   // armed for this monotonic time, in ns
}
```

| Invocation | Rights | Request | Does |
|---|---|---|---|
| `SetNotification(ntfn)` | `WRITE` on both | `mr1`: notification CPtr | Binds where expiry signals. The badge must be nonzero, as for an `IrqHandler` (RFC-0027) |
| `Arm(at)` | `WRITE` | `mr1`: absolute monotonic ns | Arms, replacing any earlier arming. An `at` already past fires on the way out of this call. Needs a binding |
| `Cancel` | `WRITE` | — | Disarms. A no-op if it isn't armed |
| `Now` | `READ` | — | Answers the monotonic time in `mr1` |

**Absolute times, not durations.** A relative timeout drifts by however long the caller was preempted between reading the
time and arming. With `Now` and `Arm(at)`, the caller says exactly when, and a retry loop re-arms against the same deadline.

**One-shot.** After it fires, a deadline is disarmed. A periodic timer is a loop around `Arm` (unresolved question 4).

**Fired is final.** A deadline that fires signals once. If the program `Cancel`s after the expiry was signalled but before it
`Wait`ed, the bit is already in its notification. Programs must treat the deadline's bit as "the time came", whatever they
did since. This is the same race any notification has, made explicit.

### 2. The kernel's timer

RFC-0027 arms the hardware timer lazily, for the current time slice's end. It now arms it for the **earliest of** the slice's
end and every armed deadline. On a timer interrupt the kernel:

1. Signals every deadline whose time has come, in order of time, and disarms each.
2. Preempts if the slice has ended, as now.
3. Re-arms for the new earliest.

In idle, the timer is armed for the earliest deadline, not disarmed, so a sleeping system still wakes for it. Finding the
earliest is a scan of `MAX_DEADLINES` (proposed 8): bounded and allocation-free, like everything in the kernel.

**Lateness:** a deadline fires no earlier than `at`, and late by at most one system call's length (the kernel is
non-preemptible, ADR-0010) plus a timer tick (100 ns on QEMU `virt`). The woken program then waits its turn like any woken
thread: up to one slice per ready thread (RFC-0027's no-priorities rule). Both bounds are stated, not hidden.

### 3. Who gets a `Deadline`

Only boot code creates them in v0, as for `IrqHandler`s: there is no way to retype one from memory, so holding memory
doesn't mean holding the clock. The loader grants one to each program that needs time (unresolved question 2). The system
image grants:

| Program | Why |
|---|---|
| Launch binder | The consent request's limit |
| Shell | To say on the console that a request timed out, and in time a prompt countdown |

No other program gets the clock yet. The keystore's sealed-token expiry would be the next.

### 4. The first user: a limit on consent

- The binder arms its deadline before it signals the shell `ASK`, bound to its answer notification with a sixth bit,
  `TIMED_OUT`. If that bit arrives before an answer, the request is declined `NO_CONSENT`, and the binder signals the shell
  with a new bit, `ASK_CANCELLED`. The shell then prints that the request timed out and drops out of prompt mode.
- An answer that arrives after the timeout is ignored: the binder isn't waiting any more. The shell knows this because of
  `ASK_CANCELLED`, so it never claims an answer counted when it didn't.
- **The limit is the binder's**, not the shell's. A shell that hangs, or never answers, can't hold grants forever. The proposed
  limit is 60 seconds, compiled into the binder (unresolved question 3).

### 5. The ABI

Syscall 15, `DeadlineInvoke`, with the four labels above. `lantern-abi` gains `sys::deadline::{set_notification, arm, cancel,
now}`, and `consent::bit::{TIMED_OUT, ASK_CANCELLED}`. `scounteren` stays clear, so U-mode can't read `time` directly: the
clock stays behind the capability.

### 6. The proof

- **Host tests** in `lantern-kernel`: arm, cancel, re-arm, an `at` in the past, ordering among several deadlines, the
  earliest-of computation with a slice, and idle arming for the earliest deadline. They use the fake controller
  `handle_trap` is already tested against.
- **A QEMU demo**, `lantern-boot-deadline-demo`: a program sleeps three times for 100 ms, checks with `Now` that each wake came
  no earlier than its deadline and within a bound after it, and reports the worst lateness. A second program runs a
  `Cancel`ed deadline that must never fire. With the `irq-idle`-style variant, the system idles between wakes and uses near
  zero CPU.
- **`console-matrix.sh`** gains a case: nobody answers, and after the limit (shortened by a test feature, as the quota was in
  RFC-0025) the console says the request timed out, the app is declined, and the audit log shows `DECLINED:NO_CONSENT`. A
  second case: an answer typed after the timeout is reported as too late, and changes nothing.
- **Mutation tests:** a kernel that never fires deadlines must fail the demo; one that disarms the timer in idle despite an
  armed deadline must fail the sleep case; and a binder that ignores `TIMED_OUT` must fail the timeout case.

### 7. Cost

- **TCB:** `lantern-kernel` gains the `Deadline` object and its invocations, the earliest-of scan, and firing in the timer path.
  That's estimated at under 200 lines. `lantern-hal` is unchanged: `set_timer` and `monotonic_time_ns` exist.
- **Time:** at most one timer interrupt per armed deadline, plus the slice's.

## Threat model impact

- **Trust boundaries affected:** time becomes something a program can hold.
- **New assets:** `Deadline` capabilities, and through `Now`, a clock with the hardware timer's resolution.
- **New adversary capabilities:**
  - A holder of `Now` can time things: its own operations, IPC round trips, and so cache or scheduling effects. That's the
    raw material of timing side channels, which is why the clock isn't ambient (unresolved question 1). Programs without a
    `Deadline` keep having no clock. They aren't timeless, since they can count their own work, but they get no reference.
  - A program can arm a deadline in the past repeatedly. Each costs it a system call, so there's no amplification.
- **Mitigated:** the consent request's indefinite stall (ADR-0032's open item). A hung or hostile shell can delay a grant by at
  most the limit.
- **Net change to attacker surface: slightly larger** (a clock for those granted one, and a new object), in exchange for
  bounded waits.

Cross-reference [Threat-Model.md](../../lantern-docs/wiki/Threat-Model.md). `lantern-kernel` gains a row for the clock as a
timing-channel source; `lantern-shell`'s SH2 notes that prompts now time out.

## TCB impact

Yes: the `Deadline` object, its invocations, and firing, in `lantern-kernel` (section 7). It can't live outside, since only the
kernel owns the timer. It reuses RFC-0027's timer path, and adds no dependency and no HAL code.

## Privacy impact

A clock lets a program correlate events in time. Gating it limits that to the programs granted one. Nothing leaves the device.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** Slightly larger. It removes an unbounded wait from
   the most security-relevant flow, consent.
2. **Can an agent use this capability without being unnecessarily trusted?** An agent gets a clock only if a deadline is
   granted to it, like any other authority.
3. **Same interface for humans and agents?** Yes. A prompt times out the same way whether a person or, one day, a Personal Agent
   would have answered it.

## Alternatives considered

- **A timeout argument on every blocking call**, as Zircon does with deadlines on waits. Every blocking syscall's ABI changes,
  and a program waiting on one notification already has a way to add a cause: a bit.
- **An ambient clock**, by setting `scounteren.TM` so every program can `rdtime`. It's simplest and conventional, but gives every
  program, including every agent, a high-resolution timer for side channels, with no way to take it back.
- **A user-space time service** owning the hardware timer and serving everyone. The kernel needs the timer for preemption, so a
  service would still need this primitive underneath. It's worth having later, to multiplex many clients' deadlines and coarsen
  their clock; it doesn't replace the primitive.
- **Durations instead of absolute times.** Easier to call, but they drift under preemption, and a retry loop compounds the drift.

## Prior art

- seL4 MCS: timeouts through scheduling contexts and a user-level timer driver, since the kernel owns the timer.
- Zircon: deadlines on waits and timer objects signalling ports.
- WASI's `wasi:clocks`, and browsers coarsening `performance.now()`, for gating and coarsening a clock against timing attacks.

## Unresolved questions

1. **Capability-gated clock, or ambient?** This RFC proposes gated: only a `Deadline`'s holder can read the time.
2. **Who creates `Deadline`s?** This RFC proposes boot code only, as for `IrqHandler`s, until a user-space launcher exists and
   needs to hand them out.
3. **The consent limit:** 60 seconds, fixed in the binder? Or should the person set it, which needs a settings story the shell
   doesn't have yet?
4. **Periodic timers:** left to a loop around `Arm` in v0?

## Future possibilities

- A user-space time service multiplexing many clients, with a coarsened clock for those who don't need precision.
- Wall-clock time, which needs persistence and an authority for the time: a separate, harder problem.
- `Deadline`s retyped from memory, once `IrqControl`-style creation exists.
- A visible countdown on consent prompts.

## Resulting ADRs

On acceptance, an ADR will record: the `Deadline` object and its invocations; absolute, one-shot deadlines; the clock gated
behind it; the timer armed for the earliest deadline or slice end, including in idle; boot-only creation in v0; and the
binder's consent limit.
