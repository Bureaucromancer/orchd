# 08 — The console (phase 2)

A web GUI for model, LoRA and engine configuration: the piece that makes orchd
usable by someone who wants local inference without hand-writing `llama-server`
and `sd-server` command lines.

## The problem it solves

Configuring these engines by hand is not hard because the flags are obscure. It
is hard because:

- A diffusion "model" is often **several files**. Flux needs a diffusion model,
  a VAE, CLIP-L and a T5 text encoder, each downloaded separately, each with
  variants that are subtly incompatible. Getting this wrong produces an error
  message about tensor shapes.
- The flags that matter are buried among the flags that do not. `llama-server`
  has well over a hundred arguments; roughly fifteen determine whether your
  model fits and runs well.
- **LoRA handling is asymmetric between the engines.** sd.cpp takes structured
  `lora[]` entries per request and explicitly refuses `<lora:...>` prompt tags.
  llama.cpp takes adapters at launch and can hot-apply them per request through
  `/lora-adapters`. Neither tells you which LoRA goes with which base model.
- Nothing tells you whether a configuration will *fit* until you run it and
  find out.

The last one is the one only orchd can answer, and it is the console's reason
to be first-party rather than left to third parties.

## Plugin or separate application: both, via one decision

The question answers itself once the real work is identified. **The
configuration surface is an API, not a UI.** `05-api.md` already specifies it:
`PUT /orchd/v1/models/{id}`, `/loras`, `/downloads`, `/models/validate`,
`/events`. That is where the engineering is.

The console is then a static single-page application against that API, shipped
two ways from one build:

1. **Embedded** (default). The bundle is compiled into the orchd binary with
   `embed` and served at `/console` when `console.enabled`. One binary, no
   second install, no extra port, no cross-process config contention. This is
   what `llama-server` does with its own web UI and it is the right default.
2. **Standalone.** The same bundle served by anything — a static host, a
   separate container, a desktop shell — pointed at an orchd base URL. Nothing
   in the SPA assumes same-origin beyond a configurable base URL and CORS.

So "plugin or separate application" is not a fork in the design. It is a
packaging flag over one artefact, and the API is public either way, which means
a third party can write a better console without asking permission. That is the
licence stance expressed in architecture.

Keeping the bundle out of the core binary entirely was considered and rejected:
it would mean a second install step for the exact user who wanted a GUI because
they did not want install steps.

## Boundary against a generation UI

The console is **not** a generation front-end. It does not own prompting,
sampling parameters at request time, galleries, or workflow. That is a
different product with different users, and conflating them is how ComfyUI
ended up being three things at once.

The line:

| Console | Generation UI |
|---|---|
| what models exist, and where their files are | what you generate with them |
| what a model costs in memory, and whether it fits | prompts, seeds, sampler choices per request |
| which LoRAs exist, what base they target, their default weight | which LoRAs to apply to *this* image |
| engine flags, residency, priority | galleries, history, iteration |
| installing, calibrating, removing | — |

The console's registry endpoints (`GET /orchd/v1/models`, `GET /orchd/v1/loras`)
are precisely what a generation UI needs in order to populate its own pickers.
So the console's API is dual-purpose by construction: it is the inventory
service any front-end wants, and the console is simply its first client. Any
generation UI — including one built later in this suite — consumes the same
endpoints rather than reimplementing model discovery. Magic Lantern, a
generation front-end built alongside orchd, is the first. It reads models,
their state and their LoRAs here, and generates through native passthrough
(`05-api.md` § "Job-aware clients").

## What the console does

### Model discovery and import

Scan `paths.models_dir`. For GGUF, read the header: architecture, parameter
count, quantisation, trained context length, chat template presence, whether an
mmproj companion exists. Populate a draft definition from it.

For diffusion models, the companion-file problem is the real work. The console
should:

- recognise single-file checkpoints versus multi-file sets;
- identify likely companions by filename convention and by file size/shape;
- let the user bind them explicitly with a picker that shows *why* a candidate
  matched;
- warn on known-incompatible pairings rather than letting sd-server fail with a
  shape error.

This is where most of the console's value lives. It is also where it will be
wrong most often, so every inference is a suggestion the user can override, and
the resulting definition is a file they can read.

