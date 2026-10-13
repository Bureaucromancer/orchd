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
- Image: `POST /sdcpp/v1/jobs/{id}/cancel` upstream, release the reservation,
  mark the orchd job cancelled.

Both are required, not optimisations: a disconnected client whose reservation
is still held is a leak in the ledger.

## Management API

All under `/orchd/v1/`.

| Endpoint | Purpose |
|---|---|
| `GET /status` | the whole picture — see below |
| `GET /models` | catalogue with state, envelope, provenance |
| `GET /models/{id}` | one entry, with full resolved argv and calibration record |
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
| `GET /events` | SSE stream of lifecycle transitions |
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

### `GET /events`

SSE, one event per lifecycle transition, from the same stream the lifecycle
goroutine emits internally. Event types: `admission`, `load_start`,
`load_complete`, `evict`, `drain`, `child_exit`, `job_state`, `yield`,
`watchdog`, `config_reload`, `ledger_reconcile`. The console subscribes to this
rather than polling; so should anything else that wants to be live.

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

Passthrough is subject to admission but not to translation: orchd does not
rewrite native request bodies, with one exception — see `engines/sdcpp.md` on
forcing VAE tiling when the coexist partition requires it. That rewrite is
logged and reported in the job record.

## Errors

A single error shape across both APIs, carrying a machine-readable reason:

```json
{"error":{"code":"never_fits","message":"model 'glm-air' needs 41200 MiB; device capacity minus headroom is 30720 MiB","model":"glm-air"}}
```

Codes: `unknown_model`, `never_fits`, `insufficient_budget`, `queue_timeout`,
`yielded`, `load_failed`, `engine_unavailable`, `upstream_error`,
`invalid_definition`, `definition_not_managed`, `device_unavailable`.

These codes are part of the contract; the console branches on them.
