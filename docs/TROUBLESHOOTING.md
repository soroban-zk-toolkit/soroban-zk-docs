# Troubleshooting Guide: Common Proof Generation Failures

This guide covers the most common errors encountered when generating ZK proofs with snarkjs and Circom, along with their root causes and fixes.

---

## Error Index

1. [Constraint Not Satisfied](#1-constraint-not-satisfied)
2. [Wrong Witness / Signal Value Mismatch](#2-wrong-witness--signal-value-mismatch)
3. [WASM Memory Limit Exceeded](#3-wasm-memory-limit-exceeded)
4. [Invalid zkey / R1CS Mismatch](#4-invalid-zkey--r1cs-mismatch)
5. [Witness Generation Hangs](#5-witness-generation-hangs)
6. [Proof Verification Fails](#6-proof-verification-fails)
7. [Trusted Setup Size Too Small](#7-trusted-setup-size-too-small)
8. [Field Overflow / BigInt Out of Range](#8-field-overflow--bigint-out-of-range)
9. [snarkjs Version Incompatibility](#9-snarkjs-version-incompatibility)
10. [Signal Not Constrained Warning](#10-signal-not-constrained-warning)

---

## 1. Constraint Not Satisfied

**Error message:**
```
Error: Assert Failed. Error in template MyCircuit_0 line: 42
```
or
```
[ERROR] snarkJS: ... Constraint 127 does not satisfy
```

**Cause:**
An assertion (`===`) in your circuit is not satisfied by the witness values. This usually means the input values are incorrect or the circuit logic has a bug.

**Fix:**
1. Print intermediate signal values in your JavaScript witness generation to trace where the mismatch occurs.
2. Check that your input values are in the same field as the circuit (BN254 prime: `21888242871839275222246405745257275088548364400416034343698204186575808495617`).
3. Ensure you are not passing JavaScript `Number` types for large values — always use `BigInt` or strings.

```js
// WRONG
const input = { a: 12345678901234567890 }; // precision loss

// CORRECT
const input = { a: "12345678901234567890" };
```

4. Run the witness calculator with verbose output:
```bash
node build/my_circuit_js/generate_witness.js \
  build/my_circuit_js/my_circuit.wasm \
  input.json witness.wtns 2>&1 | head -50
```

---

## 2. Wrong Witness / Signal Value Mismatch

**Error message:**
```
Error: Not all inputs have been set
```
or
```
Error: Signal X already set
```

**Cause:**
- Missing required input signals in `input.json`
- Input signal names in JSON do not match signal names in the circuit
- A signal is set twice (conflicting template instances)

**Fix:**
1. Run `snarkjs r1cs print build/my_circuit.r1cs build/my_circuit.sym` to list all expected input signals.
2. Confirm every listed input signal appears in your `input.json` with the exact same name (case-sensitive).
3. For arrays, provide them as JSON arrays:
```json
{
  "pathElements": ["123", "456", "789"],
  "pathIndices": [0, 1, 0]
}
```

---

## 3. WASM Memory Limit Exceeded

**Error message:**
```
RuntimeError: memory access out of bounds
```
or
```
RangeError: WebAssembly.Memory: could not allocate memory
```

**Cause:**
The WASM witness calculator ran out of memory. This happens for large circuits (>2^18 constraints) or in browser environments with constrained heap.

**Fix (Node.js):**
```bash
# Increase Node.js heap size (e.g., 8 GB)
node --max-old-space-size=8192 build/my_circuit_js/generate_witness.js ...
```

**Fix (snarkjs prove command):**
```bash
# Use the native prover instead of WASM for large circuits
snarkjs groth16 prove build/circuit_final.zkey build/witness.wtns \
  build/proof.json build/public.json --wasm=false
```

**Fix (browser):**
- Split the circuit into smaller sub-circuits (circuit decomposition).
- Use a Web Worker with increased memory allocation.
- Move large proof generation to a backend service; only do verification in the browser.

---

## 4. Invalid zkey / R1CS Mismatch

**Error message:**
```
Error: zkey curve does not match
```
or
```
Error: circuit hash mismatch. Expected X, got Y
```

**Cause:**
The `zkey` file was generated from a different version of the circuit than the one currently compiled. Any change to the circuit (even a comment change in some versions) invalidates the existing zkey.

**Fix:**
1. Recompile the circuit:
```bash
circom circuits/my_circuit.circom --r1cs --wasm --sym -o build/
```
2. Redo Phase 2 of the trusted setup:
```bash
snarkjs groth16 setup build/my_circuit.r1cs pot12_final.ptau build/circuit_0000.zkey
# ... contributions ...
snarkjs zkey export verificationkey build/circuit_final.zkey build/verification_key.json
```
3. Keep the `.r1cs` file in version control alongside the `.zkey` and use content-hash verification to detect mismatches early.

---

## 5. Witness Generation Hangs

**Symptom:**
`generate_witness.js` runs indefinitely and never produces `witness.wtns`.

**Cause:**
- An infinite loop in the circuit (e.g., a `for` loop with a variable bound that snarkjs cannot resolve at compile time)
- Extremely large circuits where witness generation legitimately takes minutes
- Missing input causing the calculator to stall

**Fix:**
1. Add a timeout and capture the process:
```bash
timeout 120 node build/my_circuit_js/generate_witness.js ... || echo "timed out"
```
2. Check that all template parameters are compile-time constants, not signals.
3. Use `snarkjs r1cs info` to confirm the number of constraints is reasonable (>10M constraints will take many minutes).
4. Profile with Node.js `--prof` flag to identify the hot loop.

---

## 6. Proof Verification Fails

**Error message:**
```
[INFO]  snarkJS: INVALID
```

**Cause (most common):**
- Public inputs passed to `verify` do not match those used during `prove`
- The verification key does not correspond to the proving key used
- The proof was generated with a different circuit version than the verification key

**Diagnosis:**
```bash
# Check that public signals match
cat build/public.json

# Verify the zkey and verification key are consistent
snarkjs zkey verify build/my_circuit.r1cs pot12_final.ptau build/circuit_final.zkey
```

**Fix:**
1. Always export `public.json` alongside `proof.json` and use both together for verification.
2. When changing the circuit, regenerate both the proving key AND the verification key.
3. Confirm the verification key on-chain matches the one exported from the zkey.

---

## 7. Trusted Setup Size Too Small

**Error message:**
```
Error: Powers of Tau is not large enough for this circuit
```

**Cause:**
The Powers of Tau file supports fewer constraints than the circuit requires. For example, `pot12_final.ptau` supports 2^12 = 4096 constraints but the circuit has 10,000.

**Fix:**
Download (or generate) a larger Powers of Tau file:

| Constraint count | Required ptau size | File |
|---|---|---|
| ≤ 4,096 | 2^12 | `powersOfTau28_hez_final_12.ptau` |
| ≤ 16,384 | 2^14 | `powersOfTau28_hez_final_14.ptau` |
| ≤ 65,536 | 2^16 | `powersOfTau28_hez_final_16.ptau` |
| ≤ 1,048,576 | 2^20 | `powersOfTau28_hez_final_20.ptau` |

Download from: `https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_XX.ptau`

---

## 8. Field Overflow / BigInt Out of Range

**Error message:**
```
Error: BigInt out of range
```
or witness values that are unexpectedly large.

**Cause:**
A value in the computation exceeded the BN254 prime field modulus. Circom arithmetic is modular, so overflows wrap around silently unless you add range checks.

**Fix:**
1. Add range checks in the circuit using `circomlib`'s `Num2Bits` or `LessThan` templates:
```circom
include "node_modules/circomlib/circuits/comparators.circom";

component rangeCheck = LessThan(32);
rangeCheck.in[0] <== mySignal;
rangeCheck.in[1] <== 2**32;
rangeCheck.out === 1;
```
2. In JavaScript, use `BigInt` and take values modulo the field prime before passing to the circuit:
```js
const FIELD_PRIME = 21888242871839275222246405745257275088548364400416034343698204186575808495617n;
const safeValue = BigInt(myValue) % FIELD_PRIME;
```

---

## 9. snarkjs Version Incompatibility

**Symptom:**
Errors change depending on which version of snarkjs is installed. Common across 0.6 / 0.7 boundary.

**Fix:**
Pin your snarkjs version in `package.json`:
```json
{
  "dependencies": {
    "snarkjs": "0.7.4"
  }
}
```

Key API differences between v0.6 and v0.7:
- v0.7 changed the PLONK prover output format
- v0.7 requires explicit curve specification in some API calls
- The CLI `fullProve` API signature changed (see the migration guide at `docs/SNARKJS_MIGRATION.md`)

---

## 10. Signal Not Constrained Warning

**Warning message:**
```
[WARN] signal X does not appear in any constraint
```

**Cause:**
A signal is declared in the circuit but never used in a constraint (`===`) or assigned via `<==`. Unconstrained signals are a security risk: a malicious prover can set them to any value without affecting proof validity.

**Fix:**
Either constrain the signal or remove it:
```circom
// WRONG — unconstrained output
signal output result;
result <-- someComputation();  // <-- is assignment only, no constraint

// CORRECT — constrained output
signal output result;
result <== someComputation();  // <== is both assignment and constraint
```

If the signal must be computed off-circuit, use `<--` for assignment and add an explicit check:
```circom
signal output result;
result <-- externalValue;
result * result === knownSquare;  // add a meaningful constraint
```

---

## Getting More Help

- Run `snarkjs --help` for a full command reference
- Check the [snarkjs GitHub issues](https://github.com/iden3/snarkjs/issues) for known bugs
- Join the Soroban ZK Toolkit Discord for community support

Related: Issue #21 — Add troubleshooting guide for common proof generation failures.
