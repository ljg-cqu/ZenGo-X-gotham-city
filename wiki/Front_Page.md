# Gotham City

| Field | Value |
|-------|-------|
| Repository | https://github.com/ZenGo-X/gotham-city |
| Tech Stack | Rust, Rocket (web framework), RocksDB, secp256k1, two-party-ecdsa |
| LOC | ~5K (estimated across 4 workspace members) |
| Updated | 2025-12-05 |

## Summary

**Problem**: Cryptocurrency private key management is a critical security challenge—single-point-of-failure custody creates risk of total asset loss through theft, hacking, or key compromise. For whom: cryptocurrency users, wallet developers, and custodial services requiring enhanced security without sacrificing usability.

**Solution**: Two-party ECDSA (2P-ECDSA) threshold signature scheme implementing Lindell's Crypto17 protocol. The private key is split between client and server; neither party alone can produce a valid signature. Supports hierarchical deterministic (HD) key derivation for address generation.

**Scope**:
- **In**: 2P-ECDSA key generation, signing, HD key derivation, Bitcoin/Ethereum wallet integration
- **Out**: Full node functionality, transaction broadcasting (delegated to external services), multi-party (>2) threshold signatures
- **Constraints**: Requires both parties online for signing; server must persist key shares
- **Assumptions**: Secure communication channel (HTTPS), honest-but-curious threat model

**Stakeholders**:
- **Users**: Cryptocurrency wallet users seeking enhanced key security
- **Maintainers**: ZenGo-X team
- **Dependents**: Mobile wallet apps (iOS/Android via FFI), Bitcoin/Ethereum applications

| Priority | Capability | Metric | Evidence |
|----------|------------|--------|----------|
| CRITICAL | 2P-ECDSA Key Generation | 762ms (M2 MacBook) | [`README.md:89-91`](../README.md#L89-L91) |
| CRITICAL | 2P-ECDSA Signing | 151ms (M2 MacBook) | [`README.md:89-91`](../README.md#L89-L91) |
| CRITICAL | HD Key Derivation | BIP32-compatible child keys | [`gotham-client/src/ecdsa/keygen.rs:102-106`](../gotham-client/src/ecdsa/keygen.rs#L102-L106) |
| IMPORTANT | Bitcoin Transaction Signing | P2WPKH support | [`demo-wallet/src/bitcoin/mod.rs:336-368`](../demo-wallet/src/bitcoin/mod.rs#L336-L368) |
| IMPORTANT | Ethereum Transaction Signing | EIP-155 support | [`demo-wallet/src/ethereum/mod.rs:155-178`](../demo-wallet/src/ethereum/mod.rs#L155-L178) |
| OPTIONAL | Key Backup/Recovery | Escrow via Centipede | [`demo-wallet/src/bitcoin/mod.rs:143-167`](../demo-wallet/src/bitcoin/mod.rs#L143-L167) |

**Architecture**: Client-Server 2P-ECDSA — chosen over multi-sig (on-chain cost, compatibility) and full MPC (complexity, latency)

**Trade-offs**:
- **Benefits**: No single point of failure, standard ECDSA signatures (chain-agnostic), HD wallet support
- **Costs**: Requires server availability for signing, additional latency vs. local signing

**Limitations**: 
- NOT recommended for high-frequency trading (latency overhead)
- NOT suitable when server availability cannot be guaranteed
- NOT appropriate for scenarios requiring >2 parties

---

## Quick Start

| Tool | Version | Verify | Install |
|------|---------|--------|---------|
| Rust | ≥1.70 | `rustc --version` | [rustup.rs](https://rustup.rs) |
| Cargo | ≥1.70 | `cargo --version` | Included with Rust |

```bash
git clone https://github.com/ZenGo-X/gotham-city.git && cd gotham-city

# Start the server (terminal 1)
cd gotham-server && cargo run                    # [`gotham-server/src/main.rs:1-5`](../gotham-server/src/main.rs#L1-L5)
# Server runs at http://127.0.0.1:8000

# Run integration tests (terminal 2)
cd integration-tests && cargo test               # [`integration-tests/tests/ecdsa.rs:107-151`](../integration-tests/tests/ecdsa.rs#L107-L151)

# Or use demo-wallet CLI
cd demo-wallet && cargo run -- --help            # [`demo-wallet/src/main.rs:54-80`](../demo-wallet/src/main.rs#L54-L80)
```

---

## Contents

- [Architecture](./Architecture.md) — System boundaries, containers, structure
- [Components](./Components/) — Detailed component documentation (prioritized by criticality)
  - [ECDSA Keygen](./Components/ECDSA_Keygen.md)
  - [ECDSA Sign](./Components/ECDSA_Sign.md)
  - [Gotham Server](./Components/Gotham_Server.md)
  - [Bitcoin Wallet](./Components/Bitcoin_Wallet.md)
  - [Ethereum Wallet](./Components/Ethereum_Wallet.md)
- [Data & APIs](./Data_APIs.md) — Data model, API contracts
- [Implementation](./Implementation.md) — Processes, data flow, cross-cutting concerns
- [Operations](./Operations.md) — Dependencies, config, testing, deployment
- [Performance & Observability](./Performance_Observability.md) — Benchmarks, metrics
- [Security](./Security.md) — Threat model, cryptographic security
- [Runbooks](./Runbooks/) — Operational playbooks
  - [Key Generation Failure](./Runbooks/Key_Generation_Failure.md)
  - [Signing Failure](./Runbooks/Signing_Failure.md)
  - [Database Recovery](./Runbooks/Database_Recovery.md)
- [Decisions](./Decisions/) — Architectural decision records
  - [001. Two-Party ECDSA Protocol Selection](./Decisions/001_Two_Party_ECDSA_Protocol.md)
  - [002. RocksDB for Key Share Storage](./Decisions/002_RocksDB_Storage.md)
  - [003. Rocket Web Framework](./Decisions/003_Rocket_Framework.md)
- [Reference](./Reference.md) — Glossary (≥10 terms), common pitfalls, sources
