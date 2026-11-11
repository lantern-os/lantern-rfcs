---
rfc: 0026
title: The trusted path and consent v0 — a person decides each grant, at a console only the shell can reach
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["shell", "ai-runtime", "boot"]
domains: ["shell", "abi", "boot", "kernel", "hal", "docs"]
created: 2026-11-10
updated: 2026-11-11
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-11.** Fixed by [ADR-0032](../adr/0032-trusted-path-and-consent.md). The unresolved questions were
> settled as proposed: interrupts first ([RFC-0027](./0027-interrupt-delivery-timer-and-idle.md)), a ceiling, no login in v0,
> said plainly, and the roadmap track, now in `Roadmap.md`. Question 3 was settled in implementation: remembered decisions
> take sector 1, and the audit ring sectors 2 to 7, so each log keeps its own erase unit. Implementation changed four things:
>
> - **The shell waits on one notification, not an endpoint.** A thread blocks on one object, and the kernel has no bound
>   notifications. So the console's interrupt, the binder's `ASK`, and the app's and system's reports are badge bits of one
>   notification. The binder signals `ASK` after writing the request into the consent `Frame`, and the answer comes back as a
>   badge bit on the binder's own notification, so no answer text crosses the `Frame`. That needed no kernel change.
> - **While a request is open, the console takes only an answer.** The binder is waiting, so a command that calls it would
>   deadlock.
> - **A revoke from the console forgets "always" instead of recording a permanent revocation**, as section 5 implies: "never"
>   is the person's permanent no. Images without consent keep RFC-0022's durable revocation.
> - **The runtime has a system mode** (the same `WITH_CONSENT` bit): its `probe` runs once, it reports to the shell, and then it
>   parks on a notification instead of spinning. A fifth badge bit, `SYSTEM_FAILED`, carries the policy store's panic report.
>
> The proof is `lantern-boot-system` and `scripts/console-matrix.sh`, 14 cases, as section 8 describes, with three more:
> commands refused during a prompt, an "always" that can't override the ceiling when the app later asks for more, and
> `forget`. Mutations caught: a binder that never asks (23 failed checks) and a sanitizer that passes everything.

# RFC-0026: The trusted path and consent v0 — a person decides each grant, at a console only the shell can reach

## Summary

Today no person sees or decides anything. The launch binder grants from a compiled-in policy, and `grant-control`, a test
script, revokes. This RFC puts a person in the loop through a **trusted path**: a serial console that **only the shell can
reach**. The shell is a confined program that maps the UART and holds nothing else of note. When a grant isn't already
settled, the binder asks the shell. The shell shows the request on the console, and the person answers. The person can also
list what each app holds, revoke it, and read the audit log, at the same console.

It runs as **one interactive system image**. That's the first boot image that is a system rather than a demo, and the first
step on a proposed roadmap track toward an OS that can be installed and run in a VM or on hardware (section 9).

The kernel can't yet wait for input, since it has no interrupt delivery. That's the main open decision (unresolved question 1).

## Motivation

The Phase 3 exit criterion asks for a local AI agent with "a visible, revocable capability set and a faithful audit log". The
audit log is now faithful ([RFC-0025](./0025-audit-log-service-operations.md)). The capability set is revocable, but only
by a script, and visible to nobody. System threat T4 lists "user consent on grant" as a mitigation, and `lantern-shell`'s
SH1 lists "trusted-path presentation for consent". Neither exists.

Principles served: **security by architecture** (consent is a capability flow, not a dialog box that apps can draw), and
**AI-native** (an agent gets exactly what a person allowed, and the person can see and end it). It is in tension with
**usability**: every unsettled grant waits on a person, which is consent fatigue (SH2) if done badly.

This needs an RFC because it creates a new trust boundary (the console), changes what the policy store holds, gives the shell
the binder's control capability, and proposes a roadmap track (`GOVERNANCE.md`).

## Guide-level explanation

```
 runtime ── BIND ──▶ binder ── policy settles it? ── yes ──────────────▶ grant, as today
                       │ no
                       └── ASK(role, ops, app's text) ──▶ shell ──▶ console ──▶ person
                                                          shell ◀── console ◀── "y" / "n" / "always"
                       ◀── ALLOW / DENY ────────────────┘
                       │ ── APPEND(CONSENTED / PERSON_DENIED) ──▶ policy-store
                       │ ── GRANTING, GRANT … (RFC-0024, unchanged)

 person ──▶ console ──▶ shell: grants | revoke N | audit [N] | help
```

