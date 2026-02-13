# Circom vs Noir

## Overview

Circom and Noir are the two most mainstream ZK circuit development languages today. Circom is the battle-tested veteran; Noir is the rapidly rising newcomer. They differ fundamentally in positioning, design philosophy, and developer experience.

---

## Circom

### Basic Info

- Developed by the iden3 team
- A fully independent DSL (Domain-Specific Language), not based on any existing language
- Compilation target: **R1CS** (Rank-1 Constraint System)
- Proving backend: **Groth16** (most common), PLONK (via snarkjs)

### Syntax Example

```circom
pragma circom 2.0.0;

include "circomlib/poseidon.circom";
include "circomlib/comparators.circom";

template Transfer() {
    signal input balance;
    signal input amount;
    signal output newBalance;

    // Manually define the constraint
    newBalance <== balance - amount;

    // Range check: amount <= balance
    component check = LessEqThan(64);
    check.in[0] <== amount;
    check.in[1] <== balance;
    check.out === 1;
}

component main = Transfer();
```

### Core Concepts

- **signal**: variables (wires) in the circuit — `input`, `output`, or intermediate signals
- **template**: reusable circuit modules (similar to functions/classes)
- **component**: an instantiation of a template
- **Constraint operators**:
  - `<==`: assign and constrain
  - `===`: constrain only (no assignment)
  - `<--`: assign only (⚠️ dangerous — does not generate constraints, a common source of vulnerabilities)

### Toolchain

- `circom`: compiler, outputs R1CS + WASM/C++ witness generator
- `snarkjs`: JavaScript-based prover/verifier
- `rapidsnark`: high-performance C++ prover
- `circomlib`: official standard library (Poseidon, MerkleTree, EdDSA, comparators, etc.)
- Trusted setup: requires Powers of Tau + per-circuit Phase 2 ceremony via snarkjs

### Production Projects

