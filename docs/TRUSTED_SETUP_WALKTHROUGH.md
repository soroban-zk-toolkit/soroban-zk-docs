# Video Walkthrough Guide: Trusted Setup Ceremony

This document is the full script and outline for a video walkthrough of the trusted setup ceremony for ZK circuits used in Soroban applications. Follow this outline when recording or attending a ceremony.

---

## Video Outline

**Total runtime:** ~45 minutes  
**Audience:** Developers integrating ZK proofs into Soroban smart contracts  
**Tools shown:** snarkjs, Node.js CLI, Powers of Tau files

---

## [00:00–02:00] Introduction

**Script:**
> "Welcome to the trusted setup ceremony walkthrough for Soroban ZK Toolkit. In this video we'll run through every step of the ceremony — from downloading the Phase 1 Powers of Tau file to exporting a final verification key. By the end, you'll understand what a trusted setup is, why it matters, and how to run one safely for your own circuit."

**Key points to cover:**
- What is a trusted setup and why is it needed for SNARKs (Groth16 / PLONK)?
- The "toxic waste" concept — the random contributions that must be discarded
- Phase 1 (universal, chain-agnostic) vs Phase 2 (circuit-specific)
- What happens if the toxic waste is not destroyed (proof forgery)

---

## [02:00–07:00] Phase 1 — Powers of Tau

**Script:**
> "Phase 1 is a one-time universal ceremony. The output is a Powers of Tau file that encodes elliptic-curve group elements raised to successive powers of a secret value τ. This file can be reused for any circuit up to the supported constraint count."

**Steps shown on screen:**

```bash
# Option A: Download an existing Phase 1 file (Hermez ceremony, trusted by the community)
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_12.ptau \
     -O pot12_final.ptau

# Verify the file hash (compare to published hash on the Hermez ceremony page)
b2sum pot12_final.ptau
```

**Timestamp 04:30 — Running your own Phase 1 (optional, for isolated testing):**

```bash
# Start a new Powers of Tau ceremony (supports up to 2^12 = 4096 constraints)
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v

# Contribute randomness (each participant does this independently)
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau \
  --name="Contributor 1" -v -e="$(cat /dev/urandom | head -c 64 | base64)"

# Add a random beacon (final phase of Phase 1)
snarkjs powersoftau beacon pot12_0001.ptau pot12_beacon.ptau \
  0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon"

# Prepare for Phase 2 (compute the Lagrange basis)
snarkjs powersoftau prepare phase2 pot12_beacon.ptau pot12_final.ptau -v
```

**What to highlight:**
- Each `contribute` step adds entropy; the toxic waste is the raw random value
- The `beacon` step uses a publicly verifiable random value (e.g., a Bitcoin block hash) to close Phase 1
- `prepare phase2` transforms the file into the format needed for circuit-specific setup

---

## [07:00–15:00] Circuit Compilation

**Script:**
> "Before running Phase 2, we need to compile our Circom circuit. This produces an R1CS file — a description of the circuit as a system of rank-1 constraints — and a WASM witness calculator."

**Steps on screen:**

```bash
# Install circom (one-time)
# circom is a standalone binary — download from https://github.com/iden3/circom/releases
# or build from source (Rust)

# Compile the circuit
circom circuits/my_circuit.circom --r1cs --wasm --sym -o build/

# Inspect the circuit
snarkjs r1cs info build/my_circuit.r1cs
```

**Expected output:**
```
[INFO]  snarkJS: Curve: bn-128
[INFO]  snarkJS: # of Wires: 1432
[INFO]  snarkJS: # of Constraints: 1127
[INFO]  snarkJS: # of Private Inputs: 4
[INFO]  snarkJS: # of Public Inputs: 2
[INFO]  snarkJS: # of Outputs: 1
```

**What to explain:**
- R1CS = Rank-1 Constraint System; each constraint is of the form (A · w) * (B · w) = (C · w)
- The number of constraints determines which Powers of Tau size you need (must satisfy 2^k ≥ constraints)
- The WASM file is the witness calculator used at proving time

---

## [15:00–25:00] Phase 2 — Circuit-Specific Setup

**Script:**
> "Phase 2 is where we bind the Phase 1 Powers of Tau file to our specific circuit. The output is a proving key (zkey) and a verification key."

**Steps on screen:**

