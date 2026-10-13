# 04 — Admission, eviction, queueing

## Two modes

**Coexist** (default). The budget is partitioned. The image model's peak
envelope is reserved *before* the text model is sized, and the text model is
launched with `--fit-target` set to leave that envelope free. Both engines stay
up; neither is evicted; an image job runs against space it already owns. The
cost is paid in context length or quantisation headroom on the text side.

**Exclusive** (fallback). One engine at a time owns the device. The resident
model is drained and killed; the other loads. This is `llama-swap` behaviour
extended across modalities. It is what admission falls back to, not what it
aims for.

The MVP implements coexist as policy and exclusive as the fallback path. That
fallback *is* the resource-aware core; everything else is plumbing.

## The algorithm

Runs on the lifecycle goroutine. Pure function of ledger state plus request.

```
admit(req) -> Admitted | Queued(reason) | Rejected(reason)

need  = req.kind == text ? resident_envelope(model)
                         : job_envelope(model, params)

if need > capacity - headroom:              return Rejected(never_fits)

free = capacity - foreign - headroom - Σ owned
if need <= free:              reserve;      return Admitted

if yielded:                                 return Rejected(yielded)

victims = evictable(owned, policy_order)
if free + Σ victims.size >= need:
    if blocked_by_inflight(victims):        return Queued(waiting_on_inflight)
    evict(victims); reserve;                return Admitted

return Queued(insufficient_budget)
```

`Rejected(never_fits)` returning immediately is non-negotiable. A request that
can never be satisfied must never sit in a queue; it gets a 4xx naming the
model, the envelope, and the capacity.

## Eviction policy

In order, with the asymmetry from `01-engine-shapes.md`:

1. **A running image job is never preempted.** No checkpointing exists;
   cancelling discards the whole job. Text requests either fit in the partition
   — which is what coexist mode is for — or queue behind it.
2. **An idle text model is always evictable**, subject to drain: stop routing,
   wait for in-flight completions up to `drain_timeout`, then stop the process.
3. **A text model with an in-flight completion is not evicted mid-stream**; the
   image job queues. Safe precisely because completions are bounded by
   `n_predict` and measured in seconds, against a diffusion job's minutes.
4. **Within a class, evict longest-idle first.**
5. `residency: never_evict` models are never victims. They can still be
   rejected at admission if what remains cannot fit the request.
6. **A model holding an unfetched result is not idle.** sd-server keeps a
   finished job's result in memory for its retention window (600 s), and the
   result dies with the process.
   - Idle TTL does not start until every result has been fetched or that
     window has passed.
   - Demand eviction waits at most `unfetched_result_hold`, then proceeds and
     logs the discarded job (`07-failure-modes.md` §11).

### Idle TTL

Two TTLs rather than a continuous pressure curve for this milestone set:
`idle_ttl_relaxed` normally, `idle_ttl_pressured` once device utilisation
crosses `pressure_threshold`. The continuous pressure-scaled TTL is a later
refinement — demand-driven eviction already does most of the work, and a model
that nothing is asking for costs only the memory it is holding.

Image models default to a short TTL (`120s`) because image use is bursty and
diffusion weights are expensive to hold; a warm-cache reload at the front of a
minute-long job is acceptable. The clock starts when the last result is
fetched, not when the last job finishes (rule 6). Text models default to
resident.

## Queueing

One FIFO per priority class. Two classes in this milestone set:
`interactive` and `batch`, assigned per model in config, overridable per
request by an `X-Orchd-Priority` header.

- A queued request is re-evaluated on every event that could change the answer:
  job completion, load completion, eviction, device poll crossing a threshold.
- `queue_timeout` bounds the wait of a request that holds a connection. On
  expiry it gets 503 with `Retry-After` derived from the estimated time for
  the blocking work to finish, not a constant.
- An async job submitted through passthrough is exempt. Its client polls and
  can cancel, so nothing is left hanging, and the job waits in
  `queued_admission` until it is admitted.
- Queue position is visible: `GET /orchd/v1/status` and, for image jobs, the
  job record.
- Starvation guard: a request that has been passed over `N` times is promoted
  ahead of its class. Without this, a stream of small text requests can keep a
  large image job waiting indefinitely.

Batch-by-model reordering — grouping queued requests for the same model to
amortise a load — is deliberately **not** in this milestone set. It is the
right optimisation and the wrong first move, because it makes queue behaviour
much harder to explain and the explanation is half the product.

## Foreign-pressure yield

On a multipurpose desktop, a large foreign allocation is not an edge case to
schedule around; it is a signal to stop.

```yaml
policy:
  yield_on_foreign_pressure: true
  yield_threshold: 0.5          # foreign > 50% of capacity
  yield_recovery_delay: 60s     # sustained below threshold before readmitting
```

When smoothed foreign usage crosses the threshold:

1. Drain and evict every model whose residency is not `never_evict`.
2. Enter `yielded` state; stop admitting.
3. Requests during a yield get 503 with `Retry-After`, **not** a queue slot. A
   game session lasts hours; a two-hour queue is a bug. Async jobs already
   waiting for admission fail with `yielded` and a null `submitted_at`, so a
   job-aware client can hold them in its own queue instead.
4. Leave `yielded` only after foreign usage stays below the threshold for
   `yield_recovery_delay`.

`/orchd/v1/status` reports `yielded` with the reading that caused it. This is
the behaviour that makes orchd tolerable on a machine that is also somebody's
PC, and it costs almost nothing: it is the existing eviction path with a
different trigger.

## Worked example

32 GB device, Windows defaults, coexist mode.

```
capacity                                32768
− headroom (windows default)             2048
− foreign (desktop, smoothed)            1400
                                        ─────
budget                                  29320

image model (Flux Q8, calibrated peak)  13900   reserved while loaded
                                        ─────
available to text                       15420

llama-server launched with --fit-target (13900 + 2048 + 1400) = 17348
  → llama.cpp trims ctx and offload to fit, lands at ~15100 measured
```

An image job arrives: its envelope is already reserved, so admission is
immediate and nothing is evicted. A second image model is requested: its
envelope does not fit, the first image model is idle, so it is evicted and the
second loads. The text model is untouched throughout — which is the point.