- Tornado Cash
- Semaphore (Ethereum anonymous signaling protocol)
- WorldID (Worldcoin's proof of personhood)
- zkEmail

---

## Noir

### Basic Info

- Developed by **Aztec Labs**
- Syntax closely resembles Rust, but is an independent language with its own compiler
- Compilation target: **ACIR** (Abstract Circuit Intermediate Representation)
- Proving backend: **Barretenberg** (UltraPlonk / UltraHonk)

### Syntax Example

```rust
use std::hash::poseidon;

fn main(
    balance: u64,
    amount: u64,
    expected_commitment: pub Field,
    secret: Field,
) {
    // Balance check — just normal code
    assert(amount <= balance);
    let new_balance = balance - amount;

    // Compute commitment
    let commitment = poseidon::bn254::hash_2([new_balance as Field, secret]);
    assert(commitment == expected_commitment);
}

#[test]
fn test_transfer() {
    // Built-in testing support
    let secret = 12345;
    let commitment = poseidon::bn254::hash_2([50, secret]);
    main(100, 50, commitment, secret);
}
```

### Core Concepts

- **Regular variables**: declare variables just like Rust; the compiler automatically generates constraints
- **`pub` keyword**: marks a parameter as public input; all other parameters default to private witness
- **`assert`**: generates constraints — the compiler converts assert conditions into circuit constraints
- **Type system**: supports `u8`/`u16`/`u32`/`u64`, `Field`, `bool`, struct, array, etc.
- **Standard library**: built-in Poseidon, Pedersen, ECDSA, Merkle tree, SHA-256, etc.

### Toolchain

- `nargo`: all-in-one CLI (compile, execute, test, prove, verify)
- `nargo test`: built-in test framework
- `nargo prove` / `nargo verify`: generate and verify proofs
- Package manager: supports GitHub dependencies

### Production Projects

- Aztec Protocol (Aztec Labs' own L2 privacy protocol)
- Growing open-source ecosystem with many hackathon projects

---

## Developer Experience Comparison

### Proving "x squared equals y"

**Circom**:

```circom
template Square() {
    signal input x;
    signal output y;
    y <== x * x;
}
component main {public [y]} = Square();
```

**Noir**:

```rust
fn main(x: Field, y: pub Field) {
    assert(x * x == y);
}
```

### Merkle Proof Verification

**Circom**:

```circom
include "circomlib/poseidon.circom";

template MerkleProof(depth) {
    signal input leaf;
    signal input pathElements[depth];
    signal input pathIndices[depth];
    signal output root;

    component hashers[depth];
    signal hashes[depth + 1];
    hashes[0] <== leaf;

    for (var i = 0; i < depth; i++) {
        hashers[i] = Poseidon(2);
        // Select left/right based on path index
        hashers[i].inputs[0] <== hashes[i] + pathIndices[i] * (pathElements[i] - hashes[i]);
        hashers[i].inputs[1] <== pathElements[i] + pathIndices[i] * (hashes[i] - pathElements[i]);
        hashes[i + 1] <== hashers[i].out;
    }

    root <== hashes[depth];
}
```

**Noir**:

```rust
use std::hash::poseidon;

fn compute_root(leaf: Field, path: [Field; 32], indices: [u1; 32]) -> Field {
    let mut current = leaf;
    for i in 0..32 {
        let (left, right) = if indices[i] == 0 {
            (current, path[i])
        } else {
            (path[i], current)
        };
        current = poseidon::bn254::hash_2([left, right]);
    }
    current
}
```

---

## Security Comparison

### Circom's Classic Pitfall: Under-Constrained Bugs

```circom
template Dangerous() {
    signal input a;
    signal input b;
    signal output c;

    // ⚠️ <-- assigns only, no constraint!
    // A malicious prover can set c to any value
    c <-- a * b;

    // Correct version should be:
    // c <== a * b;
}
```

`<--` assigns a value without generating a constraint. This is the most dangerous operation in Circom. The compiler won't flag it, but the proof system has a soundness hole. Early versions of Tornado Cash had this class of bug.

### Noir's Safety Advantages

- No "assign without constrain" operation — the compiler automatically generates constraints for all computations
- Strong type system catches type mismatches at compile time
- Built-in overflow checks (`u64` operations automatically include range constraints)
- Developers still need to ensure logical correctness (no language can fully prevent logic bugs in ZK circuits)

---

## Comparison

| Dimension | Circom | Noir |
|-----------|--------|------|
| Language type | Independent DSL | Rust-like independent language |
| Abstraction level | Low (manual signals and constraints) | High (feels like writing normal code) |
| Constraint system | R1CS | PLONKish (ACIR) |
| Proving backend | Groth16 (default) | UltraPlonk / UltraHonk |
| Trusted setup | Per-circuit ceremony | Universal SRS |
| Proof size | ~128 bytes | ~2-4 KB |
| On-chain verification gas | ~200k | ~300-500k |
| Development speed | Slow, manual constraint management | Fast, close to writing Rust |
| Testing | Requires external tools | Built-in `nargo test` |
| Package management | Manual includes | Built-in package manager |
| Security risk | `<--` under-constrained bugs | Compiler catches more issues |
| Ecosystem maturity | High (5+ years, many production projects) | Medium (growing rapidly) |
| Standard library | circomlib (mature) | std (continuously improving) |

---

## Choice for This Project

This project uses **Noir** for the following reasons:

1. **Development speed**: Rust-like syntax, low learning curve, fast iteration
2. **Safety**: compiler auto-generates constraints, reducing under-constrained risks
3. **PLONKish advantages**: lookup tables provide huge benefits for range checks and similar operations
4. **No per-circuit setup**: modifying circuits during development doesn't require re-running a trusted setup
5. **Aztec ecosystem**: Noir is Aztec's core language, ensuring long-term maintenance
6. **Maintainability**: the project will undergo continuous iteration (Parallel EVM optimizations, etc.), and Noir code is easier to maintain
