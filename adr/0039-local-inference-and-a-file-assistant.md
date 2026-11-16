---
adr: 0039
title: Local inference and a file assistant — a model behind a capability, and an agent that only has what it's granted
status: Accepted
date: 2026-11-16
deciders: ["TSC"]
rfc: ../rfcs/0034-local-inference-and-a-file-assistant.md
supersedes: []
superseded_by: null
---

# ADR-0039: Local inference and a file assistant

## Context

Phase 3's first exit criterion is "a user can run a capable local AI agent with a visible, revocable capability set and a
faithful audit log". The capability set, consent, revocation, the audit log and provenance already existed (RFC-0024 to
RFC-0026, RFC-0033). There was no model. [RFC-0034](../rfcs/0034-local-inference-and-a-file-assistant.md) proposed a
confined inference service and an agent that uses it. It has been accepted; this ADR is the durable record.

## Decision

**A language model runs in a confined native service, `inference-service`, behind a capability like any other. An agent
is an ordinary signed app: the file assistant reads one file, asks the model to continue it, and writes the result to
another, and holds those three capabilities and nothing else.**

- **The engine** is `lantern-ai-runtime`: a `no_std` Rust Llama-family transformer (RMSNorm, rotary embeddings, SwiGLU,
  grouped-query attention) over int8 weights in groups, after llama2.c's `runq.c`. It allocates nothing: the service
  gives it the model file and its buffers. Decoding is greedy, so the same model and text give the same words.
- **The model** ships as an `LMODEL01` file on its own read-only disk partition: Karpathy's TinyStories `stories15M`
  (MIT), quantized by `lantern-mkmodel`, 17.5 MB. `scripts/get-model.sh` downloads the checkpoint and tokenizer at pinned
  commits, checks each against a pinned SHA-256, converts them, and checks the result against `assets/model.digest`.
- **The service** reads the model through the volume (badge `MODELS`), checks the whole file's BLAKE3 digest against the
  one compiled in, and only then parses it. A missing or changed model leaves the service refusing every call `FAILED`.
  It grants only `generate` on the model, only over the binder's `GRANT`, each badge tied to its app slot's channel.
  It holds no store, network or console capability.
- **The capability** is `lantern:host/inference`, resource-scoped like `keystore` and `filesystem` (the RFC proposed
  link-scoped; see the RFC's note): `open(slot) -> option<model>`, `model.generate(prompt, max-tokens)`. A call is at
  most 128 tokens and a 2 KiB prompt (`lantern_abi::inference`).
- **The audit log** records each call before it runs: the badge, the model's digest prefix, the tokens asked for and the
  prompt's length (`inference::audit_subject`), under the service's own writer badge, `INFERENCE_WRITER_BADGE`. A call
  the log can't take doesn't run.
- **The file assistant** (`lantern-runtime/file-assistant-guest`) is a Wasm component signed with the demo key. Its
  manifest asks for file 1 read (`input`), file 2 write (`output`) and the model (`writer`); the binder's policy allows
  exactly those as a ceiling, and the person is asked for each.
- **Apps with several grants.** The launcher maps a channel per interface an app asks for
  (`launcher::channel_vaddr`), and the runtime's app mode binds every declared role, each into its own slot, routing
  each handle's calls by that slot (`IpcFilesystem::routed`, `IpcInference`), then calls the guest's `run` once.
- **A person's own files.** The shell's `write N TEXT` and `read N` reach the store under `STORE_CONSOLE_BADGE`, with
  their own channel. A write is recorded as the console (`provenance::CONSOLE`), and both are in the audit log.

## Consequences

- **Easier:** a person can run an AI agent, see and decide everything it's given, read what it wrote and see that it
  wrote it, and read in the log which model ran, for how much.
- **Found:** the keystore service took requests of at most 256 bytes, so the store couldn't seal a file over about 190
  bytes though it promised 256. Its buffer now holds a full block's request.
- **Harder / committed to:**
  - The system image holds a 24 MiB inference heap; `CNODE_SLOTS` is 192 and `MAX_FRAMES` 144. 256 MiB is still enough.
  - Building the image downloads 61 MB once, unless `LANTERN_NO_MODEL=1`.
  - The model is identified by a digest compiled into the image, not signed: a new model means a new image.
- **Checked:** the engine's logits track an independent f32 forward pass at every position of a synthetic model, on
  the host, closely enough that a wrongly encoded position fails.
- **Measured:** under QEMU's emulation on an x86 host, 64 tokens take about 2 seconds, and the confined RISC-V engine's
  words match the host engine's token for token. An independent f32 reference agrees with the int8 engine for the first
  50 to 71 tokens of each of three prompts, which is quantization drift, not a layout or arithmetic error.
- **Not done:**
  - Models as signed packages, more than one, or chosen by the person.
  - The tokens generated, recorded after a call.
  - Cutting a call short with a deadline, or fair scheduling across agents. A `REVOKE` waits for the call in progress.
  - The keystore for apps in the runtime's app mode.
  - Accelerators.
