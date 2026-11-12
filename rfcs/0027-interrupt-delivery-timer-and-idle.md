---
rfc: 0027
title: Interrupt delivery, a preemption timer and an idle loop — the kernel half of ADR-0004's item 5
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["kernel", "hal", "boot"]
domains: ["kernel", "hal", "boot", "abi"]
created: 2026-11-10
updated: 2026-11-11
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-11.** Fixed by [ADR-0031](../adr/0031-interrupt-delivery-timer-and-idle.md). The unresolved questions
> were settled as proposed: the timer is in this RFC, at a fixed 10 ms; only boot code creates `IrqHandler`s; no priorities in
> v0; user-space timeouts are the next RFC. Implementation changed one thing and added three:
>
> - **The timer is armed lazily** (section 3). Re-arming on every switch, as written, was measured: each `set_timer` is an SBI
>   call into firmware, and an IPC round trip nearly doubled. Now a switch only records the new slice's end. The hardware stays
>   armed for the earlier deadline, and a firing that's early re-arms instead of preempting. Every thread still gets a full
>   slice.
> - An interrupt's frame carries a `TrapCause` and an impossible syscall number, so nothing that dispatches on the syscall
>   number mistakes an interrupted program's registers for a request.
> - Invocations and deliveries queue controller actions, which the trap glue applies. So the kernel's interrupt logic, and
>   the glue itself (generic over `Hal`), are tested on the host.
> - A notification bound to a line must have a nonzero badge, or a delivery with no waiter would be lost.
>
> The proof is `lantern-boot-irq-demo` and `scripts/irq-demo.sh`, as section 6 describes, and the idle case measured 0 CPU ticks
> over 3 s. The mutation "completes the line on delivery" is caught by two host tests rather than the demo, since QEMU's UART
> behaves the same either way.

# RFC-0027: Interrupt delivery, a preemption timer and an idle loop — the kernel half of ADR-0004's item 5

## Summary

The kernel takes no interrupts. A thread runs until it blocks, and a trap other than a system call parks the hart. So no
program can wait for a device, nothing can stop a program that spins, and nothing runs when every thread is blocked. This RFC
adds the three pieces [ADR-0004](../adr/0004-kernel-responsibilities-and-tcb-boundary.md) already assigns to the kernel, and
nothing more:

1. **Interrupt delivery.** The HAL drives the platform interrupt controller (the PLIC on `riscv64`). A new capability,
   **`IrqHandler`**, binds one interrupt line to a `Notification`. The kernel signals it and masks the line until the driver
   acknowledges. Drivers stay in user space.
2. **A preemption timer.** A fixed time slice per thread, so a program that spins slows the system down instead of stopping it.
3. **An idle loop.** When nothing is ready, the hart waits for an interrupt instead of failing the block.

[RFC-0026](./0026-trusted-path-and-consent.md)'s console is the first user. Storage, networking and timeouts are the next.

## Motivation

- **No program can wait for input.** RFC-0026 needs a shell that waits for a keystroke. Every later step on the proposed
  "toward an installable system" track needs a driver that waits for a device: virtio-blk, network, a real board's UART.
- **A spinning program stops everything.** This happened twice in RFC-0025's implementation. A service's panic handler spun, and
  so did a "can't happen" fallback that a corrupt log made reachable. Each time, no other program ran again, and the only
  symptom was silence. Without preemption, every bug of that class looks like a system hang.
