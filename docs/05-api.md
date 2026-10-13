# 05 — API

Three surfaces: an OpenAI-compatible client API, a management API under
`/orchd/v1/`, and per-model native passthrough. Default listen address
`127.0.0.1:11434`.

The management API is **a product surface, not a debug surface**. The console
(`08-console.md`) is built entirely on it, and so is any third-party client.
Design it accordingly: stable shapes, explicit errors, no fields that only
make sense if you have read the source.

## Client API

| Endpoint | Notes |
|---|---|
| `POST /v1/chat/completions` | streaming and non-streaming |
| `POST /v1/completions` | |
| `POST /v1/embeddings` | routed to a model configured with the embedding flag |
| `POST /v1/images/generations` | synchronous to the client; async upstream |
| `POST /v1/images/edits` | multipart |
| `GET /v1/models` | the union of the catalogue, not any one engine's list |

`model` is resolved against orchd's catalogue ids, never passed through to the
engine for resolution. An unknown id is a 404 naming the id, with the available
ids in the error body.

### Cold-start transparency

A streaming request that triggers a load must not look like a hung connection.
While the model loads orchd emits SSE keepalive comments on
`sse_ping_interval`, plus named frames for clients that understand them:

```
: keepalive

event: progress
data: {"state":"loading","model":"qwen3-30b","elapsed_ms":4200,"queue_position":0}
```

Unmodified OpenAI clients ignore both and simply see a connection that stays
open. Non-streaming requests hold the connection with a longer timeout.

A request that is queued rather than loading emits the same frame shape with
`"state":"queued"` and a position, so a client can tell waiting-for-memory from
waiting-for-load.

### Disconnect handling

- Text: close the upstream connection so the slot frees immediately.
- Image: `POST /sdcpp/v1/jobs/{id}/cancel` upstream and mark the orchd job
  cancelled. Release the reservation when the engine reports the job ended:
  at once for a job still queued in the engine; at the end of the job if the
  build cannot cancel a generating one (`adr/0006`). Until then `/status`
  shows the reservation as `cancelling`.

Both are required, not optimisations. A disconnected client whose reservation
is still held after its work has stopped is a leak in the ledger. Releasing it
while the engine is still generating is the opposite error, and worse.

## Management API

All under `/orchd/v1/`.

| Endpoint | Purpose |
|---|---|
| `GET /status` | the whole picture — see below |
| `GET /models` | catalogue with state, envelope, provenance |
| `GET /models/{id}` | one entry, with full resolved argv, calibration record and cached engine capabilities; never loads — see below |
| `POST /models/{id}/load` | admin override; bypasses TTL, **not** the ledger |
| `POST /models/{id}/unload` | drain and stop |
| `POST /models/{id}/calibrate` | run or re-run envelope calibration as a job |
| `PUT /models/{id}` | create or replace a managed model definition |
| `DELETE /models/{id}` | remove a managed model definition |
| `POST /models/validate` | dry-run a definition without committing it |
| `GET /loras` | LoRA inventory with metadata and compatibility |
| `PUT /loras/{id}`, `DELETE /loras/{id}` | manage LoRA metadata |
| `GET /jobs`, `GET /jobs/{id}` | all job kinds: `img_gen`, `vid_gen`, `download`, `calibrate` |
| `POST /jobs/{id}/cancel` | |
| `POST /downloads` | start a download job |
| `GET /config`, `POST /config/reload` | effective merged config; explicit reload |
| `GET /events` | SSE stream of lifecycle transitions; resumable |
| `GET /device` | capacity, driver model, probe capabilities, platform warnings |
| `ANY /models/{id}/native/*` | passthrough to that model's engine |
| `GET /healthz`, `GET /metrics` | liveness; Prometheus |

`PUT /models/{id}` and friends only operate on definitions whose `source` is
`managed`. A hand-authored definition returns 409 with an explanation — orchd
never rewrites a file a human owns. See `06-config.md`.

### `GET /status`

The most important endpoint in the system. It must answer "why did that
request wait?" without recourse to logs.

