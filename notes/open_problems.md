# Open Problems & Technical Difficulties

## Overview

This document catalogs the main technical difficulties and unresolved design decisions in the Altius Privacy Module technical proposal. Items are ordered roughly by severity and dependency — the first two are immediate blockers for Phase 1 implementation; everything else depends on those choices being made first.

---

## 1. ~~Composite ZK Proof Design~~ (Resolved)

**Severity**: ~~Critical~~ → Resolved

**Decision**: Use **PLONK (Universal SRS)** as a unified proving system. All proof statements (wrap, confidential transfer, unwrap) are expressed as arithmetic circuits under a single PLONK instance with KZG polynomial commitments. This eliminates the need to compose separate sigma protocols and provides a single verifier on-chain. See `llms/260303-proof-system-decisions.md` for full rationale.

**Original problem**: The proposal specified "Bulletproofs or equivalent, TBD" for range proofs, but the actual proof required for a confidential transfer is **not a vanilla range proof**. The prover must simultaneously demonstrate:

1. Knowledge of `(balance, r)` that opens `com_sender` — a commitment opening proof
2. `balance >= amount` — a range proof on the difference
3. `amount > 0` — a range proof on the transfer amount
4. `com_transfer = Com(amount, r_transfer)` — a commitment opening proof for the transfer commitment

