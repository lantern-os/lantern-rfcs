---
adr: 0031
title: Interrupt delivery, a preemption timer and an idle loop
status: Accepted
date: 2026-11-11
deciders: ["TSC"]
rfc: ../rfcs/0027-interrupt-delivery-timer-and-idle.md
supersedes: []
superseded_by: null
---

# ADR-0031: Interrupt delivery, a preemption timer and an idle loop

## Context

The kernel took no interrupts. A thread ran until it blocked, a trap other than a system call parked the hart, and blocking
with nothing ready was an error. So no program could wait for a device, and one that spun stopped every other program. That
happened twice while implementing RFC-0025. [ADR-0004](./0004-kernel-responsibilities-and-tcb-boundary.md) already assigns
"interrupts delivered to user-space drivers as notifications bound to IRQ capabilities" to the kernel.
[RFC-0027](../rfcs/0027-interrupt-delivery-timer-and-idle.md) proposed filling that in, and has been accepted; this ADR is the
durable record. It is the prerequisite for [RFC-0026](../rfcs/0026-trusted-path-and-consent.md)'s console.

## Decision

**The kernel routes interrupts to confined drivers, preempts every thread on a fixed slice, and idles when nothing is ready.
It never handles a device itself.**

- **Where interrupts are taken:** only from U-mode, and in the idle loop without a trap. The kernel runs with them masked
  ([ADR-0010](./0010-kernel-concurrency-model.md) unchanged), and the trap entry's "from U-mode" assumption still holds.
- **The HAL** drives the platform controller (QEMU `virt`'s PLIC, hart 0's S-mode context) and the timer (the SBI TIME
  extension). Its registers are mapped S-mode-only in every VSpace. An interrupt's frame carries a `TrapCause` and
  `NOT_A_SYSCALL`.
- **`IrqHandler`:** a capability binding one line to a notification with a nonzero badge. Its invocations are
  `SetNotification`, `Clear` and `Ack`, as syscall 14. Delivery signals the notification and leaves the line claimed until `Ack`.
  A line with no bound handler is disabled and completed. Only boot code creates handlers, and they aren't retypable.
  `MAX_IRQ_HANDLERS = 4`.
- **Preemption:** a fixed 10 ms slice per thread, round robin, with no priorities. The timer is armed lazily: a switch records
  the slice's end, and a firing that's early re-arms without preempting. A system call is never cut short.
- **Idle:** blocking with nothing ready leaves no current thread. The trap glue disarms the timer and waits with `wfi` and
  interrupts masked, then claims what's pending and starts whatever it made ready.
- **Pure logic, thin glue:** the kernel queues controller actions, and `handle_trap`, generic over `Hal`, applies them. Both
  are host-tested.
- **Console:** in the system image RFC-0026 adds, the kernel and boot code stay off the UART once the shell runs.

## Consequences

- **Easier:** a confined driver can wait for its device, which every later device needs: storage, network, a real board's
  UART. A spinning program slows the system instead of stopping it. A hart with nothing to do waits instead of erroring.
- **Harder / committed to:**
  - Every demo now runs preemptively, so nothing may rely on running uninterrupted.
  - A woken driver waits up to one slice per ready thread.
  - A timer firing costs an SBI call, about once per slice.
  - The IPC benchmark measures 31,000 to 32,600 instructions per round trip, against 28,700 to 30,000 before. About 1,000
    is the glue; the rest is within its run-to-run spread.
- **TCB:** grows by the PLIC driver, the interrupt trap path and the SBI timer in `lantern-hal`, and by the `IrqHandler`
  object, delivery, the slice and the idle loop in `lantern-kernel`. All of it is Rust, with no new dependency, inside the
  boundary ADR-0004 already drew.
- **Still not done:** user-space timeouts (the next RFC, on the same timer), `IrqControl` for a user-space launcher,
  priorities, more than one hart, `Sstc`, the device tree for the PLIC's address, and the `x86-64` APIC.
