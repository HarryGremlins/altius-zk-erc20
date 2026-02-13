# Pedersen Commitment vs Poseidon Hash Commitment

## Overview

In ZK privacy protocols, a commitment is used to hide the contents of a UTXO (value, owner, etc.) while ensuring the contents cannot be tampered with. The two most common approaches are **Pedersen Commitment** and **Poseidon Hash Commitment**.

---

## Pedersen Commitment

### How It Works

Based on the discrete logarithm hardness assumption over elliptic curves:

```
C = v·G + r·H
```

- `v`: the committed value (e.g., transfer amount)
- `r`: random blinding factor, ensures hiding
- `G`, `H`: two independent generator points on the elliptic curve, where the discrete log of `H` with respect to `G` is unknown

### Core Property: Additive Homomorphism

```
C(v1, r1) + C(v2, r2) = (v1·G + r1·H) + (v2·G + r2·H)
                       = (v1 + v2)·G + (r1 + r2)·H
                       = C(v1 + v2, r1 + r2)
```

This means **balance conservation can be verified directly on-chain** without a ZK proof:

```
C_input1 + C_input2 == C_output1 + C_output2
// Just do EC point addition in the smart contract
```

### Security Properties

- **Hiding**: due to the random blinding factor `r`, it is infeasible to recover `v` from `C`
- **Binding**: after committing, it is infeasible to find another `(v', r')` such that `C(v, r) = C(v', r')` (based on the discrete log assumption)

### Cost in ZK Circuits

Elliptic curve scalar multiplication requires many field operations:

- One 256-bit scalar multiplication ≈ thousands of constraints
- Significant impact on circuit size

---

## Poseidon Hash Commitment

### How It Works

Uses the ZK-friendly hash function Poseidon to construct the commitment:

```
C = Poseidon(value, owner_pubkey, randomness)
```

- `value`: the committed value
- `owner_pubkey`: the UTXO owner's public key
- `randomness`: random nonce for hiding

### Core Property: ZK-Native

Poseidon's internal operations are entirely field additions and multiplications (`x^5 + constants`), which directly correspond to native ZK circuit operations:

```
// Poseidon S-box in one round
x → x^5  // Only 3 multiplication constraints

// Full Poseidon hash ≈ 250 constraints
```

### No Homomorphism

```
Poseidon(a, ...) + Poseidon(b, ...) ≠ Poseidon(a + b, ...)
```

Therefore, balance conservation **must be proven inside the ZK circuit**:

```rust
// Noir pseudocode
fn transfer(input_values: [u64; 2], output_values: [u64; 2]) {
    assert(input_values[0] + input_values[1] == output_values[0] + output_values[1]);
}
```

### Security Properties

- **Hiding**: with randomness, it is infeasible to recover the preimage from the hash
- **Binding**: based on Poseidon's collision resistance

---

## Comparison

| Dimension | Pedersen Commitment | Poseidon Hash Commitment |
|-----------|-------------------|------------------------|
| Math foundation | Elliptic curve discrete log | ZK-friendly hash function |
| Additive homomorphism | ✅ Yes | ❌ No |
| Balance verification | On-chain EC point addition | Inside ZK circuit |
| In-circuit cost | High (thousands of constraints) | Low (~250 constraints) |
| Design complexity | Hybrid model (on-chain EC + in-circuit hash) | Pure hash, unified architecture |
| Noir ecosystem support | `std::hash::pedersen_commitment` | `std::hash::poseidon`, Aztec's preferred choice |

---

## Choice for This Project

This project uses **Poseidon Hash Commitment** for the following reasons:

1. **Circuit efficiency**: Poseidon is extremely cheap in-circuit, leading to faster proof generation
2. **Unified architecture**: commitment, nullifier, and Merkle tree all based on Poseidon — consistent and simple
3. **Ecosystem fit**: Noir + Barretenberg's PLONKish constraint system is a natural match for Poseidon
4. **Design philosophy**: since we fully embrace ZK, we don't need Pedersen's on-chain homomorphic verification — everything is proven inside the circuit

> Note: Pedersen commitment's additive homomorphism ≠ homomorphic encryption (FHE). The former only means commitments can be added together; the latter allows arbitrary computation on ciphertexts. This project rejects FHE, but Pedersen commitment itself would not violate that principle.
