# ZK ERC20 Technical Proposal

## 1. Project Goals

Implement a zero-knowledge proof enabled ERC20 token protocol with the following capabilities:

- **Optional privacy**: senders can independently choose to hide or reveal the transfer amount and the recipient
- **Pure ZK approach**: all privacy features are built on zero-knowledge proofs, no homomorphic encryption (FHE)
- **Parallel EVM friendly**: architecture designed with future Parallel EVM executor optimizations in mind

## 2. Core Design Decisions

### 2.1 UTXO Model (Not Account Model)

This project adopts the UTXO (Unspent Transaction Output) model rather than the account model for the following reasons:

**Parallelism**: under the account model, multiple transactions targeting the same recipient compete for the same state slot, causing ZK proofs generated against stale state to become invalid. In the UTXO model, each UTXO is an independent state object with no shared state between transactions — naturally parallelizable.

```
Account model: A → C, B → C compete for C.balance → proof collision
UTXO model:    A creates UTXO_c1, B creates UTXO_c2 → no interference
```

**Privacy**: the UTXO + Nullifier pattern is a natural fit for privacy — when spending a UTXO, a nullifier marks it as spent without revealing which specific UTXO was consumed.

### 2.2 ZK Stack: Noir + Barretenberg

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Circuit language | **Noir** | Rust-like syntax, high development efficiency, compiler auto-generates constraints |
| Constraint system | **PLONKish (ACIR)** | Supports lookup tables, highly expressive |
| Proving backend | **Barretenberg (UltraPlonk/UltraHonk)** | No per-circuit trusted setup, uses universal SRS |

## 3. System Architecture

### 3.1 Dual-Balance Model

The contract **is itself an ERC20 token** with a built-in privacy layer. Each user has two types of balance:

- **Public balance**: standard ERC20 `balanceOf`, supports regular `transfer`/`approve`/`transferFrom`
- **Private balance**: exists as UTXOs in a Merkle tree, visible only to the owner

The two balances are interconvertible:

```
Public balance ←→ Private balance (UTXO)

shield:   balanceOf[msg.sender] -= amount → new UTXO commitment created
unshield: spend UTXO (nullifier + proof) → balanceOf[recipient] += amount
```

### 3.2 Overall Structure

```
┌─────────────────────────────────────────────────────┐
│                   User / Frontend                   │
│          (construct tx, generate ZK proof)          │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│            ZK ERC20 Contract (Solidity)             │
│            — is itself an ERC20 token —             │
│                                                     │
│  ┌───────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ ERC20 Standard│  │ UTXO Mgmt│  │ ZK Verifier  │  │
│  │ (balanceOf,   │  │(Merkle   │  │(Noir         │  │
│  │  transfer,    │  │ Tree +   │  │ Verifier)    │  │
│  │  approve)     │  │Nullifier)│  │              │  │
│  └───────────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 3.3 Core Components

#### UTXO (Note)

Each UTXO contains:

```
Note {
    value: u64,           // amount
    owner: Field,         // owner's public key (or its hash)
    randomness: Field,    // random blinding factor
}
```

What is stored on-chain is the commitment of the Note; the original data is known only to the owner. Commitment scheme TBD (Poseidon Hash or Pedersen Commitment).

#### Nullifier

When spending a UTXO, a nullifier is revealed publicly. The contract checks that it has not been used before. The nullifier cannot be linked back to a specific commitment, achieving spending privacy.

#### Merkle Tree (Commitment Tree)

- An on-chain append-only Merkle tree of fixed depth (e.g., depth 20, supporting ~1 million UTXOs)
- Append-only — no deletions (spent UTXOs are tracked via the nullifier set)

## 4. Transaction Types

### 4.1 Standard ERC20 Operations

The contract is a fully compliant ERC20, supporting all standard operations: `transfer`, `approve`, `transferFrom`, etc. These only involve the public balance and require no ZK proof.

### 4.2 Shield (Public Balance → Private UTXO)

User converts their public balance into the privacy layer.

```
balanceOf[msg.sender] -= amount
→ new commitment inserted into Merkle tree
No ZK proof required (input comes from public balance)
```

### 4.3 Private Transfer (UTXO → UTXO)

Spend one or more existing UTXOs to create new UTXOs for the recipient and change back to self.

```
Input:  nullifier(s) + ZK proof
Output: new commitment(s) inserted into Merkle tree
```

**Privacy dimensions are independently selectable**:

| Amount | Recipient | Description |
|--------|-----------|-------------|
| Hidden | Hidden | Fully private — only commitment + nullifier on-chain |
| Public | Hidden | Plaintext value on-chain, recipient not visible |
| Hidden | Public | Plaintext recipient on-chain, amount not visible |
| Public | Public | Fully transparent, but still goes through UTXO flow |

The two privacy dimensions (amount, recipient) are **independent** — the sender can freely combine them. Implementation: optional plaintext fields in transaction calldata, with consistency constraints in the ZK circuit (if choosing to disclose, prove that the public value matches the private value).

### 4.4 Unshield (Private UTXO → Public Balance)

Spend a private UTXO and convert it back to public balance.

```
Spend UTXO (nullifier + ZK proof)
→ balanceOf[recipient] += amount
```

## 5. Contract Design (Solidity)

### 5.1 Core Interface

```solidity
contract ZkERC20 is ERC20 {

    // --- Standard ERC20 ---
    // transfer(), approve(), transferFrom(), balanceOf() inherited from ERC20

    // --- Public balance → Private UTXO ---
    function shield(uint256 amount, bytes32 commitment) external;

    // --- Private transfer (UTXO → UTXO) ---
    function privateTransfer(
        bytes32[] calldata nullifiers,
        bytes32[] calldata newCommitments,
        bytes calldata proof
    ) external;

    // --- Private UTXO → Public balance ---
    function unshield(
        uint256 amount,
        address recipient,
        bytes32 nullifier,
        bytes calldata proof
    ) external;
}
```

### 5.2 Contract Storage

```solidity
contract ZkERC20 is ERC20 {
    // Merkle tree for commitments
    mapping(uint256 => bytes32) public filledSubtrees;
    mapping(uint256 => bytes32) public roots;
    uint32 public nextLeafIndex;

    // Nullifier set
    mapping(bytes32 => bool) public nullifierUsed;

    // Noir proof verifier (auto-generated)
    IVerifier public verifier;
}
```

## 6. Future Optimizations

### 6.1 Parallel EVM Optimization

Natural advantages of the UTXO model:

- Transactions read/write non-overlapping state (independent nullifiers + new commitments)
- Merkle tree append operations can be batched
- Compatible with access-list pre-declaration for parallel scheduling

Potential optimizations:
- Batch proof verification (verify multiple proofs in a single call)
- Merkle tree sharding (multiple subtrees with parallel appends)
- Nullifier set bucketing (reduce write contention)

### 6.2 Other

- Recursive proofs: aggregate multiple transaction proofs into one
- Compliance interface: optional view key mechanism allowing auditors to inspect specific transactions
- Cross-chain bridging: cross-chain transfer of private UTXOs

## 7. Tech Stack Summary

| Layer | Technology |
|-------|------------|
| ZK Circuit | Noir |
| Proving System | Barretenberg (UltraPlonk / UltraHonk) |
| Commitment Scheme | TBD (Poseidon Hash or Pedersen Commitment) |
| Data Model | UTXO (Note) + Nullifier |
| Smart Contract | Solidity (ERC20) |
| Merkle Tree | On-chain append-only, depth TBD |
