# 06 — Configuration

## Layering

Configuration is **not** a single file, because two parties write it: a human
with opinions and comments, and the console with a form. Merging those into one
file means either destroying comments on every save or implementing
comment-preserving YAML round-tripping, which is a tax paid forever.

```
<config_dir>/
  orchd.yaml            human-owned. orchd reads, never writes.
  models.d/
    qwen3-30b.yaml      one model per file. source: managed → console may rewrite.
    flux-dev.yaml       source: file → hand-authored, console must not touch.
  loras.d/
    *.yaml              LoRA metadata, same ownership rule
<state_dir>/
  calibration.json      measured envelopes, keyed by content hash
  downloads.json        download job manifest
```

Config directory by platform: `$XDG_CONFIG_HOME/orchd` or `/etc/orchd` on
Linux, `%PROGRAMDATA%\orchd` on Windows, `~/Library/Application Support/orchd`
on macOS. State directory analogously. All overridable by flag and environment.

Precedence, lowest to highest: built-in defaults → `orchd.yaml` → `models.d/`
entries → command-line flags. A model defined in both `orchd.yaml` and
`models.d/` is an error at load, not a silent merge.

A model definition being one file is deliberate beyond the ownership question:
it makes a working configuration something you can post in a gist, and it makes
the console's "export this model" a file copy.

## `orchd.yaml`

```yaml
listen: 127.0.0.1:11434
api_key: ""                     # optional; passed through to engines if set

device:
  index: 0
  capacity_mib: auto
  headroom_mib: auto            # platform default; see 03-device-model.md
  poll_interval: 1s
  foreign_hysteresis_samples: 3
  reconcile_interval: 30s

policy:
  mode: coexist                 # coexist | exclusive
  idle_ttl_relaxed: 30m
  idle_ttl_pressured: 60s
  pressure_threshold: 0.85
  queue_timeout: 120s
  drain_timeout: 30s
  start_timeout: auto
  image_jobs_block_text: false
  yield_on_foreign_pressure: true
  yield_threshold: 0.5
  yield_recovery_delay: 60s
  starvation_promote_after: 5

engines:
  llamacpp:
    binary: /usr/local/bin/llama-server
    common_args: [--no-webui, --metrics]
  sdcpp:
    binary: /usr/local/bin/sd-server
    common_args: []

paths:
  models_dir: /srv/models
  lora_dir: /srv/models/lora
  upscalers_dir: /srv/models/upscalers

console:
  enabled: true                 # serves the embedded bundle at /console
  bind_to_listen: true          # false → separate listener, see 08-console.md

log:
  level: info
  file: auto                    # platform default; never assume a console
```

## Model definition

One file per model in `models.d/`.

```yaml
id: qwen3-30b
engine: llamacpp
modality: text                  # text | embedding | image | video
source: managed                 # managed | file
display_name: Qwen3 30B A3B
residency: resident             # resident | on_demand | never_evict
priority: interactive           # interactive | batch
idle_ttl: null                  # null → policy default

model:
  path: /srv/models/qwen3-30b-a3b-q4_k_m.gguf

tuning:
  fit: true                     # orchd supplies --fit-target; ctx/ngl left unset
  ctx: null                     # pinning this opts out of fit — validated, not ignored
  flash_attn: auto
  cache_type_k: f16
  cache_type_v: f16
  parallel: null

envelope:
  resident_mib: auto            # auto → measured at load, persisted
  cushion_pct: auto             # platform default

extra_args: []                  # raw passthrough; orchd never rewrites these
```

For an image model, `model` carries the companion files that make sd.cpp hard
to configure by hand:

```yaml
id: flux-dev
engine: sdcpp
modality: image
source: managed
residency: on_demand
idle_ttl: 120s

model:
  diffusion_model: /srv/models/flux1-dev-q8_0.gguf
  vae: /srv/models/ae.safetensors
  clip_l: /srv/models/clip_l.safetensors
  t5xxl: /srv/models/t5xxl_fp16.safetensors

limits:
  max_resolution: 1024x1024
  max_batch_count: 4

defaults:                       # applied when the client omits them
  sample_steps: 28
  sample_method: euler
  scheduler: discrete

vae_tiling: auto                # auto | always | never — see engines/sdcpp.md

envelope:
  peak_mib: auto                # auto → calibration job on first use
  cushion_pct: auto

extra_args: []
```

### Field rules

- **`extra_args` is sacred.** It is the escape hatch, it is always available,
  and nothing — not the console, not a config migration — ever rewrites it.
  Every abstraction over engine flags leaks eventually; the escape hatch is
  what makes the leak survivable.
- **`source: file` definitions are read-only to the management API.** `PUT` and
  `DELETE` return 409. The console may read, display and offer to *copy* them
  into a managed definition, never edit in place.
- **Validation is strict and early.** Unknown keys are errors, not warnings.
  Referenced files must exist. `tuning.ctx` set together with `tuning.fit: true`
  is an error with an explanation, because silently ignoring one of them
  produces a model that is mysteriously the wrong size.

## Reload

`POST /orchd/v1/config/reload`, plus an optional filesystem watcher on
`models.d/` and `loras.d/` (`config.watch: true`, default on).

Reload is a diff, not a restart:

1. Parse and validate the whole tree. On any error, **keep the running config**
   and report the error; a bad edit must never take the service down.
2. Hash each model definition. Unchanged definitions keep their running
   process, their reservation, and their calibration record.
3. Changed definitions mark the model `stale`; it is drained and reloaded on
   next use, or immediately if `residency: resident`.
4. Removed definitions drain and stop.
5. Policy and device changes apply immediately; they do not evict anything by
   themselves, but they change the next admission decision.

Writes from the management API are atomic: write to a temporary file in the
same directory, fsync, rename. The watcher debounces 200 ms to avoid reacting
to a half-written tree.

## Calibration cache

`<state_dir>/calibration.json`, keyed by a hash over the fields that actually
affect memory — model file digest, companion file digests, max resolution,
batch, tiling settings, engine binary version.

```json
{
  "sha256:ab12…": {
    "model": "flux-dev",
    "peak_mib": 13900,
    "basis": "calibrated",
    "measured_at": "2026-10-04T18:22:11Z",
    "engine_build": "master-813-bfbef5b",
    "oom_backoff_applied": 0,
    "baseline_s_per_step": 0.84
  }
}
```

The cache is advisory: a missing or stale entry costs a calibration pass, not a
failure. It is never the authority over a live measurement.
