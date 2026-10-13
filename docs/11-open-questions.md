# 11 — Open questions

Enumerated rather than guessed. An implementer who hits one of these should
raise it, not pick an answer silently. Items marked **SPIKE** should be
resolved by a small experiment before code depends on them.

## Carried forward from the design set

1. **Queue policy.** FIFO per priority class is specified. Batch-by-model
   reordering is deferred, not rejected — it is the right optimisation and it
   makes queue behaviour much harder to explain. Revisit after real usage.
2. **Default listen port.** 11434 is specified, matching the de facto local
   convention. It also guarantees a conflict with an existing Ollama install,
   which may be a feature (drop-in replacement) or a nuisance. Keep, with a
   clear error on bind failure naming the likely cause.
3. **Embeddings as a modality.** Resolved in favour of "a text-engine model
   with its own process", because `--embedding` is a process-level flag on
   `llama-server` and cannot be shared with a chat process. This makes the
   one-model-one-process rule load-bearing rather than incidental.
4. **Windows support.** Resolved: co-equal with Linux. See `03-device-model.md`.
5. Remaining items from the earlier design set that this milestone set does not
   touch — multi-device gateway re-scoping, ComfyUI capability mapping — stay
   open and out of scope here.

## Raised by the coexistence design

6. **SPIKE — does `--fit` size against free or total device memory?** And does
   it re-check after other allocations? If it targets *free* memory at load
   time, load ordering matters: the image engine must be resident before the
   text model is launched. If it targets total capacity, the computed
   `--fit-target` must include everything else by hand. Everything in
   `04-admission.md`'s `--fit-target` arithmetic depends on the answer.

7. **SPIKE — does `--fit` interact predictably with pinned arguments?** It
   adjusts *unset* arguments. Confirm the exact set it will touch, and confirm
   that pinning `-c` disables context adjustment but not layer adjustment (or
   whatever the real behaviour is). `06-config.md` validates against pinning
   plus fit, and that validation should match reality.

8. **Compute contention as a scheduling dimension.** The ledger models memory
   only. Two engines that fit together still contend for SMs. This milestone
   set offers a boolean (`image_jobs_block_text`) and a measurement. The real
   design — priority-weighted thread allocation, deferring image jobs during
   interactive bursts, possibly MPS — is a later pass. Do not build a
   half-version of it opportunistically.

9. **Is the spill threshold model-independent?** The watchdog compares
   throughput against baseline. MoE and dense models degrade differently under
   PCIe paging, and quantisation changes the shape again. A single global
   `spill_threshold` may be too blunt. Start global, record per-model observed
   degradation, revisit.

10. **Should yield be predictive rather than reactive?** A large foreign
    allocation is usually preceded by a process launch orchd could notice.
    Reactive for now; do not paint the hook into a corner.

11. **VAE tiling default.** `auto` is specified — tiling forced on when the
    untiled envelope will not fit the partition. The speed cost of tiling at
    various tile sizes is not measured. If it turns out to be large, `auto`
    should prefer evicting the text model over tiling, which inverts the
    current preference.

## Raised by the console

12. **Companion-file inference for diffusion models.** How far should automatic
    matching go? Filename convention is weak; file size and tensor-shape
    inspection is stronger but means reading safetensors headers in orchd.
    Decide before C2 whether the console does shallow inspection (fast, often
    wrong, always overridable) or deep inspection (slow, usually right).

13. **Where do presets live?** `presets.d/` as first-class config files, or
    inside a model definition? Separate files compose better across models and
    are shareable; nested is simpler. Leaning separate, undecided.

14. **Does the console need its own auth?** The core has bearer-token
    passthrough only and binds loopback by default. The moment someone binds
    `0.0.0.0` to use the console from a laptop, the management API — which can
    rewrite configuration and start processes — is exposed. At minimum C1 must
    refuse to serve the console on a non-loopback bind without an API key set.
    Whether that is sufficient is open.

15. **Hugging Face authentication for gated repos.** Token storage, and where.
    Probably defer past C3.

## macOS

16. **Admission semantics for unified memory.** With no discrete pool, no
    foreign GPU consumer in the usual sense, and pressure rather than failure,
    what does a "reservation" mean? Candidate answer: budget against the wired
    limit and treat swap activity as the spill signal. Unvalidated. macOS ships
    tier 2 until this is answered.

17. **Does the wired limit need to be read, set, or merely respected?** orchd
    should certainly not change a system sysctl on its own. Whether it should
    *recommend* one, as it does with the Windows sysmem fallback policy, is
    open.
