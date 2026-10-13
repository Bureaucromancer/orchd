# ADR 0006 — A job's reservation is released when the engine stops, not when the client leaves

**Status.** Accepted. Supersedes the word "immediately" in `adr/0003`,
rationale point 2, and the disconnect rules in `05-api.md` and
`07-failure-modes.md` §5 that followed from it. The rest of `adr/0003` stands.

## Context

`adr/0003` argues for async submission partly because a client that
disconnects mid-generation "must release its reservation immediately, or the
ledger leaks and the queue can deadlock". `05-api.md` and
`07-failure-modes.md` turned that into a rule: on disconnect, cancel upstream
and release the reservation.

Two facts break the "immediately":

1. **Cancelling a generating job may not be possible.** The two readings of
   sd-server's API disagree. One finds a `cancel_generating` feature flag; the
   other finds a `409` for generating jobs (`11-open-questions.md` item 18). On
   a build that refuses, the job runs to the end, using the memory its
   reservation stood for.
2. **Even an accepted cancel takes effect at the engine's pace,** not orchd's.
   Until the engine reports the job cancelled, the device memory is still in
   use.

Releasing the reservation on disconnect therefore lets the next admission
spend memory that is still allocated. On Linux that is an OOM; on Windows it
is the silent paging collapse the ledger exists to prevent.

## Decision

A job reservation is released only on evidence that the engine has stopped:

- the engine reports the job terminal (completed, failed or cancelled); or
- the engine process exits.

It is never released on a client disconnect alone, and never on a local
timeout.

On disconnect, orchd still cancels upstream at once and marks its own job
cancelled. A job still queued in the engine has not begun its peak, so its
reservation is released as soon as the engine confirms the cancel. A
generating job that cannot be cancelled keeps its reservation until it ends,
and `/status` shows that reservation as `cancelling`.

## Rationale

This is invariant 4 from `CLAUDE.md` — the ledger is the truth about device
memory — applied to the release path. `adr/0003` wrote that path with the
leak in mind, and not the opposite error. A leak costs throughput until the
job ends; an early release costs the device. Between the two, holding is the
safe side, and the hold is bounded by the job's own runtime.

The same rule appears independently in Magic Lantern's job engine, orchd's
first job-aware client: a local timeout never frees a slot while the GPU may
still be busy.

## Consequences

- S11 is reworded: the reservation is released when the engine confirms the
  job has stopped, never before.
- `/status` gains a reservation state, `cancelling`: cancelled by orchd, still
  running on the engine. A wait it causes is explained by `blocked_by` like
  any other.
- The deadlock `adr/0003` worried about does not come back. The held
  reservation ends when the job does, and nothing waits on the client.
- If an engine ever cancels instantly and reliably, nothing changes: the
  confirmation simply arrives sooner.