Standard Bulletproofs prove `v ∈ [0, 2^n)` for a single committed value. Here we need a proof about the *relationship between two commitments* (the sender's balance commitment and the transfer commitment), composed with range proofs on derived values.

### Why This Is Hard

There are two broad approaches, each with trade-offs:

**Approach A — Compose sigma protocols**: build a custom proof by combining Schnorr-like proofs (for commitment openings) with Bulletproofs (for range constraints) using the Fiat-Shamir heuristic. This preserves Bulletproofs' compact ~600-byte proof size but requires careful protocol design to ensure soundness when composing multiple proof statements. The composition must share a common transcript to prevent proof-splicing attacks.

**Approach B — General-purpose ZK circuit**: express all constraints in a single circuit (e.g., using Circom/Groth16 or Noir/UltraPlonk). This is simpler to implement correctly but:
- Loses Bulletproofs' no-trusted-setup property (if using Groth16)
- Requires in-circuit EC scalar multiplication for commitment verification (thousands of constraints)
- Proof size jumps from ~600 bytes to ~128 bytes (Groth16) or ~2-4 KB (PLONK)

The proposal hasn't specified which approach to take, and the choice significantly affects proof size, verification cost, and implementation complexity.

### Additional Complexity: Three Distinct Proof Types

The proposal actually requires three different proof types:
- **Wrap proof**: prove `commitment == Com(amount, r)` for a public `amount` — essentially a proof of knowledge of discrete log
- **Confidential transfer proof**: the full composite proof described above
- **Unwrap proof**: similar to confidential transfer but with a public `amount`

If these use different proof systems (e.g., Schnorr for wrap, Bulletproofs for transfer), the contract needs multiple verification paths, increasing surface area and complexity.

---

## 2. Elliptic Curve Selection

**Severity**: Critical — cascading dependency for every other component

The curve choice is unspecified but affects every component:

| Component | Curve dependency |
|-----------|-----------------|
| Pedersen commitment | Defines the groups `G`, `H` live in; determines security level |
| Bulletproofs | Natively work on prime-order groups; pairing-friendly curves (BN254) are unnecessary overhead |
| ECIES | Key format, compatibility with EVM's secp256k1 |
| Precompiles | What EC operations the Altius engine must implement |
| On-chain verification | Gas cost of EC point addition/scalar multiplication |

### The Tension

- **secp256k1** (EVM native): natural fit for key derivation from EVM keys, but no existing precompiles for general `ecAdd`/`ecMul` — the EIP-196 precompiles (`ecAdd`, `ecMul`, `ecPairing`) only work on alt_bn128 (BN254)
- **BN254** (alt_bn128): has EVM precompiles, but is a pairing-friendly curve (unnecessary for Pedersen/Bulletproofs, and pairing-friendly curves have weaker-than-expected security margins). Also, ECIES keys would diverge from EVM keys, complicating the key derivation hierarchy
- **Curve25519 / Ristretto**: optimal for Bulletproofs (this is what Monero and Penumbra use), but requires entirely custom precompiles and has no EVM ecosystem support

Since the Altius Execution Engine can define its own precompiles, this is less constrained than on Ethereum mainnet — but the choice still needs to be made explicitly, and it blocks all implementation work.

### Resolution

**Decision**: Use **BN254 (alt_bn128)**.

The choice of PLONK with KZG polynomial commitments (problem #1) forces us onto a pairing-friendly curve — KZG verification is a pairing equation check, and secp256k1 has no pairing. The two realistic options are BN254 and BLS12-381.

BN254 is chosen over BLS12-381 for pragmatic reasons:
- **Tooling maturity**: the entire Ethereum ZK toolchain (Barretenberg, snarkjs, Solidity verifier generators) is built around BN254
- **Performance**: smaller field (254-bit vs 381-bit) means faster EC operations
- **Ecosystem alignment**: zkSync, Polygon zkEVM, Aztec, Semaphore all use BN254
- **ECIES key separation is already handled**: the key derivation hierarchy derives `sk_ecies = KDF("ecies" || evm_sk)` on BN254, while the EVM signing key stays on secp256k1

The ~100-bit security level (vs BLS12-381's ~128-bit) is a known trade-off, but no practical attack exists. Migration to BLS12-381 later would be a curve swap within the same PLONK + KZG framework if stronger security is needed.

---

## 3. Receiver State Synchronization

**Severity**: Medium — engineering reliability problem

The ECIES memo mechanism creates a dependency chain for receiver state:

```
Transfer 1: Bob receives (v1, r_tx1) → balance = v1, r = r_tx1
Transfer 2: Bob receives (v2, r_tx2) → balance = v1 + v2, r = r_tx1 + r_tx2
Transfer 3: Bob receives (v3, r_tx3) → balance = v1 + v2 + v3, r = r_tx1 + r_tx2 + r_tx3
```

If Bob's client **misses transfer 2** (client crash, indexer lag, RPC failure), his local state becomes permanently desynchronized — he cannot construct valid proofs for any future transaction because his `(balance, r)` pair no longer opens `com_bob`.

### Edge Cases Not Addressed in the Proposal

- **Corrupted calldata**: if a memo's ciphertext is malformed (truncated, wrong encoding), decryption fails and the entire subsequent blinding factor chain breaks
- **Chain reorgs**: a transaction Bob already processed gets reorganized out, but the replacement block includes the same transfer with a different transaction ordering — does Bob's state need to roll back?
- **Multiple pending transactions**: if Bob receives two confidential transfers in the same block, the order in which he processes them matters for his cumulative `r` value
- **Adversarial memos**: a malicious sender could submit a transaction with a valid on-chain proof but an intentionally wrong encrypted memo — Bob's commitment updates correctly on-chain, but he receives garbage `(v, r_tx)` values and cannot recover his balance

### The Proposal's Recovery Mechanism

The proposal states: "scan the chain for all shielded operations, replay them in chronological order to reconstruct the nonce sequence." This works in theory but:

- Requires decrypting **every** incoming memo in order — O(n) in the number of transactions to the user
- If a single decryption fails (corrupted memo, adversarial sender), the entire recovery halts
- No on-chain checkpoint or commitment-to-local-state mechanism exists to detect or recover from desync

### Possible Mitigations (Not in Proposal)

- Store a hash of the user's `(balance, r)` on-chain after each operation as a checkpoint
- Include a sequence number in the encrypted memo so the receiver can detect gaps
- Allow the sender to prove memo correctness on-chain (expensive but prevents adversarial memos)

---

## 4. Nonce Management Under Adversarial Conditions

**Severity**: Medium — affects recovery and determinism

The key derivation for Pedersen randomness uses a client-side nonce:

```
r_tx = KDF("pedersen" || evm_sk || nonce)
```

The proposal states the nonce is "implicitly determined by the user's on-chain transaction history," but several scenarios break this assumption:

- **Reverted transactions**: a transaction that consumes nonce `n` gets submitted but reverts (e.g., insufficient balance). Did the nonce increment? If the user's client incremented it locally but the on-chain state didn't change, the nonce sequences diverge.
- **Concurrent transactions**: the user submits two shielded operations in rapid succession (before the first confirms). Both are constructed with the same nonce value, leading to identical `r_tx` — this is a blinding factor reuse, which can leak information.
- **Chain reorgs**: the user's transactions get reordered in a different sequence after a reorg, changing the implicit nonce mapping.

### Possible Mitigations

- Use an explicit on-chain nonce counter (simple `uint256` incremented per successful shielded operation) — adds one SSTORE but eliminates ambiguity
- Derive `r_tx` from transaction-specific entropy (e.g., `KDF(evm_sk || tx_hash)`) instead of a sequential nonce — but then recovery requires knowing all transaction hashes

---

## 5. Wrap Proof — Verification Path Design

**Severity**: Medium — straightforward in isolation, tricky in integration

The wrap operation requires proving: "I know `r` such that `commitment == amount·G + r·H`" where `amount` is public. Since `amount` is known, the verifier can compute `amount·G` and check that `commitment - amount·G` is a valid multiple of `H`. This reduces to a **proof of knowledge of discrete log** of `(commitment - amount·G)` with respect to `H` — essentially a Schnorr proof.

This is cryptographically simple, but the integration question remains: if confidential transfers use Bulletproofs (or a composite proof system) and wrap uses a Schnorr proof, the contract needs **two separate verification codepaths**. Each verifier is an additional attack surface and audit target.

Ideally, all proof types would use a unified proving system, but that may force suboptimal trade-offs (e.g., using a full circuit for a simple Schnorr proof).

---

## 6. Phase 2 Commitment Bridge

**Severity**: High — but deferred to Phase 2

The dual-track architecture requires both representations to stay consistent:

```
ct_balance:  FHE.Enc(balance)              // FHE track
com_balance: PedersenCommit(balance, r)     // ZK track
```

The proposal lists three bridging approaches:

| Approach | Guarantee | Cost |
|----------|-----------|------|
| Verify at wrap/unwrap only | Weak (only checked at boundaries) | Low |
| Execution engine threshold-decrypt and verify | Medium (requires trust in validators) | Medium |
| ZK-prove FHE encryption correctness | Strong (trustless) | Extremely high |

### Why the Strong Option Is Hard

Proving "this TFHE ciphertext encrypts the same value as this Pedersen commitment" inside a ZK circuit requires:

- Encoding LWE encryption (modular arithmetic over large integers) as field arithmetic constraints
- The TFHE ciphertext dimension `n` is typically 500-1000 — each LWE sample involves an inner product of that dimension
- Estimated circuit size: millions of constraints, orders of magnitude larger than the range proof circuit

This is an active research area. Some projects (e.g., Sunscreen, PESCA) are exploring ZK-FHE bridges, but none are production-ready.

---

## Summary

| # | Difficulty | Severity | Status | Blocks |
|---|-----------|----------|--------|--------|
| 1 | ~~Composite ZK proof design~~ | ~~Critical~~ | **Resolved: PLONK (Universal SRS)** | — |
| 2 | ~~Elliptic curve selection~~ | ~~Critical~~ | **Resolved: BN254** | — |
| 3 | Receiver state synchronization | Medium | Partially addressed | Production readiness |
| 4 | Nonce management under reorgs/reverts | Medium | Not addressed | Recovery mechanism |
| 5 | Wrap proof verification path | Medium | Not detailed | Contract design |
| 6 | Phase 2 commitment bridge | High | Deferred | Phase 2 |
