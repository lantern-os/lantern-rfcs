---
rfc: 0012
title: A monotonic clock read primitive for lantern-hal
status: Draft
authors: ["TheNewAutonomy"]
stewards: ["hal", "kernel"]
domains: ["hal", "kernel", "crypto"]
created: 2026-08-23
updated: 2026-08-23
supersedes: []
superseded_by: null
tracking_issue: null
---

# RFC-0012: A monotonic clock read primitive for lantern-hal

## Summary

This RFC adds one narrow primitive to the [`Hal`](../../lantern-hal/src/lib.rs) trait —
`monotonic_time_ns() -> u64`, a read-only monotonic clock — implemented on `riscv64` via
the `time` CSR (a read-only shadow of the platform timer, directly readable from S-mode
under standard OpenSBI configuration, no trap or SBI call needed) and left `unimplemented!()`
on `x86-64` pending real TSC calibration work. It deliberately does **not** add timer
interrupts, scheduler ticks, or an IRQ/notification path — `lantern-hal/ARCHITECTURE.md`'s
"Timekeeping" abstraction table entry names all of these as one category, but this RFC scopes
only the read-only clock slice a concrete consumer needs today.

## Motivation

[RFC-0011](./0011-sealed-capability-token-format.md)/[ADR-0015](../adr/0015-sealed-capability-token-format.md)
added `Caveat::ExpiresAt` to sealed capabilities and `Keystore::unseal`'s `now: Option<u64>`
parameter, both explicitly designed around "no real clock source exists yet" — `now: None`
is treated as **unsatisfied**, not unchecked, specifically so correctness never depended on
one showing up. `lantern-crypto/STATUS.md` has carried "a real clock source for
`Caveat::ExpiresAt`" as a Next item since. Separately, `lantern-hal/STATUS.md` already flags
timer support as "now concretely motivated, not just abstractly listed": `lantern-boot` found
`wfi` doesn't behave safely without it.

This is an RFC-required change per [`GOVERNANCE.md`](../../GOVERNANCE.md) — it adds new code
to `lantern-hal`, which is in the Trusted Computing Base
([ADR-0004](../adr/0004-kernel-responsibilities-and-tcb-boundary.md)) — even though the slice
itself is small. "Timekeeping" is not a new kernel responsibility invented here:
`lantern-hal/ARCHITECTURE.md` has listed it in the HAL's abstraction table since Phase 0, and
RFC-0002's kernel responsibility list implies it (responsibility 1, "Scheduling," needs a time
source; responsibility 5, "Interrupt handling," is where a future timer-interrupt half would
eventually live) — this RFC is the first real implementation of an already-scoped item, the
same relationship [RFC-0008](./0008-vspace-frame-capabilities-and-elf-loader.md) had to
"Memory isolation" (responsibility 2) when it first implemented VSpace/Frame capabilities.

