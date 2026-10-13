# 13 — Implementation plan

Where coding starts. This document sequences work; it does not re-specify
design. Every "what" is in docs 00–12 and the ADRs, and where this document
and those disagree, those win.

Read before starting: `00-scope.md`, `01-engine-shapes.md`, `03-device-model.md`,
`04-admission.md`, and `CLAUDE.md` at the repo root for the invariants.

---

## How to use this

Work units are `W0`–`W13`, in dependency order, each with a definition of done
that is checkable without judgement. They are sized to be one commit or one
small PR each. The milestone boundaries from `09-roadmap.md` are unchanged —
this is M1 decomposed, plus the handoff into M2 and the manager.

Two rules carried from `CLAUDE.md` and worth repeating because they are easy to
skip under momentum:

- **The admission path must be testable without a GPU.** If a test for
  scheduling behaviour needs hardware, the abstraction is wrong. This is why
  W6 (the fake engine) comes before the lifecycle loop rather than after it.
- **When implementation contradicts a document, change the document in the same
  commit.** If the contradiction touches an ADR, add a superseding ADR; do not
  edit the old one.

---

## Decisions to make before the first commit

Three, all trivial to change later except the first, which is annoying to
change later.

1. **Module path.** Needs the real repository URL —
   `module github.com/<owner>/orchd`. Everything imports from it, so set it in
   the first commit rather than renaming a hundred import lines in the fifth.
2. **Go version.** 1.23 or newer. Pin in `go.mod` and in CI.
3. **Dependency budget.** Standard library plus `gopkg.in/yaml.v3` is enough
   for M1. The YAML choice is not arbitrary: its `Decoder` has
   `KnownFields(true)`, which is what makes `06-config.md`'s "unknown keys are
   errors" cheap instead of hand-rolled. Resist a router, a logging framework
   and a config framework — `net/http`'s pattern-based mux, `log/slog` and a
   struct are sufficient, and each dependency is a thing to audit before a
   release.

NVML binding is an M2 concern. Do not add a CGo NVML dependency now; the plan
is runtime dynamic loading (`libnvidia-ml.so.1` / `nvml.dll`) so a GPU-less
build still runs.

---

## W0 — Preflight: pin the engines

Before any Go code. The design documents were written from published API
surfaces, not from running binaries, and two of their claims are load-bearing.

1. Build or download `llama-server` and `sd-server`. Record exact commits or
   release tags in `docs/engines/*.md` under a "Pinned build" heading. Every
   later observation is relative to these.
2. Capture `llama-server --help` and `sd-server --help` output into
   `testdata/engines/` as reference. Flags move; a captured baseline makes the
   next surprise diagnosable.
3. Run `llama-server` with a small model and capture, as test fixtures:
   `GET /health` while loading (expect 503) and when ready (expect 200
   `{"status":"ok"}`); `GET /props`; `GET /slots`; one streamed
   `/v1/chat/completions` response including its `timings` block.
4. Run `sd-server` and capture `GET /sdcpp/v1/capabilities`, a
   `POST /sdcpp/v1/img_gen` 202 response, and a job polled through to
   `completed`. **Confirm the known-issue note in `engines/sdcpp.md`** — there
   is an upstream report of `capabilities` returning 500 on some builds while
   the OpenAI and A1111 families work. If it reproduces on the pinned build,
   the readiness gate needs its documented fallback from day one rather than as
   a later fix.
5. Pin `sd-server` to the build Lemonade ships, `master-894-017cc8e` (Sep 22,
   2026), to start with.
   - Magic Lantern's first spike records fixtures from that build, so one set
     serves both projects, and `fake-sd` (W6) is checked against them.
   - While it runs, try cancelling a generating job, and look for step counts
     with a preview enabled (`11-open-questions.md` items 18 and 19).

**Not blocking M1:** the two `--fit` spikes (`11-open-questions.md` items 6 and
7). They block M4, because the `--fit-target` arithmetic in `04-admission.md`
depends on their answers. Worth running early anyway, while a model is loaded
and attention is on the engine — the experiment is twenty minutes and the
answer changes load ordering.

**Done when.** Both engines run from a recorded invocation, fixtures are in
`testdata/`, and the pinned builds are written into the engine docs.

---

## W1 — Repository skeleton

