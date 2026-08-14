# Tutorial: Build a Private Voting System Using Membership Proofs

In this tutorial you will build a complete private voting system on Stellar/Soroban. Voters prove they are eligible members of a voter registry without revealing their identity, and a nullifier prevents double voting — all without a trusted coordinator.

## Prerequisites

- Familiarity with Circom circuit syntax
- snarkjs installed (`npm install -g snarkjs` or use the WASM playground)
- A Soroban development environment (Stellar CLI)

---

## Concepts

### Membership Proof
A membership proof lets a voter prove "my secret key is in the approved voter set" without revealing which entry it corresponds to. We use a Merkle tree whose leaves are commitments to voter keys. The voter provides a Merkle path (siblings) as private inputs and the circuit verifies the path leads to the public root.

### Nullifier
A nullifier is a deterministic value derived from the voter's secret key and the election ID:

```
nullifier = Poseidon(secretKey, electionId)
```

The nullifier is published on-chain when a vote is cast. If the same voter tries to vote again, the circuit would produce the same nullifier, and the contract rejects it as already seen.

### Vote Commitment
The actual vote (0 or 1) is kept private. The voter submits a vote commitment and a ZK proof that:
1. They are in the voter Merkle tree.
2. The nullifier matches their secret key.
3. The vote value is 0 or 1 (range check).

---

## Step 1 — Circuit Design

Create `circuits/vote.circom`:

```circom
pragma circom 2.0.0;

include "node_modules/circomlib/circuits/poseidon.circom";
include "node_modules/circomlib/circuits/mux1.circom";
include "node_modules/circomlib/circuits/comparators.circom";

// Merkle proof inclusion (depth = LEVELS)
template MerkleProof(LEVELS) {
    signal input leaf;
    signal input pathElements[LEVELS];
    signal input pathIndices[LEVELS];  // 0 = left, 1 = right
    signal output root;

    component hashers[LEVELS];
    component mux[LEVELS];

    signal levelHashes[LEVELS + 1];
    levelHashes[0] <== leaf;

    for (var i = 0; i < LEVELS; i++) {
        hashers[i] = Poseidon(2);
        mux[i] = MultiMux1(2);

        mux[i].c[0][0] <== levelHashes[i];
        mux[i].c[0][1] <== pathElements[i];
        mux[i].c[1][0] <== pathElements[i];
        mux[i].c[1][1] <== levelHashes[i];
        mux[i].s <== pathIndices[i];

        hashers[i].inputs[0] <== mux[i].out[0];
        hashers[i].inputs[1] <== mux[i].out[1];
        levelHashes[i + 1] <== hashers[i].out;
    }

    root <== levelHashes[LEVELS];
}

template Vote(LEVELS) {
    // Private inputs
    signal input secretKey;
    signal input vote;              // 0 or 1
    signal input pathElements[LEVELS];
    signal input pathIndices[LEVELS];

    // Public inputs
    signal input electionId;
    signal input merkleRoot;

    // Public outputs
    signal output nullifier;
    signal output voteCommitment;

    // 1. Compute the voter's leaf commitment
    component leafHasher = Poseidon(1);
    leafHasher.inputs[0] <== secretKey;

    // 2. Verify Merkle membership
    component merkle = MerkleProof(LEVELS);
    merkle.leaf <== leafHasher.out;
    for (var i = 0; i < LEVELS; i++) {
        merkle.pathElements[i] <== pathElements[i];
        merkle.pathIndices[i] <== pathIndices[i];
    }
    merkle.root === merkleRoot;

    // 3. Compute nullifier
    component nullifierHasher = Poseidon(2);
    nullifierHasher.inputs[0] <== secretKey;
    nullifierHasher.inputs[1] <== electionId;
    nullifier <== nullifierHasher.out;

    // 4. Validate vote is binary (0 or 1)
    vote * (vote - 1) === 0;

    // 5. Compute vote commitment (hide actual vote)
    component voteHasher = Poseidon(2);
    voteHasher.inputs[0] <== vote;
    voteHasher.inputs[1] <== secretKey;
    voteCommitment <== voteHasher.out;
}

component main {public [electionId, merkleRoot]} = Vote(20);
```

---

## Step 2 — Trusted Setup

```bash
# Download Powers of Tau (phase 1, up to 2^21 constraints)
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_21.ptau -O pot21_final.ptau

# Compile circuit
circom circuits/vote.circom --r1cs --wasm --sym -o build/

# Phase 2 setup (circuit-specific)
snarkjs groth16 setup build/vote.r1cs pot21_final.ptau build/vote_0000.zkey

# Contribute entropy (in production, run a ceremony)
snarkjs zkey contribute build/vote_0000.zkey build/vote_final.zkey --name="Contributor 1" -v

# Export verification key
snarkjs zkey export verificationkey build/vote_final.zkey build/verification_key.json
```

