# Experimental Pancake modinv buffer

This isolated consumer uses Flapjack's actual `panValueFfiExtCall`, `callFfi`,
and production RISC-V aligned-cell byte access model. Flapjack is pinned to
`034bb5a1ed0b757082768d5406206bd2f8e31b7c`; Lean is 4.33.1. Its Mathlib
dependency stays outside the parent Mathlib-free package.

From this directory:

```sh
lake build Op6Pancake
lake env lean --run AxiomSweep.lean
```

Lake resolves the parent package through `../..`. The checked-in manifest pins
all transitive dependencies. A cold checkout must download and build those
packages; local build artifacts are not required source inputs.

`modinv-v1` takes empty configuration and a 24-byte array: little-endian RV64
words `a`, `p`, `w` at offsets 0, 8, 16. The raw stateful handler returns exactly
eight bytes. The adapter preserves the first sixteen bytes and replaces the
last eight; it does not certify the witness. Malformed replies fail before
write-back. `incorrect_candidate_transported` and `incorrect_candidate_rejected`
show this separation for witness 5 with inputs 3 and 17. `malformed_call`
checks failure through the real FFI boundary.

`demo_runs` kernel checks a complete concrete memory call at address `0x100`:
input `(3,17,0)` becomes `(3,17,6)` and the host count becomes 1.
`pancake_extCall` proves the symbolic call, including the exact recorded event.
`pancake_write_frame` proves that Flapjack's writer preserves every aligned
memory cell outside its footprint (`Avoid`), including failed-store rollback.
`pancake_checked_call` combines this boundary and frame with soundness of the
15-instruction RV64 checker. The checker requires `p>1`, `a,w<2^32`, `w<p`, and
`a*w mod p=1`; the range checks exclude multiplication overflow.

The parent `Logic/ModInvHints.lean` additionally connects canonical bytes to
`MachineState.readBytes`, original input words, and installation of the witness
in actual RV64 memory. The checker initializer loads registers from these words
as a mathematical boundary operation. A compiled three-LD call stub, Pancake
compiler/linker simulation, Sail/platform interception ABI, and zkVM backend
refinement are separate obligations. The concrete demo and symbolic call do
not prove those refinements. No fixed evm.fast Challenge contract is changed or
claimed proved; this is an experimental ordinary-FFI integration.
