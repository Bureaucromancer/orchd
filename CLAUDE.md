# Working in this repository

Instructions for a coding agent. Read this before touching anything.

## State

Design documents only. There is no Go module, no source tree, no tests. The
first implementation task is to create them, following `docs/09-roadmap.md`.

## Read first

`docs/00-scope.md`, `docs/01-engine-shapes.md`, `docs/03-device-model.md`,
`docs/04-admission.md`. Everything else assumes the reasoning in those four.
The ADRs in `docs/adr/` record decisions that are settled; do not relitigate
them in code without raising it.

## Non-negotiables

These are the invariants the design rests on. Breaking one silently is worse
than failing to implement a feature.

1. **orchd owns process lifecycle.** Engine-internal residency behaviour is
   disabled, always: no llama-server router mode, no autoload, no
   `--sleep-idle-seconds`. One process, one model. See `adr/0001`.
2. **All admission decisions happen on one goroutine.** The lifecycle loop is
   single-threaded and owns the ledger. Nothing else mutates reservations.
   Loads are dispatched to workers and report back by event.
3. **No blocking work on the lifecycle goroutine.** No process spawns, no HTTP
   calls, no disk I/O. If a decision needs data, it was gathered beforehand.
4. **The ledger is the truth about device memory.** If a mechanism would free
   or consume device memory without the ledger knowing, that mechanism is not
   used.
5. **Never reproduce a child's state from a stale read.** Child state changes
   only by event from the supervisor.
6. **Platform differences live in one package.** `internal/platform` and
   `internal/device`. No `runtime.GOOS` checks scattered through the
   scheduler.

## Proposed layout

Nothing is built yet; this is the intended shape, not a constraint to defend.

```
cmd/orchd/            main, flag parsing, signal handling
internal/config/      load, merge, validate, watch; models.d handling
internal/registry/    model catalogue, definitions, hashing
internal/device/      DeviceProbe implementations (nvml, pdh, metal)
internal/ledger/      capacity, reservations, foreign attribution
internal/lifecycle/   the single-goroutine loop; admission and eviction
internal/supervisor/  child process spawn, health gate, kill, reap
internal/backend/     Backend interface + llamacpp, sdcpp implementations
internal/jobs/        generic job store (image, download, calibration)
internal/proxy/       HTTP front end, streaming, SSE
internal/api/         /v1 and /orchd/v1 handlers
internal/watchdog/    throughput baselines and spill detection
```

## Style

- Go 1.23+. Standard library first. No framework.
- No cgo where it can be avoided. NVML is loaded dynamically at runtime
  (`nvml.dll` / `libnvidia-ml.so.1`), so a build without a GPU still runs and
  reports an unavailable device rather than failing to link.
- Errors wrap with context and are returned, not logged-and-swallowed.
- Structured logging (`log/slog`). Every lifecycle transition logs at info
  with the model id, the decision, and the ledger numbers that produced it.
  This is the main debugging surface; treat it as an interface.
- Tests: table-driven. The admission logic must be testable without a GPU —
  the ledger and lifecycle take a `DeviceProbe` interface, and the test suite
  supplies a scripted fake. **If a change to admission requires a GPU to
  test, the abstraction is wrong.**

## What "done" means

Each milestone in `docs/09-roadmap.md` has a definition of done tied to
scenarios in `docs/10-testing.md`. A milestone is complete when its scenarios
pass on both Linux and Windows. Do not mark a milestone done on one platform.

## When the documents are wrong

They will be. They were written from the engines' published API surfaces, not
from a running system. When implementation contradicts a document, change the
document in the same commit, and if the contradiction touches a decision in
`docs/adr/`, add a new ADR superseding the old one rather than editing it.

Items already known to be unverified are collected in `docs/11-open-questions.md`
and marked **SPIKE** in the text. Resolve those before building on them.
