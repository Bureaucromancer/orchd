# ADR 0003 — Always submit image work to the engine's async API

**Status.** Accepted.

## Context

`sd-server` exposes three API families. The OpenAI-compatible `/v1/...` and the
A1111-compatible `/sdapi/v1/...` are synchronous from the caller's point of
view. The native `/sdcpp/v1/...` family is asynchronous and job-based:
submission returns 202 with a job id and a poll URL, status is polled, and
cancellation is explicit for both queued and generating jobs.

orchd serves an OpenAI-shaped synchronous endpoint to clients. The obvious
implementation is to proxy straight through to the engine's matching
synchronous endpoint.

## Decision

orchd always submits to `/sdcpp/v1/img_gen` (or `vid_gen`) and polls, even when
the client's request is synchronous. The synchronous families are exposed only
through native passthrough, for clients that ask for them explicitly.

## Rationale

The synchronous path forfeits three things orchd needs:

1. **Job identity.** Without an id there is nothing to cancel, nothing to
   report in `/status`, and nothing to correlate a reservation against.
2. **Cancellation.** A client that disconnects mid-generation must release its
   reservation immediately, or the ledger leaks and the queue can deadlock. On
   the synchronous path the only lever is closing a connection, which does not
   reliably stop the work.
3. **Queue visibility.** `queue_position` lets orchd report why a request is
   waiting, and lets it distinguish waiting-for-memory from waiting-in-engine.

It also lets orchd hold a job *before* submission, in its own
`queued_admission` state, which the engine has no concept of. That state is
what makes cross-engine arbitration expressible to the client at all.

The cost is a poll loop per in-flight job. At the concurrency this system
targets — single digits — that is negligible.

## Consequences

- orchd's job state machine is a superset of the engine's:
  `queued_admission → submitted → queued → generating → terminal`.
- `JobBackend` and `Cancelable` are exercised by the first image backend, while
  `ProgressReporter` is not, since the engine offers no per-step fractions.
  This is a useful early proof that optional capabilities are genuinely
  optional.
- The engine's own queue is configured small (1–2) so backpressure lands in
  orchd's queue where it can be reordered and cancelled. A `429` from the
  engine therefore signals an orchd accounting bug.
- Any future image backend must either offer async submission or be wrapped in
  a shim that provides equivalent identity and cancellation. ComfyUI does offer
  this; a backend that does not would be a poor fit regardless.