```json
{
  "mode": "coexist",
  "state": "running",
  "device": {
    "capacity_mib": 32768,
    "total_used_mib": 30100,
    "foreign_mib": 1400,
    "foreign_confidence": "measured_at_quiescence",
    "headroom_mib": 2048,
    "free_mib": 2520,
    "driver_model": "WDDM",
    "sysmem_fallback": "unknown",
    "last_poll_age_ms": 420,
    "last_reconcile_divergence_mib": 60
  },
  "reservations": [
    {"model":"qwen3-30b","kind":"resident","mib":15100,"basis":"measured","state":"ready","idle_s":12},
    {"model":"flux-dev","kind":"resident","mib":13900,"basis":"calibrated","state":"ready","idle_s":340},
    {"model":"flux-dev","kind":"job","job":"job_01HT…","mib":0,"basis":"calibrated","state":"generating"}
  ],
  "queue": [
    {"request":"req_01HT…","model":"sdxl","class":"interactive","reason":"insufficient_budget",
     "position":1,"waiting_ms":8200,"blocked_by":["flux-dev"]}
  ],
  "watchdog": {"qwen3-30b": {"baseline_tps": 48.2, "current_tps": 44.9, "state": "ok"}}
}
```

`blocked_by` is the field that turns a support question into a glance.

### `GET /models/{id}`

Introspection never loads. A client that wants to know whether a model is
loaded, which files it is made of, or what its engine can do asks here. It
never asks through native passthrough, which loads the model it addresses.

```json
{
  "id": "flux-dev",
  "engine": "sdcpp",
  "modality": "image",
  "source": "managed",
  "state": "unloaded",
  "definition_hash": "sha256:9f3c…",
  "engine_build": "master-894-017cc8e",
  "limits": {"max_resolution": "1024x1024", "max_batch_count": 4},
  "envelope": {"peak_mib": 13900, "basis": "calibrated"},
  "argv": ["…"],
  "engine_capabilities": {
    "captured_at": "2026-10-12T19:04:51Z",
    "definition_hash": "sha256:9f3c…",
    "body": {"model": {"name": "…", "stem": "…", "path": "…"}, "…": "…"}
  }
}
```

- `state` is the lifecycle state: `unloaded`, `loading`, `ready`,
  `draining`, `stale`, `failed` or `unavailable`. It is not `residency`,
  which is the eviction class.
- `definition_hash` is the registry's content hash. It changes exactly when
  something that affects behaviour changes, companion files included, so a
  client can use it as the model's identity.
- `engine_build` is the build orchd launched, as configured or as a digest of
  the binary.
- `engine_capabilities` is the document the readiness gate read at the last
  load: `/sdcpp/v1/capabilities` for sd.cpp, `/props` for llama.cpp. It is
  null if the model has never loaded. Its own `definition_hash` says which
  definition it was read under; a client treats a mismatch as unknown.

### `GET /events`

SSE, one event per lifecycle transition, from the same stream the lifecycle
goroutine emits internally. Event types: `admission`, `load_start`,
`load_complete`, `evict`, `drain`, `child_exit`, `job_state`, `yield`,
`watchdog`, `config_reload`, `ledger_reconcile`, `reset`. The console
subscribes to this rather than polling; so should anything else that wants to
be live.

The stream resumes:

- Event ids are `<boot id>:<seq>`.
- A client reconnecting with `Last-Event-ID` receives the events it missed
  from the ring of recent events.
- If the id is from an earlier boot, or has left the ring, the client
  receives a `reset` event instead and refetches `/status` and `/models`.
- Keepalive comments hold idle connections open.

Build it in M1 even though nothing consumes it until later. It is cheap when
designed in and awkward when retrofitted, and it is the best debugging tool in
the system.

## Native passthrough

`ANY /orchd/v1/models/{id}/native/*` proxies to that model's engine with the
prefix stripped, after an admission check that loads the model if necessary.