What the person sees, on a serial console:

```
lantern: confined-probe (publisher 927308f5…00b7) asks for
         keystore role "probe-signing": sign
         the app says: "x"
         allow? [y]es once, [a]lways, [n]o, n[e]ver >
```

**Why only the shell writes the console.** The person trusts what the console shows. If any other program could write
to it, that program could draw a fake prompt or a fake `grants` listing. So exactly one program maps the UART, and the
loader gives no other program a path to it. Everything an app supplies, such as its name and its justification, is shown
inside quotation marks with every control byte replaced, so it can't move the cursor, clear the screen, or print a line that
looks like the shell's own.

## Reference-level explanation

### 1. The shell program

A new confined program, `lantern-shell/shell`, replaces `grant-control` in the system image. It holds:

| Capability | Why |
|---|---|
| The UART, one device `Frame` | The console. No other program maps it |
| The binder's endpoint under `CONTROL_BADGE` | `grants`, `revoke` (RFC-0021), as `grant-control` has today |
| The policy store under `READER_BADGE` | `audit` (RFC-0024/0025) |
| Its own endpoint, for the binder's `ASK` | Consent |

It mints nothing, holds no service endpoint, and can't grant. It can only answer the binder, which still holds every
granter capability. **The shell can't grant authority it doesn't hold** (`ARCHITECTURE.md`): a compromised shell can say yes
to anything the binder's ceiling allows (section 4), and nothing beyond it.

The UART driver is part of the shell, as the flash driver is part of `policy-store`. That needs no new program, so the system
image stays within today's limits: the shell takes `grant-control`'s place.

### 2. Waiting for input

The kernel has no interrupt delivery and no timer, and a thread runs until it blocks. A console has to wait for a keystroke.
Two ways (unresolved question 1):

- **A. Interrupt delivery, in its own RFC first.** The HAL drives the PLIC. The kernel gains an IRQ-handler capability that
  binds one interrupt line to a `Notification` (the seL4 shape), and an idle loop for when nothing is ready. The shell
  `Wait`s on that notification. This adds kernel and HAL code to the TCB, but every later device needs it: storage, network,
  a timer for timeouts. It's also the foundation the installable-OS track needs.
- **B. Polling with `Yield`, for v0 only.** The shell reads the UART's status register and `Yield`s when nothing has arrived.
  This needs no TCB change and works today. But it keeps the hart busy whenever the shell is the only ready thread, and it gives
  no way to time out a prompt nobody answers.

This RFC proposes **A**, as a separate kernel RFC that this one depends on, and then implementing consent on it. B would ship
consent sooner, but on a mechanism that every later step would replace.

### 3. The consent protocol

The binder asks over a shared `Frame` between binder and shell, using RFC-0019's framing, since the request carries text.

| Op | From | Request | Reply |
|---|---|---|---|
| `ASK` | Binder | Publisher key, app name, interface, role name, ops, justification, revocation key | `ALLOW_ONCE`, `ALLOW_ALWAYS`, `DENY`, `DENY_ALWAYS` |

The binder asks only when the manifest's request is within the ceiling (section 4) and no remembered decision settles it.
While it waits, it serves nothing else. That's acceptable for v0, since one person answers one prompt at a time.

The binder records the answer before acting on it, as it does for grants:

- New events: `CONSENTED`, with `reason` = once or always, and a new decline reason `PERSON_DENIED`.
- `GRANTING` and the rest follow as today.
- A shell that can't be reached, or answers anything else, is `DENY`. The decline reason is `NO_CONSENT`.

### 4. What the person decides within

The binder's compiled-in `POLICY` becomes a **ceiling**: what may be asked for at all on this system. Anything outside it is
declined without a prompt, as now. Anything inside it is asked, unless a remembered decision settles it (unresolved question 2).
So a person can narrow what an app gets, and allow what's within the ceiling, but nothing an app declares can reach a
person's prompt if the system's policy forbids it.

### 5. Remembered decisions