This serves **security by architecture** indirectly (a real clock is what turns
`ExpiresAt`'s deny-by-default placeholder into an actually-enforceable expiry) without itself
touching any capability or trust boundary — see "Threat model impact."

## Guide-level explanation

Any code with access to a `Hal` implementation (today: privileged, same-address-space code —
`lantern-kernel`, `lantern-boot`, and, once this RFC is implemented, whatever eventually calls
`Keystore::unseal` with a real `now`) can call `monotonic_time_ns()` to get a monotonically
increasing nanosecond count since an arbitrary, unspecified epoch (never wall-clock time — no
consumer needs one yet, and wall-clock time is a separate, harder problem involving
persistence and drift that this RFC doesn't take on). Two calls a known duration apart differ
by approximately that duration; the value never decreases and never repeats.

Example: `Keystore::unseal(badge, &token, Some(lantern_hal::Riscv64::monotonic_time_ns()),
requested_rights)` replaces today's `now: None` placeholder once a caller has a `Hal`
reference to read from.

## Reference-level explanation

### The trait addition

```rust
pub trait Hal {
    // ...existing methods...

    /// Nanoseconds since an arbitrary, monotonic epoch — never wall-clock time.
    /// Never decreases across calls; wraps only after ~584 years at 1 ns
    /// resolution (`u64::MAX` ns), not a practical concern for Phase 2.
    fn monotonic_time_ns(&self) -> u64;
}
```

### `riscv64`

RISC-V's `time` CSR is a read-only shadow of the platform timer register (`mtime` in
M-mode), made readable from S-mode by the `mcounteren.TM` bit — set by default under
OpenSBI, the firmware this project's `lantern-boot` already depends on for the boot hand-off
(`lantern-hal/src/lib.rs`'s own doc comments already assume OpenSBI's default configuration
elsewhere, e.g. trap-enable state). Reading it is a single unprivileged CSR read, `rdtime`
(the standard pseudo-instruction for `csrrs rd, time, x0`) — no trap, no SBI call, no
new privilege boundary crossed:

```rust
fn monotonic_time_ns(&self) -> u64 {
    let ticks: u64;
    unsafe { core::arch::asm!("rdtime {0}", out(reg) ticks, options(nomem, nostack)) };
    ticks * NS_PER_TICK
}
```

`time` counts platform timer ticks, not nanoseconds directly — QEMU's `virt` machine clocks
its timer at 10 MHz (100 ns/tick), a fixed, empirically-confirmed constant for this project's
existing QEMU environment (the same environment `lantern-hal/STATUS.md` already documents
platform-specific constants for, e.g. the Sv39 walk limitation). A real board would need this
sourced from the device tree instead (`lantern-hal/ARCHITECTURE.md`'s own "platform
discovery" open question) — out of scope here, tracked as a "Next" item, not silently
hardcoded as if it were portable.

### `x86-64`

Left `unimplemented!()`, matching `Hal::initial_trap_frame`/`enter_thread`'s existing
precedent for `x86-64` methods with no real boot work to validate them against yet
(`lantern-hal/STATUS.md`). A real implementation needs TSC calibration (the TSC's frequency
isn't architecturally fixed and must be measured or read from CPUID/MSRs) — separable work,
not blocking `riscv64`'s slice, which is this project's strategic target
([ADR-0002](../adr/0002-riscv-target-isa.md)).

### What this does not do

- **No timer interrupts.** `wfi`'s safety issue (`lantern-boot/STATUS.md`) and any real
  scheduler tick need a programmable timer (`sstimecmp`/`stimecmp` or an SBI timer call) and
  the interrupt-controller/IRQ-capability story RFC-0002's responsibility 5 already reserves
  for user-space delivery — genuinely separate, larger work, deferred (see "Future
  possibilities").
- **No syscall exposing time to confined user-space code.** `lantern-kernel`'s syscall table
  is untouched by this RFC. Every current and near-term consumer (`Keystore::unseal`'s
  eventual caller) is privileged, same-address-space code with direct `Hal` access, the same
  category `lantern-capabilities`/`lantern-crypto`/`lantern-filesystem` are all still in
  today.

## Threat model impact

- **Trust boundaries affected:** none. This is a read-only primitive inside the TCB, callable
  only by code that already has a `Hal` reference — no new capability, no new caller category,
  no new cross-boundary data flow.
- **New assets introduced and who can reach them:** the monotonic time value itself — not
  sensitive (it reveals only "how long since boot," already inferable by anyone able to time
  their own operations).
- **New adversary capabilities, if any:** none directly. Indirectly, once a real consumer
  wires this into `Keystore::unseal`, an adversary who can influence what `now` a caller
  supplies could try to bypass `ExpiresAt` — but that's a property of the *caller's* trust in
  its own clock reading, not of this primitive, which just reads real hardware state
  faithfully.
- **Mitigations:** the primitive is read-only (no `set_time`/adjustment operation exists or is
  proposed); `riscv64`'s `time` CSR is hardware-enforced read-only from S-mode.
- **Net change to attacker surface:** neutral. No new caller can reach this that couldn't
  already read *something* time-correlated (the existing IPC latency benchmark already reads
  `cycle`/`instret`, per ADR-0013) — this adds a named, portable-contract way to do it, not a
  new capability.

## TCB impact

- **Does this add code to the Trusted Computing Base?** Yes — `lantern-hal` is in the TCB
  (ADR-0004). The addition is one trait method plus a ~5-line `riscv64` implementation (a
  single inline-asm CSR read) and an `unimplemented!()` stub on `x86-64`.
- **Does this add a dependency to the TCB?** No.
- **Effect on TCB size and auditability:** minimal — smaller than any prior HAL addition
  (trap entry, paging); no new `unsafe` pattern beyond the inline-asm-behind-a-safe-method
  discipline `lantern-hal/ARCHITECTURE.md` already mandates and every other `riscv64` HAL
  method already follows.

## Privacy impact

None. Monotonic time since an arbitrary epoch carries no user data, identity, or wall-clock
correlation.

## Alternatives considered

- **An SBI timer call (`sbi_set_timer`) instead of reading `time` directly.** Rejected for
  this RFC's scope: that's the *interrupt* half ("Future possibilities"), not needed just to
  *read* current time, and would pull in SBI call plumbing this crate doesn't have yet for no
  benefit to the one consumer motivating this RFC.
- **Wall-clock time (RTC-backed) instead of monotonic.** Rejected: `ExpiresAt`'s only real
  requirement is "later calls read a value ≥ earlier calls" (monotonicity); wall-clock time
  adds persistence-across-reboot and drift-correction problems this RFC's motivating consumer
  doesn't need, and `lantern-hal/ARCHITECTURE.md`'s "platform discovery" already flags RTC
  access as unstarted, separate work.
- **Do nothing — leave `Keystore::unseal`'s `now` permanently caller-supplied with no real
  source.** Rejected: this was always documented as a placeholder
  (`lantern-crypto/STATUS.md`), not a permanent design; leaving `ExpiresAt` permanently
  unenforceable defeats the caveat's own purpose.

## Prior art

seL4's own minimal timer driver model (a thin, portable read primitive with interrupt
handling left to user space) — this RFC takes the same "read primitive first, interrupts
later, as a genuinely separate concern" split. RISC-V's own privileged architecture
specification defines `time`/`rdtime` for exactly this purpose (a cheap, trap-free monotonic
read available to less-privileged modes).

## Unresolved questions

- The QEMU `virt` machine's 10 MHz timer frequency is hardcoded for now — device-tree-sourced
  frequency discovery is real future HAL work (`lantern-hal/ARCHITECTURE.md`'s "platform
  discovery"), not resolved here.
- Exact epoch semantics (boot time vs. some other reference) — left unspecified since no
  consumer needs one yet; `ExpiresAt`-style relative comparisons don't care, only that the
  value is monotonic.

## Future possibilities

- Timer interrupts + scheduler ticks, once `lantern-boot`'s `wfi` safety issue needs solving
  for real (`lantern-boot/STATUS.md`) — a substantially bigger RFC (programmable timer setup,
  the interrupt-controller/IRQ-capability delivery path RFC-0002 already reserves).
- A syscall exposing time to confined user-space services, once one is real enough to need it.
- Wiring `monotonic_time_ns()` into `Keystore::unseal`'s `now` parameter at whatever call site
  ends up calling it — `lantern-crypto`/`lantern-filesystem`'s own `STATUS.md`s track this as
  their own "Next," not this RFC's job to implement.

## Resulting ADRs

On acceptance, an ADR will fix `monotonic_time_ns()` as the HAL's canonical read-only clock
primitive, `riscv64`'s `time`-CSR implementation, and the explicit scoping decision (read-only
now, timer interrupts deferred as separate future work).
