# 03 — The device model and the ledger

## The finding that shapes this

`nvmlProcessInfo_t.usedGpuMemory` always reports `NVML_VALUE_NOT_AVAILABLE`
under WDDM, because the Windows kernel-mode driver manages the memory rather
than the NVIDIA driver. Any consumer GPU with a display attached must run in
WDDM mode. So on exactly the machine this project targets — a multipurpose
desktop that is also somebody's PC — per-process VRAM attribution through NVML
does not exist.

A second limitation applies on every platform: NVML's compute-process
enumeration returns only processes holding an active CUDA context. Graphics
applications using OpenGL or DirectX are not listed at all. A game, a browser
compositor, a 3D viewport — invisible.

Taken together: per-process telemetry can identify neither the foreign
consumers that matter most, nor, on Windows, orchd's own children. It cannot
be the mechanism.

## The mechanism: delta attribution

Invert the arithmetic. orchd knows what it started; it measures the total; it
derives everything else.

**Quiescent calibration.** Whenever orchd holds zero reservations,
`total_used` *is* `foreign`, exactly. This is a free and exact reading, and on
a desktop it happens often. Every such moment re-anchors the ledger.

**Load-time delta.** Measure `total_used` before spawn and again after the
child reports ready plus a settle window. The difference is that child's cost.
Sample across the settle window; if variance suggests a foreign consumer moved
during the measurement, discard the attribution, fall back to the declared or
previously calibrated envelope with a widened cushion, and flag the reservation
as `estimated` rather than `measured`.

**Release delta.** On eviction, measure before and after the child exits. A
release that does not return approximately what was attributed is a signal
worth logging — it usually means the attribution was wrong, not that memory
leaked.

**Per-process telemetry becomes a cross-check**, where it exists. On Linux it
feeds reconciliation and drift detection for compute processes. Where it does
not exist, nothing breaks.

This is a simplification, not a Windows compromise. It needs only `capacity`
and `total_used`, which every platform provides.

## The DeviceProbe contract

Mirrors the backend capability contract: a small mandatory interface plus
declared optional capabilities.

```go
type DeviceProbe interface {
    Capacity() (bytes uint64, err error)   // total installed device memory
    TotalUsed() (bytes uint64, err error)  // allocated device memory, all consumers
    Describe() DeviceInfo                  // name, driver, index, capability flags
}

// Optional, type-asserted:
type PerProcessReporter interface { PerProcessUsage() (map[int]uint64, error) }
type DriverModelReporter interface { DriverModel() (DriverModel, error) } // WDDM | MCDM | TCC | N/A
type SpillSignaller     interface { SpillSignal() (bool, error) }          // if a platform ever offers one
```

| Platform | Implementation | Capacity / TotalUsed | PerProcess | Notes |
|---|---|---|---|---|
| Linux + NVIDIA | NVML, dynamically loaded | yes | compute processes only | reference implementation |
| Windows + NVIDIA | NVML for totals; PDH `GPU Process Memory` counters optionally for per-process | yes | via PDH, not NVML | WDDM blocks NVML per-process |
| macOS | Metal `recommendedMaxWorkingSetSize`, `currentAllocatedSize`; `iogpu.wired_limit_mb` | different semantics — see below | no | tier 2 |

NVML is loaded dynamically at runtime (`libnvidia-ml.so.1`, `nvml.dll`) so that
a build without a GPU still starts and reports the device as unavailable.

## The ledger

```
free = capacity − foreign − headroom − Σ(owned reservations)
```

- **capacity** — from `Capacity()`, cached; the driver reserves a little for
  bookkeeping, which is already reflected in the totals.
- **foreign** — `total_used − Σ(owned attributions)`, clamped at zero, smoothed
  across `foreign_hysteresis_samples` consecutive polls. Exactly recomputed at
  every quiescent moment.
- **headroom** — never allocated into. Platform-defaulted, larger on Windows
  (see below).
- **owned reservations** — one per loaded or loading model, plus one per
  admitted in-flight image job (the job's peak above its model's resident
  footprint).

Reservations have a state: `estimated` (from config or a widened fallback),
`measured` (from a clean load delta), or `calibrated` (from a deliberate
calibration pass). `/orchd/v1/status` reports which, because an admission
decision made from estimates deserves less confidence than one made from
measurements.

### Reconciliation

Every 30 s, and at every quiescent moment, compare the ledger's expectation of
`total_used` against the actual reading. Divergence beyond a threshold is
logged with the per-model attributions that produced it. Sustained divergence
is a bug in the envelopes, and it is the single most useful metric for finding
one.

## Platform differences that reach beyond the probe

### Windows: no hard failure

WDDM virtualises video memory. When an allocation does not fit, the driver can
page GPU memory to system RAM instead of failing. The consequence: **over-commit
on Windows does not produce an out-of-memory error, it produces a silent
ten-to-fifty-fold slowdown** with every health check still green.

Three consequences, developed in `07-failure-modes.md`:

1. The throughput watchdog is a first-class component on Windows, not a nicety.
   It is the only over-commit signal available.
2. orchd documents and detects NVIDIA's per-application "CUDA — Sysmem Fallback
   Policy" setting (driver 536.40+), which converts silent paging back into a
   catchable allocation failure. With it off, the ledger is the *only* thing
   standing between the user and the slow path.
3. Default headroom and envelope cushions are larger on Windows, because there
   is no backstop to correct a too-small estimate.

### Windows: foreign pressure is the normal case

On a Linux inference host, foreign consumption is a compositor taking 300 MB.
On a gaming desktop it is a title taking 16 GB the instant it launches. The
right response is not to reshuffle reservations but to get out of the way —
see the yield policy in `04-admission.md`.

### macOS: unified memory (tier 2)

- No discrete pool and no meaningful "foreign GPU consumer" — everything
  contends for system RAM.
- The ceiling is the wired-memory limit (`iogpu.wired_limit_mb`), not a device
  capacity; Metal's `recommendedMaxWorkingSetSize` is the number to read.
- The failure mode is neither OOM nor PCIe paging but memory pressure —
  compression and swap. Degradation again, so the watchdog built for Windows
  earns its keep twice.
- Coexistence pressure is simply lower on a large unified-memory machine, which
  is the honest reason to tier it: holding a text model and a diffusion model
  resident at once is often just *possible*.

The code must therefore not assume a discrete pool, a compute-process
enumeration, or a hard allocation failure anywhere outside the platform
package. macOS ships building and running, with admission semantics for
unified memory recorded as an open question rather than answered.

## Platform defaults

| Setting | Linux | Windows | macOS |
|---|---|---|---|
| `headroom_mib` | 1024 | 2048 | 4096 (of wired limit) |
| envelope cushion | 10% | 20% | 20% |
| `start_timeout` | 120 s | 240 s (anti-malware scanning inflates process start) | 120 s |
| spill detection | secondary | **primary** | primary |
