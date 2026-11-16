---
rfc: 0034
title: Local inference and a file assistant — a model behind a capability, and an agent that only has what it's granted
status: Accepted
authors: ["TheNewAutonomy"]
stewards: ["ai-runtime", "runtime", "shell", "boot"]
domains: ["ai-runtime", "runtime", "abi", "shell", "boot", "filesystem", "sdk", "docs"]
created: 2026-11-15
updated: 2026-11-16
supersedes: []
superseded_by: null
tracking_issue: null
---

> **Accepted 2026-11-16, implemented and recorded in [ADR-0039](../adr/0039-local-inference-and-a-file-assistant.md).**
> Built as proposed, with these changes:
> - `inference` is resource-scoped (`open(slot) -> option<model>`), like `keystore` and `filesystem`, not link-scoped, so
>   the model is one role an app asks for, and the person grants and revokes it alone, through the same binder path.
> - The per-call limit is the service's (128 tokens, a 2 KiB prompt), not a ceiling in each grant. The file assistant
>   asks for 64.
> - The audit record is written before the call, with the model's digest prefix, the tokens asked for and the prompt's
>   length. How many were generated isn't recorded after.
> - The shell gained `write N TEXT` and `read N`, so a person can give the assistant text and read what it wrote. The
>   store records a console write as the person's, and both are audited.
> - Section 6's comparison is against an independent f32 reference written for this RFC (`lantern-reference`), not
>   llama2.c's `runq.c`, which wasn't run here: greedy decoding agrees for the first 50 to 71 tokens of three prompts,
>   as int8 drift predicts. The confined RISC-V engine matches the host engine token for token.
> - Revoking a grant during a call isn't tested under QEMU: a 64-token call ends in about 2 seconds. The service's
>   grant table, including revocation, is host-tested.
>
> **Found while implementing:** the keystore service's request buffer held 256 bytes, so the store couldn't seal a
> file over about 190 bytes; now it holds a full block's request.
>
> **Direction agreed 2026-11-15.** The unresolved questions were settled as proposed:
> the image ships `stories15M`, the agent continues the text of its file, and the model's digest is recorded in the audit
> log only, not in the file's provenance.

# RFC-0034: Local inference and a file assistant

## Summary

Phase 3's first exit criterion is: **"a user can run a capable local AI agent with a visible, revocable capability set and
a faithful audit log."** The capability set, its consent, revocation and audit log already exist (RFC-0024 to RFC-0026),
and provenance records what wrote each file (RFC-0033). What's missing is the AI.

This RFC adds two things:

1. **`inference`, a confined native service** that runs a language model on the CPU. It reads the model from its own disk
   partition, checks it against a digest, and serves `generate` to whoever holds a grant to it. It has no network, no
   console and no access to the store.
2. **`file-assistant`, an agent**: an ordinary signed Wasm app. It reads one granted file, asks the model to continue it,
   and writes the result to a second granted file. Each of those three capabilities is asked for at the console, can be
   revoked, and every use is in the audit log. The file it writes says it wrote it.

```
lantern> run file-assistant
"file-assistant" from publisher 3b1e9a40..77c2 asks to use:
  the language model            (generate, up to 128 tokens a call)
allow? [y]es once, [a]lways, [n]o, n[e]ver > y
  ...
lantern: "file-assistant" finished.
lantern> files
  file 1  412 bytes  written by "file-assistant" (publisher 3b1e9a40..77c2), write 7, boot 2
lantern> audit
  ...  inference  GENERATE  grant 41  "file-assistant"  model 6f0c..e2  96 tokens
```

An agent is an app. It gets no new path, no new privilege, and nothing the binder's policy and the person didn't allow.

## Motivation

- It's a Phase 3 exit criterion, and the [AI](../../lantern-docs/wiki/AI.md) page's three jobs (local inference, model
  management, capability-scoped agency) start here.
- The argument that capabilities defeat prompt injection is only an argument until an agent that reads untrusted text
  exists and can be shown not to exceed its grants. This makes it testable: a file that tells the model to do something
  else changes the words the agent writes, never what the agent can reach.

## Guide-level explanation

- The image gains a **models** partition, holding model files written by `mkdisk --model`. A model file is a header, the
  weights and the tokenizer. Its identity is the BLAKE3 digest of the whole file.
- At start, `inference` reads its model through the volume (a new `MODELS` badge, partition 4), checks the digest against
  the one compiled into the system image, and refuses to serve if they differ.
- An app declares `lantern:host/inference` in its manifest, as it declares `filesystem`. The binder asks, grants a badge on
  `inference`'s endpoint, and the runtime wires it to the guest's `inference` import.
- `generate(prompt, max-tokens)` returns text. The service records each call in the audit log before running it, with the
  grant, the model digest and the token count.

## Reference-level explanation

### 1. The engine

- A Llama-architecture transformer (RMSNorm, rotary embeddings, SwiGLU, grouped-query attention), in `no_std` Rust, after
  llama2.c's `runq.c`: int8 weights in groups of 64 with an f32 scale per group, f32 activations.
- Greedy decoding by default; a sampling temperature with a seed the caller gives. No randomness the caller didn't supply,
  so the same prompt, model and seed give the same output, which is what makes it testable.
