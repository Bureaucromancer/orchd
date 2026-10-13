# Engine: stable-diffusion.cpp (`sd-server`)

Verified against `examples/server/api.md` upstream as of October 2026.

## Role

The first image backend. Chosen over ComfyUI deliberately: engine-only, MIT,
ggml-based, no internal scheduler to fight, and on the same architectural
trajectory as llama.cpp — betting on convergence across DiT variants, flow
matching and LLM text encoders, which compresses porting lag and means new
models arrive within days.

ComfyUI remains the later proving case for the capability contract precisely
because it is hostile: engine, workflow runtime and UI in one, GPL-3, and an
internal memory manager that will contend with orchd's.

## Three API families

| Family | Shape | orchd's use |
|---|---|---|
| `/v1/...` | OpenAI-compatible images; **synchronous** to the caller | not used internally; orchd serves this shape itself |
| `/sdapi/v1/...` | A1111-compatible txt2img/img2img; **synchronous** | exposed only through passthrough |
| `/sdcpp/v1/...` | native, **async**, job-based | **what orchd uses for everything** |

Both compatibility families accept native parameters embedded in `prompt` via
`<sd_cpp_extra_args>{...}</sd_cpp_extra_args>`. orchd does not use that
mechanism; it speaks the native schema directly.

## Native endpoints

```
GET  /sdcpp/v1/capabilities
POST /sdcpp/v1/img_gen            → 202 {id, kind, status, created, poll_url}
POST /sdcpp/v1/vid_gen            → 202, same shape
GET  /sdcpp/v1/jobs/{id}          → 200 | 404 | 410
POST /sdcpp/v1/jobs/{id}/cancel   → 200 | 404 | 409 | 410
```

Job states: `queued`, `generating`, `completed`, `failed`, `cancelled`. The job
object carries `id`, `kind`, `status`, `created`, `started`, `completed`,
`queue_position`, `result`, `error`.

**There are no per-step progress fractions.** Queue position and coarse state
are all the engine offers. This is why `ProgressReporter` is unimplemented for
this backend, and it is a useful proof that the optional capabilities are
genuinely optional.

Submission errors: `400` for bad parameters or an unsupported model mode,
`429` when the job queue is full, `500` otherwise.

## Why orchd always submits async

Even when serving a synchronous `/v1/images/generations` to a client. The
synchronous families give orchd no job identity, no cancellation handle and no
queue visibility, and they tie up a connection for the duration. The async path
gives all three for the cost of a poll loop. See `adr/0003`.

## Capabilities discovery

`GET /sdcpp/v1/capabilities` returns the loaded model's metadata (`name`,
`stem`, `path`), `supported_modes` (`img_gen`, `vid_gen`), per-mode defaults,
output formats and feature flags, plus `samplers`, `schedulers`, `loras`,
`upscalers` and `limits` (min/max width and height, `max_batch_count`,
`max_queue_size`).

Two uses:
1. The load gate — a 200 here is sd-server's readiness signal.
2. The console's capability picker. `features_by_mode` tells it whether this
   model supports `init_image`, `mask_image`, `control_image`,
   `ip_adapter_image`, `ref_images`, `lora`, `vae_tiling`, `hires`, `cache`,
   `cancel_queued` and `cancel_generating`.

The top-level `defaults`, `output_formats` and `features` fields are deprecated
mirrors of `current_mode`. Use the `*_by_mode` forms.

## One model per process

`capabilities.model` is a single object and `/v1/models` returns a fixed
`sd-cpp-local` id. There is no model-switch endpoint and no unload. Therefore:
**a model change is a process change.** Eviction of an image model is killing
its process; loading another is spawning a new one.

This is why image models default to `residency: on_demand` with a short TTL —
holding diffusion weights resident between bursty uses is expensive, and a
warm-cache reload at the front of a minute-long job is tolerable.

## Sizing: calibration, not computation

sd-server reports nothing about memory. The envelope is measured.

`envelope.peak_mib: auto` triggers a calibration job on first use: with no other
owned process resident, submit a probe `img_gen` at the model's configured
maximum resolution and batch, sample the device at 100 ms intervals across the
job, take the peak, add the platform cushion. Cache keyed by model and
companion file digests, max resolution, batch, tiling settings and engine build.

## VAE tiling is policy

On most models the **decode** peak, not sampling, sets the envelope. The native
schema exposes `vae_tiling_params`: `enabled`, `temporal_tiling`, `tile_size_x`,
`tile_size_y`, `target_overlap`, `rel_size_x`, `rel_size_y`,
`extra_tiling_args`.

`vae_tiling: auto | always | never` in the model definition. Under `auto`, orchd
enables tiling when the untiled envelope would not fit the coexist partition,
trading some decode speed for a much flatter peak. **This is the one case where
orchd rewrites a native request body** — it is logged, and the job record
reports that it happened.

The speed cost at various tile sizes is unmeasured; see
`11-open-questions.md` item 11.

## Queue bound

Set the engine's queue small (1–2) via its startup options so that backpressure
lands in orchd's queue, where jobs can be reordered, prioritised and cancelled,
rather than in the engine's FIFO where they cannot. A `429` from the engine
therefore indicates an orchd accounting bug, not a client problem.

## LoRA

Structured only: `lora[]` entries with `path`, `multiplier` and `is_high_noise`.
Prompt-embedded `<lora:...>` tags are intentionally unsupported by the engine,
across all three API families. Available adapters are scanned from the LoRA
directory and reported in `capabilities.loras[]`.

The console's LoRA library maps onto this directly, and the asymmetry with
llama.cpp — launch-time adapters applied per request by index — is hidden
behind one console concept.

## Images and results

All image fields accept raw base64 or a data URL. Channel expectations:
`init_image`, `ref_images[]`, `control_image`, `ip_adapter_image` are 3-channel;
`mask_image` is 1-channel.

Completed `img_gen` results carry `result.images[]` with `index` and
`b64_json`. Completed `vid_gen` results carry a single encoded container as
`result.b64_json` with `mime_type`, `output_format`, `fps` and `frame_count`
(note: the core normalises frame count to the largest `4n+1` not exceeding the
request, so the returned count may differ from the one asked for).

Base64 image payloads through orchd's proxy should be streamed to disk or
relayed rather than accumulated in memory per job — a batch of 1024×1024 PNGs
is tens of megabytes and the job store must not hold them.

## Known issue

There is an upstream report of `GET /sdcpp/v1/capabilities` returning 500 on
some builds while the OpenAI and A1111 families work. Since orchd uses
capabilities as its readiness gate, the load gate should fall back to a second
signal (a 200 from `GET /v1/models`) and log loudly rather than failing the
load outright. Verify against the pinned build.
