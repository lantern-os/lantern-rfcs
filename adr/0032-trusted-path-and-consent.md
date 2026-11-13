---
adr: 0032
title: The trusted path and consent v0 — a person decides each grant, at a console only the shell can reach
status: Accepted
date: 2026-11-11
deciders: ["TSC"]
rfc: ../rfcs/0026-trusted-path-and-consent.md
supersedes: []
superseded_by: null
---

# ADR-0032: The trusted path and consent v0 — a person decides each grant, at a console only the shell can reach

## Context

Until now, no person saw or decided anything. The launch binder granted from a compiled-in policy, and `grant-control`, a test
script, revoked. The Phase 3 exit criterion asks for a "visible, revocable capability set", system threat T4 for "user
consent on grant", and `lantern-shell`'s SH1 for "trusted-path presentation for consent".
[RFC-0026](../rfcs/0026-trusted-path-and-consent.md) proposed a console only the shell can reach. Its prerequisite, interrupt
delivery, is [ADR-0031](./0031-interrupt-delivery-timer-and-idle.md). The RFC has been accepted; this ADR is the durable record.

## Decision

**In the interactive system image, a person decides each grant within the system's policy, at a serial console that only the
shell can read or write.**

- **The trusted path is exclusivity.** The shell is the only program with the UART's 4 KiB page and its `IrqHandler`, and the
  kernel and boot code stay off the console once it runs. Every byte a person reads there is the shell's.
- **App text is sanitized.** Names and justifications from a signed manifest pass only as printable ASCII, with quote marks
  and backslashes replaced, inside quotes, after a label the shell wrote. An app can say anything, but can't look like the
  shell.
- **The binder decides, and the shell only answers.** The binder keeps every granter capability. Its compiled-in policy is a
  **ceiling**: a request outside it is declined with no prompt. Inside it, a remembered "always" or "never" settles the
  request; otherwise the binder writes the request into a shared consent `Frame`, signals the shell, and waits. A compromised
  shell can say yes to anything within the ceiling, and no further.
- **One notification.** The shell waits on one notification whose badge bits are its inputs: the console, `ASK`, the app done
  or failed, and a system service failed. Answers return as one badge bit each on the binder's notification. No kernel change
  was needed.
- **Recorded first.** `CONSENTED` (once, always, remembered), or a decline for `PERSON_DENIED`, `REMEMBERED_DENIAL` or
  `NO_CONSENT`, is appended before anything else happens.
- **Remembered decisions:** a person's "always" and "never" go in their own log in the policy store's sector 1; the audit ring
  moves to sectors 2 to 7, in format 3. **This ends ADR-0027's "narrow-only".** A remembered "always" widens, by skipping
  the prompt, but never past the ceiling, and it covers only the ops it allowed: an app that asks for more is asked again.
- **Revoking means "ask again".** A revoke from the console ends the grant now and forgets any "always". "Never" is the
  permanent no, and `forget` clears it. Images without consent keep ADR-0027's durable revocation.
- **Console commands:** `grants`, `revoke N`, `audit [N]` (shown only after verifying the chain from `OLDEST`), `decisions`,
  `forget N`, `help`. There's no command to run anything. While a request is open, only an answer is taken.
- **No login.** Whoever is at the console is the person. That's said plainly rather than hidden behind a PIN that would be a
  public constant.
- **The roadmap gains a track, "Toward an installable system"**, changing no phase's exit criteria.

## Consequences

- **Easier:** a person sees each request, decides it, sees what each app holds, revokes it, and reads a verified audit log.
  With ADR-0030, that's the "visible, revocable capability set and faithful audit log" the Phase 3 exit criterion names, for
  this image's one app.
- **Harder / committed to:**
  - Every unsettled grant waits on a person, and the binder serves nothing else meanwhile. There's no timeout yet; that's the
    next kernel RFC. *(Done by [ADR-0033](./0033-deadlines-and-a-capability-clock.md): a request is declined
    `NO_CONSENT` after 60 seconds.)*
  - The system image runs one fixed app until a user-space launcher exists.
- **Trust boundary:** a new one, the console. Nothing enters the ADR-0004 TCB.
- **Not mitigated, and on [RFC-0023](../rfcs/0023-hardware-key-custody-in-phase-4.md)'s checklist:** someone who can write
  the flash can plant an "always".
- **Not mitigated in v0:** persuasive app text (SH2), and anyone at the console counts as the person.
