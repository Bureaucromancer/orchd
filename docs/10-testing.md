# 10 — Acceptance scenarios

Thirty-four scenarios, mapped to milestones. These are integration-level: they
run against a real orchd with real or faked engines. Admission *unit* tests are
separate and must run without a GPU.

## Harness requirements

- **Fake probe.** A scripted `DeviceProbe` returning a programmed sequence of
  `TotalUsed` readings. Every admission and eviction scenario must be runnable
  against it, on any machine, in CI.
- **Fake engines.** Small binaries that mimic `llama-server` and `sd-server`
  health, job and streaming semantics, with programmable load time, memory
  claim, failure injection and throughput. Most scenarios run against these.
- **Synthetic foreign consumer.** A tool that allocates and holds a specified
  amount of device memory on demand, so foreign-pressure and yield scenarios
  are reproducible in CI rather than requiring someone to launch a game. This
  is required for Windows acceptance and is the one piece of the harness that
  needs a real GPU.
- **Real-engine suite.** A smaller subset run against actual llama-server and
  sd-server on a GPU host, nightly rather than per-commit.

---

## M1 — lifecycle basics

**S1** Cold start, streaming. Request to an unloaded model streams tokens; SSE
keepalives are observed during the load window; total time to first token is
within load time plus a small constant.

**S2** Cold start, 30-second-timeout client. A client with a 30 s read timeout
survives a 60 s load.

**S3** Idle TTL. A loaded model with no traffic unloads after
`idle_ttl_relaxed`; the process is gone and the port is released.

**S4** Child death mid-stream. `kill -9` during generation produces a 502 with
diagnostic text, the reservation is released, and the next request reloads
successfully.

## M2 — the ledger

**S5** Attribution accuracy. After loading two models, per-model attributions
sum to within tolerance of the device-wide delta.

**S6** Quiescent re-anchor. With all models unloaded, `foreign` equals
`total_used` exactly, and a previously drifted estimate is corrected.

**S7** Foreign transient. A 2 GB foreign allocation lasting less than the
hysteresis window does not change admission behaviour.

**S8** Foreign sustained. A 4 GB foreign allocation held past the hysteresis
window reduces the budget and is reflected in `/status` within one poll
interval plus the hysteresis.

## M3 — the image engine

**S9** Image generation, OpenAI shape. A `POST /v1/images/generations` returns
a valid image; internally the native async endpoint was used.

**S10** Passthrough job lifecycle. A client submits via passthrough, polls, and
sees orchd's `queued_admission` state before the engine's `queued`.

**S11** Cancel on disconnect. Client disconnects mid-generation; an upstream
cancel is observed within 1 s and the reservation is released.

**S12** Explicit cancel. `POST /orchd/v1/jobs/{id}/cancel` cancels both a
queued and a generating job.

**S13** Calibration stability. Three calibration runs of the same model at the
same settings produce peaks within 5% of each other.

## M4 — arbitration

**S14** Coexistence, the headline case. Text model resident; image job admitted
within the partition. Both complete. No eviction. Text throughput during the
image job is **recorded**.

**S15** Image arrives, must evict. Text model holds the full budget; image job
needs room; text model drains after its in-flight completion, image runs, text
model reloads on the next chat request.

**S16** Text arrives during image job, fits. Served immediately, no queue. This
is the scenario that justifies the project.

**S17** Text arrives during image job, does not fit. Queued; served after the
job completes; within `queue_timeout`.

**S18** Running image job is never preempted. A high-priority text request that
cannot fit queues rather than cancelling the job, even under `interactive`
priority.

**S19** Never-fits rejection. A model whose envelope exceeds capacity minus
headroom is rejected immediately with `never_fits`; it never enters a queue.

**S20** Two image models. Concurrent requests for different image models: the
second queues, the first model is evicted after its job completes, the second
loads.

**S21** Queue timeout. A request blocked past `queue_timeout` gets 503 with a
`Retry-After` derived from the blocking work's estimated remaining time.

**S22** Starvation promotion. A continuous stream of text requests does not
prevent a queued image job from running within `starvation_promote_after`
passes.

**S23** Yield. Foreign usage crosses `yield_threshold`; all evictable models
drain; requests get 503 `yielded`, not queue slots; after
`yield_recovery_delay` below threshold, normal admission resumes.

**S24** Exclusive mode. With `mode: exclusive`, text and image never hold
reservations simultaneously, and switching between them is bounded by drain
plus load time.

## M5 — hardening

**S25** OOM backoff. An induced allocation failure widens the envelope,
persists it, and the retried request succeeds.

**S26** Spill detection. With a fake engine programmed to drop to 20% of
baseline throughput, the watchdog raises a spill alarm within `spill_window`
and the eviction path runs.

**S27** Spill false-positive guard. Legitimate concurrent-load slowdown does
**not** raise a spill alarm, because the comparison uses the concurrent
baseline.

**S28** Orphan reaping. orchd is `kill -9`ed with two children running; on
restart, no orphans survive and the ledger rebuilds from a quiescent reading.

**S29** Job object / pdeathsig. Children die automatically with orchd on
Windows and Linux without the reaping scan being needed.

**S30** Device loss. The probe fails; orchd enters `device_unavailable`, does
not evict within the grace window, recovers cleanly when the probe returns.

**S31** No-GPU start. orchd starts on a machine with no GPU, reports
`device_unavailable`, serves `/status` and `/console`, and rejects inference
requests with a clear reason.

**S32** Circuit breaker. Five consecutive start failures mark the model
unavailable; an explicit load resets it.

**S33** Bad config reload. An invalid edit to `models.d/` is rejected, the
running configuration survives, the error is reported on `/events` with file
and line.

**S34** Unchanged-definition reload. A reload that does not change a model's
hash leaves its process, reservation and calibration untouched.

---

## What is measured rather than asserted

Some outputs are numbers to record, not thresholds to pass. The test suite
writes them to a results file and the roadmap treats changes in them as
signals:

- Text tokens/s solo versus during an image job, per model (S14).
- Image seconds/step solo versus with a text model resident and active.
- Cold-load time per model, warm and cold page cache.
- Calibration peak versus the engine's actual peak during a real job.
- Reconciliation divergence over an hour of mixed load.

Asserting thresholds on these before the real numbers are known would only
encode a guess.
