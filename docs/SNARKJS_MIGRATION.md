# Migration Guide: snarkjs v0.6 → v0.7

This guide covers all breaking changes when upgrading from snarkjs v0.6.x to v0.7.x, including API changes, trusted setup file format differences, and updated CLI commands.

---

## Overview of Changes

snarkjs v0.7 introduced several breaking changes focused on:
1. Improved PLONK prover and updated proof format
2. Revised API for `fullProve` and related functions
3. Updated trusted setup file format for PLONK (`zkey` v2)
4. CLI command renames and new flags
5. Better TypeScript types and ESM support

**Minimum Node.js version:** v16 (v0.7 dropped support for Node.js 14)

---

## Step 1 — Update the Package

```bash
# Check current version
npx snarkjs --version

# Update (use your package manager)
npm install snarkjs@latest
# or
yarn add snarkjs@latest
```

Verify:
```bash
npx snarkjs --version
# Should print 0.7.x
```

---

## Step 2 — API Changes

### `fullProve` — input format changed for PLONK

**v0.6:**
```js
const { proof, publicSignals } = await snarkjs.plonk.fullProve(
  input,
  wasmFile,
  zkeyFile
);
```

**v0.7:**
```js
// Same call signature for Groth16 — no change
const { proof, publicSignals } = await snarkjs.groth16.fullProve(
  input,
  wasmFile,
  zkeyFile
);

// PLONK: proof object structure changed (see proof format section below)
const { proof, publicSignals } = await snarkjs.plonk.fullProve(
  input,
  wasmFile,
  zkeyFile
);
```

The call signature is unchanged but the **shape of the returned `proof` object** differs for PLONK (see Step 4).

---

### `verify` — now returns a boolean (was Promise<bool> with side effects)

**v0.6:**
```js
const isValid = await snarkjs.groth16.verify(vKey, publicSignals, proof);
// isValid: boolean
```

**v0.7:** Identical call signature; the return type is still `Promise<boolean>`. No change needed for Groth16. For PLONK:

**v0.6:**
```js
const isValid = await snarkjs.plonk.verify(vKey, publicSignals, proof);
```

**v0.7:**
```js
// Same, but proof must be in the new v0.7 format (see Step 4)
const isValid = await snarkjs.plonk.verify(vKey, publicSignals, proof);
```

---

### `exportSolidityCallData` — removed from top-level, moved to named export

**v0.6:**
```js
import snarkjs from "snarkjs";
const calldata = await snarkjs.groth16.exportSolidityCallData(proof, pub);
```

**v0.7:** No change for Groth16. For PLONK, the function was renamed:
```js
// v0.7 PLONK
const calldata = await snarkjs.plonk.exportSolidityCallData(proof, pub);
```

---

### `zKey.exportVerificationKey` — curve parameter now required

**v0.6:**
```js
const vKey = await snarkjs.zKey.exportVerificationKey(zkeyFilename);
```

**v0.7:**
```js
// The curve is now inferred automatically from the zkey file
// No code change required, but the function is now async and may log differently
const vKey = await snarkjs.zKey.exportVerificationKey(zkeyFilename);
```

---

## Step 3 — Trusted Setup File Format

### Groth16 zkey — no format change

Groth16 `zkey` files from v0.6 are fully compatible with v0.7. No regeneration needed.

### PLONK zkey — format updated (zkey v2)

PLONK `zkey` files from v0.6 are **not compatible** with v0.7. You must regenerate your PLONK zkeys.

**Why:** v0.7 changed the internal layout of the PLONK constraint system to support improved custom gates. The proving and verification key formats are incompatible.

**Migration steps for PLONK:**

```bash
# 1. Recompile your circuit (no change to .circom source needed)
circom circuits/my_circuit.circom --r1cs --wasm --sym -o build/

# 2. Regenerate the zkey from your existing Powers of Tau file
snarkjs plonk setup build/my_circuit.r1cs pot12_final.ptau build/circuit_plonk_0000.zkey

# 3. Re-export the verification key
snarkjs zkey export verificationkey build/circuit_plonk_final.zkey build/verification_key_plonk.json
```

> **Note:** The Powers of Tau (`.ptau`) file from v0.6 is still fully compatible. You do not need to redo Phase 1.

---

## Step 4 — PLONK Proof Format Changes

In v0.7, the PLONK proof JSON has additional fields for the updated polynomial commitment scheme.

