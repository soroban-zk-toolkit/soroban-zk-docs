# Comparison: Groth16 vs PLONK vs STARKs

This page compares three widely used zero-knowledge proof systems available to Soroban developers. Use it to choose the right system for your application.

---

## Quick-Reference Comparison Table

| Property | Groth16 | PLONK | STARKs |
|---|---|---|---|
| **Proof type** | zk-SNARK | zk-SNARK | zk-STARK |
| **Proof size** | ~200 bytes (3 G1 + 1 G2) | ~400–600 bytes | 10–200 KB (scales with circuit size) |
| **Verification time** | Very fast (~2 ms) | Fast (~5–10 ms) | Slower (~50–200 ms, logarithmic) |
| **Trusted setup** | Required, circuit-specific | Required, universal (reusable) | None — transparent |
| **Setup size** | Small (per circuit) | Large SRS (once, shared) | N/A |
| **Prover time** | Fast | Moderate | Slower for small circuits, competitive for large |
| **Quantum resistance** | No (elliptic curve) | No (elliptic curve) | Yes (hash-based) |
| **Recursion support** | Difficult | Good (native folding) | Good (FRI-based) |
| **Arithmetic field** | BN254 / BLS12-381 | BN254 / BLS12-381 | Binary / prime fields |
| **Widely audited** | Yes (Zcash, Tornado Cash) | Yes (Aztec, Polygon) | Yes (StarkWare) |

---

## Groth16

### How It Works
Groth16 is a pairing-based SNARK introduced in 2016. The prover constructs three elliptic-curve group elements (proof π = (A, B, C)) and the verifier checks a pairing equation. The proving algorithm is highly optimized, making it the fastest prover among the three systems for most circuit sizes.

### Trusted Setup
Groth16 requires a **circuit-specific** trusted setup (Phase 2). Every time you modify the circuit, you must rerun the setup ceremony. The setup produces a proving key and verification key tied to that exact circuit. A compromised setup allows forging of proofs.

### Proof Size and Verification
The proof is ~200 bytes regardless of circuit complexity — the smallest among the three. Verification requires exactly 3 pairing operations, making it extremely fast and cheap on-chain.

### Best Use Cases
- Applications where proof size and verification cost are critical (e.g., on-chain verification on resource-limited chains)
- Mature, well-audited circuits (the setup is one-time for a stable circuit)
- Confidential token transfers, private identity proofs

### Libraries & Tools
- `snarkjs` (JavaScript/WASM)
- `bellman` (Rust)
- `gnark` (Go)

---

## PLONK

### How It Works
PLONK (Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge) uses a universal trusted setup that is reusable across all circuits. The prover commits to the circuit's wire values using polynomial commitments (KZG by default), and the verifier checks a series of polynomial identities.

### Trusted Setup
PLONK uses a **universal, updateable** trusted setup (Structured Reference String, SRS). A single ceremony can be used for all circuits up to a maximum size, and new participants can add randomness at any time, making the setup more trust-distributed than Groth16's circuit-specific setup.

### Proof Size and Verification
Proofs are ~400–600 bytes (larger than Groth16 but still compact). Verification requires KZG multi-point evaluations — slightly slower than Groth16's pairings but still practical on-chain.

### Best Use Cases
- Frequently updated circuits (no new ceremony needed per circuit change)
- Applications needing recursive proof composition
- Development environments where circuit iteration speed matters

### Libraries & Tools
- `snarkjs` (PLONK mode)
- `jellyfish` (Rust, Espresso Systems)
- `halo2` (variant with different commitment scheme)

---

## STARKs

### How It Works
STARKs (Scalable Transparent ARguments of Knowledge) use only cryptographic hash functions — no elliptic curves or pairings. The prover commits to the execution trace over a large field using Merkle trees (via the FRI protocol), and the verifier checks random query positions. This makes STARKs transparent (no trusted setup) and post-quantum secure.

### Trusted Setup
**None.** STARKs are transparent: the only public parameters are standard hash function parameters. Anyone can verify that no trapdoor exists.

### Proof Size and Verification
Proof size scales logarithmically with circuit complexity and is much larger than SNARKs — typically 10 KB to 200 KB+. Verification is also slower, though still sublinear in circuit size.

### Best Use Cases
- High-security applications where eliminating trusted setup risk is paramount
- Post-quantum security requirements
- Large computations where prover scalability matters more than proof size (e.g., L2 validity proofs)
- zkEVM implementations (StarkWare, Polygon Miden)

### Libraries & Tools
- `stone-prover` (StarkWare, C++)
- `miden-vm` (Polygon, Rust)
- `winterfell` (Facebook/Meta, Rust)
- `plonky2` (Polygon, uses FRI; hybrid SNARK-STARK)

---

## Decision Guide

```
Do you need a trusted setup?
├── No → STARKs
└── Yes is acceptable
    ├── Is your circuit stable and won't change often?
    │   ├── Yes → Groth16 (smallest proof, fastest verification)
    │   └── No  → PLONK (universal setup, reusable)
    └── Do you need recursion / folding?
        ├── Yes → PLONK or STARKs
        └── No  → Groth16
```

---

## Detailed Properties Comparison

### Proof Size Breakdown

**Groth16** (BN254 curve):
- A: 1 G1 point = 64 bytes (compressed: 32 bytes)
- B: 1 G2 point = 128 bytes (compressed: 64 bytes)
- C: 1 G1 point = 64 bytes (compressed: 32 bytes)
- **Total: ~200 bytes (uncompressed) / ~128 bytes (compressed)**

**PLONK** (BN254 curve, KZG):
- 7 wire commitments + 1 quotient + 3 evaluations + 2 opening proofs
- **Total: ~400–600 bytes**

**STARKs**:
- Depends on the number of FRI queries (security parameter λ) and circuit depth
- Typical: 40–200 KB for 80-bit security with a 2^20 trace length

### Verification Cost (Ethereum as reference, scaled to Stellar)

| System | Approx. on-chain cost |
|---|---|
| Groth16 | ~250,000 gas (ETH) — very low |
| PLONK | ~450,000 gas (ETH) — moderate |
| STARKs | 2–5M+ gas (ETH) — high |

On Stellar/Soroban, absolute gas costs differ, but the relative ordering holds: Groth16 < PLONK << STARKs for verification cost.

---

## Summary

| Your priority | Best choice |
|---|---|
| Smallest proof & fastest on-chain verification | **Groth16** |
| Reusable setup, moderate proof size | **PLONK** |
| No trusted setup, post-quantum security | **STARKs** |
| Recursive proofs / proof aggregation | **PLONK or STARKs** |

Related: Issue #19 — Add comparison page between Groth16, Plonk, and STARKs.
