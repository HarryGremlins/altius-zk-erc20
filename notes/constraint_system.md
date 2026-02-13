# R1CS vs PLONKish Constraint Systems

## Overview

The core idea of ZK proof systems is: **convert a computation into mathematical constraints, then prove you know inputs satisfying all constraints without revealing those inputs.**

Different proof systems use different constraint representations. The two most prominent are **R1CS** and **PLONKish**.

---

## From Computation to Constraints

Regardless of the constraint system, the pipeline is:

```
High-level code (Circom / Noir)
    ↓ compile
Arithmetic constraints (R1CS / PLONKish gates)
    ↓ proving system (Groth16 / PLONK)
ZK Proof
    ↓ verify
On-chain verifier contract
```

All computations — hashing, signature verification, balance checks — must ultimately be decomposed into additions and multiplications over a finite field.

---

## R1CS (Rank-1 Constraint System)

### Constraint Form

Each constraint takes the form:

```
⟨a, w⟩ × ⟨b, w⟩ = ⟨c, w⟩
```

Where `w` is the vector of all variables (witness), and `a`, `b`, `c` are coefficient vectors.

Simplified: **each constraint can only express one multiplication**.

### Example

Computing `x³`:

```
// Cannot write x * x * x = y (two multiplications)
// Must decompose into single multiplications:

x * x = t       // Constraint 1: introduce intermediate variable t
t * x = y       // Constraint 2: t × x = x³
```

Computing `(a + b) * c`:

```
(a + b) * c = d  // Valid — the parenthesized part is a linear combination (addition is free)
```

### Characteristics

- **Addition is free**: linear combinations don't consume constraints
- **Each multiplication costs one constraint**: this is the core metric for circuit size
- Proof systems: **Groth16** (most common), Marlin, etc.

### Users

- Circom compiles to R1CS
- snarkjs / rapidsnark as provers

---

## PLONKish Constraint System

### Constraint Form

Each gate has the general expression:

```
q_L·a + q_R·b + q_O·c + q_M·(a·b) + q_C = 0
```

- `a`, `b`, `c`: the gate's input/output wires
- `q_L`, `q_R`, `q_O`, `q_M`, `q_C`: selectors (fixed at compile time by the circuit)

### Expressing Different Operations via Selectors

**Addition gate** (`a + b = c`):

```
q_L=1, q_R=1, q_O=-1, q_M=0, q_C=0
→ 1·a + 1·b + (-1)·c + 0 + 0 = 0
→ a + b = c
```

**Multiplication gate** (`a × b = c`):

```
q_L=0, q_R=0, q_O=-1, q_M=1, q_C=0
→ 0 + 0 + (-1)·c + 1·(a·b) + 0 = 0
→ a·b = c
```

**Constant gate** (`a = 5`):

```
q_L=1, q_R=0, q_O=0, q_M=0, q_C=-5
→ 1·a + 0 + 0 + 0 + (-5) = 0
→ a = 5
```

### Extended Capabilities

PLONKish also supports:

- **Custom gates**: define more complex gates, e.g., one gate performing both addition and multiplication
- **Lookup tables**: verify by table lookup (e.g., range check `0 ≤ x < 2^64`) instead of decomposing into per-bit constraints
- **Copy constraints**: use permutation arguments to ensure wire values are consistent across different gates

### Users

- Noir compiles to ACIR, backend Barretenberg uses UltraPlonk/UltraHonk
- Halo2 is also PLONKish

---

## Comparison

| Dimension | R1CS | PLONKish |
|-----------|------|----------|
| Constraint form | `a × b = c` (one multiplication only) | `q_L·a + q_R·b + q_O·c + q_M·(a·b) + q_C = 0` |
| Expressiveness | Limited, multiplication gates only | Flexible, supports custom gates |
| Addition | Free (linear combination) | Also free, or use a dedicated addition gate |
| Lookup tables | ❌ Not supported | ✅ Supported (huge advantage for range checks) |
| Trusted setup | Groth16 requires per-circuit setup | PLONK uses universal SRS, one-time setup |
| Proof size | Groth16 is minimal (~128 bytes) | Larger (~2-4 KB) |
| Verification cost | Groth16 is cheap (~200k gas) | Higher (~300-500k gas) |
| Representative systems | Groth16, Marlin | PLONK, UltraPlonk, Halo2 |

---

## The Power of Lookup Tables

A core advantage of PLONKish over R1CS is the **lookup argument**.

Consider range checking `0 ≤ x < 2^64`:

**R1CS approach**: decompose x into 64 bits, constrain each bit to 0 or 1:

```
b_i * (1 - b_i) = 0    // For each bit, 64 constraints total
x = Σ b_i * 2^i         // Plus the recomposition constraint
// Total: ~64+ constraints
```

**PLONKish + Lookup approach**: precompute a table `{0, 1, 2, ..., 2^16 - 1}`, decompose x into four 16-bit limbs, verify each limb via table lookup:

```
x = limb_0 + limb_1 * 2^16 + limb_2 * 2^32 + limb_3 * 2^48
// 4 lookups + 1 recomposition constraint, far more efficient than 64 constraints
```

---

## Choice for This Project

This project uses **Noir + PLONKish (UltraPlonk/UltraHonk)** for the following reasons:

1. **Lookup tables**: massive benefits for range checks, Poseidon optimizations, etc.
2. **Universal SRS**: no per-circuit trusted setup, more convenient for iterative development
3. **Custom gates**: can define specialized gates for high-frequency operations like Poseidon
4. **Industry direction**: PLONKish is the mainstream evolution path in the ZK space