- **A block with nothing ready is an error today** (`error.rs`'s "would strand the hart"). That forces demos into busy loops.

ADR-0004 lists "interrupt handling: interrupts delivered to user-space drivers as notifications bound to IRQ capabilities" as
one of the kernel's responsibilities. ADR-0008 reserved IRQ-handler invocations for implementation. This RFC fills in that
boundary. It doesn't move it.

Principles served: **security by architecture** (drivers are confined; the kernel only routes a signal), and the near-term
thesis that the kernel stays small. It is in tension with **TCB size**: this is the first new kernel object type since Phase 1's
VSpace and Frame, and the first new trap path.

## Guide-level explanation

```
 device ──▶ PLIC ──▶ trap (from U-mode, or the idle loop)
                       │ kernel: claim line N
                       │   bound? ── Signal(notification, badge) ── driver becomes ready
                       │   line N stays masked until the driver's Ack
                       └── return to the interrupted thread, or switch if its slice is up

 driver:  loop { Wait(notification); read and handle the device; IrqHandler::Ack }
```

A driver is an ordinary confined program. It holds a device `Frame` for its registers, an `IrqHandler` capability for its line,
and a `Notification`. It never runs in S-mode, and it can't touch another device's line or registers.

**The timer** fires every quantum. If the running thread has used its slice and another is ready, the kernel switches. It's the
same switch a blocking system call makes today, in the same place: on the way back to user space.

**Idle:** when the last ready thread blocks, the kernel waits for an interrupt with interrupts still masked. `wfi` wakes on a
pending interrupt anyway. The kernel then handles it, the same way as one taken from user space, and switches to whatever it woke.

## Reference-level explanation

### 1. What the HAL gains

- **Trap causes.** `scause`'s interrupt bit now dispatches: supervisor external (9) and supervisor timer (5) go to the kernel's
  new interrupt entry. Synchronous traps other than `ecall` still park, as now. Page faults are out of scope.
- **The PLIC**, behind a small trait so `x86-64` can supply an APIC later: `enable(line)`, `disable(line)`, `claim() -> line`,
  `complete(line)`, and a threshold, for hart 0's S-mode context. Its registers are mapped S-mode-only in every VSpace, as the
  kernel megapages and the UART are today (`launch::map_kernel_shared`). On QEMU `virt` they're 2 megapages from `0x0c00_0000`.
  A real board's address comes from the device tree, as RAM's does (`lantern-boot/src/fdt.rs`).
- **The timer:** `set_timer(deadline)` through the SBI TIME extension, which is portable to any board with SBI firmware. The
  `Sstc` extension, which QEMU has, is a later optimisation. `rdtime` reads the clock, as RFC-0012's monotonic clock does.
- **Interrupt enables.** `sie.SEIE` and `sie.STIE` are set once at boot. `sstatus.SIE` stays clear in S-mode, as now, and
  threads enter U-mode with `SPIE` set, so interrupts are taken only from U-mode. The kernel stays non-preemptible
  ([ADR-0010](../adr/0010-kernel-concurrency-model.md)), and the trap entry's "from U-mode" assumption still holds.

### 2. The `IrqHandler` object

```rust
pub struct IrqHandler {
    line: u16,
    notification: Option<(NotificationId, u64 /* badge bits */)>,
    state: IrqState, // Masked, Armed, Pending
}
```

| Invocation | Rights | Does |
|---|---|---|
| `SetNotification(ntfn)` | `WRITE` on both | Binds the line to a notification capability, whose badge bits the kernel ORs in on delivery |
| `Clear` | `WRITE` | Unbinds; the line is masked |
| `Ack` | `WRITE` | Completes the line at the PLIC, so the next interrupt can arrive |

- **Delivery:** on a claimed line, the kernel looks up its handler. If bound and armed, it signals the notification, marks the
  handler `Pending`, and doesn't complete the line, so it can't fire again until `Ack`. An unbound line is completed and masked.
  An out-of-range or spurious claim (0) is ignored.
- **Creation:** only the boot-time loader creates handlers, through a new `admin` function, as it creates device Frames, and
  only for lines it names. There is no `IrqControl` capability in v0: nothing creates programs after boot yet. The user-space
  launcher, step 3 of RFC-0026's track, will need one (unresolved question 3).
- **Limits:** `MAX_IRQ_HANDLERS = 4`. A handler is a capability like any other: badged, copyable, deletable, and revocable once
  the derivation tree exists.

### 3. Preemption

- **Quantum:** 10 ms (unresolved question 2). Every scheduling context gets the same slice. That's ADR-0009's
  `budget == period` with replenishment at the period, which is all Phase 1's model needed in order to mean something.
- **On a timer interrupt** taken from U-mode: if another thread is ready, the current one goes to the back of the queue and the
  next one runs. If none is, it carries on. The timer is re-armed for one quantum from now.
- **On a blocking system call**, the next thread gets a full quantum.
- **A woken driver** goes to the back of the ready queue, like any other woken thread. There are no priorities in v0, so its
  latency is at most one quantum per ready thread (unresolved question 4).
- The kernel itself is never preempted. A system call runs to completion with interrupts masked, as ADR-0010 requires.

### 4. Idle

When a block finds nothing ready, the kernel no longer returns an error. It enters `idle`:

```
loop {
    wfi                         // sstatus.SIE = 0: wakes on a pending, enabled interrupt without trapping
    handle_pending_interrupts() // the same path as a trap from U-mode: claim, deliver, re-arm the timer
    if has_ready() { switch to it; return to U-mode }
}
```

This needs no idle thread and no S-mode trap entry: interrupts stay masked, and `wfi` only waits. The hart is never left
running user code that can't be interrupted. Every existing "would strand the hart" error path becomes a wait.

### 5. The console, and who writes it

The kernel already maps the UART for its own messages: boot narration, the demos' trap-handler reports, and panics. In the
system image RFC-0026 adds, that would break the trusted path. So in that image, the kernel and boot code write to the UART
only before the first program runs, and in a panic. Everything else goes through the shell. The existing demos keep their
narration.

The UART is a 4 KiB device at `0x1000_0000`. Its megapage also holds the virtio devices, so a driver needs a **4 KiB device
Frame**. The loader can create those today only as megapages, and gains small device Frames.

### 6. The proof

- **Host tests** in `lantern-kernel`: handler binding and delivery, `Ack` re-arming, masked lines staying silent, quantum
  accounting, and idle switching. These are pure logic, as the existing scheduler and IPC tests are.
- **A new demo, `lantern-boot-irq-demo`.** A confined UART driver holds only the UART's 4 KiB Frame, its `IrqHandler` and a
  notification. It echoes every byte typed into QEMU's serial input, and counts them. A second program spins forever, and must
  not stop the driver. A script types into QEMU and checks the echo.
- **Idle:** with every program blocked, QEMU's host CPU use over two seconds must stay under a threshold. Today's busy loops would
  pin it at 100%.
- **Regression:** every existing demo, `binder-matrix.sh` and `dirty-ram.sh` must pass with preemption on. Preemption changes
  every interleaving, so this is the real test of whether anything relied on running uninterrupted.
- **Mutation tests:** a kernel that completes the line on delivery (not on `Ack`), and one that never re-arms the timer, must
  each fail a case.

### 7. Cost

- **TCB:** in `lantern-hal`, the PLIC driver, the interrupt trap path, and SBI `set_timer`. In `lantern-kernel`, the
  `IrqHandler` object and its invocations, delivery, quantum accounting, and idle. That's estimated at a few hundred lines of Rust
  in total, with no new dependencies.
- **Kernel memory:** one small pool of handlers.
- **Time:** a timer interrupt every 10 ms, and a switch only when something else is ready.

## Threat model impact

- **Trust boundaries affected:** user-space drivers now receive device interrupts. The kernel–driver boundary is a notification
  and a mask, with no driver code in the kernel.
- **New assets:** `IrqHandler` capabilities. Holding one lets a program learn when that line fires, and delay its next interrupt
  by not acknowledging. That's a denial of service to that device only.
- **New adversary capabilities:** an interrupt storm from a device. Mitigated by masking until `Ack`: a storm costs one delivery
  per acknowledgement.
- **Mitigated by preemption:** a confined program that spins can no longer stop every other program. That turns a class of bugs
  and a class of attacks from "system hang" into "slow".
- **Net change to attacker surface: slightly larger in the kernel** (a new object, a new trap path), and smaller in practice,
  since nothing can monopolise the hart any more.

Cross-reference [Threat-Model.md](../../lantern-docs/wiki/Threat-Model.md). `lantern-kernel/THREAT_MODEL.md` gains rows for an
interrupt storm and for a driver that never acknowledges.

## TCB impact

**Yes, this adds to the TCB:** the code in section 7, in Rust, in `lantern-hal` and `lantern-kernel`. It can't live outside:
taking an interrupt and switching threads are privileged by the ISA. It's kept to the minimum by routing, not handling. The
kernel never touches a device's own registers, only the interrupt controller's and the timer's. ADR-0004 already places this
responsibility in the kernel, so the boundary is unchanged and only filled in. No dependency is added.

## Privacy impact

None directly. An interrupt's timing can carry information, such as keystroke timing to whoever holds the UART's handler. That
is the shell, by RFC-0026's design, and no one else.

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** Larger, by the minimum ADR-0004 calls for. Safer,
   since a spinning program can't stop the system, and blocking with nothing ready stops being an error.
2. **Can an agent use this capability without being unnecessarily trusted?** Agents never hold an `IrqHandler`. Only drivers do,
   one line each.
3. **Same interface for humans and agents?** Yes. Preemption treats every program alike, and no program, agent or not, gets a
   separate path to a device.

## Alternatives considered

- **Polling with `Yield`.** It needs no TCB change, but it wastes the hart, gives no timeouts, and every later device replaces it.
  RFC-0026 considered and set it aside.
- **Drivers in the kernel**, as a monolithic kernel does. That's faster, but it's exactly what ADR-0004 rules out.
- **An idle thread in U-mode.** `wfi` is illegal in U-mode, so it would spin, using the hart it exists to save. The S-mode loop
  with interrupts masked needs no new thread and no nested trap path.
- **`Sstc` directly, not SBI.** It's faster, but it's not on every board. SBI first, with `Sstc` as a later optimisation.
- **Priorities, so a woken driver runs at once.** That's better latency, but it's a scheduling-policy decision that deserves its
  own RFC, with the MCS-style scheduling contexts ADR-0009 points at.
- **No timer in this RFC, only external interrupts.** That's smaller, but it leaves the bug class above in place, and every
  timeout waiting.

## Prior art

- seL4: `IRQControl`/`IRQHandler` capabilities, notifications, and `Ack`. The shape of section 2 is seL4's, minus `IRQControl` for now.
- The RISC-V PLIC specification (claim and complete), and the SBI TIME extension.
- Fuchsia's interrupt objects, bound to ports, as a later variant.

## Unresolved questions

1. **The timer in this RFC, or its own?** This RFC proposes here, because preemption fixes a bug class this project has hit twice,
   and the console's timeouts will want the same timer.
2. **The quantum.** This RFC proposes 10 ms, fixed, for v0.
3. **`IrqControl` now, or with the user-space launcher?** This RFC proposes later. Only boot code creates handlers until then.
4. **Should a woken driver run first?** This RFC proposes not in v0, with no priorities, and latency bounded by one quantum
   per ready thread.
5. **Timeouts for user space.** A notification signalled at a deadline would give the console its prompt timeout, and any
   program a sleep. This RFC proposes the next RFC, on the same timer.

## Future possibilities

- `IrqControl`, for the user-space launcher.
- User-space deadlines and sleeps.
- Priorities and MCS-style budgets (ADR-0009's direction).
- MSI and the AIA on newer RISC-V cores; the APIC for `x86-64`.
- Multiple harts, with per-hart PLIC contexts.

## Resulting ADRs

On acceptance, an ADR will record: interrupts taken only from U-mode and the idle loop, never in the kernel; the `IrqHandler`
object, its invocations, and masking until `Ack`; boot-only creation in v0; the SBI timer and the fixed quantum; the masked-`wfi`
idle loop; and the kernel's silence on the console in the system image.
