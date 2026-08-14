# Community Resources: ZK Learning Materials

A curated list of zero-knowledge proof learning resources — from beginner-friendly introductions to advanced academic papers — plus tools, communities, and Stellar ecosystem links.

---

## Foundational Papers

| Paper | Authors | What It Covers |
|---|---|---|
| [Groth16: On the Size of Pairing-Based Non-Interactive Arguments](https://eprint.iacr.org/2016/260.pdf) | Groth (2016) | The most widely deployed SNARK; basis for snarkjs Groth16 |
| [PLONK: Permutations over Lagrange-Bases](https://eprint.iacr.org/2019/953.pdf) | Gabizon, Williamson, Ciobotaru (2019) | Universal SNARK used in Aztec, Polygon |
| [STARKs: Scalable, Transparent Arguments of Knowledge](https://eprint.iacr.org/2018/046.pdf) | Ben-Sasson et al. (2018) | Hash-based, no trusted setup, post-quantum |
| [Bulletproofs](https://eprint.iacr.org/2017/1066.pdf) | Bünz et al. (2017) | Short range proofs without trusted setup |
| [Halo / Halo2](https://eprint.iacr.org/2019/1021.pdf) | Bowe, Grigg, Hopwood (2019) | Recursive proof composition without trusted setup |
| [Nova: Recursive Zero-Knowledge Arguments](https://eprint.iacr.org/2021/370.pdf) | Kothapalli, Setty, Tzialla (2021) | Folding-based incremental verification |

---

## Introductory Articles and Blog Posts

- **[ZK Proofs: An Illustrated Primer](https://blog.cryptographyengineering.com/2014/11/27/zero-knowledge-proofs-illustrated-primer/)** — Matthew Green. The clearest non-technical introduction to ZK concepts.
- **[ZK-SNARKs: Under the Hood](https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6)** — Vitalik Buterin. Three-part series covering the math step by step.
- **[Understanding PLONK](https://vitalik.ca/general/2019/09/22/plonk.html)** — Vitalik Buterin. Accessible explanation of the PLONK protocol.
- **[An Introduction to ZK Rollups](https://ethereum.org/en/developers/docs/scaling/zk-rollups/)** — Ethereum documentation. Explains how ZK proofs power L2 scaling.
- **[The MoonMath Manual](https://leastauthority.com/community-matters/moonmath-manual/)** — Least Authority. Free book covering ZK math from the ground up.

---

## Video Courses and Lectures

- **[Zero Knowledge Proofs MOOC](https://zk-learning.org/)** — Stanford / Berkeley / CMU collaborative course. Free, 12 weeks, covers theory and practice.
- **[ZK Whiteboard Sessions](https://www.youtube.com/playlist?list=PLj80z0cJm8QErn3akRcqvxUsyXWC81OGq)** — ZKProof community. 20+ sessions with leading researchers.
- **[Circom and snarkjs Workshop](https://www.youtube.com/watch?v=I7I-qVAI3Tc)** — iden3 team. Hands-on circuit writing and proof generation.
- **[Dan Boneh's Cryptography Course](https://crypto.stanford.edu/~dabo/courses/OnlineCrypto/)** — Stanford. Prerequisite cryptography; covers elliptic curves and pairings.
- **[0xPARC ZK Education](https://learn.0xparc.org/)** — Practical ZK program with hands-on Circom exercises.

---

## Developer Tools

### Circuit Languages
- **[Circom](https://docs.circom.io/)** — Most widely used ZK circuit language. Used with snarkjs.
- **[Noir](https://noir-lang.org/)** — Rust-inspired ZK language by Aztec; compiles to PLONK.
- **[Cairo](https://www.cairo-lang.org/)** — StarkWare's ZK-native language for STARKs.
- **[Leo](https://developer.aleo.org/leo/language/)** — ZK language for the Aleo blockchain.
- **[Lurk](https://github.com/lurk-lab/lurk-rs)** — Lisp-inspired language for recursive SNARKs.

### Proof Libraries
- **[snarkjs](https://github.com/iden3/snarkjs)** — JavaScript/WASM library for Groth16 and PLONK.
- **[bellman](https://github.com/zkcrypto/bellman)** — Rust library for building circuits and provers.
- **[gnark](https://github.com/ConsenSys/gnark)** — Go library; supports Groth16 and PLONK.
- **[arkworks](https://github.com/arkworks-rs)** — Rust ecosystem for ZK cryptography primitives.
- **[halo2](https://github.com/zcash/halo2)** — ZCash's recursive SNARK library (no trusted setup).
- **[circomlibjs](https://github.com/iden3/circomlibjs)** — JavaScript implementation of circomlib primitives (Poseidon, MiMC, etc.).

### Utilities
- **[circomlib](https://github.com/iden3/circomlib)** — Standard library of Circom circuit templates (hash functions, comparators, Merkle trees).
- **[zkREPL](https://zkrepl.dev/)** — Browser-based Circom IDE with instant compilation.
- **[snarkjs ceremony tools](https://github.com/weijiekoh/perpetualpowersoftau)** — Tools for running trusted setup ceremonies.
- **[Poseidon hash calculator](https://github.com/iden3/poseidon)** — Reference implementation of the Poseidon ZK-friendly hash function.

---

## Communities and Forums

- **[ZKProof Community](https://zkproof.org/)** — Standards body and community for ZK proof practitioners. Hosts annual workshops.
- **[0xPARC Discord](https://discord.gg/0xparc)** — Active community for ZK application developers.
- **[ZK Hack Discord](https://discord.gg/zkhack)** — Runs regular ZK puzzle competitions and workshops.
- **[Ethereum Research Forum — ZK](https://ethresear.ch/c/zero-knowledge-proofs/55)** — Technical research discussions.
- **[iden3 Forum](https://github.com/iden3/snarkjs/discussions)** — snarkjs and Circom questions.
- **[Reddit r/zk](https://www.reddit.com/r/zk/)** — General ZK discussion.

---

## Stellar / Soroban Ecosystem

- **[Soroban Documentation](https://soroban.stellar.org/docs)** — Official Soroban smart contract platform docs.
- **[Stellar Developers Discord](https://discord.gg/stellar)** — #zk-proofs channel for Soroban ZK discussion.
- **[Soroban ZK Toolkit (this repo)](https://github.com/soroban-zk-toolkit/soroban-zk-docs)** — Tools and documentation for integrating ZK proofs into Soroban contracts.
- **[Stellar GitHub](https://github.com/stellar)** — Core Stellar/Soroban protocol repositories.
- **[Stellar Community Forum](https://community.stellar.org/)** — Developer discussions and proposals.
- **[Stellar Quest](https://quest.stellar.org/)** — Interactive learning challenges for Stellar developers.

---

## Practice and Challenges

- **[ZK Hack Puzzles](https://www.zkhack.dev/puzzles.html)** — Real-world ZK vulnerabilities to find and exploit. Excellent for deepening understanding.
- **[Noir by Example](https://noir-by-example.org/)** — Hands-on Noir circuit examples.
- **[circomlib exercises](https://github.com/0xPARC/circom-ecdsa)** — Real-world circomlib usage in ECDSA signature verification.
- **[Awesome Zero Knowledge Proofs](https://github.com/matter-labs/awesome-zero-knowledge-proofs)** — Comprehensive curated list maintained by Matter Labs.

---

## Newsletters and Blogs to Follow

- **[ZK Newsletter](https://zknewsletter.com/)** — Weekly digest of ZK research, tools, and ecosystem news.
- **[Paradigm Research Blog](https://www.paradigm.xyz/writing)** — Deep technical posts on ZK and cryptography.
- **[Espresso Systems Blog](https://www.espressosys.com/blog)** — PLONK, HotShot consensus, and ZK infrastructure.
- **[StarkWare Blog](https://starkware.co/blog/)** — STARKs, Cairo, and ZK scaling.
- **[Aztec Blog](https://aztec.network/blog)** — PLONK, noir, and private smart contracts.

---

## Reference Implementations

| System | Reference Implementation |
|---|---|
| Groth16 | [snarkjs (JS)](https://github.com/iden3/snarkjs), [bellman (Rust)](https://github.com/zkcrypto/bellman) |
| PLONK | [snarkjs (JS)](https://github.com/iden3/snarkjs), [jellyfish (Rust)](https://github.com/EspressoSystems/jellyfish) |
| STARKs | [stone-prover (C++)](https://github.com/starkware-libs/stone-prover), [winterfell (Rust)](https://github.com/facebook/winterfell) |
| Halo2 | [zcash/halo2 (Rust)](https://github.com/zcash/halo2) |
| Nova | [microsoft/Nova (Rust)](https://github.com/microsoft/Nova) |

Related: Issue #23 — Add community resources page with links to ZK learning materials.