`go.mod`; `cmd/orchd/main.go` that parses flags and exits; the `internal/`
tree from `CLAUDE.md` with a `doc.go` per package stating that package's one
responsibility; `Makefile`; `.golangci.yml`; CI.

Makefile targets: `build`, `test`, `test-race`, `lint`, `fakes`, `acceptance`,
`cross`.

CI matrix: `ubuntu-latest` and `windows-latest`, both running build, vet, lint
and unit tests. Windows is in the matrix from the first commit, not added at
M5 — the whole point of the platform work in `03-device-model.md` is that it
is not a retrofit, and a Windows job that has been green since W1 is how that
stays true. GPU-dependent tests are a separate manually-triggered or nightly
workflow against a self-hosted runner.

**Done when.** `make build test lint` passes on both platforms in CI with
nothing implemented.

---

## W2 — Configuration

The types, loader and validator from `06-config.md`. Layering
(defaults → `orchd.yaml` → `models.d/` → flags), strict decoding, path
existence checks, contradiction checks (`tuning.ctx` pinned together with
`tuning.fit: true` is an error, not a silent precedence rule), and the
`source: managed | file` distinction recorded on every definition.

Ship a subcommand: `orchd config check [--print-effective]`. It parses, reports
errors with file and line, and dumps the merged configuration. This is a real
diagnostic tool, it is the first thing that will be used in anger, and it makes
the config layer testable with golden files before anything else exists.

**Done when.** Table-driven tests cover each precedence rule and each
validation failure; a deliberately broken `models.d/` entry produces an error
naming the file and the key; `--print-effective` output is golden-tested.

---

## W3 — Registry

The catalogue derived from config: id resolution, modality, engine, residency,
priority, resolved argv template, and the content hash used by reload diffing
(`06-config.md`). Hash over the fields that affect behaviour, deliberately
excluding cosmetic ones like `display_name` — an edit to a label must not
restart a model.

**Done when.** Two definitions differing only in `display_name` hash equal; any
change to `model`, `tuning`, `extra_args` or `limits` hashes differently;
duplicate ids across `orchd.yaml` and `models.d/` are rejected at load.

---

## W4 — Event bus and `/orchd/v1/events`

Early on purpose. Everything after this emits into it, and retrofitting an
event stream means revisiting every call site.

A typed internal event with a kind, a timestamp, a model id where applicable
and a small payload; an in-process fan-out to subscribers; a bounded ring of
recent events so a late subscriber sees context; the SSE handler from
`05-api.md`. Event kinds as listed there: `admission`, `load_start`,
`load_complete`, `evict`, `drain`, `child_exit`, `job_state`, `yield`,
`watchdog`, `config_reload`, `ledger_reconcile`, `reset`. Emit the ones that
exist yet; the vocabulary is fixed now because the manager will depend on it
(`adr/0005`). Event ids are `<boot id>:<seq>`. The handler resumes from
`Last-Event-ID`, or sends `reset` when it cannot (`05-api.md`).

**Done when.**
- A subscriber receives events in order with no publisher blocking.
- A slow subscriber is dropped rather than stalling the emitter.
- The SSE endpoint survives a client disconnecting mid-stream.
- A reconnect with a recent `Last-Event-ID` misses nothing; one with a stale
  id gets `reset`.

---

## W5 — Platform package

Everything `runtime.GOOS` touches, in one place, behind one interface.

| Concern | Linux | Windows |
|---|---|---|
| child tree lifetime | `PR_SET_PDEATHSIG` | Job Object with `KILL_ON_JOB_CLOSE` |
| graceful stop | `SIGTERM` | terminate after drain (no signal equivalent) |
| config / state dirs | `$XDG_CONFIG_HOME`, `/etc/orchd` | `%PROGRAMDATA%\orchd` |
| default log destination | file, never assume a console | file, never assume a console |

Drain-then-terminate is the uniform abstraction (`07-failure-modes.md`):
neither engine has persistent state to flush, so a hard terminate after a drain
is correct everywhere, and the signal differences never leak out of this
package.

**Done when.** Killing orchd with `SIGKILL` / `TerminateProcess` leaves no
surviving children on either platform, verified by a test that spawns a sleeper
and checks afterwards.

---

## W6 — Fake engines

