---
adr: 0013
title: IPC latency benchmark — methodology, Phase 1 target, and a known QEMU-only bug
status: Accepted
date: 2026-08-17
deciders: ["kernel", "boot"]
rfc: null
supersedes: []
superseded_by: null
---

# ADR-0013: IPC latency benchmark — methodology, Phase 1 target, and a known QEMU-only bug

## Context

[RFC-0004](../rfcs/0004-phase-0-to-phase-1-transition.md) fixes Phase 1's exit criterion
as "a confined user-space 'hello service' reachable only via a granted capability, with
IPC latency benchmarked **and within target**." The confinement half was met and recorded
in [ADR-0012](./0012-vspace-frame-capabilities-and-elf-loader.md); the benchmark was not
done. No prior RFC or ADR ever fixed a *numeric* target — [RFC-0002](../rfcs/0002-microkernel-architecture.md)'s
"Performance posture" section says only "benchmarks gate IPC changes," and
[ADR-0009](./0009-phase1-scheduling-context-model.md) refers to "a benchmarked IPC
latency" without a number. This is a decision too small and too implementation-bound to
need its own RFC (`GOVERNANCE.md`'s "what requires an RFC" list — no public interface,
TCB, or threat-model change), so it's recorded directly as an ADR per
[`adr/README.md`](./README.md)'s standalone-ADR allowance.

## Decision

**Methodology.** `lantern-boot`'s `hello-service` (already RFC-0004/ADR-0012's demo) now
repeats its `Call`/`Recv`+`Reply` round trip: one untimed warm-up (unchanged from the
original single-shot demo, still narrated), then 2000 timed ones, plus a small safety
margin (see "the round-trip-loss bug" below). `lantern-boot/src/main.rs`'s
`boot_trap_handler` reads the riscv64 `cycle`/`instret` counters (`rdcycle`/`rdinstret`,
unprivileged CSRs `0xC00`/`0xC02`, read from S-mode — no `lantern-hal` timer/perf-counter
primitive exists yet, so this is narrow, throwaway instrumentation local to the benchmark,
not a new HAL surface) at the top of the `Call` trap that starts a round trip, and again
right after dispatch finishes the matching `Reply` (the direct-switch fast path means that
single `Reply` dispatch is what resumes the client). This measures **kernel-side IPC
dispatch latency**: capability lookup, endpoint rendezvous, and the direct context switch
(including the real Sv39 address-space reactivation, `satp`/`sfence.vma`, each direction)
— it excludes the fixed hardware trap-entry/exit assembly on either end, which is small,
symmetric, and shared by every syscall alike, not specific to IPC.

**Result, and the Phase 1 target.** Confirmed under real QEMU (`qemu-system-riscv64
-machine virt -bios default`), release profile:

```
boot: IPC benchmark done (2000 timed Call+Reply round trips, direct-switch fast path):
boot:   cycles/round-trip  min=26477 avg=27035 max=129804
boot:   instret/round-trip min=26507 avg=27074 max=129923
```

