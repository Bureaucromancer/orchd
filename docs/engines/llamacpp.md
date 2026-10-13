# Engine: llama.cpp (`llama-server`)

Verified against the upstream server README and manual pages as of October 2026.
Flags move; re-verify against the pinned build before relying on anything here.

## Role

The reference text backend. Chat, completion and embedding modalities.

## Invocation

One process, one model, an ephemeral loopback port. orchd supplies:

```
llama-server
  --model <path>
  --alias <orchd model id>
  --host 127.0.0.1 --port <ephemeral>
  --fit-target <computed MiB>        # when tuning.fit
  --no-webui
  --metrics
  [--embedding]                      # when modality: embedding
  [--lora <path> ...] [--lora-init-without-apply]
  <tuning flags>
  <extra_args verbatim, last>
```

`extra_args` goes last so a user can always override orchd's choice.

## Flags that matter

| Flag | Use |
|---|---|
| `-m, --model` | model path |
| `-a, --alias` | the name clients see; set to orchd's model id |
| `-c, --ctx-size` | context. **Leave unset when `fit: true`** |
| `-ngl, --gpu-layers` | `auto`, `all`, or a count. Leave unset when `fit: true` |
| `-fit, --fit [on\|off]` | adjust unset arguments to fit device memory (default on) |
| `-fitt, --fit-target MiB` | target margin per device, default 1024. **This is orchd's main lever** |
| `-fitc, --fit-ctx N` | minimum context `--fit` may fall to, default 4096 |
| `-fa, --flash-attn [on\|off\|auto]` | |
| `-ctk` / `-ctv` | KV cache quantisation; a cheap way to shrink the resident envelope |
| `-np, --parallel` | server slots |
| `-cram, --cache-ram` | host-side prompt cache cap |
| `-dev, --device` | device selection |
| `--api-key` | passthrough when configured |
| `--jinja`, `--chat-template` | template handling; default jinja on |
| `--no-webui` | orchd serves its own console; the engine's UI is noise |
| `--metrics` | Prometheus endpoint, feeds the watchdog |

## Flags orchd deliberately does not use

| Flag | Why not |
|---|---|
| `--models-dir`, `--models-preset`, `--models-max`, `--models-autoload` | router mode. An engine that loads and unloads models on its own schedule makes the ledger a fiction. See `adr/0001` |
| `--sleep-idle-seconds` | idle sleep frees memory invisibly and wakes without an admission check. Same reason |
| `--tools`, `--agent`, `--mcp-servers-config` | orchd is a scheduler, not an agent host. These also narrow CORS defaults and spawn child processes orchd does not supervise |

If a user puts any of these in `extra_args`, orchd logs a warning naming the
flag and the invariant it breaks, and proceeds. The escape hatch stays open;
it is just signposted.

## Readiness and health

`GET /health` returns 503 with a loading message while the model loads and 200
`{"status":"ok"}` once ready. That transition is the load gate: the load worker
polls it until 200 or `start_timeout`.

## Introspection

- `GET /props` — model alias and path, `total_slots`, `modalities`,
  `build_info`, default generation settings including `n_ctx`, chat template
  capabilities, and sleep state. Read once after readiness and cache; it is how
  orchd learns the context size `--fit` actually chose.
- `GET /slots` — per-slot state; enabled by default.
- `GET /metrics` — Prometheus, with `--metrics`. Feeds watchdog baselines.
- Completion responses carry a `timings` block including tokens per second.
  This is the cheapest throughput signal and the watchdog's primary input.

## Sizing

Delegate to `--fit` for this milestone set rather than modelling GGUF memory.

```
fit_target = headroom + Σ(other reservations incl. image envelopes) + foreign_smoothed
```

Leave `ctx` and `ngl` unset; llama.cpp trims them to fit the margin. After
readiness plus settle, measure the actual delta and rewrite the reservation
from `estimated` to `measured`, and read back the chosen `n_ctx` from `/props`
for display.

Two caveats, both **SPIKE** items in `11-open-questions.md`: whether `--fit`
targets free or total memory (which decides load ordering), and exactly which
unset arguments it will adjust.

## LoRA

Adapters are passed at launch. With `--lora-init-without-apply` they are loaded
but inactive, and a request may then apply them by index and scale:

```json
{"lora": [{"id": 0, "scale": 0.8}]}
```

This is the mechanism the console's LoRA library targets for text models: load
the catalogue's adapters at launch, apply per request. Note that requests with
different LoRA configurations are not batched together, so heavy per-request
LoRA variation costs throughput.

## Cancellation and draining

No explicit cancel endpoint; closing the client connection frees the slot.
Draining is therefore: stop routing, poll `/slots` until idle or
`drain_timeout`, then stop the process.

## Streaming

Standard SSE. `--sse-ping-interval` controls the engine's own keepalive, and a
request may override it. orchd emits its own keepalives during the load window,
before any upstream connection exists.

`return_progress` adds prompt-processing progress to streamed responses — this
is what makes `ProgressReporter` implementable for this backend, in contrast to
sd.cpp.

## Notes

- `-cl, --cache-list` lists models in the engine's own cache; useful to the
  console's discovery, not to the scheduler.
- `-hf` can download models. orchd does not use it: downloads are orchd jobs so
  that progress, cancellation and the ledger's view of disk are consistent
  across both engines.
- `--host` accepts a path ending in `.sock` for a Unix socket. Not used —
  Windows has no equivalent and the divergence buys nothing.
