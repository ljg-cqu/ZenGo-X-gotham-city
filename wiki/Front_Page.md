# Gotham City

| Field | Value |
|-------|-------|
| Repository | https://github.com/ZenGo-X/gotham-city |
| Tech Stack | Rust, Rocket 0.5.0-rc.1, RocksDB, secp256k1, two-party-ecdsa |
| License | GPL-3.0 |
| Coverage | Integration tests (protocol verification) |
| Updated | 2025-12-06 |

### Codebase Metrics

| Metric | Value | Command/Source |
|--------|-------|----------------|
| Total LOC | ~5K | `cloc --vcs=git .` (estimated) |
| Source Files | 15+ | Rust files across 4 workspace members |
| Test Files | 2 | `gotham-server/src/tests.rs`, `integration-tests/tests/ecdsa.rs` |
| Workspace Members | 4 | gotham-server, gotham-client, demo-wallet, integration-tests |
| Contributors | ZenGo-X | GitHub contributors |

## Summary

**Problem**: Cryptocurrency private key management creates a critical single-point-of-failure—loss, theft, or compromise of a private key results in total asset loss. For whom: cryptocurrency wallet developers, custodial services, and security-conscious users requiring enhanced key protection without sacrificing usability.

**Solution**: Two-party ECDSA (2P-ECDSA) threshold signature scheme implementing Lindell's Crypto17 protocol. The private key is cryptographically split between client and server; neither party alone can produce a valid signature. Supports BIP32-compatible hierarchical deterministic (HD) key derivation.

**Scope**:
- **In**: 2P-ECDSA key generation, transaction signing, HD key derivation, Bitcoin/Ethereum wallet integration, key backup/recovery via Centipede protocol
- **Out**: Full node functionality, transaction broadcasting (delegated to Electrum/RPC), multi-party (>2) threshold signatures
- **Constraints**: Both parties must be online for signing; server must persist Party1 key shares
- **Assumptions**: Secure transport (HTTPS), honest-but-curious adversary model

**Stakeholders**:
- **Users**: Cryptocurrency wallet users seeking enhanced key security
- **Maintainers**: ZenGo-X team
- **Dependents**: Mobile wallet apps (iOS/Android via FFI bindings), Bitcoin/Ethereum applications