Run-to-run variance under QEMU/TCG is real (a repeat run measured min=26872 avg=32081
max=179226) — expected for software emulation without `-icount`, and part of why the
target below is set well above the observed average rather than pinned to one run.
`cycles` and `instret` track each other almost exactly 1:1 in both runs, which
confirms this environment's QEMU/TCG `cycle` counter is effectively an instruction-count
proxy here, not real-hardware cycle-accurate timing — expected for software emulation
without `-icount`, and consistent with this project's prior QEMU-environment findings
(`lantern-hal/STATUS.md`'s Sv39-walk entry). This number is **not** a real-hardware
performance claim. It **is** a real, reproducible, regression-detectable baseline for this
specific dev environment (QEMU 10.2.1, `-machine virt`, `-bios default`, this host), which
is exactly what RFC-0002's "benchmarks gate IPC changes" needs at this stage.

**Phase 1 target, fixed here for the first time: kernel-side IPC dispatch latency stays
at or below ~40,000 cycles/instret per round trip (roughly 25% over today's observed
avg), measured this same way, in this same class of environment.** A regression past this
without an accompanying explanation in the PR/RFC that caused it should be treated as a
bug. This target is deliberately loose and environment-local — the max above (179,226,
one outlier — see below) already exceeds it, which is fine: the target gates the *typical*
path, not every sample. Re-baselining against real RISC-V hardware is Phase 4 work
(`Roadmap.md`); this target only has to hold Phase 1's own line, per its own exit
criterion, until then.

## The round-trip-loss bug (found while building this benchmark)

Building a benchmark that repeats the round trip (the original demo only ever did it
once) surfaced a real, reproducible bug: the *first* `Call` a thread issues immediately
after being resumed via `ipc::reply`'s direct `state.switch_to` occasionally never
completes its switch to the receiver. `lantern-kernel`'s own bookkeeping says it
switched — `block_current` returns `true`, `scheduler.current` genuinely becomes the
receiver, confirmed by temporarily replacing the normal `debug_assert!` with a hard
`assert!` that never fired — and QEMU's own `-d int` trace shows no spurious trap (every
entry is a plain `user_ecall`, `scause=8`, nothing else). Yet execution resumes the
*original caller* anyway, silently dropping that one message.

Investigated and ruled out this session (see `lantern-boot/STATUS.md` and
`lantern-kernel/STATUS.md` for the full record): the kernel's own dispatch logic (a
host-side unit test, `two_call_reply_round_trips_in_a_row_client_runs_first`, replays the
identical sequence against portable `KernelState` with no real paging, and passes); a
spurious/timer interrupt; `sfence.vma`/TLB staleness on VSpace reactivation (`fence.i`
after `activate()` made no difference); and a stack overflow from the diagnostic code
itself (the bug reproduces identically with all diagnostics removed, in the original
minimal benchmark). **Not root-caused.**

Observed **exactly once per ~2000-round-trip run, always immediately after the untimed
warm-up round trip**, never again afterward. Worked around, not fixed: both
`hello-service` and `boot_trap_handler` tolerate `BENCH_SAFETY_MARGIN` (8) extra round
trips beyond the target, and the lost round trip (if any) is simply excluded from the
reported statistics rather than hanging the benchmark. This does not retroactively
validate the measured numbers above as bug-free — it means the *one* known-bad sample
never entered the min/avg/max in the first place, by construction (it's the round trip
that's missing a `Reply`, not one with a bad timing value).

This is accepted as a **known, documented, unresolved Phase 1 gap** — in the same spirit
as the Sv39-walk QEMU limitation this project already lives with
(`lantern-hal/STATUS.md`) — rather than blocking the benchmark on fully root-causing it.
It may or may not share a root cause with that earlier bug; both are QEMU/hardware-level
mysteries in the address-space-switch path that took (or are taking) real debugging effort
disproportionate to Phase 1's stated scope.

## Consequences

- **Easier:** RFC-0004's Phase 1 exit criterion is now fully met — a Phase 1 → Phase 2
  transition RFC (RFC-0004's own "Future possibilities") can proceed without this item
  outstanding.
- **Harder:** the round-trip-loss bug is now a tracked, real liability. Any future IPC
  change needs to consider whether it interacts with the (still unknown) root cause; the
  safety-margin workaround only holds because the loss is empirically rare and one-shot,
  not because it's understood.
- **Committed to:** this ADR's numeric target (~40,000 cycles/instret, this environment)
  is the Phase 1 regression gate until superseded — a real change to the target, or to the
  methodology, needs its own ADR (append-only per `adr/README.md`), not a silent edit here.
- **Still open:** the round-trip-loss bug's actual root cause (`lantern-boot/STATUS.md`'s
  "Next" has candidate next steps: QEMU GDB-stub single-stepping the failing trap, QEMU
  monitor inspection of `RAW_FRAME` at the moment of the bad resume, differential testing
  across QEMU versions/`-cpu` flags). Re-baselining this target against real hardware
  remains Phase 4 work.
