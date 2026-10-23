# Operation 6 — work in progress

Base: PR #18 at `06736b268f`; branch `op6-riscv-resumable-hints`.
This report is a progress record, not a completion certificate.

## Current verified declarations

`RiscvZkvm/Rv64/ResumableHints.lean` adds an operational interface without
changing `MachineState`, `step`, `stepN`, instruction constructors or the old
finite-tree interface. `Event`, `Control`, `run`, `resume`, `drive`, and `execute`
implement return, progress, pending request, explicit response, trap and fuel
exhaustion. Exhaustion retains the guest state; callers replenish fuel explicitly.
The control state contains no transcript: one saved state and at most one request.
This is a structural bound on protocol storage, not a bound on arbitrary user
supplied types or external handler storage.

`Safe`, `Valid`, `run_safe`, `drive_safe`, and `adversarial_stateful_sound`
separate guest safety from handler correctness. External state `H` is arbitrary,
so it may encode the entire interaction history. Every response must preserve
the guest invariant. Receiving a value never supplies a mathematical certificate.
`execute` covers arbitrarily many bounded slices and responses, including
repeated/adaptive questions. It is not yet a recording/replay refinement proof.

`rvTick` connects ordinary progress directly to the authoritative `step`.
Interception has priority over completion and ordinary stepping; suspension
saves the current state without retiring an instruction. The adapter's supplied
`accept` function determines resumption effects; no concrete ABI is claimed yet.
`Ordinary` and `stepN_lift` establish exact preservation on unaffected segments.
`Logic/ResumableHints.lean` provides `CpsWithin`, `cps_lift`, and `wp_lift`,
retaining the original frame, budget, precondition and postcondition. Their
non-interception hypothesis covers the segment selected by the existing proof.

## Mapping to the seven staged criteria of issue #20

The live issue was read on 2026-10-23. Its seven **staged acceptance criteria**
are broader than the five-item summary in the operation brief.

1. **Interfaces, options and theorem statements:** partially implemented above.
   Concrete codec, completion/progress contracts, replay and refinement remain.
2. **Executable request/suspend/answer/resume with malformed data and footprint:**
   generic operational path implemented; concrete buffer adapter remains.
3. **Independent contract and checker proof:** existing `IsCanonicalModInv` and
   `fp_inv_hint_sound` are reusable. New end-to-end checker connection remains.
4. **Repeated/adaptive and recording/replay, error/boundary cases:** repeated
   interactions covered by `adversarial_stateful_sound`; concrete recording,
   replay matching, error cases and replay preservation remain.
5. **Pancake buffer wrapper and reproducibility:** inspected
   `op4-stateless-pancaketh/Guest/AccelFfi.lean` and `docs/ACCEL-FFI.md`.
   Convention: empty configuration, self-contained fixed-offset operands,
   same-length array replacement, no writes outside array. Companion remains.
   No proof under the fixed Challenge contract is claimed.
6. **Execution connection and remaining obligations:** `rvTick` / `stepN_lift`
   connect ordinary execution to this repository's model. Concrete hint ABI,
   guest checker lowering, Pancake evaluator, compiler/linker, Sail platform and
   backend refinements remain; ECALL trapping/CSR semantics alone do not supply
   them. No Sail undefined-value choices are used as hints.
7. **Repository/downstream checks:** focused kernel checks and
   `turbo-lean build RiscvZkvm.Rv64` passed. Model pin, forbidden-tactic and
   unimported-module gates passed (102 modules reachable, 2 allowlisted).
   Full five-library build is in progress. Axiom sweep and downstream consumer
   validation remain; no final seven-axiom or zero-sorry census is claimed yet.

## Inspection notes

The prepared `op2-riscv-lean435/docs` currently contains no hint-interface
assessment; do not assume the requested assessment is present there.
No generated Sail files, dependency pins or toolchain pins have been edited.
No Maxwell/research-governor records have been submitted.
