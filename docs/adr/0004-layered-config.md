# ADR 0004 — Layered configuration with an ownership boundary

**Status.** Accepted.

## Context

The first design assumed a single hand-edited YAML file. The console changes
that: configuration acquires a second author, a web form, writing to the same
tree a human maintains with comments, ordering and intent.

Single-file approaches both fail. Rewriting the file destroys comments and
formatting on every save. Preserving them requires comment-aware YAML
round-tripping, which is a permanent tax and still loses arguments about
ordering.

## Decision

Split by ownership, one model per file.

```
orchd.yaml          human-owned; orchd reads, never writes
models.d/*.yaml     one model per file, each carrying source: managed | file
loras.d/*.yaml      same rule
```

`source: managed` means the management API and the console may rewrite that
file. `source: file` means hand-authored: `PUT` and `DELETE` return 409, and
the console may display and offer to copy it into a managed definition, never
edit it in place.

Precedence: defaults → `orchd.yaml` → `models.d/` → flags. A model defined in
both `orchd.yaml` and `models.d/` is an error at load, not a silent merge.

## Rationale

The ownership boundary is explicit and mechanically enforced, so neither author
can surprise the other. A human who writes a definition by hand keeps it
exactly as written, forever, and still sees it in the console.

One file per model has benefits beyond the ownership question:

- A working configuration becomes something you can post in a gist or paste
  into an issue.
- "Export this model" is a file copy; "import" is a file drop plus a reload.
- Reload can diff by definition hash and leave unchanged models completely
  untouched — process, reservation and calibration record intact — which is
  what makes config editing safe while models are serving.

## Consequences

- Reload must be non-destructive and must survive invalid input: on any
  validation error, keep the running configuration and report. With the console
  writing to this directory, a bad edit taking the service down would be a
  terrible failure mode.
- Writes are atomic: temp file in the same directory, fsync, rename. The
  filesystem watcher debounces to avoid reacting to a half-written tree.
- Validation is strict and early. Unknown keys are errors. Referenced files
  must exist. Contradictory combinations — `tuning.ctx` pinned together with
  `tuning.fit: true` — are errors with an explanation, because silently
  ignoring one produces a model that is mysteriously the wrong size.
- `extra_args` is never rewritten by anything, ever. Every abstraction over
  engine flags leaks eventually; an inescapable one is worse than none.
