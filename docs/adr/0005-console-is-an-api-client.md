# ADR 0005 — The console is an API client, shipped two ways

**Status.** Accepted.

## Context

The configuration GUI was framed as "either a plugin or a separate application
running alongside". Both have obvious costs. A separate application needs its
own install, its own port, its own authentication, and write access to orchd's
configuration — which means either a config-write API anyway, or two processes
contending over the same files. A built-in UI puts a web asset pipeline into
the core binary's build.

## Decision

Treat the configuration surface as an **API**, and the console as one client of
it.

- The management API in `05-api.md` is specified as a product surface, with
  stable shapes and machine-readable error codes, independent of any UI.
- The console is a static single-page application built once and shipped two
  ways from the same artefact: **embedded** in the orchd binary via `embed` and
  served at `/console` when `console.enabled` (the default), or **standalone**,
  served by anything and pointed at an orchd base URL.
- Nothing in the SPA assumes same-origin beyond a configurable base URL.

## Rationale

The question dissolves once the real work is located. The engineering is the
API; the delivery mode is a packaging flag.

Embedded-by-default is right because the user who wants a GUI is precisely the
user who does not want a second install step. One binary, no extra port, no
cross-process contention over the config tree.

Standalone matters for the licence stance: the API is public either way, so a
third party can write a better console — or a different kind of client — without
asking permission. Open in architecture, not only in file headers.

## Consequences

Six requirements land on the core *before* the console starts, and are already
reflected in docs 00–07:

1. Layered config with a `managed` / `file` ownership distinction (ADR 0004).
2. Non-destructive, diff-based reload that survives invalid input.
3. Management API with stable shapes and error codes.
4. `GET /orchd/v1/events` from M1, with no consumer until C1.
5. Generic job kinds from the first job implementation, so `download` and
   `calibrate` are not special cases later.
6. `POST /models/validate` — admission arithmetic callable as a side-effect-free
   what-if. This falls out of well-factored admission and is painful if
   admission is entangled with mutation.

Also:

- A contributor must be able to build orchd without a Node toolchain. The built
  bundle is checked in; `make console` regenerates it.
- The console must refuse to serve on a non-loopback bind without an API key
  configured. The management API can rewrite configuration and start processes;
  exposing it to a LAN by accident is the obvious foot-gun.
- The console is explicitly **not** a generation front-end — see the boundary
  table in `08-console.md`. Its registry endpoints are exactly what a
  generation UI needs to populate its own pickers, so the two compose rather
  than overlap.