```bash
# Initialize Phase 2 with our compiled circuit and Phase 1 file
snarkjs groth16 setup build/my_circuit.r1cs pot12_final.ptau build/circuit_0000.zkey

# Contribute randomness to Phase 2
# Each contributor downloads the current zkey, adds entropy, and passes it to the next
snarkjs zkey contribute build/circuit_0000.zkey build/circuit_0001.zkey \
  --name="Alice" -v -e="$(cat /dev/urandom | head -c 64 | base64)"

# Second contributor
snarkjs zkey contribute build/circuit_0001.zkey build/circuit_0002.zkey \
  --name="Bob" -v -e="$(cat /dev/urandom | head -c 64 | base64)"

# Apply a random beacon to finalize Phase 2
snarkjs zkey beacon build/circuit_0002.zkey build/circuit_final.zkey \
  0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon"

# Verify the zkey is valid
snarkjs zkey verify build/my_circuit.r1cs pot12_final.ptau build/circuit_final.zkey
```

**Timestamp 22:00 — Exporting the verification key:**

```bash
snarkjs zkey export verificationkey build/circuit_final.zkey build/verification_key.json
```

**What to highlight:**
- As long as at least one contributor destroys their randomness, the ceremony is secure
- The final zkey is the proving key; it contains all the information needed to generate proofs
- The verification key is small (~2 KB) and is what goes on-chain
- Show the `verification_key.json` contents: `alpha`, `beta`, `gamma`, `delta`, `IC` arrays

---

## [25:00–32:00] Testing the Setup — Generate and Verify a Proof

**Script:**
> "Let's confirm our setup works by generating a test proof and verifying it."

**Steps on screen:**

```bash
# Create a test input file (values must satisfy the circuit)
cat > build/input.json << 'EOF'
{
  "a": "3",
  "b": "4"
}
EOF

# Generate witness
node build/my_circuit_js/generate_witness.js \
  build/my_circuit_js/my_circuit.wasm \
  build/input.json \
  build/witness.wtns

# Generate proof
snarkjs groth16 prove build/circuit_final.zkey build/witness.wtns \
  build/proof.json build/public.json

# Verify proof locally
snarkjs groth16 verify build/verification_key.json \
  build/public.json build/proof.json
```

**Expected output:**
```
[INFO]  snarkJS: OK!
```

---

## [32:00–38:00] Exporting for On-Chain Verification

**Script:**
> "The final step is exporting the Solidity or Soroban-compatible verifier and the call data for the proof."

**Steps on screen:**

```bash
# Export a Solidity verifier (for reference — Soroban uses a Rust verifier)
snarkjs zkey export solidityverifier build/circuit_final.zkey build/Verifier.sol

# Export call data (for testing the Solidity verifier)
snarkjs zkey export soliditycalldata build/public.json build/proof.json
```

**Soroban integration note:**
> "For Soroban, you'll pass the proof bytes and public signals to the on-chain `verify_proof` entry point we covered in the private voting tutorial. The verification key is embedded in the contract during deployment."

---

## [38:00–42:00] Security Checklist

**Script:**
> "Before shipping your ceremony to production, run through this checklist."

- [ ] At least 3 independent contributors participated in Phase 2
- [ ] Each contributor used a different machine and operating environment
- [ ] Contributors destroyed their randomness after contributing (deleted temp files, wiped RAM if possible)
- [ ] The beacon value is from a publicly verifiable source (Bitcoin block hash, Ethereum block hash, drand)
- [ ] The final zkey was verified with `snarkjs zkey verify`
- [ ] The verification key hash is published and pinned in the documentation
- [ ] The proving key is distributed to provers via a verifiable content-addressed URL (IPFS, Arweave)

---

## [42:00–45:00] Conclusion and Next Steps

**Script:**
> "You've now completed the trusted setup ceremony. Keep your proving key secure — it's large (~50–500 MB for typical circuits) and should be distributed via IPFS or Arweave. Pin the verification key hash in your documentation so users can verify they have the right key. For PLONK circuits, the process is similar but uses a universal SRS — check the PLONK docs for differences."

**Links to show:**
- Hermez Powers of Tau ceremony transcript
- `snarkjs` official documentation
- Soroban ZK Toolkit verification key registry

---

## File Reference

| File | Description |
|---|---|
| `pot12_final.ptau` | Phase 1 Powers of Tau (reusable) |
| `circuit_final.zkey` | Phase 2 proving key (circuit-specific) |
| `verification_key.json` | Verification key (public, on-chain) |
| `proof.json` | Generated proof (per-transaction) |
| `public.json` | Public signals (per-transaction) |
| `witness.wtns` | Witness (private, never shared) |

Related: Issue #20 — Create video walkthrough guide for the trusted setup ceremony.
