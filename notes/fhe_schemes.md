# Fully Homomorphic Encryption (FHE) Schemes

## Overview

Fully Homomorphic Encryption allows computation on encrypted data without decryption. The result, when decrypted, matches the result of performing the same computation on the plaintext.

```
FHE.Enc(a) ⊕ FHE.Enc(b) = FHE.Enc(a + b)
FHE.Enc(a) ⊗ FHE.Enc(b) = FHE.Enc(a × b)
```

This is fundamentally different from **Pedersen Commitment's additive homomorphism**, which only supports addition:

```
Pedersen:  Com(a) + Com(b) = Com(a + b)     ✅ addition
           Com(a) × Com(b) = ???             ❌ no multiplication
           Com(a) > Com(b)  = ???             ❌ no comparison

FHE:       Enc(a) + Enc(b) = Enc(a + b)     ✅ addition
           Enc(a) × Enc(b) = Enc(a × b)     ✅ multiplication
           Enc(a) > Enc(b) → Enc(bool)       ✅ comparison (scheme-dependent)
```

---

## Key Concepts

### Noise and Bootstrapping

FHE ciphertexts contain **noise** that grows with each operation. After too many operations, the noise overwhelms the plaintext and decryption fails.

**Bootstrapping** is the process of "refreshing" a ciphertext — homomorphically evaluating the decryption circuit to reduce noise. This is the most expensive FHE operation.

```
Fresh ciphertext:     noise = small     ✅ can decrypt
After many ops:       noise = large     ⚠️ approaching limit
After bootstrapping:  noise = small     ✅ refreshed, can continue computing
```

### Leveled vs Fully Homomorphic

- **Leveled HE**: supports a fixed number of operations (depth) without bootstrapping. Parameters must be set upfront based on the computation depth.
- **Fully HE**: uses bootstrapping to support unlimited operations. More flexible but slower.

---

## TFHE (Torus Fully Homomorphic Encryption)

### How It Works

TFHE operates on the **torus** T = R/Z (real numbers modulo 1). It encrypts individual bits or small integers using LWE (Learning With Errors) samples:

```
LWE ciphertext: (a₁, a₂, ..., aₙ, b)
where b = Σ(aᵢ · sᵢ) + e + m · q/p

s: secret key vector
e: small noise
m: plaintext message
q: ciphertext modulus
p: plaintext modulus
```

### Core Feature: Programmable Bootstrapping

TFHE's signature innovation is **programmable bootstrapping** (PBS): it simultaneously refreshes the ciphertext noise AND evaluates a lookup table function. This means:

```
// In one bootstrapping operation:
Enc(x) → Enc(f(x))     // refresh + compute arbitrary function f

// Example: compare with threshold
Enc(balance) → Enc(balance > 100)  // one PBS operation
```

This makes TFHE uniquely suitable for arbitrary boolean and integer operations on encrypted data.

### Characteristics

- **Operand types**: bits, integers (2-bit, 4-bit, 8-bit, up to ~16-bit natively)
- **Bootstrapping speed**: ~10-20ms per PBS on modern hardware
- **Strength**: arbitrary operations via PBS; very flexible
- **Weakness**: slow for large arithmetic; each operation requires bootstrapping
- **Key project**: Zama (fhEVM, Concrete, TFHE-rs)

### Blockchain Relevance

Zama's **fhEVM** uses TFHE as the core FHE scheme. It provides:
- Encrypted unsigned integers (`euint8`, `euint16`, `euint32`, `euint64`)
- Homomorphic operations: `+`, `-`, `*`, `<`, `>`, `==`, `if/else`
- Solidity-like developer experience via `TFHE.add(a, b)`, `TFHE.lt(a, b)` syntax

---

## BGV/BFV (Brakerski-Gentry-Vaikuntanathan / Brakerski-Fan-Vercauteren)

### How It Works

BGV and BFV are closely related schemes based on the **Ring-LWE** problem. They encrypt integers modulo a plaintext modulus `t`, using polynomial rings:

```
Ciphertext: (c₀, c₁) ∈ Rq × Rq
where Rq = Zq[x]/(xⁿ + 1)

Decryption: m = (c₀ + c₁ · s) mod q mod t
```

### Core Feature: SIMD Batching

Through the **Chinese Remainder Theorem (CRT)**, a single ciphertext can encode **multiple plaintext values** in parallel "slots":

```
One ciphertext = [v₁, v₂, v₃, ..., vₙ]   // n values packed together

// Operations apply element-wise to all slots simultaneously:
Enc([a₁, a₂, ..., aₙ]) + Enc([b₁, b₂, ..., bₙ]) = Enc([a₁+b₁, a₂+b₂, ..., aₙ+bₙ])
```

