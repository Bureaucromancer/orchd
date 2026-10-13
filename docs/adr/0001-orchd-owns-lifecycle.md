# ADR 0001 — orchd owns process lifecycle; engine autonomics are disabled

**Status.** Accepted.

## Context

Both target engines have grown their own residency management.

`llama-server` now has a router mode — `--models-dir`, `--models-preset`,
`--models-max N` (default 4), `--models-autoload` (default on) — which loads
models on demand and caps how many are resident. It also has
`--sleep-idle-seconds`, which puts the server to sleep after a period of
idleness, with sleep state visible through `/props`.

On its face this is helpful: orchd could run one router process and let it
manage the text side.

## Decision

orchd does not use any of it. One process, one model, spawned and killed by
orchd. Router mode off, autoload off, idle sleep disabled. The same principle
applies to any future engine: **if a mechanism would free or consume device
memory without the ledger knowing, orchd does not use it.**

## Rationale

The ledger is the only thing that makes cross-engine arbitration possible. An
engine that loads a model on its own initiative consumes budget orchd did not
allocate; an engine that sleeps on its own schedule frees budget orchd cannot
re-allocate, and then wakes without passing an admission check. Either way the
ledger becomes a fiction, and every decision built on it becomes a guess.

This is the same failure already identified in ComfyUI, where an internal
scheduler contends with external memory management. Recognising it in a
cooperative engine is easier than recognising it in a hostile one, and the rule
is the same.

There is also a scope argument. Router mode solves intra-engine multi-model
residency, which is the part of the problem `llama-swap` already solved. It
does nothing about the image engine, because it cannot know the image engine
exists. Adopting it would mean orchd arbitrating between its own ledger and the
router's policy — two schedulers, one device.

## Consequences

- orchd implements process-per-model supervision itself, including health
  gating, drain and kill. That work was always in scope.
- orchd cannot benefit from improvements to router mode without revisiting this
  decision. Acceptable: the benefit is duplicated functionality.
- A user may still put these flags in `extra_args`. orchd logs a warning naming
  the flag and the invariant it breaks, and proceeds. The escape hatch stays
  open; it is signposted.
- If an engine ever exposes *observable* residency — an event stream or a
  queryable per-model memory state orchd could subscribe to — this decision is
  worth reopening, because the objection is to invisibility, not to delegation.

## Note on what orchd does adopt

`--fit` / `--fit-target` is the opposite case and is used enthusiastically. It
does not take autonomous action over time; it is a one-shot sizing computation
at load, driven by a number orchd supplies, with a result orchd then measures.
Delegation of arithmetic is fine. Delegation of policy is not.