| Priority | Capability | Metric | Evidence |
|----------|------------|--------|----------|
| P1 | 2P-ECDSA Key Generation | 762ms (M2 MacBook localhost) | [`README.md:90`](../README.md#L90) |
| P1 | 2P-ECDSA Signing | 151ms (M2 MacBook localhost) | [`README.md:91`](../README.md#L91) |
| P1 | HD Key Derivation | BIP32-compatible child keys | [`gotham-client/src/ecdsa/keygen.rs:102-106`](../gotham-client/src/ecdsa/keygen.rs#L102-L106) |
| P2 | Bitcoin Transaction Signing | P2WPKH (SegWit) support | [`demo-wallet/src/bitcoin/`](../demo-wallet/src/bitcoin/) |
| P2 | Ethereum Transaction Signing | EIP-155 replay protection | [`demo-wallet/src/ethereum/`](../demo-wallet/src/ethereum/) |
| P3 | Key Backup/Recovery | Verifiable encryption via Centipede | [`gotham-client/src/ecdsa/recover.rs`](../gotham-client/src/ecdsa/recover.rs) |

**Architecture**: Client-Server 2P-ECDSA — chosen over:
- **Multi-sig** (on-chain cost, reduced compatibility, visible threshold structure)
- **Full MPC (n-of-n)** (higher complexity, increased latency, harder mobile integration)

**Trade-offs**:
- **Benefits**: No single point of failure, standard ECDSA signatures (chain-agnostic), HD wallet support, mobile FFI bindings
- **Costs**: Server availability required for signing, ~150ms latency overhead vs. local signing, server persistence requirements

**Limitations**:
- NOT recommended for high-frequency trading (latency overhead ~150ms per signature)
- NOT suitable when server availability cannot be guaranteed
- NOT appropriate for scenarios requiring more than 2 parties (use n-of-n MPC instead)

---

## Quick Start

| Tool | Version | Verify | Install |
|------|---------|--------|---------|
| Rust | ≥1.70 | `rustc --version` | [rustup.rs](https://rustup.rs) |
| Cargo | ≥1.70 | `cargo --version` | Included with Rust |

```bash
git clone https://github.com/ZenGo-X/gotham-city.git && cd gotham-city

# Terminal 1: Start the server
cd gotham-server && cargo run                    # [`gotham-server/src/main.rs:1-5`](../gotham-server/src/main.rs#L1-L5)
# Server runs at http://127.0.0.1:8000

# Terminal 2: Run integration tests
cd integration-tests && cargo test               # [`integration-tests/tests/ecdsa.rs`](../integration-tests/tests/ecdsa.rs)

# Or use demo-wallet CLI
cd demo-wallet && cargo run -- --help            # [`demo-wallet/src/main.rs:54-80`](../demo-wallet/src/main.rs#L54-L80)
```

### Demo Wallet Commands

```bash
# Bitcoin operations
cargo run -- bitcoin create    # Generate new 2P-ECDSA wallet
cargo run -- bitcoin balance   # Query wallet balance via Electrum
cargo run -- bitcoin send      # Sign and broadcast Bitcoin transaction

# Ethereum operations
cargo run -- evm create        # Generate new 2P-ECDSA wallet
cargo run -- evm balance       # Query ETH balance via RPC
cargo run -- evm send          # Sign and broadcast Ethereum transaction
```

---

## Versioning Strategy

| Aspect | Strategy | Evidence |
|--------|----------|----------|
| **Versioning Scheme** | Semantic Versioning implied (no explicit version file) | [`CHANGELOG.md`](../CHANGELOG.md) |
| **Breaking Change Policy** | Major changes documented in CHANGELOG | [`CHANGELOG.md:3-20`](../CHANGELOG.md#L3-L20) |
| **Backward Compatibility** | Protocol-level compatibility with two-party-ecdsa crate | [`Cargo.toml:25`](../Cargo.toml#L25) |

---

## Knowledge Transfer Path

| Day | Focus Area | Essential Reading | Practical Exercise | Success Checkpoint | Time Budget |
|-----|------------|------------------|-------------------|-------------------|-------------|
| **Day 1** | Architecture Overview | `Front_Page.md`, `Architecture.md` | Reproduce C4 L2 diagram from memory | Can explain client-server 2P-ECDSA split | 4 hours |
| **Day 2** | Cryptographic Protocol | `Security.md`, `Implementation.md` § Keygen/Sign | Trace 4-round keygen protocol with debugger | Can explain why neither party can sign alone | 6 hours |
| **Day 3** | API & Data Flow | `Data_APIs.md`, `Components/` | Write test calling keygen + sign endpoints | Can construct valid API requests | 4 hours |
| **Day 4** | Development Workflow | `Configuration.md`, `Deployment.md` | Fix a "good first issue" or add wallet feature | Can run tests, build locally | 6 hours |
| **Day 5** | Operations | `Performance_Observability.md`, `Runbooks/` | Run benchmarks, simulate signing failure | Can interpret latency metrics, follow runbook | 4 hours |

**80/20 Reading Priority**:
1. **P1 Critical** (20% of content → 80% of value): `Front_Page.md` § Summary, `Architecture.md` § C4 L1-L2, `Security.md` § Cryptographic Security
2. **P2 High** (30% of content → 15% of value): `Implementation.md` § Keygen/Sign Processes, `Data_APIs.md`
3. **P3 Reference** (50% of content → 5% of value): Remaining sections (consult as-needed)

---

## Wiki Navigation Guide

| If You Want to... | Start Here | Then Go To | Why This Path |
|-------------------|------------|------------|---------------|
| **Understand the big picture** | `Front_Page.md` § Summary | `Architecture.md` § C4 diagrams | Summary provides context; C4 diagrams show structure |
| **Trace a signing request** | `Architecture.md` § Containers | `Implementation.md` § Signing Process | Containers show parties; Implementation shows protocol flow |
| **Debug a protocol error** | `Implementation.md` § Error Handling | `Runbooks/Signing_Failure.md` | Error handling explains types; runbook provides steps |
| **Add blockchain support** | `Architecture.md` § Structure | `Components/Bitcoin_Wallet.md` | Structure shows layout; Bitcoin shows integration pattern |
| **Review cryptographic security** | `Security.md` § Cryptographic Primitives | External: [Lindell 2017](https://eprint.iacr.org/2017/552) | Security shows implementation; paper provides proofs |
| **Deploy to production** | `Deployment.md` | `Security.md` § Deployment Checklist | Deployment shows how; Security shows hardening |
| **Optimize performance** | `Performance_Observability.md` § Benchmarks | `Implementation.md` § Keygen/Sign | Benchmarks show metrics; Implementation shows hot paths |

---

## Contents

- [Architecture](./Architecture.md) — System boundaries, containers, structure, project graphs
- [Components](./Components/) — Detailed component documentation
  - [ECDSA Keygen](./Components/ECDSA_Keygen.md) [P1]
  - [ECDSA Sign](./Components/ECDSA_Sign.md) [P1]
  - [Gotham Server](./Components/Gotham_Server.md) [P1]
  - [Gotham Client](./Components/Gotham_Client.md) [P1]
  - [Bitcoin Wallet](./Components/Bitcoin_Wallet.md) [P2]
  - [Ethereum Wallet](./Components/Ethereum_Wallet.md) [P2]
- [Data & APIs](./Data_APIs.md) — Data model, API contracts
- [Implementation](./Implementation.md) — Processes, data flow, cross-cutting concerns, algorithms
- [Dependencies](./Dependencies.md) — Package dependencies, health matrix
- [Configuration](./Configuration.md) — Environment variables, settings
- [Testing](./Testing.md) — Test strategy, coverage
- [Deployment](./Deployment.md) — Build, deploy, infrastructure
- [Performance & Observability](./Performance_Observability.md) — Benchmarks, SLIs, capacity
- [Security](./Security.md) — Threat model, cryptographic security, auth
- [Runbooks](./Runbooks/) — Operational playbooks
  - [Key Generation Failure](./Runbooks/Key_Generation_Failure.md)
  - [Signing Failure](./Runbooks/Signing_Failure.md)
  - [Database Recovery](./Runbooks/Database_Recovery.md)
- [Decisions](./Decisions/) — Architectural decision records
  - [001. Two-Party ECDSA Protocol Selection](./Decisions/001_Two_Party_ECDSA_Protocol.md)
  - [002. RocksDB for Key Share Storage](./Decisions/002_RocksDB_Storage.md)
  - [003. Rocket Web Framework](./Decisions/003_Rocket_Framework.md)
- [Reference](./Reference.md) — Glossary (≥10 terms), common pitfalls, sources

---

## Skipped Sections

The following sections are **not applicable** to this repository:

| Section | Reason |
|---------|--------|
| Events & Messaging | Synchronous HTTP request-response only; no async event bus |
| Consumer Guide | Application repository, not a published library/SDK |
| Feature Flags | Configuration via TOML/environment only; no feature flag system |
| Economic Model | Open-source project; no pricing or tokenomics |
| Roadmap | No public roadmap documented |
