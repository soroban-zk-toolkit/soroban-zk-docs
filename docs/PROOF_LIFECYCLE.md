# Full Proof Lifecycle: Circuit to On-Chain Verification

This document traces the complete lifecycle of a zero-knowledge proof — from writing a circuit to verifying it on the Soroban blockchain — with explanations of each stage and an ASCII diagram showing the data flow.

---

## Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ZK PROOF LIFECYCLE                                   │
│                                                                             │
│  DEVELOPER SIDE                          PROVER SIDE                        │
│  ─────────────                           ────────────                       │
│                                                                             │
│  ┌─────────────────┐                    ┌──────────────────────┐            │
│  │  1. Write       │                    │  4. Generate         │            │
│  │  Circuit        │                    │  Witness             │            │
│  │  (.circom)      │                    │                      │            │
│  └────────┬────────┘                    │  circuit.wasm        │            │
│           │ circom compile              │  + input.json        │            │
│           ▼                             │  ──────────────────► │            │
│  ┌─────────────────┐                    │  witness.wtns        │            │
│  │  2. Compile     │                    └──────────┬───────────┘            │
│  │  to R1CS        │                               │                        │
│  │  + WASM         │─────────────────────────────► │                        │
│  └────────┬────────┘   circuit.wasm, .r1cs         │                        │
│           │                                         ▼                        │
│           ▼                             ┌──────────────────────┐            │
│  ┌─────────────────┐                    │  5. Generate Proof   │            │
│  │  3. Trusted     │                    │                      │            │
│  │  Setup          │                    │  snarkjs.groth16     │            │
│  │  (Phase 1+2)    │─────────────────►  │  .fullProve()        │            │
│  └─────────────────┘   circuit_final   │                      │            │
│                         .zkey          │  proof.json          │            │
│                                         │  public.json         │            │
│                                         └──────────┬───────────┘            │
│                                                    │                        │
│  ON-CHAIN (Soroban)                                │                        │
│  ────────────────                                  │                        │
│                                                    ▼                        │
│  ┌─────────────────┐                    ┌──────────────────────┐            │
│  │  6. Submit Tx   │◄───────────────────│  Prover sends        │            │
│  │  to Contract    │   proof.json +     │  proof + signals     │            │
│  │                 │   public signals   │  to smart contract   │            │
│  └────────┬────────┘                    └──────────────────────┘            │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │  7. On-Chain    │                                                        │
│  │  Verification   │                                                        │
│  │  (Soroban Rust) │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                  │
│    ┌──────┴───────┐                                                         │
│    ▼              ▼                                                          │
│  VALID          INVALID                                                     │
│  → state        → tx                                                        │
│    updated        rejected                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Stage-by-Stage Explanation

### Stage 1 — Write the Circuit

**Location:** Developer's machine  
**Input:** Application logic (e.g., "prove knowledge of a preimage", "prove set membership")  
**Output:** `.circom` source file

The circuit is written in the Circom language. It describes the computation as a set of constraints over field elements. Every constraint takes the form:

```
(A · w) * (B · w) = (C · w)
```

where `w` is the witness vector (all signal values). The circuit declares:
- **Private inputs** — known only to the prover (secret key, vote value, etc.)
- **Public inputs** — known to both prover and verifier (Merkle root, nullifier, etc.)
- **Intermediate signals** — internal computation values
- **Outputs** — public outputs committed to in the proof

---

### Stage 2 — Compile to R1CS + WASM

**Location:** Developer's machine  
**Tool:** `circom` compiler  
**Input:** `.circom` source file  
**Output:** `.r1cs` (constraint system), `.wasm` (witness calculator), `.sym` (symbol map)

```bash
circom circuits/my_circuit.circom --r1cs --wasm --sym -o build/
```

The R1CS (Rank-1 Constraint System) file encodes all constraints in a compact binary format. The WASM file is a compiled witness calculator that takes inputs and computes all intermediate signal values.

---

### Stage 3 — Trusted Setup

**Location:** Ceremony (multi-party computation)  
**Input:** R1CS file, Powers of Tau file  
**Output:** `circuit_final.zkey` (proving key), `verification_key.json` (verification key)

The trusted setup binds the circuit to a set of cryptographic parameters:

```
Phase 1 (universal):
  pot_final.ptau  ←  Powers of Tau ceremony
                     (any circuit up to 2^N constraints)

Phase 2 (circuit-specific):
  circuit.r1cs + pot_final.ptau
        │
        ▼
  circuit_final.zkey   ─────► verification_key.json
  (proving key, large) │       (public, goes on-chain, ~2KB)
                        │
                        └── distributed to all provers
```