`ALLOW_ALWAYS` and `DENY_ALWAYS` are remembered across boots, by revocation key (RFC-0022's role identity):

- `policy-store` gains a **decision log** beside the revocation log, with records of key, decision and ops.
- This ends RFC-0022's "the store can only narrow". A remembered `ALLOW` widens: it skips the prompt. It still can't exceed
  the ceiling, and a later revocation still wins.
- The store's flash layout changes: sector 0 is split between revocations and decisions, or the decision log takes the
  audit ring's first sector (unresolved question 3).
- `revoke` at the console also forgets an `ALLOW_ALWAYS` for that role. A `forget N` command clears a remembered `DENY`.

### 6. The console commands

The shell's command language is deliberately tiny. It isn't a general shell, and has no command to run programs (section 9).

| Command | Does |
|---|---|
| `grants` | Lists the binder's ledger: index, app, interface, role, ops, revoked or not, remembered or not |
| `revoke N` | Revokes ledger entry N (RFC-0021) and forgets its `ALLOW_ALWAYS` |
| `audit [N]` | Shows the last N audit records, joined as `audit-dump` does: role, badge, each use. It verifies the chain from `OLDEST` first, and says so if it doesn't verify |
| `decisions` / `forget N` | Lists and clears remembered decisions |
| `help` | Lists these |

Names come from the package's manifest via the binder's ledger. The ledger gains the app name and role name, as text,
alongside each entry.

### 7. Text from apps

The app name, role name and justification are publisher-supplied. The shell:

- Shows them inside quotation marks, after a label the shell wrote ("the app says:").
- Replaces every byte outside printable ASCII, including every control and escape byte, with `?`. No UTF-8 in v0.
- Truncates each to a fixed length, and says it did.

So an app can lie in its justification, but it can't make its words look like the shell's.

### 8. The proof

A new boot image, `lantern-boot-system`, runs the shell, the binder, `policy-store`, both services, and one runtime hosting a
guest. A new script, `scripts/console-matrix.sh`, drives QEMU's serial input with scripted keystrokes, and checks the
console's output and the audit log:

| Case | Expected |
|---|---|
| Answer `y` | The guest runs; `CONSENTED(once)` precedes `GRANTING` |
| Answer `n` | Declined with `PERSON_DENIED`; the guest sees `none` |
| Answer `a`, then reboot | No prompt the second time; the grant follows the remembered decision |
| Answer `e`, then reboot | No prompt; declined |
| `revoke 0` at the console | Revoked, durable, recorded; next boot prompts again |
| A justification full of escape bytes | Shown with `?`s; the screen isn't cleared and no line imitates the shell |
| A request outside the ceiling | No prompt; declined `NOT_IN_POLICY` |
| `audit` after a run | The run's records, with a verified chain |

Mutation tests: a binder granting without asking, a shell passing control bytes through, and a store remembering an `ALLOW`
beyond the ceiling must each fail a case. `binder-matrix.sh` keeps running on the existing demos, with `grant-control`.

### 9. Toward an installable system (a roadmap proposal)

This RFC proposes adding a track to the roadmap, alongside the phases and changing no exit criterion: **Toward an installable
system**. Each step needs its own RFC:

1. **An interactive system image**: this RFC.
2. **Interrupts, a timer and an idle loop**: the prerequisite in section 2.
3. **Launching from user space**: a confined launcher that starts programs from packages after boot, so a person can run an
   app at the console. Today only boot code can create a program.
4. **Storage**: a confined virtio-blk driver, and the encrypted store and packages on a disk image instead of in the boot image.
5. **Installing**: a bootable disk image that a VM boots directly, and an install flow that writes one.
6. **Real hardware**: Phase 4's RISC-V board work, with RFC-0023's checklist.

`x86-64` boot stays deferred. QEMU's `riscv64 virt` is the VM target until a decision says otherwise.

## Threat model impact

- **Trust boundaries affected:** a new one, the console. The person trusts every byte on it to come from the shell.
- **New assets:** the UART, held only by the shell. Remembered decisions on flash.
- **New adversary capabilities:**
  - A malicious app can put persuasive text in a prompt (SH2). It can't make it look like the shell.
  - Someone at the console can grant within the ceiling. That person is the user, by definition here. There's no login in v0
    (unresolved question 4).
  - Someone who can write the flash can plant an `ALLOW_ALWAYS`. That's the known offline-tampering limit, now able to widen;
    it goes on RFC-0023's checklist.
- **Mitigations:** exclusive UART mapping, sanitized and labelled app text, a ceiling a prompt can't exceed, every decision
  recorded before it acts, and denial by default on any failure.
- **Net change to attacker surface: larger**, by a person-facing input path and a parser for it, in exchange for the property
  the exit criterion asks for.

Cross-reference [Threat-Model.md](../../lantern-docs/wiki/Threat-Model.md) T4. `lantern-shell`'s SH1 and SH2 get their first
real mitigations.

## TCB impact

None from this RFC itself: the shell, binder and store are confined. **Option A in section 2 adds to the TCB**: PLIC handling in
`lantern-hal`, an IRQ-handler object and an idle loop in `lantern-kernel`. That is decided in its own RFC, not here.

## Privacy impact

Prompts show which app asks for what, on a screen anyone nearby can read. Remembered decisions are a durable record of what a
person allowed, on unencrypted flash until Phase 4. The `audit` command makes the log readable at the console to anyone at it.
No login means no separation between people (unresolved question 4).

## The three vision questions

1. **Does this make the kernel smaller, safer, and easier to reason about?** This RFC adds nothing to the kernel. Its
   prerequisite (option A) does, in the standard microkernel shape: one capability type, binding an interrupt to a notification,
   and no drivers in the kernel.
2. **Can an agent use this capability without being unnecessarily trusted?** Yes. An agent can ask, but never answer, and never
   reach the console. What it gets is what a person allowed, within what the system allows.
3. **Same interface for humans and agents?** Yes. A human-launched app and an agent ask the same binder, and get the same prompt.
   A future Personal Agent that answers prompts on a person's behalf would have to go through this same path, holding a
   capability a person granted it. It wouldn't get a side door.

## Alternatives considered

- **Every program gets console output, and the shell marks its own lines.** That's familiar, but marks can be imitated on a
  serial terminal. Exclusivity is simpler, and provably holds.
- **A secure attention key, like Ctrl-Alt-Del.** It's needed when several programs share a display. With one writer, it adds
  nothing yet. It becomes necessary when apps get output (future).
- **The shell grants directly, holding the granters.** That's fewer hops, but it makes the person-facing program, the one with
  the parser for untrusted input, also the one with authority to mint. The binder keeps that, and the shell only answers.
- **Drop the ceiling; the person decides everything the manifest asks.** It's simpler, and closer to "the user owns the
  machine". But one wrong "yes" to an over-broad manifest then has no backstop.
- **Polling now, interrupts later (option B).** It ships sooner, but builds consent on a mechanism the next steps replace.

## Prior art

- seL4's IRQ handler capabilities and notifications, for option A.
- Trusted path in the Orange Book (B2+), Windows' secure attention sequence, and macOS's TCC prompts, which only the system draws.
- Android's runtime permissions: allow once, allow always, deny. Also its evidence on permission fatigue.
- Terminal escape-sequence injection (CVE-2003-0063 and successors), for section 7.

## Unresolved questions

1. **How the shell waits for input.** This RFC proposes interrupt delivery, in its own kernel RFC first (option A), over
   polling with `Yield` (option B).
2. **A ceiling, or not?** This RFC proposes keeping the compiled-in policy as a ceiling that prompts can't exceed.
3. **Where remembered decisions live on flash.** Split sector 0, or take the audit ring's first sector, leaving six for the audit
   log. An engineering choice, left to implementation, with the reason recorded in the ADR.
4. **Who is "the person"?** v0 has no login: whoever is at the console decides. Identity and personas (`lantern-identity`)
   come later. This RFC proposes saying so plainly, and not adding a PIN that would be a public constant.
5. **The roadmap track (section 9).** Add it to `Roadmap.md` as proposed?

## Future possibilities

- App output through the shell, marked, and then a secure attention key.
- Scoped and time-bounded grants (`ARCHITECTURE.md`), and standard capability shapes, for prompts that say what they mean.
- A Personal Agent that drafts answers to prompts for a person to confirm.
- A graphical surface, which brings back the trusted-path problem in a harder form.

## Resulting ADRs

On acceptance, an ADR will record: the shell as sole console holder; the `ASK` protocol and its recording; the ceiling;
remembered decisions and the end of narrow-only; the console commands; the text-sanitizing rules; and the roadmap track.
