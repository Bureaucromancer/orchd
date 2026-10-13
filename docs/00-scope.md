# 00 — Scope

## What is being built

A single Go binary that supervises `llama-server` and `sd-server` child
processes on one machine with one GPU, arbitrates their use of video memory
against a single budget, and serves OpenAI-compatible HTTP to clients.

"Minimal usable" is defined concretely: **point orchd at a GPU holding a
resident chat model, ask it for an image, and have both engines survive — no
out-of-memory, no silent collapse into system-memory paging, and no tearing
down the chat model for every picture.** A build that does that is useful to
its author on day one. Everything else is elaboration.

## In scope

| Area | Included |
|---|---|
| Engines | llama.cpp (`llama-server`), stable-diffusion.cpp (`sd-server`) |
| Modalities | text generation, embeddings, image generation (and `vid_gen` passthrough) |
| Devices | one NVIDIA GPU |
| Platforms | Linux, Windows — co-equal, both gated in CI. macOS tier 2 |
| Scheduling | admission control, eviction, cross-engine queueing, idle TTL, foreign-pressure yield |
| Sizing | per-model envelopes, measured by calibration; `--fit` delegation for text |
| API | `/v1/*` OpenAI shapes, `/orchd/v1/*` management, per-model native passthrough |
| Config | layered YAML; a `models.d/` directory that is machine-writable |
| Phase 2 | the console: model / LoRA / engine configuration GUI |

## Out of scope

Multi-device and multi-GPU. ComfyUI and vLLM backends. ROCm, Intel, and
CPU-only scheduling. The gateway and chat-UI packages of the wider suite.
Authentication beyond bearer-token passthrough. Batch-by-model queue
reordering. Per-step progress reporting for image jobs (orchd relays whatever
step counts the engine reports, but builds nothing on them). Any persistence
beyond the calibration cache and the config tree.

These are omissions of sequencing, not of intent. The abstractions are shaped
so that adding a backend or a second device does not require reopening the
scheduler — but nothing in this milestone set is built for them, and
speculative generality for their sake is a defect, not a virtue.

## Why this order

The wider suite has three packages: the orchestrator, a self-hosted API
gateway, and a chat UI. The orchestrator is first because it is the only one
that fills a gap nothing else fills. `llama-swap` swaps models for one engine.
Lemonade Server manages process-per-model lifecycles with an explicit API.
LiteLLM routes across providers. None of them arbitrate memory *between*
engines on one device, which is precisely the situation on a machine that is
also somebody's desktop.

Within the orchestrator, image generation comes before anything else because
it is the cheapest honest test of the backend abstraction. stable-diffusion.cpp
is engine-only, MIT, ggml-based, and structurally cooperative. ComfyUI is the
opposite on every count — it conflates engine, workflow runtime and UI, it is
GPL-3, and it runs an internal scheduler that will fight external memory
management. Validating the capability contract against the cooperative engine
first, and treating the hostile one as a later proving case, is deliberate.

The console comes after the core rather than alongside it because the console
is a client of an API that does not exist yet. Building it earlier would mean
designing that API against a UI mockup instead of against the scheduler's real
state. It does, however, impose requirements on the core *now* — see
`08-console.md` § "What the core must do first". A read-only console can still
follow M1, against the endpoints that exist. What waits for the core is
anything that writes, or that depends on admission arithmetic. See
`09-roadmap.md`.

The first job-aware client is Magic Lantern, a generation front-end built
alongside orchd. It submits image jobs through native passthrough and reads
model state from the management API. `05-api.md` § "Job-aware clients" lists
what it depends on, and its needs are why the image half of M3 follows M1.

## What the engines already do

Both engines have moved. Two developments matter enough to state here, because
they change what orchd should build rather than merely how.

`llama-server` has acquired a **router mode** — `--models-dir`,
`--models-preset`, `--models-max`, `--models-autoload` — plus
`--sleep-idle-seconds` for idle unloading, with sleep state visible through
`/props`. This is overlapping functionality, and orchd deliberately does not
use it. An engine that decides on its own when to hold and release memory makes
orchd's ledger a fiction. See `adr/0001`.

`llama-server` has also acquired `--fit` / `--fit-target`, which adjusts unset
arguments to fit a target margin of free device memory. This is genuinely
useful and orchd *does* use it, to avoid modelling GGUF memory arithmetic in
the first milestone set. See `engines/llamacpp.md`.

## Success criteria for the milestone set

1. The coexistence scenario works unattended for a week on the author's
   machine without manual intervention.
2. No configuration file in that week needed hand-editing of engine flags to
   recover from an out-of-memory or a paging collapse.
3. `GET /orchd/v1/status` answers "why did that request wait?" without
   recourse to logs.
4. The admission logic is covered by tests that run without a GPU.