This makes BGV/BFV extremely efficient for **batch operations** on many values.

### Characteristics

- **Operand types**: integers mod t (can be large), batched via SIMD slots
- **Strength**: very fast for large batches of additions and multiplications
- **Weakness**: comparison/branching requires bit-level decomposition (expensive); leveled — depth must be predetermined
- **Key libraries**: Microsoft SEAL, HElib, OpenFHE
- **BGV vs BFV**: BGV scales noise mod q down after each operation (scale-then-encrypt); BFV keeps the noise management in decryption (encrypt-then-scale). BFV is simpler to implement; BGV can be more efficient.

### Blockchain Relevance

Less commonly used for on-chain computation because:
- Comparison operations are expensive (critical for balance checks)
- SIMD batching is most useful for batch processing, less so for individual transactions
- Could be useful for batch settlement or batch proof aggregation

---

## CKKS (Cheon-Kim-Kim-Song)

### How It Works

CKKS encodes **real/complex numbers** (floating-point approximations) into polynomial rings. Unlike BGV/BFV which operate on exact integers, CKKS treats the encryption noise as part of the approximation:

```
Encode: (3.14159, 2.71828, ...) → polynomial
Encrypt: polynomial → ciphertext

// After computation, results are approximate:
Enc(3.14) + Enc(2.71) ≈ Enc(5.85)   // not exactly 5.85000...
```

### Core Feature: Approximate Arithmetic

CKKS can efficiently handle floating-point-like operations:

```
// Matrix multiplication on encrypted data:
Enc(A) × Enc(B) ≈ Enc(A × B)

// Machine learning inference on encrypted data:
Enc(input) → model → Enc(prediction)
```

### Characteristics

- **Operand types**: real/complex numbers (approximate)
- **Strength**: natural fit for ML inference, scientific computation, statistics
- **Weakness**: results are approximate — errors accumulate; NOT suitable for exact arithmetic
- **Key use cases**: privacy-preserving ML, encrypted analytics
- **Key libraries**: Microsoft SEAL, Lattigo, OpenFHE

### Blockchain Relevance

**Not suitable for blockchain balance operations** — financial transactions require exact arithmetic. A balance of 100.00 must remain exactly 100.00 after zero-value operations, not 99.99999997.

---

## Comparison

| Dimension | TFHE | BGV/BFV | CKKS |
|-----------|------|---------|------|
| Plaintext type | Bits / small integers | Integers mod t | Approximate real numbers |
| Addition | ✅ (with bootstrapping) | ✅ (native, fast) | ✅ (native, fast) |
| Multiplication | ✅ (with bootstrapping) | ✅ (increases depth) | ✅ (increases noise) |
| Comparison | ✅ (via PBS, efficient) | ⚠️ (bit decomposition, expensive) | ❌ (approximate, unreliable) |
| Branching / if-else | ✅ (via PBS) | ⚠️ (expensive) | ❌ |
| Exact arithmetic | ✅ | ✅ | ❌ (approximate) |
| Batching (SIMD) | ❌ | ✅ (major advantage) | ✅ |
| Bootstrapping speed | Fast (~10-20ms) | Slow (seconds) | Moderate |
| Per-operation cost | Higher (each op bootstraps) | Lower (batched) | Lower (batched) |
| Best for | Arbitrary integer logic | Batch integer arithmetic | ML / statistics |
| Blockchain fit | ✅ Best | ⚠️ Possible for batch | ❌ Not suitable |

---

## Why TFHE is the Leading Candidate for Blockchain

1. **Exact integer arithmetic**: financial operations require precision — no approximation
2. **Efficient comparison**: balance checks (`balance >= amount`) are a core operation; TFHE handles this via programmable bootstrapping
3. **Flexible operations**: any boolean or integer function can be expressed through PBS
4. **Zama ecosystem**: fhEVM provides a ready-made integration path for EVM-compatible chains
5. **Individual operation model**: blockchain transactions are typically individual (not batched), aligning with TFHE's per-operation model rather than BGV/BFV's SIMD batching

---

## FHE vs Pedersen Commitment (Recap)

For the Altius privacy module:

| | Pedersen Commitment | FHE (TFHE) |
|---|---|---|
| What it provides | Hides values inside commitments | Encrypts values for computation |
| Operations | Addition only | Arbitrary (add, mul, compare, branch) |
| Who computes | No one computes on committed values; users prove properties via ZK | Execution engine computes on ciphertexts |
| Key management | None (user remembers randomness) | Network-wide key (pk/evk/sk) with threshold |
| Phase 1 (hide amounts) | ✅ Sufficient | Overkill |
| Phase 2 (encrypted DEX, etc.) | ❌ Insufficient | ✅ Required |
