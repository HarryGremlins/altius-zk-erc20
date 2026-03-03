# ECIES (Elliptic Curve Integrated Encryption Scheme)

## Overview

In confidential transfer protocols, after updating the receiver's on-chain commitment, the receiver needs to learn the transfer amount `v` and blinding factor `r_tx` to reconstruct their plaintext balance. The problem: how to deliver this secret data through a public blockchain?

**ECIES** solves this by combining elliptic curve Diffie-Hellman (ECDH) key agreement with symmetric authenticated encryption. It is a hybrid encryption scheme — the EC part establishes a shared secret, and a symmetric cipher (AES-GCM) does the actual encryption.

---

## Why Not Encrypt Directly with EC Public Keys?

Unlike RSA, elliptic curve keys **cannot directly encrypt arbitrary data**. EC cryptography is built on the discrete log problem — it provides:

- **Signing** (ECDSA): prove you know the private key
- **Key agreement** (ECDH): two parties derive a shared secret

But there is no native "encrypt a message to a public key" operation. ECIES bridges this gap by turning ECDH key agreement into a full encryption scheme.

---

## The ECDH Foundation

ECDH (Elliptic Curve Diffie-Hellman) allows two parties to derive the same shared secret without transmitting it:

```
Alice has: (a, a·G)    — private key a, public key a·G
Bob has:   (b, b·G)    — private key b, public key b·G

Alice computes: secret = a · (b·G) = ab·G
Bob computes:   secret = b · (a·G) = ab·G
                         ↑ same point!
```

The shared secret is the same EC point `ab·G`. Anyone observing `a·G` and `b·G` on the wire cannot compute `ab·G` — this is the Elliptic Curve Diffie-Hellman Problem (ECDHP).

**Limitation**: standard ECDH uses long-lived keys. If Alice sends multiple messages using the same `(a, b)` pair, they all derive the same shared secret, making the scheme deterministic and vulnerable to replay attacks.

---

## ECIES: ECDH with Ephemeral Keys

ECIES solves the determinism problem by having the sender generate a **fresh ephemeral key pair for every message**. This ensures every encryption produces a unique shared secret.

### Algorithm (step by step)

**Setup**: Bob has a long-lived key pair `(sk, PK)` where `PK = sk·G`. Bob's `PK` is publicly known (registered on-chain in our system).

**Encryption** (Alice → Bob):

```
1. Generate ephemeral key pair
   ek  ← random scalar              // ephemeral private key (used once)
   EPK = ek·G                       // ephemeral public key

2. ECDH key agreement
   S = ek · PK                      // shared secret point = ek · sk · G

3. Key derivation
   (enc_key, mac_key) = KDF(S)      // derive symmetric keys from shared secret

4. Symmetric encryption
   ciphertext = AES-GCM(enc_key, plaintext)
   // AES-GCM provides both encryption and authentication (MAC)

5. Output
   Send: (EPK, ciphertext)          // EPK = ek·G is public, ek is discarded
```

**Decryption** (Bob):

```
1. Recover shared secret
   S = sk · EPK                     // sk · ek·G = ek · sk·G  (same point)

2. Key derivation
   (enc_key, mac_key) = KDF(S)      // same keys as Alice derived

3. Symmetric decryption
   plaintext = AES-GCM.decrypt(enc_key, ciphertext)
   // Also verifies integrity — rejects tampered ciphertext
```

### Why This Works

The critical insight is the ECDH identity:

```
Alice's computation:  ek · PK     = ek · (sk · G) = (ek · sk) · G
Bob's computation:    sk · EPK    = sk · (ek · G) = (sk · ek) · G
                                                     ↑ same point
```

Both parties arrive at the same shared secret `S` without ever transmitting it. An eavesdropper sees `EPK = ek·G` and `PK = sk·G` but cannot compute `ek·sk·G` (ECDHP).

---

## Key Roles in ECIES

| Key | Owner | Lifetime | Purpose |
|-----|-------|----------|---------|
| `sk` | Receiver | Permanent | Long-lived private key; used to decrypt all incoming messages |
| `PK = sk·G` | Receiver | Permanent | Long-lived public key; published so senders can encrypt to this receiver |
| `ek` | Sender | Single-use | Ephemeral private key; generated fresh per message, **discarded after encryption** |
| `EPK = ek·G` | Sender | Single-use | Ephemeral public key; attached to the ciphertext so receiver can recover the shared secret |

**The sender never uses their own long-lived key in ECIES encryption.** The ephemeral key `ek` is the sender's contribution. This means:

- The ciphertext is **not bound to the sender's identity** (no sender authentication by default)
- Every message has a unique shared secret (even same sender → same receiver)
- Compromising `ek` after encryption reveals nothing — it was already discarded

---

## Security Properties

### IND-CCA2 (Ciphertext Indistinguishability)

An attacker who sees `(EPK, ciphertext)` cannot determine which of two possible plaintexts was encrypted. This holds even if the attacker can request decryption of other ciphertexts (adaptive chosen-ciphertext attack).

The ephemeral key is what makes this possible — even if the same plaintext is encrypted twice to the same receiver, the two ciphertexts are completely different because `ek` is different each time.

### Forward Secrecy (Partial)

If Bob's long-lived key `sk` is compromised in the future, an attacker can decrypt all past ECIES messages to Bob (since `S = sk · EPK` can be recomputed from stored `EPK` values). This is **not** forward-secret with respect to the receiver's key.

However, compromising the sender's long-lived key reveals nothing about past ECIES messages, since the sender used ephemeral keys.

### Authentication

Standard ECIES does **not** authenticate the sender — any party who knows Bob's `PK` can encrypt a message to Bob. In our system, sender authentication comes from the blockchain transaction signature (ECDSA), not from ECIES itself.

---

## ECIES vs Other Approaches

| Dimension | ECIES | RSA Encryption | Symmetric-only (shared secret) |
|-----------|-------|---------------|-------------------------------|
| Key type | Elliptic curve | RSA key pair | Pre-shared symmetric key |
| How it works | ECDH + AES hybrid | RSA-OAEP encrypts directly | AES-GCM with shared key |
| Key size for 128-bit security | 256-bit EC | 3072-bit RSA | 128-bit AES |
| Ciphertext overhead | ~33 bytes (compressed EPK) + AES overhead | ≥384 bytes (RSA-3072) | AES overhead only |
| Per-message unique key | ✅ Ephemeral key | ✅ Random padding | ❌ Same key reused |
| On-chain suitability | ✅ Compact | ❌ Large keys and ciphertext | ❌ Requires key exchange channel |

---

## Application in This Project

In the Altius privacy module, ECIES is used to deliver encrypted transfer memos:

```
Confidential transfer: Alice → Bob

1. Alice wraps the transfer metadata: plaintext = (v, r_tx)
   v:    transfer amount
   r_tx: blinding factor for the transfer commitment Com(v, r_tx)

2. Alice encrypts with ECIES:
   - Bob's PK = PK_ecies_bob  (looked up from on-chain registry)
   - ek ← fresh random scalar
   - S = ek · PK_ecies_bob
   - sym_key = KDF(S)
   - ciphertext = AES-GCM(sym_key, [v, r_tx])
   - Attach (ek·G, ciphertext) to the transaction as calldata

3. Bob decrypts:
   - S = sk_ecies_bob · ek·G
   - sym_key = KDF(S)
   - [v, r_tx] = AES-GCM.decrypt(sym_key, ciphertext)
   - Updates local state: balance_new = balance_old + v, r_new = r_old + r_tx
```

### Key Mapping to Our System

| ECIES generic | Our system | Notes |
|---------------|------------|-------|
| Receiver's `sk` | `sk_ecies` | Derived from EVM key: `KDF("ecies" \|\| evm_sk)` |
| Receiver's `PK` | `PK_ecies` | Registered on-chain via `register()` |
| Sender's `ek` | `ek` | Fresh per transaction, discarded after encryption |
| Ephemeral public key | `ek·G` | Stored in transaction calldata (~33 bytes compressed) |
| Shared secret | `secret` | `ek · PK_ecies_receiver` = `sk_ecies_receiver · ek·G` |
| Symmetric key | `sym_key` | `KDF(secret)` |
| Plaintext | `[v, r_tx]` | Transfer amount and blinding factor |

### On-Chain Overhead

Each confidential transfer adds ~80 bytes of calldata:
- `ek·G`: 33 bytes (compressed EC point)
- AES-GCM ciphertext: ~48 bytes (16 bytes data + 12 bytes nonce + 16 bytes auth tag)

### Why ECIES and Not Simpler Alternatives?

1. **No pre-shared key needed**: the sender only needs the receiver's public key (available on-chain)
2. **Compact**: ~80 bytes overhead vs hundreds for RSA
3. **Standard construction**: well-analyzed, used in Ethereum's devp2p (RLPx), SECG SEC 1, and IEEE 1363a
4. **Fits the account model**: each user registers one ECIES public key; any sender can encrypt to any registered receiver