`cmd/fake-llama` and, later, `cmd/fake-sd`. Small HTTP servers mimicking the
real engines' observable behaviour, with everything that matters programmable
by flag or environment: load delay before `/health` turns 200, declared memory
claim, token emission rate, `timings` values, failure injection (exit code,
stderr text, hang, flapping health), and a slot model good enough to exercise
draining.

This is infrastructure, not a toy. It is what lets the scheduler's behaviour be
tested deterministically in CI on a machine with no GPU, and the quality of
every later test depends on it. Build it before the lifecycle loop, so the loop
is written against something runnable.

**Done when.** A fake can be driven through: slow load, ready, stream, idle,
drain, exit; and through each injected failure. `make fakes` builds them.

---

## W7 — Supervisor

Spawn with the resolved argv, an ephemeral loopback port, a generation marker in
the child environment, a bounded stderr ring buffer per child, the readiness
gate, drain, stop, and child-exit events.

Two specifics worth settling here rather than discovering:

**Port allocation.** Bind `127.0.0.1:0`, read the assigned port, close, pass it
to the child. There is a race — another process can take the port in the gap.
It is narrow, and the mitigation is to detect the child failing to bind and
retry with a fresh port up to a small limit. State the race in a comment; do
not pretend it is absent.

**Readiness gating.** Three distinguishable states, and conflating them
produces bad diagnostics: connection refused (process up, listener not yet
bound), 503 (listener up, model loading), 200 (ready). Poll with backoff, treat
only 200 as ready, and report which state the timeout expired in.

**Done when.** Scenario S4 passes against a fake: `kill -9` mid-stream produces
a child-exit event, a 502 carrying stderr, and a clean reload on the next
request.

---

## W8 — Lifecycle loop

The single goroutine from `02-architecture.md`. One event channel, all state
transitions, load dispatch to workers, the idle-TTL ticker.

