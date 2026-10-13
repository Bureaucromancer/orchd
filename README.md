# orchd

A resource-aware model lifecycle orchestrator for local AI inference.

orchd runs inference engines as child processes, decides which models may be
resident on a device at any moment, and routes OpenAI-compatible requests to
them. Its distinguishing feature is **cross-engine arbitration**: it knows that
a resident language model and an in-flight image generation are competing for
the same video memory, and it schedules them against a single budget.

This repository contains the design documents for the first milestone set. No
code yet — start with `docs/00-scope.md`.

---

## The problem this exists for

A single machine with one GPU, running a chat model and an image model at the
same time. Neither `llama.cpp` nor `stable-diffusion.cpp` knows the other
exists. Each will happily allocate until the device is full. The usual answers
are to run one at a time, to over-provision, or to hand-tune flags until the
numbers happen to fit, and then to re-tune them every time a model changes.

orchd's answer is a ledger: one accounting of device memory, one admission
decision point, and process lifecycle owned by the orchestrator rather than by
the engines.

## Scope of this milestone set

| | |
|---|---|
| **Engines** | llama.cpp (`llama-server`), stable-diffusion.cpp (`sd-server`) |
| **Devices** | one NVIDIA GPU |
| **Platforms** | Linux and Windows co-equal; macOS tier 2 |
| **Language** | Go |
| **Interface** | OpenAI-compatible client API, `/orchd/v1/` management API, per-model native passthrough |
| **Phase 2** | the console — a web GUI for model, LoRA and engine configuration |

Explicitly out: multi-device, multi-GPU, ComfyUI, vLLM, ROCm, the gateway and
chat-UI packages of the wider suite.

## Document map

```
docs/
  00-scope.md            what is and is not being built, and why this order
  01-engine-shapes.md    why the two engines conflict; the asymmetry that drives everything
  02-architecture.md     processes, components, control flow
  03-device-model.md     the DeviceProbe contract, the ledger, platform differences
  04-admission.md        admission, eviction, queueing, yield
  05-api.md              client API, management API, passthrough
  06-config.md           layered configuration and the model definition schema
  07-failure-modes.md    what goes wrong and what orchd does about it
  08-console.md          the configuration GUI (phase 2)
  09-roadmap.md          milestones M1–M5, C1–C3, with definitions of done
  10-testing.md          acceptance scenario catalogue
  11-open-questions.md   decisions deliberately not made yet
  12-glossary.md         terms used precisely throughout
  13-implementation-plan.md  work units for M1, and what flows into the next milestones
  engines/
    llamacpp.md          verified API surface and integration notes
    sdcpp.md             verified API surface and integration notes
  adr/
    0001 … 0006          architecture decision records
```

Read `00`, `01`, `03` and `04` before writing any code. They contain the
reasoning that the rest of the documents assume.

## Licence

MIT. The project commits to support-and-services commercialisation: no
relicensing to more restrictive terms, no feature gating, no open-core split.
Every feature described in these documents ships under this licence or does
not ship.

The design documents were first published under Apache 2.0 and moved to MIT
before any code was written.
