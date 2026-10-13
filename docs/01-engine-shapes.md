# 01 — Engine shapes and the asymmetry

Everything in the scheduler follows from one observation: the two engines have
incompatible resource shapes, and the scheduler exists to reconcile them.

## The two shapes

| | llama.cpp | stable-diffusion.cpp |
|---|---|---|
| Occupancy | continuous and steady-state: weights plus KV cache, held from load to unload | weights resident between jobs, plus a large transient peak during sampling and especially VAE decode |
| Footprint known in advance | yes — from GGUF metadata and KV arithmetic, or by delegating to `--fit` | no — the server reports no memory information at all |
| Latency profile | interactive, streaming, measured in seconds | throughput, measured in tens of seconds to minutes |
| Unit of work | a request against a slot; cheap to interrupt | a job; not checkpointable, so cancelling discards everything |
| Models per process | one | one |
| Runtime unload | none | none |
| Introspection | `/health`, `/props`, `/slots`, `/metrics` | `GET /sdcpp/v1/capabilities`, job polling |
| Queueing | continuous batching across slots | internal FIFO with a configurable bound; returns 429 when full |
| Cancellation | close the connection | explicit for queued jobs; for generating jobs only where the build advertises `cancel_generating` (**SPIKE**) |

A resident steady-state footprint versus a transient peak. That is the whole
problem.

## Why this is hard rather than merely fiddly

If both engines had steady-state footprints, the answer would be a static
partition computed once. If both were job-shaped, the answer would be a queue.
The mixture means:

- The text model's footprint must be chosen with the image model's *peak* in
  mind, not its average, because the peak is what causes the failure.
- The image model's peak cannot be computed from anything the engine exposes,
  so it has to be measured.
- The failure is asymmetric in cost. Evicting an idle text model costs a
  reload. Cancelling a diffusion job at step 25 of 28 costs the entire job.
- The failure is asymmetric in *time*. A completion is bounded by `n_predict`
  and usually finishes in seconds; a diffusion job can run for minutes. So
  "wait for the in-flight work to finish" is a reasonable policy in one
  direction and an unreasonable one in the other.

That last asymmetry is the one that sets eviction policy in `04-admission.md`:
**a running image job is never preempted; a text request either fits in the
partition or queues behind it.**

## Memory coexistence is not performance coexistence

The ledger models device *memory*. It says nothing about streaming-multiprocessor
contention. Two engines that fit in memory simultaneously will still contend
for compute, and a diffusion job running concurrently with token generation
will slow both — possibly a lot.

This is not a problem orchd solves in this milestone set, and pretending
otherwise would be dishonest. What orchd does instead:

1. Measures it. The calibration pass establishes a solo baseline for each
   model; the acceptance tests record degradation under concurrent load as a
   number rather than a feeling.
2. Offers the knob. `policy.image_jobs_block_text` (default `false`)
   serialises image jobs against text requests for anyone who values
   interactive latency over throughput.

Treating compute contention as a scheduling dimension — priority-weighted
thread counts, deferring image jobs during interactive bursts — is a later
design pass. See `11-open-questions.md`.

## What this means for the backend contract

The capability-contract approach already chosen for backends holds up well
here. The mandatory `Backend` interface is small; capabilities are optional and
declared:

| Capability | llama.cpp | sd.cpp | Note |
|---|---|---|---|
| `Sizer` | yes, delegated to `--fit` plus measured correction | yes, by calibration only | two very different implementations of one idea |
| `Drainable` | yes — stop routing, wait for slots to clear | partial — wait for the current job, refuse new | |
| `JobBackend` | no | yes | native async submission with job ids |
| `Cancelable` | by connection close | yes, explicit; generating jobs only where advertised | capabilities report advertises both flags; a reservation is released only when the engine confirms (`adr/0006`) |
| `ProgressReporter` | yes — `return_progress` gives prompt-processing progress | **no** | sd-server gives queue position and coarse state; newer builds add step counts only while a preview is enabled (**SPIKE**) |

sd.cpp exercising `JobBackend` and `Cancelable` while leaving
`ProgressReporter` unimplemented is useful: it proves the optional capabilities
are genuinely optional, and it means the first image backend is not a
degenerate copy of the text one.