This keeps the full engine surface reachable — `/sdcpp/v1/img_gen` and the
job endpoints, `/props`, `/slots`, `/sdapi/v1/*` — for clients that want
native control. It is also how a native-async image client gets real job
semantics with orchd's pre-admission states included.

Passthrough is subject to admission but not to translation. orchd does not
rewrite native request bodies, with one exception: it fills in VAE tiling when
the coexist partition requires it (see `engines/sdcpp.md`).

- The rewrite applies only when the request leaves `vae_tiling_params` unset.
  A value the request sets is never changed; such a request waits for room
  instead.
- A model's `defaults:` and the console's presets do not apply here at all.
  They belong to the `/v1` surface.
- Any rewrite is logged and listed in the job record.

### Job endpoints are intercepted

For sd.cpp, the job endpoints are not passed through. orchd has a state the
engine cannot have — waiting for memory — and the engine has no job id until
orchd submits, so orchd serves these endpoints itself:

- `POST …/native/sdcpp/v1/img_gen` (and `vid_gen`) creates an orchd job and
  returns 202 at once. The response has the engine's shape, with an orchd job
  id and an orchd `poll_url`. Any other answer means nothing was created.
- `GET …/native/sdcpp/v1/jobs/{id}` and `POST …/jobs/{id}/cancel` resolve the
  orchd id.
  - Before submission, orchd answers from its own record.
  - Afterwards it relays the engine's job object, keeping every field it does
    not recognise (step counts and previews included), and adds an `orchd`
    block.
- `status` stays in the engine's vocabulary, so an unmodified sd-server client
  keeps working: orchd reports `queued` while the job waits for admission. The
  superset is in `orchd.state`.

```json
{
  "id": "job_01J9ZB…",
  "kind": "img_gen",
  "status": "queued",
  "queue_position": 0,
  "result": null,
  "error": null,
  "poll_url": "/orchd/v1/models/flux-dev/native/sdcpp/v1/jobs/job_01J9ZB…",
  "orchd": {
    "state": "queued_admission",
    "reason": "insufficient_budget",
    "blocked_by": ["qwen3-30b"],
    "submitted_at": null,
    "engine_job_id": null,
    "rewrites": []
  }
}
```

Results are relayed from the engine on each poll, never held by orchd
(`engines/sdcpp.md` § "Images and results"). A poll that returns a completed
job's result marks it fetched, which is what releases the model to its idle
TTL (`04-admission.md`).

### Job-aware clients

A job-aware client runs its own queue and records exactly what it sent. Magic
Lantern is the first. Such a client depends on these rules:

| Rule | Where |
|---|---|
| Introspection never loads | `GET /models/{id}` above |
| Job ids are ULIDs, never reused, even across restarts | `02-architecture.md` |
| A job that fails with a null `orchd.submitted_at` never reached the engine | Errors, below |
| Async jobs are exempt from `queue_timeout`; a yield fails them before start | `04-admission.md` |
| Unfetched results hold the model | `04-admission.md` |
| A reservation is released when the engine stops, not when the client leaves | `adr/0006` |
| A value the request sets is never rewritten | above |
| Events resume from `Last-Event-ID` | `GET /events` above |

Breaking one of these is a breaking change to the API, however small the code
change.

## Errors

A single error shape across both APIs, carrying a machine-readable reason:

```json
{"error":{"code":"never_fits","message":"model 'glm-air' needs 41200 MiB; device capacity minus headroom is 30720 MiB","model":"glm-air"}}
```

Codes: `unknown_model`, `never_fits`, `insufficient_budget`, `queue_timeout`,
`yielded`, `load_failed`, `engine_unavailable`, `upstream_error`,
`invalid_definition`, `definition_not_managed`, `device_unavailable`.

These codes are part of the contract; the console branches on them.

For jobs, the reliable test of whether work reached the engine is
`orchd.submitted_at`, not the code.

- A job that fails while it is null never reached the engine, whatever its
  code, and its client may submit it again without generating twice.
- `yielded`, `queue_timeout` and `never_fits` always arise there.
- `upstream_error` and `engine_unavailable` may come after.
