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
repeated/adaptive questions. `HintHandlers.recording_preserves_execution` now proves preservation of the
entire guest control for stateful recording, using `HandlerProjection`,
`drive_projection`, and `execute_projection`. `replay`, `replay_match`,
`replay_mismatch`, and `replay_missing` give typed exact-question replay with
explicit errors and non-consuming retries. `generatedTranscript` follows the
authoritative scheduler. `generated_replay_preserves` proves exact replay of
that transcript and consumption up to an arbitrary remaining suffix, including
repeated/adaptive requests and every terminal/suspended control outcome.
Connecting this generated list to the recording adapter's final list remains.

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

1. **Interfaces, options and theorem statements:** operational and safety API
   implemented above. `ModInvHints` defines an experimental 24-byte version-1
   buffer (three little-endian words a, p, w), empty configuration, and eight-byte
   raw replies. The checker restricts a and w to 32 bits to prove RV64 MUL cannot
   overflow; p > 1 and w < p are checked. No instruction ABI number is claimed.
2. **Executable request/suspend/answer/resume with malformed data and footprint:**
   generic operational path implemented. New `ModInvHints.accept` decodes raw
   replies without mathematical checking; `tick` runs the RISC-V checker as an
   explicitly costed 15-instruction macrostep. Malformed replies trap.
   `replaceBuffer_same_length` proves the array length contract. Full Pancake
   evaluator binding and byte-memory bridge remain. `validBuffer` and
   `checkedCompleteResponse` now validate the entire aligned, non-wrapping
   address footprint before any write. The build includes checked invalid-size,
   unaligned, zero and overflowing-address examples.
3. **Independent contract and checker proof:** existing `IsCanonicalModInv` is
   reused; the old natural-number `fp_inv_hint_sound` remains available. `Contract`, `check_sound`, `check_complete`,
   `checker_stepN`, `checker_flag`, `guest_checker_sound`, and
   `guest_checker_complete` connect the concrete 15-instruction RISC-V checker
   to canonical modular inversion. Codec inverse `decode_encode` is proved.
   `install_frame`, `install_byte_frame`, `install_state_frame` establish exact
   replacement effects; `checker_memory_frame` shows checking performs no writes.
   `protocol_safe`, `protocol_adversarial_sound`, `honest_path`,
   `malformed_reply_traps`, `noninvertible_cannot_accept`, and
   `accepted_call_frame` are kernel checked in the Rv64 library build.
   `example_inverse` supplies a non-vacuous instance (3 inverse mod 17 is 6).
4. **Repeated/adaptive and recording/replay, error/boundary cases:** repeated
   interactions covered by `adversarial_stateful_sound`; recording preservation
   and replay matching/error cases are kernel checked in `HintHandlers`.
   `generated_replay_preserves` is kernel checked; identifying its generated
   transcript with the recording adapter output and actual Pancake boundary
   checking remain.
5. **Pancake buffer wrapper and reproducibility:** inspected
   `op4-stateless-pancaketh/Guest/AccelFfi.lean` and `docs/ACCEL-FFI.md`.
   Convention: empty configuration, self-contained fixed-offset operands,
   same-length array replacement, no writes outside array. `ModInvHints.buffer`,
   `replaceBuffer`, `replaceBuffer_canonical`, and `replaceBuffer_same_length`
   implement this byte-level convention. Actual Pancake companion remains; the
   prepared consumer has empty `flapjack`/`evm-asm` submodule directories and
   no `.lake` dependencies, so evaluator checking needs a separate setup.
   No proof under the fixed Challenge contract is claimed.
6. **Execution connection and remaining obligations:** `rvTick` / `stepN_lift`
   connect ordinary execution to this repository's model. Concrete hint ABI,
   buffer/register loading glue, Pancake evaluator, compiler/linker, Sail platform and
   backend refinements remain; ECALL trapping/CSR semantics alone do not supply
   them. No Sail undefined-value choices are used as hints.
7. **Repository/downstream checks:** focused kernel checks and
   `turbo-lean build RiscvZkvm.Rv64` passed. Model pin, forbidden-tactic and
   unimported-module gates passed (104 modules reachable, 2 allowlisted).
   Full five-library build is still live in generated `Sail.RvfiDii` (session
   23219; never restarted). The axiom sweep passed for the committed interface:
   3751 declarations, only the seven documented axioms. A fresh expanded sweep
   is running in session 59913. Downstream validation remains.

## Inspection notes

The prepared `op2-riscv-lean435/docs` currently contains no hint-interface
assessment; do not assume the requested assessment is present there.
No generated Sail files, dependency pins or toolchain pins have been edited.
No Maxwell/research-governor records have been submitted.

## Audit coverage correction

The axiom gate now explicitly imports the `RiscvZkvm.Rv64` root; importing
only equivalence and logic did not guarantee that newly added optional base
modules were in the audit environment. The documented seven-element allowlist
is unchanged. A fresh sweep after this correction is required.
