# ADR 001: Two-Party ECDSA Protocol Selection

## Header

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | ~2018 (project inception) |
| Deciders | ZenGo-X team |
| Impact | Core protocol - affects entire architecture |

---

## Context

Cryptocurrency wallet security requires protecting private keys from theft or compromise. Traditional approaches have significant drawbacks:

1. **Single-key wallets**: Complete loss if key is compromised
2. **Hardware wallets**: Physical device required, limited programmability
3. **Multi-signature (on-chain)**: Higher transaction fees, visible threshold structure, limited blockchain support

The project needed a cryptographic protocol that:
- Eliminates single points of failure
- Produces standard ECDSA signatures (blockchain-agnostic)
- Works with mobile devices (limited computational power)
- Supports hierarchical deterministic (HD) key derivation

---

## Decision

**Adopt Lindell's Two-Party ECDSA protocol (Crypto 2017)** for threshold key generation and signing.

Reference: [Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)

### Protocol Characteristics

| Aspect | Lindell 2P-ECDSA |
|--------|------------------|
| Parties | 2 (client + server) |
| Threshold | 2-of-2 |
| Keygen Rounds | 4 + 2 (chain code) |
| Sign Rounds | 2 |
| Output | Standard ECDSA signature |
| Security Model | Honest-but-curious |

---

## Alternatives Considered

### Option 1: Multi-Signature (Rejected)

| Pro | Con |
|-----|-----|
| Simple implementation | On-chain cost (multiple signatures) |
| No trusted setup | Visible threshold structure |
| | Not supported on all chains |
| | No HD derivation support |

### Option 2: Threshold ECDSA (GG18/GG20) (Rejected)

| Pro | Con |
|-----|-----|
| Flexible n-of-m threshold | Higher complexity |
| Full MPC security | More computation |
| | Longer latency |
| | Harder mobile integration |

### Option 3: Schnorr Signatures (Rejected)

| Pro | Con |
|-----|-----|
| Native threshold support | Not supported on Bitcoin (at time) |
| Simpler MPC | Not supported on Ethereum |
| | Limited blockchain adoption |

---

## Consequences

### Positive

- **No single point of failure**: Neither party can sign alone
- **Standard signatures**: Works with any ECDSA-based blockchain
- **Efficient**: Only 2 rounds for signing (~150ms)
- **Mobile-friendly**: Client computations are lightweight
- **HD support**: BIP32-compatible key derivation

### Negative

- **2-of-2 only**: Cannot extend to n-of-m threshold
- **Server required**: Cannot sign offline
- **Honest-but-curious**: Malicious server could potentially learn information through deviation
- **Latency overhead**: ~150ms per signature vs. instant local signing

### Neutral

- **Paillier dependency**: Requires 2048-bit Paillier encryption (adds latency to keygen)
- **Protocol complexity**: ZK proofs required for security

---

## Evidence

| Claim | Verification |
|-------|--------------|
| Protocol implemented | [`two-party-ecdsa` crate](https://github.com/ZenGo-X/two-party-ecdsa) |
| Keygen 4 rounds | [`keygen.rs:40-80`](../../gotham-client/src/ecdsa/keygen.rs#L40-L80) |
| Sign 2 rounds | [`sign.rs:36-65`](../../gotham-client/src/ecdsa/sign.rs#L36-L65) |
| HD derivation | [`keygen.rs:102-106`](../../gotham-client/src/ecdsa/keygen.rs#L102-L106) |
| Performance | [`README.md:89-91`](../../README.md#L89-L91) |

---

## References

- Lindell, Y. (2017). "Fast Secure Two-Party ECDSA Signing". CRYPTO 2017. [ePrint 2017/552](https://eprint.iacr.org/2017/552)
- Gennaro, R., & Goldfeder, S. (2018). "Fast Multiparty Threshold ECDSA". CCS 2018. (GG18)
- ZenGo-X. "Threshold Signatures: The Future of Private Keys". [Medium](https://medium.com/kzen-networks/threshold-signatures-private-key-the-next-generation-f27b30793b)
