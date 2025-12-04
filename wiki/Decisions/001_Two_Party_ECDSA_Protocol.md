# 001. Two-Party ECDSA Protocol Selection

**Date**: 2018 (original implementation)  
**Status**: Accepted  
**Deciders**: ZenGo-X Team

## Context

Cryptocurrency wallets require private key management that balances security and usability. Traditional approaches have significant limitations:

1. **Single-key custody**: User holds entire private key
   - Risk: Single point of failure (theft, loss, hacking)
   - Benefit: Full control, no dependencies

2. **Multi-signature (on-chain)**: Multiple keys required to sign
   - Risk: On-chain costs, limited blockchain support
   - Benefit: No single point of failure

3. **Full MPC (n-of-n)**: Key split among multiple parties
   - Risk: Complexity, latency, coordination overhead
   - Benefit: Distributed trust

The goal was to provide enhanced security without sacrificing usability or blockchain compatibility.

## Decision

We will implement **Lindell's Two-Party ECDSA protocol** (Crypto17) for key generation and signing.

Key characteristics:
- Private key is split between client (Party2) and server (Party1)
- Neither party alone can produce a valid signature
- Output is a standard ECDSA signature (compatible with all blockchains)
- 4-round keygen, 2-round signing protocol

## Consequences

**Positive**:
- **No single point of failure**: Compromising either client or server alone is insufficient
- **Standard signatures**: Works with any blockchain using ECDSA (Bitcoin, Ethereum, etc.)
- **Reasonable performance**: ~762ms keygen, ~151ms signing on modern hardware
- **HD wallet support**: Compatible with BIP32-style key derivation
- **Provable security**: Based on peer-reviewed cryptographic protocol

**Negative**:
- **Server dependency**: Signing requires server availability
- **Latency overhead**: ~150ms vs. local signing (<1ms)
- **Protocol complexity**: Cryptographic implementation requires expertise
- **Trust model**: Requires honest-but-curious assumption

**Risks**:
- **Implementation bugs**: Cryptographic code is difficult to audit
  - Mitigation: Use established `two-party-ecdsa` library
- **Server compromise**: Attacker gains partial key share
  - Mitigation: Key rotation capability, escrow backup

## Evidence

- Protocol implementation: [`gotham-client/src/ecdsa/keygen.rs`](../../gotham-client/src/ecdsa/keygen.rs)
- Academic paper: [Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
- Performance benchmarks: [`README.md:89-91`](../../README.md#L89-L91)