For M1 the admission function is a stub: a concurrency limit of one loaded
model, no ledger, no eviction policy beyond "evict the incumbent if a different
model is wanted". **Give it the real signature now** — `admit(req) → Admitted |
Queued(reason) | Rejected(reason)` — so that M4 replaces a body rather than
rewriting call sites, and so the queueing and rejection paths through the proxy
are exercised from the start.

Nothing in the loop blocks. No spawns, no HTTP, no disk. Enforce it with a
review habit and, if it helps, a lint comment on the function.

**Done when.** S3 passes (idle TTL unloads); requesting a second model evicts
the first after drain; a test with a deliberately blocking handler is caught by
a watchdog assertion on loop latency.

---

## W9 — Proxy and cold-start transparency

Streaming reverse proxy with no buffering. During a load or a queue wait, emit
SSE keepalive comments plus the named `progress` frames from `05-api.md`, with
`state` distinguishing `loading` from `queued`. On client disconnect, close the
upstream connection.

**Done when.** S1 and S2 pass: a cold start streams, and a client with a 30 s
read timeout survives a 60 s load (test it with a fake configured for a 60 s
load and a client with an explicit 30 s timeout — this is the scenario that
makes or breaks first impressions, and it is easy to believe it works without
having checked).

---

## W10 — HTTP API

`/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/models`;
`/orchd/v1/status` with the fields that exist at M1; `/orchd/v1/models` and
`/orchd/v1/models/{id}`, which never load anything; `load` / `unload`; `/orchd/v1/config` and
`config/reload`; native passthrough; `/healthz`.

Implement the full error vocabulary from `05-api.md` now, even where some codes
cannot yet occur. The manager branches on these codes (`adr/0005`), and a code
added later is a breaking change to a client that was written against its
absence.

Model ids resolve against orchd's catalogue only, never passed through for the
engine to resolve.

**Done when.**
- An unknown model id returns 404 with the available ids.
- Every error path returns the documented shape.
- Passthrough reaches `/props` on a loaded model.
- `GET /orchd/v1/models/{id}` on an unloaded model starts no process.

---

## W11 — Job store skeleton

The generic store from `02-architecture.md`: a `Job` with id, kind, state,
timestamps, result, error; kinds `img_gen`, `vid_gen`, `download`, `calibrate`
declared but only the store implemented; `/orchd/v1/jobs` and `jobs/{id}` and
`jobs/{id}/cancel` wired to it.

Two details are fixed now because clients depend on them (`05-api.md` §
"Job-aware clients"):
- Ids are ULIDs, never reused, even across restarts.
- A job carries `submitted_at` from the first version, so a failure before
  submission is distinguishable from one after.

Generic from the start is a deliberate cost paid now. `download` and
`calibrate` are the manager's progress surfaces, and bolting them onto an
image-specific store later is the retrofit `adr/0005` exists to avoid.

**Done when.** A synthetic job can be created, listed, polled and cancelled
through the API with no engine involved.

---

## W12 — Observability

`log/slog` configured per platform (file by default, never assuming a console);
every lifecycle transition logged at info with the model id, the decision and
the numbers behind it; `/metrics` with the counters that exist.

The log is an interface, not a diary. `07-failure-modes.md` and the manager
both assume a decision can be reconstructed from it.

**Done when.** A full load → serve → idle → unload cycle is reconstructible
from the log alone, and `/events` and the log agree.

---

## W13 — M1 acceptance

S1–S4 from `10-testing.md` green against the fakes in CI on both platforms, and
run by hand against the real pinned `llama-server` on the GPU box. Record the
measured numbers named in `10-testing.md` — cold load times, tokens per second
— in `testresults/`. They are a baseline, not an assertion.

---

## First light

The shortest path to something running, for morale and for flushing out
integration problems early: **W1 → W2 (minimal) → W6 → W7 → W8 (stub
admission) → W9**, with the API reduced to `/v1/chat/completions` alone.

That is a request arriving at orchd, a fake engine being spawned on demand, and
tokens streaming back. Everything after it is filling in. Reaching it before
W3–W5 are complete is fine; just do not let the stubs calcify — W4's event bus
in particular gets harder the longer the call sites exist without it.

---

## After M1: the image slice

`09-roadmap.md` puts the image half of M3 next, ahead of M2, because Magic
Lantern needs it. Its work units are not decomposed here yet. When they are,
they follow the same pattern as M1:

- `fake-sd` comes before the backend.
- Job interception and `GET /models/{id}` (`05-api.md`) come before any client
  is pointed at orchd.
- S35–S40 join S9–S12 as the definition of done.

---

## Flowing into M2

M2 is the ledger (`03-device-model.md`), and it is a self-contained addition if
M1 leaves it room:

- `DeviceProbe` as an interface from its first use, with the scripted fake
  landing in the same PR as the real NVML implementation. Not afterwards: the
  fake is what makes M4 testable, and a probe written without one tends to
  acquire assumptions only real hardware satisfies.
- The stub admission function in W8 replaced body-only.
- `/orchd/v1/status` and `/device` gain the ledger fields; the shapes already
  exist.
- Delta attribution, quiescent re-anchoring and reconciliation per `adr/0002`.
  Per-process telemetry is a cross-check, never the mechanism — on Windows it
  does not exist at all.

## Flowing into the manager

The manager is a separate tool above orchd (`adr/0005`, and the suite framing
in `00-scope.md`): it consumes orchd's API, and orchd's own embedded console
remains the component-level surface. Nothing about it requires orchd to change
shape — provided M1 and M2 land these five, which they do if the work units
above are followed:

1. `GET /orchd/v1/events` with a fixed event vocabulary — W4.
2. Machine-readable error codes on every failure path — W10.
3. Generic job kinds, so `download` and `calibrate` are ordinary — W11.
4. `models.d/` with enforced `managed` / `file` ownership — W2.
5. Non-destructive diff-based reload that survives invalid input — W2, with the
   reload path exercised in M1 even though little depends on it yet.

The sixth, `POST /orchd/v1/models/validate`, arrives with M4, because a
fit preview is meaningless before admission arithmetic is real. Keep the
admission function free of side effects so that endpoint is a call rather than
a reimplementation.

Component boundaries are HTTP with a configurable base URL, always, including
when everything is on one machine. Loopback is fast enough, and co-location is
never a different code path.

---

## Working agreements

- One work unit per PR where practical. A PR that touches the lifecycle loop
  and the config loader is two PRs.
- Tests in the same commit as the code. The fakes make this cheap; there is no
  excuse of "needs hardware" for anything above the device package.
- Both platforms green before merge. A unit that only passes on Linux is not
  done (`09-roadmap.md`).
- Update the affected document in the same commit as the behaviour, and add a
  superseding ADR rather than editing a settled one.
- `11-open-questions.md` is live. Resolving an item means editing that file in
  the commit that resolves it; discovering a new one means adding it rather
  than choosing silently.