**v0.6 PLONK proof (abbreviated):**
```json
{
  "A": ["0x...", "0x...", "1"],
  "B": ["0x...", "0x...", "1"],
  "C": ["0x...", "0x...", "1"],
  "Z": ["0x...", "0x...", "1"],
  "T1": ["0x...", "0x...", "1"],
  "T2": ["0x...", "0x...", "1"],
  "T3": ["0x...", "0x...", "1"],
  "eval_a": "0x...",
  "eval_b": "0x...",
  "eval_c": "0x...",
  "eval_s1": "0x...",
  "eval_s2": "0x...",
  "eval_zw": "0x...",
  "Wxi": ["0x...", "0x...", "1"],
  "Wxiw": ["0x...", "0x...", "1"],
  "protocol": "plonk",
  "curve": "bn128"
}
```

**v0.7 PLONK proof (abbreviated) — new fields:**
```json
{
  "A": ["0x...", "0x...", "1"],
  "B": ["0x...", "0x...", "1"],
  "C": ["0x...", "0x...", "1"],
  "Z": ["0x...", "0x...", "1"],
  "T1": ["0x...", "0x...", "1"],
  "T2": ["0x...", "0x...", "1"],
  "T3": ["0x...", "0x...", "1"],
  "Wxi": ["0x...", "0x...", "1"],
  "Wxiw": ["0x...", "0x...", "1"],
  "eval_a": "0x...",
  "eval_b": "0x...",
  "eval_c": "0x...",
  "eval_s1": "0x...",
  "eval_s2": "0x...",
  "eval_zw": "0x...",
  "eval_r": "0x...",    // NEW in v0.7
  "protocol": "plonk",
  "curve": "bn128"
}
```

**Impact:** If you store or serialize PLONK proofs, update your schema to include `eval_r`. Old v0.6 proofs will fail verification under v0.7.

---

## Step 5 — Updated CLI Commands

### Witness Generation — no change

```bash
# v0.6 and v0.7 — identical
node circuit_js/generate_witness.js circuit.wasm input.json witness.wtns
```

### Groth16 Prove — no change

```bash
snarkjs groth16 prove circuit_final.zkey witness.wtns proof.json public.json
```

### PLONK Prove — no change in command, new output format

```bash
snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json
```

### New: `snarkjs zkey verify` with explicit protocol flag

**v0.6:**
```bash
snarkjs zkey verify circuit.r1cs pot.ptau circuit_final.zkey
```

**v0.7:** Same command; the protocol (Groth16 or PLONK) is now auto-detected from the zkey. A new `--verbose` flag prints more ceremony contribution details:
```bash
snarkjs zkey verify circuit.r1cs pot.ptau circuit_final.zkey --verbose
```

### New: `snarkjs r1cs export json`

v0.7 adds a new CLI command to export the R1CS as human-readable JSON for debugging:
```bash
snarkjs r1cs export json circuit.r1cs circuit.r1cs.json
```

### Renamed: `wtns check` (was `wtns verify` in some builds)

**v0.6:**
```bash
snarkjs wtns verify circuit.r1cs witness.wtns
```

**v0.7:**
```bash
snarkjs wtns check circuit.r1cs witness.wtns
```

---

## Step 6 — TypeScript / ESM Changes

v0.7 ships improved TypeScript definitions and native ESM support.

**v0.6 (CommonJS import):**
```js
const snarkjs = require("snarkjs");
```

**v0.7 (ESM import — preferred):**
```js
import * as snarkjs from "snarkjs";
```

If your project uses CommonJS, the `require` form still works. For ESM projects, update your imports.

**TypeScript:** The type definitions for `ProofData` and `PublicSignals` are now exported from the package root:
```ts
import type { Groth16Proof, PlonkProof, PublicSignals } from "snarkjs";
```

---

## Migration Checklist

- [ ] Updated `snarkjs` to `^0.7.0` in `package.json`
- [ ] Node.js version is 16 or higher
- [ ] If using PLONK: regenerated all `.zkey` files
- [ ] If using PLONK: updated proof storage schema to include `eval_r`
- [ ] Updated import style to ESM if applicable
- [ ] Updated TypeScript types (if applicable)
- [ ] Ran `snarkjs zkey verify` on all newly generated keys
- [ ] Re-exported all verification keys and updated on-chain contracts if VK changed
- [ ] Updated CLI scripts using `wtns verify` → `wtns check`

---

## Common Migration Errors

| Error | Cause | Fix |
|---|---|---|
| `Error: invalid zkey version` | Old PLONK zkey used with v0.7 | Regenerate zkey with v0.7 |
| `TypeError: snarkjs.plonk.verify is not a function` | Using CJS default import | Use `import * as snarkjs` |
| `invalid proof: missing eval_r` | Old v0.6 PLONK proof passed to v0.7 verifier | Re-generate proof with v0.7 |
| `cannot find module 'snarkjs'` | ESM/CJS mismatch | Add `"type": "module"` to package.json or use CJS require |

Related: Issue #22 — Write migration guide from snarkjs v0.6 to v0.7.
