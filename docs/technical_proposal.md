# Altius Privacy Module — Technical Proposal

## 1. Project Goals

Implement a modular privacy layer for the Altius Execution Engine that enables confidential token transfers while maintaining institutional compliance:

- **Confidential transfers**: hide transfer amounts on-chain while keeping sender/recipient visible
- **Phased approach**: Phase 1 uses Pedersen Commitment + ZK Range Proofs; Phase 2 introduces FHE for general-purpose confidential computation
- **Institutional auditability**: native viewing key support for compliance and regulatory oversight
- **Toggleable privacy**: users can seamlessly transition between public and shielded states

## 2. Core Design Decisions

### 2.1 Account Model with Wrapped Accounts

This project adopts the **account model** with a wrapped account architecture rather than the UTXO model.

**Rationale**: UTXO/mixer-style models carry market perception risks associated with money laundering (e.g., Tornado Cash precedent). The account model aligns naturally with institutional compliance standards and existing EVM account abstractions.

**Wrapped Account Flow**:

```
A  ──wrap──▶  A'  ──confidential transfer──▶  B'  ──unwrap──▶  B
(public)    (shielded)                      (shielded)        (public)
```

- **Wrap** (A → A'): convert public tokens into a shielded balance — visible on-chain
- **Confidential Transfer** (A' → B'): transfer between shielded accounts — amount hidden
- **Unwrap** (B' → B): convert shielded balance back to public tokens — visible on-chain

**Limited Privacy**: since wrap and unwrap are transparent, this provides **amount confidentiality** rather than full anonymity. This is an accepted trade-off for Phase 1, prioritizing compliance compatibility over maximum privacy.

### 2.2 Phase 1 Tech: Pedersen Commitment + ZK Range Proofs

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Balance representation | **Pedersen Commitment** | Additive homomorphism enables on-chain balance updates without decryption |
| Validity proofs | **ZK Range Proofs** | Sender knows plaintext, proves balance sufficiency and amount positivity off-chain |
| Range proof scheme | **TBD (Bulletproofs or equivalent)** | No trusted setup, compact proofs (~600 bytes) |

**Why not FHE for Phase 1?**

For simple transfers, the sender knows all plaintext values (their balance, the transfer amount). The sender can generate a ZK proof off-chain; the execution engine only needs to verify the proof and perform elliptic curve point arithmetic on commitments. FHE is unnecessary when a single party holds all the plaintext — it becomes essential only when the execution engine must compute on encrypted data from multiple parties without any party revealing their inputs (see Phase 2).

### 2.3 Execution Environment

The privacy module runs on the **Altius Execution Engine** — not native Ethereum. The execution engine provides:

- Precompiled opcodes for elliptic curve operations (EC point addition, scalar multiplication)
- Precompiled opcodes for ZK proof verification
- Future: FHE-related precompiled opcodes (Phase 2)

## 3. System Architecture

### 3.1 Dual-Balance Model

Each user has two types of balance:

- **Public balance**: standard ERC20 `balanceOf`, supports regular `transfer`/`approve`/`transferFrom`
- **Shielded balance**: stored as a Pedersen commitment `Com(balance, r) = balance·G + r·H`, visible only to the owner

The two balances are interconvertible:

```
Public balance ←→ Shielded balance (Pedersen Commitment)

wrap:   balanceOf[msg.sender] -= amount → shielded commitment created
unwrap: spend shielded balance (proof) → balanceOf[recipient] += amount
```

### 3.2 Overall Structure

```
┌─────────────────────────────────────────────────────────────┐
│                      User / Frontend                        │
│        (construct tx, generate ZK range proof)              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Altius Execution Engine                         │
│          — with privacy precompiled opcodes —                │
│                                                             │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │ ERC20 Standard│  │ Shielded Acct  │  │ ZK Verifier    │  │
│  │ (balanceOf,   │  │ Management     │  │ (Range Proof   │  │
│  │  transfer,    │  │ (Pedersen      │  │  Verification) │  │
│  │  approve)     │  │  Commitments)  │  │                │  │
│  └───────────────┘  └────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Core Components

#### Pedersen Commitment

Each shielded balance is stored as a Pedersen commitment:

```
Com(v, r) = v·G + r·H
```

- `v`: the balance value
- `r`: random blinding factor (known only to the owner)
- `G`, `H`: two independent elliptic curve generator points (discrete log relationship unknown)

**Additive homomorphism** enables balance updates without decryption:

```
Com(a, r1) + Com(b, r2) = Com(a + b, r1 + r2)

// On-chain balance update for transfer of amount v:
com_sender'   = com_sender   - Com(v, r_v)   // EC point subtraction
com_receiver' = com_receiver + Com(v, r_v)   // EC point addition
// The execution engine never sees the plaintext values
```

**Security properties**:
- **Hiding**: given `Com(v, r)`, it is infeasible to recover `v` (due to the random blinding factor `r`)
- **Binding**: it is infeasible to find `(v', r')` such that `Com(v, r) = Com(v', r')` (based on the discrete log assumption)

#### ZK Range Proofs

The sender generates a ZK proof off-chain to prove transaction validity:

1. **Balance sufficiency**: `balance >= amount` (the sender has enough funds)
2. **Amount positivity**: `amount > 0` (prevents zero or negative transfers)

The proof is submitted on-chain and verified by the execution engine. The verifier learns nothing about the actual balance or amount — only that the constraints are satisfied.

**Candidate: Bulletproofs**
- No trusted setup required
- Proof size: ~600 bytes for a 64-bit range proof
- Verification: logarithmic in the range size
- Battle-tested in Monero (since 2018) and Penumbra

#### Encrypted Memo (ECIES)

When a confidential transfer occurs, the receiver's commitment is updated on-chain, but the receiver does not know the transfer amount `v` or the randomness `r_v`. Without these values, the receiver cannot compute their new balance or generate proofs for future transactions.

**Solution**: the sender encrypts `(v, r_v)` using **ECIES** (Elliptic Curve Integrated Encryption Scheme) and attaches it to the transaction as calldata:

```
Sender encrypts memo to Receiver:
  1. Generate ephemeral key pair: (ek, ek·G)
  2. ECDH shared secret:  secret = ek · PK_receiver
  3. Derive symmetric key: sym_key = KDF(secret)
  4. Encrypt memo:         ciphertext = AES-GCM(sym_key, [v, r_v])
  5. Attach to tx:         (ek·G, ciphertext)

Receiver decrypts:
  1. ECDH shared secret:  secret = sk_receiver · ek·G     // same value
  2. Derive symmetric key: sym_key = KDF(secret)
  3. Decrypt memo:         [v, r_v] = AES-GCM.decrypt(sym_key, ciphertext)
  4. Update local state:   balance_new = balance_old + v, r_new = r_old + r_v
```

On-chain overhead: ~80 bytes (32-byte compressed EC point + ~48-byte AES-GCM ciphertext).

#### Key Derivation Hierarchy

The system involves multiple secrets (EVM signing key, ECIES encryption key, Pedersen randomness), but the user only needs to manage **a single private key**. All other secrets are deterministically derived:

```
EVM Private Key (single master seed)
  │
  ├── ECIES Private Key = KDF("ecies" || evm_sk)
  │     Used for: encrypting/decrypting transfer memos
  │     Derived once, reused across all transactions
  │
  └── Pedersen Randomness = KDF("pedersen" || evm_sk || nonce)
        Used for: blinding factor r in each commitment
        Derived per-transaction using an incrementing nonce
```

**Recovery**: if the user has their EVM private key + on-chain transaction history, they can reconstruct all ECIES keys and Pedersen randomness values, fully recovering their shielded balance state.

#### Viewing Keys

For institutional compliance, users can grant auditors the ability to inspect their transactions:

- The owner shares `(value, randomness)` pairs for specific commitments, allowing the auditor to verify `Com(v, r) = v·G + r·H`
- Alternatively, sharing the ECIES private key allows the auditor to decrypt all transfer memos and reconstruct the full transaction history
- Viewing keys are **opt-in** and **granular** — users control exactly what is disclosed and to whom

## 4. Transaction Types

### 4.1 Standard ERC20 Operations

The contract is a fully compliant ERC20, supporting all standard operations: `transfer`, `approve`, `transferFrom`, etc. These only involve the public balance and require no ZK proof.

### 4.2 Wrap (Public Balance → Shielded Balance)

User converts public tokens into the shielded layer.

```
balanceOf[msg.sender] -= amount
→ new Pedersen commitment Com(amount, r) added to sender's shielded balance
No ZK proof required (amount is known from the public input)
```

The sender provides the commitment `Com(amount, r)`. The contract verifies the commitment is well-formed (or trusts the precompile) and records it.

### 4.3 Confidential Transfer (Shielded → Shielded)

Transfer between shielded accounts with the amount hidden.

```
Input:  sender's current shielded commitment + ZK range proof + transfer commitment
Output: updated sender commitment + updated receiver commitment

On-chain operations:
  com_sender'   = com_sender   - com_transfer    // EC subtraction
  com_receiver' = com_receiver + com_transfer    // EC addition
  verify(range_proof) == true                    // ZK verification
```

**What the ZK proof proves**:
- The sender knows `(balance, r)` that opens `com_sender`
- `balance >= transfer_amount`
- `transfer_amount > 0`
- `com_transfer = Com(transfer_amount, r_transfer)` for the claimed transfer commitment

**What is visible on-chain**: sender address, receiver address, updated commitments
**What is hidden**: transfer amount, sender's balance, receiver's balance

### 4.4 Unwrap (Shielded Balance → Public Balance)

Convert shielded balance back to public tokens.

```
Spend shielded balance (commitment + ZK proof)
→ balanceOf[recipient] += amount
Amount is revealed during unwrap (public balance is transparent)
```

## 5. Contract Design

### 5.1 Core Interface

```solidity
contract ConfidentialERC20 is ERC20 {

    // --- Standard ERC20 ---
    // transfer(), approve(), transferFrom(), balanceOf() inherited from ERC20

    // --- Public balance → Shielded balance ---
    function wrap(uint256 amount, bytes32 commitment) external;

    // --- Confidential transfer (shielded → shielded) ---
    function confidentialTransfer(
        address recipient,
        bytes32 transferCommitment,
        bytes calldata rangeProof,
        bytes calldata encryptedMemo   // ECIES-encrypted (value, randomness) for recipient
    ) external;

    // --- Shielded balance → Public balance ---
    function unwrap(
        uint256 amount,
        bytes calldata proof
    ) external;
}
```

### 5.2 Contract Storage

```solidity
contract ConfidentialERC20 is ERC20 {
    // Shielded balances: address → Pedersen commitment (stored as EC point)
    mapping(address => bytes32) public shieldedBalance;

    // ZK range proof verifier (precompile or deployed contract)
    address public verifier;
}
```

## 6. Phase 2 Roadmap: FHE Integration

### 6.1 When FHE is Needed

FHE becomes necessary when the **execution engine must compute on encrypted data from multiple parties** — scenarios where no single party holds all the plaintext:

| Scenario | Why FHE | Why Pedersen+ZK is Insufficient |
|----------|---------|--------------------------------|
| Encrypted DEX order matching | Engine compares encrypted bid/ask prices from different users | Neither user knows the other's price; neither can generate a ZK proof about the comparison |
| Encrypted liquidation checks | Engine evaluates `collateral × price < threshold` on encrypted data | User doesn't want to reveal collateral amount; oracle provides price — multi-party inputs |
| Confidential auctions | Engine determines winner from encrypted bids | No bidder should see others' bids |

### 6.2 FHE Scheme Candidates

| Scheme | Operation Types | Performance | Use Case Fit |
|--------|----------------|-------------|--------------|
| **TFHE** | Boolean/integer ops, fast bootstrapping | Best for arbitrary integer computation | Leading candidate (Zama/fhEVM) |
| **BGV/BFV** | Batched integer arithmetic | Good for bulk operations | Possible for batch settlement |
| **CKKS** | Approximate arithmetic | Fast but inexact | Not suitable for precise balances |

Final FHE scheme selection is deferred to Phase 2 planning.

### 6.3 Key Management

FHE requires a network-wide key set:

- **Public Key (pk)**: shared by all users for encryption
- **Evaluation Key (evk)**: used by the execution engine for homomorphic computation
- **Secret Key (sk)**: split via threshold secret sharing among validators (t-of-n)

No single validator can decrypt. Users read their own balances via proxy re-encryption (network re-encrypts to user's personal key).

### 6.4 Dual-Track Architecture (Phase 2)

When FHE is introduced, each shielded account stores both representations:

```
Shielded Account {
    ct_balance:  FHE.Enc(balance)              // FHE track: execution engine computation
    com_balance: PedersenCommit(balance, r)     // ZK track: range proofs and validity
}
```

The "commitment bridge" ensures consistency between the two tracks. Details to be designed in Phase 2.

## 7. Tech Stack Summary

| Layer | Phase 1 | Phase 2 |
|-------|---------|---------|
| Balance Representation | Pedersen Commitment | Pedersen + FHE Ciphertext |
| Validity Proofs | ZK Range Proofs (Bulletproofs or equivalent) | ZK Range Proofs + FHE computation |
| Commitment Scheme | Pedersen (`v·G + r·H`) | Pedersen + FHE scheme TBD |
| Data Model | Account (wrapped accounts) | Account (dual-track) |
| Execution | Altius Engine + EC precompiles | Altius Engine + EC + FHE precompiles |
| Compliance | Viewing Keys | Viewing Keys + Proxy Re-Encryption |
