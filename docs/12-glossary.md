# 12 — Glossary

Terms used precisely throughout these documents. Where a word has a loose
everyday meaning and a specific meaning here, the specific one wins.

**Admission** — the decision to allow a request to proceed, made on the
lifecycle goroutine against the ledger. Returns `Admitted`, `Queued` or
`Rejected`. Not the same as routing.

**Attribution** — assigning a quantity of device memory to an owned child, by
measuring `TotalUsed` before and after that child's load. The alternative to
per-process telemetry, which is unavailable on WDDM.

**Backend** — an implementation of the engine interface. One backend per
engine, not per model. `llamacpp` and `sdcpp` are the two in this milestone
set.

**Basis** — how a reservation's size was arrived at: `estimated` (config or
fallback), `measured` (a clean load delta), or `calibrated` (a deliberate
calibration pass). Reported in `/status`.

**Budget** — `capacity − foreign − headroom`. What orchd may distribute among
reservations.

**Calibration** — a deliberate measurement pass that establishes a model's peak
envelope and throughput baseline, cached by content hash.

**Capability** — an optional interface a backend or device probe may implement,
type-asserted at runtime. `Sizer`, `Drainable`, `JobBackend`, `Cancelable`,
`ProgressReporter` for backends; `PerProcessReporter`, `DriverModelReporter`
for probes.

**Coexist mode** — the default policy, in which the image engine's envelope is
reserved and the text model sized to fit around it, so neither is evicted for
the other's normal operation. Contrast **exclusive mode**.

**Console** — the configuration web GUI, phase 2. A client of the management
API, shipped embedded in the binary or standalone. Not a generation front-end.

**Drain** — stopping new routing to a model and waiting for in-flight work to
complete, up to `drain_timeout`, before stopping its process. Precedes every
orderly eviction.

**Envelope** — the device memory a model or job is expected to occupy.
*Resident envelope* for a loaded model's steady state; *job envelope* for the
transient peak of an image job above its model's resident footprint.

**Eviction** — draining and stopping a model's process to free its reservation.
Always deliberate; never a consequence of something else.

**Exclusive mode** — the fallback policy, in which only one engine holds
reservations at a time.

**Foreign** — device memory consumed by anything orchd does not own. Derived as
`TotalUsed − Σ owned attributions`, and exactly equal to `TotalUsed` at
quiescence.

**Headroom** — memory never allocated into, held back against measurement error
and transient foreign spikes. Larger on Windows, where there is no hard-failure
backstop.

**Job** — a unit of asynchronous work with an id and a terminal state. Kinds:
`img_gen`, `vid_gen`, `download`, `calibrate`. Image generation is a job even
when the client's request is synchronous.

**Ledger** — the single accounting of device memory. Owned exclusively by the
lifecycle goroutine.

**Lifecycle goroutine** — the single-threaded loop that owns the ledger and
makes every admission and eviction decision. Nothing in it blocks.

**Managed definition** — a model definition whose `source` is `managed`,
meaning the management API and the console may rewrite its file. Contrast a
`file` definition, which is hand-authored and read-only to the API.

**Modality** — `text`, `embedding`, `image`, `video`. A property of a model
definition; determines which client endpoints route to it.

**Native passthrough** — proxying a request to an engine's own API unchanged,
under `/orchd/v1/models/{id}/native/*`, subject to admission.

**Owned** — a process orchd started and supervises. Everything else is foreign.

**Quiescence** — the state in which orchd holds zero reservations, and
therefore `TotalUsed` is exactly `foreign`. Used to re-anchor the ledger.

**Reconciliation** — periodic comparison of the ledger's expectation against
the device's actual reading. Divergence indicates wrong envelopes.

**Reservation** — a claim on budget held by a loaded or loading model, or by an
admitted in-flight job. Created at admission, released at eviction, job
completion, or client disconnect.

**Residency** — a model's eviction class: `resident` (loaded eagerly, evictable
under pressure), `on_demand` (loaded on first use, short TTL), `never_evict`
(never a victim).

**Spill** — the Windows failure mode in which WDDM pages device memory to
system RAM rather than failing an allocation, producing severe slowdown with no
error. Detected by the throughput watchdog.

**Watchdog** — the component that maintains throughput baselines and detects
sustained degradation. The primary over-commit signal on Windows and macOS.

**Yield** — the state entered when foreign memory pressure crosses a threshold:
everything evictable is drained, admission stops, and requests are rejected
rather than queued. The behaviour that makes orchd tolerable on a machine that
is also somebody's PC.
