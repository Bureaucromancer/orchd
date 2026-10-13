# 02 — Architecture

## Process model

One orchd process. N child processes, each running one model.

```
                        clients
                           │
                    ┌──────▼──────┐
                    │    orchd    │  :11434
                    │             │
                    │  proxy ─────┼──► llama-server (model A)   127.0.0.1:ephemeral
                    │  lifecycle  │
                    │  ledger ────┼──► llama-server (model B)   127.0.0.1:ephemeral
                    │  supervisor │
                    │             ├──► sd-server    (model C)   127.0.0.1:ephemeral
                    └──────┬──────┘
                           │
                      DeviceProbe ──► NVML / PDH / Metal
```

Children bind ephemeral ports on loopback, never a fixed range, and never a
Unix socket — `llama-server` supports one when the host ends in `.sock`, but
that path does not exist on Windows and the divergence buys nothing.

## Components

**registry** — the model catalogue, derived from config. Each entry carries id,
engine, modality, argv template, residency class, envelope (declared or
calibrated), and a content hash used to detect definition changes on reload.

**device** — the `DeviceProbe` implementations. Mandatory `Capacity()` and
`TotalUsed()`; optional `PerProcessUsage()`, `DriverModel()`, `SpillSignal()`.
See `03-device-model.md`.

**ledger** — the single accounting of device memory:
`free = capacity − foreign − headroom − Σ owned reservations`. Mutated only by
the lifecycle goroutine.

**lifecycle** — one goroutine, one event channel, all admission decisions.
Events: request arrival, load complete, load failed, job complete, health
change, child exit, TTL tick, device poll, config reload, watchdog alarm.
Nothing in this loop blocks.

**loadworkers** — a small pool that performs the actual spawn-and-health-gate
and reports back as an event. Loading a 20 GB model takes seconds to tens of
seconds; doing it inline would stall every other decision.

**supervisor** — child process management. Platform-specific kill semantics
behind one interface (`Start`, `Stop(drain)`, `Kill`, `Wait`). Captures child
stderr into a bounded ring buffer per child for diagnostics. Enforces start
timeouts.

**proxy** — the HTTP front end. Streaming reverse proxy for text with no
buffering; job shim for image.

**jobs** — a generic job store. Image generation is one job kind; model
download and calibration are others. Generic from the start because the
console needs download jobs to look like image jobs, and retrofitting that is
worse than designing for it. Kinds: `img_gen`, `vid_gen`, `download`,
`calibrate`.

**watchdog** — throughput baselines per model and detection of sustained
degradation. On Windows this is the primary over-commit signal, because the
WDDM driver pages rather than failing. See `07-failure-modes.md`.

**api** — handlers for `/v1/*`, `/orchd/v1/*` and the native passthrough.

## Control flow: a text request

```
POST /v1/chat/completions (model=A)
  │
  ├─ api: resolve model id against registry           → 404 if unknown
  ├─ lifecycle: admit(A, text)                        → Admitted | Queued | Rejected
  │     ├─ Admitted + already resident                → proxy immediately
  │     ├─ Admitted + not resident                    → reserve, dispatch load,
  │     │                                               hold request, emit SSE
  │     │                                               keepalives while loading
  │     ├─ Queued                                     → wait, re-enter on event
  │     └─ Rejected                                   → 4xx/503 with a reason
  │
  ├─ proxy: stream upstream → client, no buffering
  └─ on client disconnect: close upstream so the slot frees
```

## Control flow: an image request

The important asymmetry. Even for a synchronous OpenAI-shaped request, orchd
submits to the engine's **native async** endpoint internally.

```
POST /v1/images/generations (model=C)
  │
  ├─ jobs: create orchd job, state queued_admission
  ├─ lifecycle: admit(C, img_gen, params)
  ├─ on Admitted: POST /sdcpp/v1/img_gen  → 202 + upstream job id
  ├─ poll GET /sdcpp/v1/jobs/{id} until terminal
  ├─ on client disconnect: POST /sdcpp/v1/jobs/{id}/cancel, release reservation
  └─ on completion: return OpenAI-shaped response
```

Using the synchronous endpoints would tie up a connection and forfeit job
identity, cancellation and queue visibility. See `adr/0003`.

orchd's job state is a superset of the engine's, because orchd has a state the
engine cannot have — waiting for memory:

```
queued_admission → submitted → queued(upstream) → generating → completed
                                                             ↘ failed
                                                             ↘ cancelled
```

## Threading rules

- The lifecycle goroutine owns the ledger and all child state. Everything else
  reads via snapshots delivered on request, or sends events.
- Request handlers are ordinary goroutines. They send an admission request and
  select on a reply channel plus the request context.
- The device poller runs on its own ticker and sends samples as events. It
  never blocks on a slow NVML call — if a sample is late, the previous one
  stands and staleness is recorded.
- The supervisor's `Wait` goroutines send child-exit events. A child that dies
  is always discovered by event, never by polling.

## Why single-threaded admission

Two concurrent admissions that each independently observe 8 GB free and each
need 6 GB will both succeed and the device will be over-committed. Locking
around the ledger would fix the race but not the harder problem: an admission
decision may trigger an eviction, which changes what a concurrent decision
should have concluded. Serialising the decisions makes each one see a
consistent world, and the cost — a single goroutine handling what are
microsecond-scale decisions — is irrelevant at this scale.