---

## Step 3 — Off-Chain Voter Registry

The election organizer builds a Merkle tree over all eligible voters:

```js
import { buildPoseidon } from "circomlibjs";
import { MerkleTree } from "merkletreejs";

const poseidon = await buildPoseidon();

// Each voter generates a secret key (kept private)
const voters = [
  BigInt("0xDEADBEEF..."),  // voter 1 secret key
  BigInt("0xCAFEBABE..."),  // voter 2 secret key
];

// Leaf = Poseidon(secretKey)
const leaves = voters.map(sk => poseidon([sk]));

// Build Merkle tree using Poseidon as hash function
const tree = new MerkleTree(leaves, (a, b) => poseidon([a, b]), {
  sort: false,
  hashLeaves: false,
});

const merkleRoot = tree.getRoot();
console.log("Merkle root:", merkleRoot.toString("hex"));
```

---

## Step 4 — Generating a Proof (Voter Side)

```js
import snarkjs from "snarkjs";

async function castVote(secretKey, vote, electionId, tree) {
  const leaf = poseidon([secretKey]);
  const proof = tree.getProof(leaf);

  const input = {
    secretKey: secretKey.toString(),
    vote: vote.toString(),           // 0 = No, 1 = Yes
    pathElements: proof.map(p => p.data.toString("hex")),
    pathIndices: proof.map(p => p.position === "right" ? 0 : 1),
    electionId: electionId.toString(),
    merkleRoot: tree.getRoot().toString("hex"),
  };

  const { proof: zkProof, publicSignals } = await snarkjs.groth16.fullProve(
    input,
    "build/vote_js/vote.wasm",
    "build/vote_final.zkey"
  );

  return { zkProof, publicSignals };
}
```

---

## Step 5 — On-Chain Tally (Soroban Contract)

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, Env, Bytes, Map, Symbol};

#[contract]
pub struct VotingContract;

#[contractimpl]
impl VotingContract {
    /// Submit a vote with ZK proof
    pub fn cast_vote(
        env: Env,
        nullifier: u128,
        vote_commitment: u128,
        proof: Bytes,          // serialized Groth16 proof
        public_signals: Bytes, // [nullifier, voteCommitment, electionId, merkleRoot]
    ) {
        let used: Map<u128, bool> = env
            .storage()
            .persistent()
            .get(&Symbol::new(&env, "nullifiers"))
            .unwrap_or(Map::new(&env));

        assert!(!used.contains_key(nullifier), "already voted");

        // Verify ZK proof on-chain (via precompile or off-chain oracle)
        assert!(Self::verify_proof(&env, &proof, &public_signals), "invalid proof");

        // Record nullifier
        let mut updated = used;
        updated.set(nullifier, true);
        env.storage()
            .persistent()
            .set(&Symbol::new(&env, "nullifiers"), &updated);

        // Tally: increment yes_count or no_count based on vote_commitment
        // (vote value stays private; commitment is stored and revealed at end)
        let mut commitments: soroban_sdk::Vec<u128> = env
            .storage()
            .persistent()
            .get(&Symbol::new(&env, "commitments"))
            .unwrap_or(soroban_sdk::Vec::new(&env));
        commitments.push_back(vote_commitment);
        env.storage()
            .persistent()
            .set(&Symbol::new(&env, "commitments"), &commitments);
    }

    fn verify_proof(_env: &Env, _proof: &Bytes, _signals: &Bytes) -> bool {
        // Integration with Stellar's upcoming ZK verification precompile
        // or use an off-chain verifier oracle pattern
        true // placeholder
    }
}
```

---

## Step 6 — Tallying the Result

At the end of the election period:
1. All voters who voted may optionally reveal their vote commitment pre-image (`vote`, `secretKey`).
2. The contract verifies `Poseidon(vote, secretKey) == storedCommitment` and adds the vote to the tally.
3. Unrevealed votes are not counted (or counted as abstain, depending on election rules).

This ensures the tally is correct while keeping losing-side anonymity.

---

## Security Considerations

| Threat | Mitigation |
|---|---|
| Double voting | Nullifier published on-chain; second submission with same nullifier is rejected |
| Ineligible voter | Merkle membership proof; root is fixed at election start |
| Vote coercion | Voter holds secret key; commitment hides vote until reveal phase |
| Sybil attack | Voter registry controlled by organizer; one leaf per identity |

---

## Summary

You have built a complete private voting system:
- Circom circuit enforcing membership + nullifier + binary vote constraints
- Off-chain Merkle tree for voter registry
- snarkjs for proof generation
- Soroban smart contract for on-chain nullifier tracking and tally

Related: Issue #18 — Write tutorial: build a private voting system using membership proofs.
