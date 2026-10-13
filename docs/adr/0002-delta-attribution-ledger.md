# ADR 0002 — Delta attribution, not per-process telemetry

**Status.** Accepted. Supersedes the earlier "total-used minus owned PIDs"
formulation.

## Context

The original ledger design computed foreign usage as device total minus the sum
of orchd's own processes' usage, read per-PID from NVML.

Two findings break it:

1. `nvmlProcessInfo_t.usedGpuMemory` always reports `NVML_VALUE_NOT_AVAILABLE`
   under WDDM, because the Windows kernel-mode driver manages the memory rather
   than the NVIDIA driver. Any consumer GPU with a display attached must run in
   WDDM mode. With Windows a co-equal platform, the owned side of the
   subtraction is unavailable on the machine the project most targets.
2. NVML's compute-process enumeration lists only processes with an active CUDA
   context. Graphics applications — a game, a compositor, a 3D viewport — are
   invisible on *every* platform. These are exactly the foreign consumers that
   matter on a multipurpose desktop.

## Decision

Derive owned usage by measuring change, not by querying processes.

- **Quiescent calibration.** When orchd holds zero reservations, `total_used`
  *is* `foreign`, exactly. Re-anchor there.
- **Load-time delta.** Measure `total_used` before spawn and after readiness
  plus a settle window; the difference is that child's cost. If variance across
  the settle window suggests a foreign consumer moved, discard and fall back to
  a widened declared envelope, flagged `estimated`.
- **Release delta** on eviction, as a consistency check.
- **Per-process telemetry becomes an optional cross-check** where it exists,
  feeding reconciliation and drift detection only.

The device probe therefore needs only `Capacity()` and `TotalUsed()` as
mandatory operations, with per-process reporting as a declared optional
capability — mirroring the backend capability contract.

## Rationale

The minimal mandatory surface is available everywhere: NVML on Linux and
Windows, Metal on macOS. Making the portable mechanism primary and the
platform-specific one advisory means Windows is a first-class citizen rather
than a degraded one, and it removes a dependency rather than adding one.

The exactness of the quiescent reading is the part that makes this work. A pure
delta scheme would accumulate error indefinitely; an occasional exact
observation bounds it, and on a desktop those observations are frequent.

## Consequences

- Attribution is wrong when a foreign consumer changes during a load window.
  Mitigated by variance detection and by reconciliation, not eliminated.
- Reservations carry a `basis` (`estimated` / `measured` / `calibrated`) so that
  confidence is visible in `/status` and in the console.
- Reconciliation divergence becomes the headline health metric: sustained drift
  means the envelopes are wrong.
- The ledger and admission logic depend only on the `DeviceProbe` interface, so
  the whole scheduler is testable with a scripted fake on a machine with no GPU.
  That property is worth more than the telemetry it replaces.
