# Resumable prover hints (experimental)

The optional operational API is in `RiscvZkvm.Rv64.ResumableHints`; the
existing finite-tree interface remains available. It does not change RISC-V
instructions, `MachineState`, `step`, or the existing CPS/WP definitions.

A guest `tick : S → Event S Q A` returns a result, makes ordinary progress,
requests `q : Q` while saving one `S`, or traps. `run tick fuel s` stops at
return, trap, pending request, or fuel exhaustion. Exhaustion retains `s`.
`resume accept q s r` supplies `r : R q` and resumes the guest. It establishes
no mathematical property of that response.

`Control` contains one guest state or one pending request and saved state;
it contains no transcript. This bounds protocol storage structurally, not the
size of caller-chosen types. `Handler H Q R := (q : Q) → H → H × R q` stores
history externally. `drive` schedules one fuel slice or one response;
`execute` repeats that operation a finite number of times. Fuel counts guest
`tick` calls, so any multi-instruction tick must document its instruction cost.

The all-response safety boundary is explicit:

```lean
Safe tick accept I P
```

requires return to establish `P`, ordinary progress to preserve `I`, and **every**
response at a pending request to re-establish `I`. Then
`adversarial_stateful_sound` proves that `execute` preserves `Valid accept I P`
for any handler and arbitrary handler state. On successful return, `Valid`
means `P`. On trap it means `True`: this is partial correctness, not guaranteed
success, termination, or availability of an honest witness.

For ordinary machine execution, `rvTick intercept done` checks interception
first, then completion, then `step`. It saves the current state without retiring
an instruction on interception. `stepN_lift` identifies non-intercepting
`stepN n s = some t` segments with `run ... n s = .exhausted t` under `Ordinary`.
`Logic/ResumableHints.lean` provides `cps_lift` and `wp_lift`, preserving the original frame,
precondition, postcondition, and budget, with explicit non-interception and
non-completion hypotheses along the selected segment. These are theorem names
in namespace `RiscvZkvm.Rv64.ResumableHints`, not changes to the original logic.

`HintHandlers` provides external recording and dependent typed transcripts.
`recording_preserves_execution` erases recording instrumentation;
`recording_transcript` identifies its accumulated output precisely.
`recorded_replay_preserves` reproduces guest execution from that output.
`generated_replay_preserves` also preserves an arbitrary unconsumed suffix.
Missing and mismatched requests have explicit non-consuming replay errors;
the total `replayOr` adapter uses its supplied fallback on those errors.

The [Pancake modinv companion](../examples/pancake-modinv/README.md) checks
actual ordinary-FFI buffer replacement, production memory access, a concrete
post-memory-to-RV64-checker execution, malformed replies, and a memory frame.
`checkedCall_sound` also proves soundness for every stateful handler after
running the actual FFI and checking its read-back words. The contract concerns
those observed words; general preservation of original inputs across arbitrary
Flapjack memory representations remains separate.
Its Mathlib dependency is isolated from this package. The protocol uses a
24-byte little-endian `(a,p,w)` array and an eight-byte raw witness reply.
The guest checks the independent modular-inverse contract with 15 RV64IM
instructions; range checks prevent multiplication overflow.

This is an experimental ordinary-FFI PoC. Compiler/linker simulation, an
emitted register-loading/interception stub, generic Flapjack-to-RV64 memory
refinement, and Sail/platform/backend integration remain separate obligations.
It neither changes nor proves evm.fast's fixed Challenge oracle contract.