The verification key is the small public key that the Soroban contract uses for verification. The proving key is larger (~50–500 MB) and used by provers.

---

### Stage 4 — Generate Witness

**Location:** Prover's machine (or browser, via WASM)  
**Input:** `circuit.wasm`, `input.json` (private + public signals)  
**Output:** `witness.wtns` (binary witness file)

```bash
node build/circuit_js/generate_witness.js \
  build/circuit.wasm \
  input.json \
  witness.wtns
```

The witness is the full assignment of values to all signals in the circuit — both private and public. It proves that the prover knows a valid assignment satisfying all constraints. The witness is never shared; only the proof derived from it is.

**Data flow:**
```
input.json
  ├── private: { secretKey: "...", vote: "1" }
  └── public:  { merkleRoot: "...", electionId: "42" }
            │
            ▼ (circuit.wasm evaluates all constraints)
witness.wtns
  └── all signal values: [1, secretKey, vote, leafHash, ..., nullifier]
```

---

### Stage 5 — Generate Proof

**Location:** Prover's machine (or browser, via WASM)  
**Input:** `circuit_final.zkey`, `witness.wtns`  
**Output:** `proof.json`, `public.json`

```js
const { proof, publicSignals } = await snarkjs.groth16.fullProve(
  inputJson,
  "build/circuit.wasm",
  "build/circuit_final.zkey"
);
```

The prover runs the Groth16 (or PLONK) proving algorithm over the witness and proving key to produce a compact proof that:
- The prover knows a witness satisfying all constraints
- The public signals match the declared public inputs

**Groth16 proof structure:**
```
proof.json = {
  "pi_a": [G1_x, G1_y, "1"],  // 64 bytes
  "pi_b": [[G2_x1, G2_x2], [G2_y1, G2_y2], ["1", "0"]], // 128 bytes
  "pi_c": [G1_x, G1_y, "1"],  // 64 bytes
  "protocol": "groth16",
  "curve": "bn128"
}

public.json = ["nullifier_value", "merkleRoot_value", "electionId_value"]
```

Total proof size: ~200 bytes (three elliptic-curve points).

---

### Stage 6 — Submit Transaction to Soroban

**Location:** Prover's client application  
**Input:** `proof.json`, `public.json`  
**Output:** Soroban transaction

The prover serializes the proof and public signals and submits them as arguments to the Soroban smart contract's `verify_and_act` entry point:

```rust
// Transaction call (conceptual)
contract.cast_vote(
    nullifier,       // public signal: u128
    vote_commitment, // public signal: u128
    proof_bytes,     // serialized pi_a, pi_b, pi_c
    public_bytes,    // serialized public signals
)
```

---

### Stage 7 — On-Chain Verification

**Location:** Soroban validator nodes  
**Input:** proof bytes, public signals, verification key (embedded in contract)  
**Output:** Accept or reject the transaction

The Soroban contract performs the Groth16 pairing check:

```
e(pi_a, pi_b) == e(alpha, beta) * e(vk_x, gamma) * e(pi_c, delta)

where:
  vk_x = vk_IC[0] + Σ (public_signal[i] * vk_IC[i+1])
```

If the equation holds, the proof is valid — the prover knows a satisfying witness — and the contract proceeds with its state update (recording a nullifier, updating a tally, etc.). If not, the transaction is rejected with no state change.

---

## End-to-End Data Summary

```
Files produced at each stage:

Stage 1: circuit.circom              (source code, text)
Stage 2: circuit.r1cs                (binary, constraint system)
         circuit.wasm                (binary, witness calculator)
Stage 3: circuit_final.zkey          (binary, proving key, large)
         verification_key.json       (JSON, ~2KB, public)
Stage 4: witness.wtns                (binary, NEVER shared)
Stage 5: proof.json                  (~200B, sent to chain)
         public.json                 (public signals, sent to chain)
Stage 6: Stellar transaction         (proof + signals as tx args)
Stage 7: contract state update       (nullifiers, tallies, etc.)
```

---

## Security Boundaries

```
┌──────────────────────────────────────┐
│ SECRET (never leaves prover's device)│
│   • circuit.circom private inputs    │
│   • witness.wtns                     │
│   • raw random contributions (setup) │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ PUBLIC (shared openly)               │
│   • circuit.r1cs                     │
│   • circuit.wasm                     │
│   • verification_key.json            │
│   • proof.json                       │
│   • public.json                      │
│   • pot_final.ptau                   │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ DISTRIBUTED (provers only)           │
│   • circuit_final.zkey (proving key) │
└──────────────────────────────────────┘
```

Related: Issue #24 — Create diagram showing full proof lifecycle from circuit to on-chain.