- The KV cache and activations are sized from the model's header when the service starts, in memory the loader gives it.
- One request at a time. A second caller waits.
- The same engine is a host-built library, tested against the reference outputs of llama2.c for the same model and prompt.

### 2. The model file

- `mkdisk --model <llama2.c .bin> --tokenizer <tokenizer.bin>` quantizes the f32 checkpoint to int8 and writes
  `LMODEL01`: dimensions, then the quantized weights, then the tokenizer.
- The system image carries the model's digest. A different model on the disk is refused, until a model is a signed
  package of its own (future work).
- Which model the image ships is unresolved question 1.

### 3. The capability

- A new interface, `INTERFACE_INFERENCE = 3`, and the WIT interface `lantern:host/inference`: link-scoped, like
  `monotonic-clock`, since there's one model and nothing to scope a handle to.
- `generate(prompt: string, max-tokens: u32) -> result<string, error-code>`.
- A grant carries a per-call token ceiling (the manifest's request, capped by the binder's policy) and the binder's
  principal, as every grant does since RFC-0033.
- The service checks the badge on every call; a revoked grant is refused at once.
- Each call is recorded through `policy::audited` before it runs: grant, principal, model digest (first 8 bytes), and the
  tokens asked for. The service also records, after, how many it generated.

### 4. The file assistant

- A Wasm component built with `lantern-sdk build`, signed, on the disk like the other packages.
- Its manifest asks for: read on one file, write on another, and `inference`.
- It reads its input file, asks for a continuation, and writes the prompt plus the continuation to its output file.
- It has nothing else. Its input can say anything; the test in section 6 gives it a file that tells it to write elsewhere.

### 5. Memory

- The service's model and buffers are retyped from the general Untyped by the system loader, in megapages: about
  `model size + KV cache`, rounded up. The system image's `-m` grows with the model.

### 6. The proof

- Host tests: the engine matches llama2.c's output for the shipped model on fixed prompts, token for token; int8
  quantization round-trips within tolerance; the model file parses and rejects a bad digest.
- `console-matrix.sh`:
  - `run file-assistant`, answered yes three times, writes its output; `files` names `file-assistant` as the writer;
  - `audit` shows the `GENERATE` record with the model digest;
  - `revoke` of its inference grant, then `run file-assistant` with "once" for the files and "no" for the model: the app
    is told `access`, writes nothing, and the audit says why;
  - an input file that tells the model to write to another file: the output file holds the model's words, and no other
    file changes;
  - a disk with a different model: `inference` refuses, and the agent's call is refused.
- Mutation tests: the service skips the badge check; the service skips the digest check; the audit record is dropped.

## Threat model impact

- **The model is untrusted input to the agent, and the agent is untrusted to the system.** The service is in neither the
  agent's nor the store's trust; it sees prompts and returns text.
- **The service sees every prompt.** It holds no store, network or console capability, so it can't pass them on. A
  malicious service could return wrong text.
- **Prompt injection** changes what an agent writes, never what it can reach: section 6 demonstrates it.
- **Resource use:** a caller can hold the service for up to its token ceiling per call. The deadline object (RFC-0028)
  isn't used to cut a call short; that's future work, with fair scheduling across agents.
- **The model is identified, not authenticated:** the digest is compiled into the image, so a changed model is refused,
  but there's no model signature yet.

## TCB impact

None. The engine and the service run confined in U-mode; the kernel only gains nothing, and the loader only places one
more program.

## The three vision questions

1. **Kernel smaller, safer, easier to reason about?** Unchanged: no kernel change.
2. **Can an agent use this without being unnecessarily trusted?** Yes: the model is a capability like any other, asked for,
   revocable and audited, and the agent holds three caps and nothing else.
3. **Same interface for humans and agents?** Yes: the agent is an app, granted through the same binder and console, and
   a person's app could hold the same `inference` grant.

## Alternatives considered

- **Inference inside the runtime (a Wasm guest or a host import).** No new service, but the model's memory and work would
  share the agent's process, and the agent's runtime would hold the weights. A separate service is the capability
  boundary.
- **A model in a Wasm component.** Portable, but much slower under Pulley's interpreter, which is already slow.
- **Cloud inference.** Not local-first; and there's no network yet.

## Unresolved questions

1. **Which model ships in the image:**
   - `stories15M` (Karpathy's TinyStories, MIT): about 17 MB as int8. It writes coherent short stories and continues text,
     and nothing else: it doesn't follow instructions. A few tokens a second under QEMU.
   - `stories260K`: 1 MB and fast, but barely coherent.
   - An instruction-following model such as SmolLM2-135M-Instruct (Apache-2.0): it actually answers requests, but it's about
     140 MB as int8 and probably under a token a second under QEMU.
2. **What the agent does:** continue the text of a file (fits any model), or follow an instruction in the file (needs an
   instruction model).
3. **Whether the model's digest joins the file's provenance entry**, beside the principal, or stays in the audit log only.

## Future possibilities

- Models as signed packages, chosen by the person, and more than one.
- Accelerators (GPU/NPU) behind a confined driver and an IOMMU, behind the same `inference` interface.
- Fair scheduling and deadlines across agents; streaming tokens.
- An agent that proposes actions and asks consent for each.

## Resulting ADRs

On acceptance, an ADR will record: the inference service and its model file, `INTERFACE_INFERENCE` and the WIT interface,
the audit record for a model call, and the file assistant.
