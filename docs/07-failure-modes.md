# 07 — Failure modes

Specified, not discovered. Each entry: the signal, the response, the user-visible
result.

## 1. Over-commit with a hard failure (Linux, or Windows with sysmem fallback disabled)

**Signal.** Child exits non-zero; stderr matches a CUDA allocation-failure
pattern. The supervisor's stderr ring buffer carries the evidence.

**Response.** Mark the model `failed_oom`. Multiply its envelope by
`oom_backoff_factor` (default 1.25), persist to the calibration cache with
`oom_backoff_applied` incremented, release the reservation, requeue the request
once. On a second failure, do not requeue: reject with `load_failed` and the
stderr tail.

**Result.** Self-correcting without operator involvement, and the corrected
envelope survives a restart.

## 2. Over-commit with silent paging (Windows, default driver settings)

The dangerous one. WDDM pages GPU memory to system RAM rather than failing, so
nothing crashes and everything crawls — health checks green, ten-to-fifty-fold
slowdown.

**Signal.** The throughput watchdog. Calibration establishes a solo baseline
per model (tokens/s for text, seconds/step for image). The watchdog compares a
rolling window against baseline; a sustained drop below `spill_threshold`
(default 0.4 of baseline) for `spill_window` (default 20 s) raises a spill
alarm.

Text sources: the `timings` block on every completion, `/slots`, `/metrics`.
Image source: job wall-time against the calibrated per-step estimate.

**Response.** Identical to an OOM: widen the envelope, mark the ledger
over-committed, evict by policy, log the event with both baselines. Report the
alarm in `/status` and on `/events`.

**Also.** Detect and report NVIDIA's per-application "CUDA — Sysmem Fallback
Policy" (driver 536.40+). Setting it to prefer no fallback for the engine
binaries converts this failure back into case 1, and that is the documented
recommended Windows configuration. `/orchd/v1/device` reports whether orchd
could determine the setting; if it could not, say `unknown` rather than
guessing.

**Caveat.** A low throughput reading is not proof of paging — a concurrent
image job legitimately slows text generation. The watchdog must therefore
compare against the baseline *for the current concurrency state*, not the solo
baseline, or it will cry wolf every time the two engines overlap. Maintaining
two baselines per model (solo and concurrent) is the minimum; see
`11-open-questions.md` on whether the threshold can be model-independent.

## 3. Start timeout

**Signal.** No `/health` 200 (llama) or `/sdcpp/v1/capabilities` 200 (sd) within
`start_timeout`.

**Response.** Kill the process group / job object, release the reservation, mark
`failed_start`, record the stderr tail. Do not retry automatically — a timeout
usually means a bad path or a bad flag, and retrying wastes minutes.

**Note.** Windows anti-malware scanning can add tens of seconds to first start
of a large binary or model file; hence the larger platform default.

## 4. Crash mid-request

**Signal.** Child exit event while requests are in flight.

**Response.** 502 to affected clients with the stderr tail. Release the
reservation. Restart with exponential backoff (1 s, 2 s, 4 s … capped at 60 s),
circuit-break after 5 failures in 10 minutes and mark the model `unavailable`
until a reload or an explicit load.

## 5. Client disconnect

**Signal.** Request context cancelled.

**Response.**

- **Text:** close the upstream connection, which frees the slot at once.
- **Image:** cancel the upstream job, then release the reservation when the
  engine reports it ended. That is immediate for a job still queued in the
  engine, and at the end of the job if the build cannot cancel a generating
  one (`adr/0006`).

Neither waits for the next scheduled poll. A held reservation for abandoned
work is a ledger leak, and in a queueing system it is a deadlock waiting to
happen. But releasing memory the engine is still using is the worse error,
because the next admission is then made against a fiction.

## 6. Upstream queue full (sd-server 429)

**Signal.** `429` from `POST /sdcpp/v1/img_gen`.

**Response.** Treat as an orchd bug: orchd configures the engine's queue bound
and should never exceed it. Log at error, keep the job in orchd's own queue,
retry on the next completion event. Backpressure belongs in orchd's queue,
where jobs can be reordered and cancelled, not in the engine's FIFO where they
cannot.

## 7. Orphans after an orchd restart

**Signal.** Children from a previous orchd generation still running.

**Response.** Platform-specific, and Windows is the best case:

| Platform | Mechanism |
|---|---|
| Windows | Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` — children die with orchd automatically |
| Linux | `PR_SET_PDEATHSIG` on the child — same effect, free |
| macOS | Neither exists. Children carry an env marker and a generation id; on startup, scan and terminate anything bearing a prior generation before rebuilding the ledger |

The macOS scan runs on all platforms as a belt-and-braces check, because a
child that escaped its job object or pdeathsig is exactly the case that corrupts
the ledger silently.

orchd's job store is in memory, so a restart also forgets every job. Their ids
answer 404 afterwards and are never reissued, which a job-aware client reads
as lost work.

## 8. Ledger drift

**Signal.** Reconciliation finds `total_used` diverging from the ledger's
expectation beyond `reconcile_tolerance_mib`.

**Response.** Log with every per-model attribution. At the next quiescent
moment, reset `foreign` from the exact reading. Sustained drift means the
envelopes are wrong; it is the headline metric for finding that out, and it
belongs in `/metrics`.

## 9. Device unavailable

**Signal.** NVML load fails, or the device disappears (driver reset, eGPU
unplug).

**Response.** Enter `device_unavailable`. Stop admitting; do not evict
immediately, since a driver reset may resolve. After `device_grace` (default
60 s), drain everything. Return `device_unavailable` to clients. Recover
automatically when the probe succeeds again, rebuilding the ledger from a
quiescent reading.

A build with no GPU present starts normally and reports this state. That is a
supported configuration, not an error — it is how the test suite and the
console development loop run.

## 10. Config error on reload

**Signal.** Parse or validation failure.

**Response.** Keep the running configuration. Report the error on the API and
on `/events` with the file and line. Never take the service down over a bad
edit — the console will be writing to this directory, and a round trip through
a dead service is a terrible failure mode.

## 11. Unfetched results at eviction

**Signal.** An image model is due for eviction, by idle TTL or by demand,
while a completed job's result has not been fetched.

**Response.** sd-server keeps results only in memory, so killing the process
destroys them.

- **Idle TTL:** the model is not idle until every result is fetched or the
  engine's retention window (600 s) has passed.
- **Demand eviction:** wait up to `unfetched_result_hold`, then evict. The
  job's record answers 410 from then on, as the engine's own purged jobs do,
  and the discard is logged with the job id.

**Result.** A client that polls promptly never loses a result to orchd's
housekeeping. One that has gone away costs at most a bounded wait.