### Install and download

Download jobs are the same job kind as everything else (`02-architecture.md`),
so progress, cancellation and history come free. Source by Hugging Face repo
and quantisation, or by direct URL. Resumable, into the managed models
directory, with a manifest. Companion files for a diffusion model download as a
set, and the resulting model definition is written with all bindings already
made — which is the single biggest improvement over doing it by hand.

Shared companions (a T5 encoder used by three models) are stored once and
referenced, not copied.

### LoRA library

Metadata orchd otherwise has no reason to know: base-model compatibility,
trigger words, default weight, a note, a thumbnail, and the file's SHA-256 so
a client can recognise the same LoRA on another route. Stored in `loras.d/`.

Because the engines differ, the console presents one concept and translates:

- sd.cpp: entries become `lora[]` objects with `path`, `multiplier`, and
  `is_high_noise` where relevant.
- llama.cpp: adapters are passed at launch with `--lora-init-without-apply`, so
  they can be applied per request by scale without reloading the model.

### Presets

A named binding of model plus LoRA stack plus default sampling parameters. This
is the A1111-style parameter paradigm rather than a node graph, and it is
deliberately the shallow version: a preset is a saved set of defaults, not a
workflow engine. Presets surface in `GET /orchd/v1/models` as selectable ids so
that a generation UI can offer them without knowing what a LoRA is.

Presets apply only to orchd's own `/v1` surface, never to native passthrough.
A client that records what it sent never has hidden defaults underneath it.
A generation UI with presets of its own, such as Magic Lantern, does not need
these at all. Whether the first console ships them is open
(`11-open-questions.md` item 13).

### Flags with guardrails

The ~15 flags that matter as typed fields with validation and tooltips, plus
the raw `extra_args` escape hatch, always visible and never rewritten. An
abstraction over engine flags that cannot be escaped is worse than no
abstraction.

### The resource view

The thing nothing else has. A live picture of the ledger from `/orchd/v1/events`
and `/status`: capacity, each reservation, foreign usage, headroom, queue.

And, crucially, **before saving a definition**: call `POST /models/validate` and
show whether this model will fit alongside the others, in coexist mode, at the
configured resolution — with the partition drawn. Changing `ctx` and watching
the bar move is the feature that makes the scheduler's behaviour
comprehensible rather than mysterious.

### Calibration and diagnostics

Trigger calibration, show results and history, show watchdog baselines and
spill events, show recent evictions with reasons, surface the
sysmem-fallback-policy recommendation on Windows with an explanation of what it
does.

## What the core must do first

These are MVP requirements that exist *because* the console is coming. They are
cheap now and expensive to retrofit, and they are already reflected in docs
00–07:

1. **Layered config with `models.d/`** and a `source: managed | file`
   distinction, so the console has somewhere to write that it unambiguously
   owns (`06-config.md`).
2. **Non-destructive reload**, diffed by definition hash, that never takes the
   service down over a bad edit (`06-config.md`).
3. **Management API designed as a product surface**, with stable shapes and
   machine-readable error codes (`05-api.md`).
4. **`GET /orchd/v1/events`** from M1, even with no consumer.
5. **Generic job kinds** from the first job implementation, so `download` and
   `calibrate` are not special cases bolted on later (`02-architecture.md`).
6. **`POST /models/validate`**: admission arithmetic callable as a pure
   what-if against a hypothetical definition, without side effects. This falls
   out of a well-factored admission function and is painful if admission is
   entangled with mutation.

If the core ships with those six, the console is a front-end project rather
than a core redesign.

## Non-goals for the console's first release

No multi-user accounts or RBAC. No remote model repositories beyond Hugging
Face and direct URL. No workflow graphs. No image editing. No telemetry. No
cloud account of any kind.

## Technology

Keep it boring and keep the build out of the critical path: a small SPA (the
framework is open — `11-open-questions.md` item 21), no server-side
rendering, no runtime framework in the Go build, one `npm run build` producing static assets committed or built in CI and
embedded with `embed`. A contributor must be able to build orchd without a
Node toolchain — the embedded bundle is checked in, and a `make console`
regenerates it.
