# Operation 6 — work in progress

Base: PR #18 at `06736b268f`; branch `op6-riscv-resumable-hints`.
This report is a progress record, not a completion certificate.

Public API and scope documentation: `docs/resumable-hints.md`; runnable
consumer documentation: `examples/pancake-modinv/README.md`.

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
`recording_transcript` identifies the generated list with the recording adapter's final list,
including an existing prefix. `recorded_replay_preserves` gives direct exact replay of the recorded
output.

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
   `replaceBuffer_same_length` proves the array length contract. Actual Pancake ordinary-FFI binding
   and byte-memory bridge are now checked in the isolated companion and `Logic/ModInvHints.lean`.
   `validBuffer` and
   `checkedCompleteResponse` now validate the entire aligned, non-wrapping
   address footprint before any write. The build includes checked invalid-size,
   unaligned, zero and overflowing-address examples.
3. **Independent contract and checker proof:** existing `IsCanonicalModInv` is
   reused; the old natural-number `fp_inv_hint_sound` remains available. `Contract`, `check_sound`,
   `check_complete`,
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
   `generated_replay_preserves` is kernel checked; `recording_transcript` and
   `recorded_replay_preserves` now connect it to actual recording output. The companion checks the
   real Pancake boundary, including malformed and incorrect replies.
5. **Pancake buffer wrapper and reproducibility:** inspected
   `op4-stateless-pancaketh/Guest/AccelFfi.lean` and `docs/ACCEL-FFI.md`.
   Convention: empty configuration, self-contained fixed-offset operands,
   same-length array replacement, no writes outside array. `ModInvHints.buffer`,
   `replaceBuffer`, `replaceBuffer_canonical`, and `replaceBuffer_same_length`
   implement this byte-level convention. The isolated `examples/pancake-modinv` companion pins
   Flapjack at `034bb5a1ed0b757082768d5406206bd2f8e31b7c` and uses its actual `panValueFfiExtCall`,
   `callFfi`, and RISC-V byte-memory accessor. `pancake_extCall`, `pancake_write_frame`,
   `pancake_checked_call`, and `demo_runs` are kernel checked. See the companion README for commands
   and exact scope.
   `demo_checked_runs` composes actual FFI post-memory reads with `stepN 15`;
   `demo_memory_contract` establishes the inverse contract for the extracted words.
   `checkedCall` composes the actual FFI with post-memory reads and `stepN 15`.
   `checkReadback_sound` and `checkedCall_sound` prove the independent contract
   for the words actually read, for every stateful handler, with no supplied
   q,w or honest-response premise. The symbolic `pancake_checked_call` uses
   supplied q,w for its precise event/frame statement. A generic equality
   between arbitrary post-memory input words and original input words remains
   separate; the concrete demo and core RV64 installation bridge cover their
   respective original-input cases. `checkedCall_demo` proves a successful call
   of the same generic composed function, including read-back words, acceptance
   flag, and updated host state.
   No proof under the fixed Challenge contract is claimed.
6. **Execution connection and remaining obligations:** `rvTick` / `stepN_lift`
   connect ordinary execution to this repository's model. `Logic/ModInvHints.lean` proves
   `readBytes_word`, `readBytes_buffer`, `buffer_memory_replacement`, and
   `installed_memory_checker_sound`: actual RV64 memory bytes, witness installation, unchanged
   original inputs, and checker contract. The companion connects the actual Pancake ordinary FFI.
   Concrete interception ABI, an emitted register-loading call stub, compiler/linker simulation,
   Sail platform and backend refinements remain; ECALL trapping/CSR semantics alone do not supply
   them. No Sail undefined-value choices are used as hints.
7. **Repository/downstream checks:** focused kernel checks and
   `turbo-lean build RiscvZkvm.Rv64` passed. Model pin, forbidden-tactic and
   unimported-module gates passed (105 modules reachable, 2 allowlisted).
   `check-no-warnings.sh` passed (190 jobs; no warnings in hand-owned sources).
   Full five-library build is still live in generated `Sail.RvfiDii` (session
   23219; never restarted). The final expanded core axiom sweep passed with 3945 declarations, only
   the seven documented axioms, and no offenders. The companion audit passed for all 52 declarations
   with the same allowlist and zero offenders. Downstream validation remains: the prepared op2
   consumer is modified for Lean 4.35.0; its source HEAD and the op4 consumer use 4.33.0, while this
   candidate is 4.33.1. No incompatible-toolchain build is reported as validation.
   A fresh isolated consumer at `../op6-evm-asm-consumer` now builds evm-asm
   `7e65e4d02` against this candidate path, Lean 4.33.1, and Mathlib 4.33.1,
   using independent dependency copies and `turbo-lean build --max-memory-mb 12000 EvmAsm`.
   Session 88927 was terminated by the disk guard when available space fell
   below 3 GiB. It produced 64 consumer oleans, but did not complete; this is
   **not** a passing downstream compatibility result. The isolated checkout and
   artifacts are preserved for resumption once disk capacity is available.
   Only its three dependency/toolchain configuration files differ from HEAD.
   The original full Sail build was not interrupted. Disk availability later
   recovered to approximately 60 GiB, but the isolated consumer's `.lake`
   directories had disappeared. Its independent dependencies were restored
   from the verified companion, and a fresh downstream build is live in
   session 86863. No source files were lost; no downstream pass is yet claimed.

## Inspection notes

The prepared `op2-riscv-lean435/docs` currently contains no hint-interface
assessment; do not assume the requested assessment is present there.
No generated Sail files, dependency pins or toolchain pins have been edited.
No Maxwell/research-governor records have been submitted.

## Audit coverage correction

The axiom gate now explicitly imports the `RiscvZkvm.Rv64` root; importing
only equivalence and logic did not guarantee that newly added optional base
modules were in the audit environment. The documented seven-element allowlist
is unchanged. The expanded sweep passed after this correction (3945 declarations).

## Companion setup incident

During dependency setup, a mistaken package-directory symlink let `lake update`
remove the shared op3 Flapjack checkout. It was restored at its recorded exact
revision `034bb5a1ed0b757082768d5406206bd2f8e31b7c`, with matching build
artifacts copied from op1. An op3 import check passed after restoration. The
companion now uses independent package directories; no package symlinks remain.
The incident was reported to the user when discovered.
