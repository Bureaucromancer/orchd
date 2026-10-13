# 09 — Roadmap

Eight milestones: five for the core, three for the console. Each has a
definition of done tied to scenarios in `10-testing.md`. **A milestone is done
when its scenarios pass on Linux and Windows.** Not one of them.

---

## M1 — One engine, end to end

Supervisor, registry, config loading, streaming proxy, on-demand load, fixed
idle TTL. llama.cpp only. No ledger: a single hard-coded concurrency limit of
one model stands in for admission.

Also in M1, out of sequence on purpose: `GET /orchd/v1/events` and the job store
skeleton. Both are cheap now and awkward later.

**Done when.** A cold chat completion streams without client timeout; the SSE
keepalive prevents a 30 s-timeout client from giving up on a 60 s load; the
model unloads on TTL; `kill -9` on the child produces a 502 and a clean
restart; scenarios S1–S4 pass.

---

## M2 — The ledger

`DeviceProbe` contract with NVML (Linux) and NVML+PDH (Windows)
implementations. Delta attribution, quiescent re-anchoring, foreign smoothing,
reconciliation. `GET /orchd/v1/status` and `/device`.

The fake probe for tests lands here, and from this point admission logic is
testable without a GPU.

**Done when.** `/status` tracks `nvidia-smi` within tolerance across a scripted
load including a synthetic foreign consumer; the ledger re-anchors exactly at
quiescence; reconciliation divergence stays under tolerance for an hour of
mixed load; scenarios S5–S8 pass.

---

## M3 — The image engine

sdcpp backend. Native async submission, job shim, OpenAI-shaped synchronous
wrapper, cancel-on-disconnect, envelope calibration, VAE tiling policy.
Passthrough routes, with the job endpoints intercepted, and introspection that
never loads (`05-api.md`).

**Done when.**
- An image generates through `/v1/images/generations`.
- The same job is reachable and cancellable through passthrough.
- A disconnected client's job is observably cancelled upstream within a
  second.
- Calibration produces a stable envelope across three runs within 5%.
- Scenarios S9–S13 and S35–S40 pass.

---

## M4 — Arbitration

Admission, eviction, drain, cross-engine queueing, priority classes, starvation
promotion, coexist/exclusive modes, `--fit-target` computation, yield policy.

This is the milestone the project exists for. Everything before it is
scaffolding and everything after it is hardening.

**Done when.** Scenarios S14–S24 pass, including the measured text-throughput
degradation number under concurrent image load recorded in the test output —
not asserted against a threshold, but recorded, because the honest answer is
not yet known.

---

## M5 — Hardening

OOM backoff, throughput watchdog and spill detection, orphan reaping, device
loss and recovery, circuit breakers, `/metrics`, structured logging review,
sysmem-fallback detection and documentation.

**Done when.** Scenarios S25–S34 pass; a week of unattended use on a
multipurpose desktop with no manual intervention; no leaked VRAM after 100
induced child deaths.

---

## C1 — Console: read-only

The SPA, embedded and standalone builds, live resource view from `/events`,
model catalogue browsing, job and queue visibility, calibration and eviction
history. Writes nothing.

Shipping read-only first is deliberate: it is useful immediately, it validates
the API shapes against a real consumer before those shapes are load-bearing for
writes, and it is the fastest way to find out whether `/status` actually
answers the questions it was designed to answer.

**Done when.** A user can answer "why is this slow", "what is loaded", "why did
that wait" and "what did it cost" without a terminal.

---

## C2 — Console: configuration

`PUT`/`DELETE` on managed definitions, `POST /models/validate` with the
fit preview, flag editing with guardrails and the raw escape hatch, LoRA
library, presets, local model discovery including diffusion companion-file
binding.

**Done when.** A new GGUF in the models directory can be discovered,
configured, fit-previewed, saved and served without editing a file by hand; a
multi-file Flux set can be bound and served the same way; a `source: file`
definition is visibly read-only and offers a copy-to-managed path.

---

## C3 — Console: install

Download jobs with progress and resume, Hugging Face repo/quant selection,
companion-set downloads with bindings pre-made, shared companion storage,
removal with reference checking.

**Done when.** A user with an empty models directory reaches a working chat
model and a working image model entirely through the GUI, on Windows, without
a terminal.

---

## Sequencing notes

- **The image half of M3 comes straight after M1.** orchd is built alongside
  Magic Lantern, which needs the image path first, and `00-scope.md` already
  puts image generation first.
  - The sd.cpp backend, the job shim and interception, cancellation, the
    OpenAI wrapper and introspection need no device probe, so they run on
    M1's stub admission.
  - Calibration and the VAE tiling policy need the ledger, so they follow M2.
  - The order is M1 → M3 (backend half) → M2 → M3 (the rest) → M4. M3 is done
    only when all of it is.
- **A read-only console may start after M1.** It consumes `/status`,
  `/events`, `/models` and `/jobs` as they exist and grows with them.
  - Those shapes stabilise at M4, so expect churn until then.
  - Running it against the live API is the best available review of that API.
  - It writes nothing until C2.
- **C2 must not start before M4 is done.** The fit preview is the console's
  distinguishing feature and it is meaningless until admission arithmetic is
  real.
- **Do not build download management before C3.** It is the most obviously
  useful feature and the least architecturally interesting, and building it
  early tends to grow a parallel model registry that then has to be merged
  with the real one.
- The ComfyUI backend, vLLM, multi-device and the rest of the suite are all
  after C3, and all out of scope for this document set.
