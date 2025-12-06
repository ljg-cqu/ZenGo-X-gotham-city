# Reference

## Glossary

| Priority | Term | Definition | Usage | Evidence |
|----------|------|------------|-------|----------|
| P1 | **2P-ECDSA** | Two-Party Elliptic Curve Digital Signature Algorithm — a threshold signature scheme where two parties collaboratively produce a standard ECDSA signature without either possessing the complete private key | Core protocol | Protocol design |
| P1 | **Party1** | The server side of the 2P-ECDSA protocol; holds server key share, executes MPC rounds | Server component | [`public_gotham.rs`](../gotham-server/src/public_gotham.rs) |
| P1 | **Party2** | The client side of the 2P-ECDSA protocol; holds client key share, initiates MPC rounds | Client component | [`gotham-client/`](../gotham-client/) |
| P1 | **MasterKey2** | The client's (Party2) complete key share including public key, chain code, and private share | Client state | [`keygen.rs:108`](../gotham-client/src/ecdsa/keygen.rs#L108) |
| P1 | **PrivateShare** | Container type holding session ID and MasterKey2 for client-side key storage | Client type | [`types.rs`](../gotham-client/src/ecdsa/types.rs) |
| P2 | **Paillier** | A probabilistic asymmetric encryption scheme with additive homomorphic properties; used in 2P-ECDSA for secure computation | Crypto primitive | two-party-ecdsa |
| P2 | **PDL** | Paillier Discrete Log — zero-knowledge proof that a Paillier ciphertext encrypts a discrete log | Keygen verification | [`keygen.rs:59-80`](../gotham-client/src/ecdsa/keygen.rs#L59-L80) |
| P2 | **Chain Code** | BIP32 component used for hierarchical deterministic key derivation; shared between parties | HD derivation | [`keygen.rs:102-106`](../gotham-client/src/ecdsa/keygen.rs#L102-L106) |
| P2 | **HD Derivation** | Hierarchical Deterministic key derivation (BIP32) — deriving child keys from a master key using derivation path | Address generation | BIP32 spec |
| P2 | **Session ID** | UUID v4 identifier linking keygen and sign operations for the same key pair | State management | gotham-engine |
| P3 | **ClientShim** | HTTP client wrapper in gotham-client that handles communication with the server | Client abstraction | [`lib.rs:19-24`](../gotham-client/src/lib.rs#L19-L24) |
| P3 | **Db Trait** | Server-side trait abstracting key share storage; implemented by PublicGotham | Storage abstraction | [`public_gotham.rs:57`](../gotham-server/src/public_gotham.rs#L57) |
| P3 | **Centipede** | Verifiable encryption protocol used for key backup and recovery | Backup system | [`recover.rs`](../gotham-client/src/ecdsa/recover.rs) |
| P3 | **recid** | Recovery ID (0 or 1) — additional signature component enabling public key recovery from signature | Signature output | [`sign.rs`](../gotham-client/src/ecdsa/sign.rs) |
| P3 | **SignatureRecid** | ECDSA signature with recovery ID: (r, s, recid) | Sign output | [`sign.rs:83`](../gotham-client/src/ecdsa/sign.rs#L83) |

---

## Common Pitfalls

| Pitfall | Symptom | Solution | Evidence |
|---------|---------|----------|----------|
| **Server not running** | Connection refused | Start server with `cargo run` in `gotham-server/` | Quick start |
| **Session ID mismatch** | 400 "Key not found" | Use session ID from keygen for signing | [`sign.rs:41`](../gotham-client/src/ecdsa/sign.rs#L41) |
| **Auth always grants** | Unauthorized access | Implement `Db::granted()` for production | [`public_gotham.rs:93-95`](../gotham-server/src/public_gotham.rs#L93-L95) |
| **No HTTPS** | MITM vulnerability | Deploy behind reverse proxy with TLS | [Security.md](./Security.md) |
| **Keygen interrupted** | Partial state | Must restart keygen from round 1 | Protocol design |
| **Old reqwest version** | Missing features | Plan upgrade to reqwest 0.11+ | [`Cargo.toml:15`](../Cargo.toml#L15) |
| **Panic on HTTP failure** | Crashes on network error | Wrap calls in error handling | [`keygen.rs:41`](../gotham-client/src/ecdsa/keygen.rs#L41) |
| **Wrong HD indices** | Invalid signature | Ensure x_pos, y_pos match address derivation | [`sign.rs:151`](../gotham-client/src/ecdsa/sign.rs#L151) |
| **DB name validation** | Server panic | Use only alphanumeric characters | [`public_gotham.rs:38-40`](../gotham-server/src/public_gotham.rs#L38-L40) |

---

## Quick Command Reference

### Development Commands

| Task | Command | When to Use | Prerequisites | Evidence |
|------|---------|-------------|---------------|----------|
| Build workspace | `cargo build --workspace` | Initial setup, code changes | Rust ≥1.70 | Cargo.toml |
| Build release | `cargo build --workspace --release` | Production deployment | Rust ≥1.70 | Cargo.toml |
| Start server | `cd gotham-server && cargo run` | Development | None | [`main.rs`](../gotham-server/src/main.rs) |
| Start server (release) | `cd gotham-server && cargo run --release` | Production-like | None | — |
| Run wallet CLI | `cd demo-wallet && cargo run -- --help` | Wallet operations | Server running | [`main.rs`](../demo-wallet/src/main.rs) |

### Testing Commands

| Task | Command | When to Use | Duration | Evidence |
|------|---------|-------------|----------|----------|
| Unit tests | `cargo test -p gotham-server` | Code changes | ~5s | [`tests.rs`](../gotham-server/src/tests.rs) |
| Integration tests | `cargo test -p integration-tests` | Protocol verification | ~10s | [`ecdsa.rs`](../integration-tests/tests/ecdsa.rs) |
| All tests | `cargo test --workspace` | Before commit | ~15s | Cargo.toml |
| Keygen benchmark | `cargo bench --bench keygen_bench` | Performance analysis | ~60s | [`README.md:90`](../README.md#L90) |
| Sign benchmark | `cargo bench --bench sign_bench` | Performance analysis | ~30s | [`README.md:91`](../README.md#L91) |

### Debugging Commands

| Task | Command | When to Use | Output | Evidence |
|------|---------|-------------|--------|----------|
| Check dependencies | `cargo tree` | Dependency analysis | Dependency tree | Cargo |
| Security audit | `cargo audit` | Security check | CVE report | cargo-audit |
| Outdated deps | `cargo outdated` | Maintenance | Update list | cargo-outdated |
| Format check | `cargo fmt --check` | Before commit | Format diff | rustfmt |
| Lint | `cargo clippy` | Code quality | Warnings | clippy |

### Database Commands

| Task | Command | When to Use | Safety | Evidence |
|------|---------|-------------|--------|----------|
| Inspect DB | `ldb scan --db=db/` | Debugging | ✅ Safe (read-only) | RocksDB tools |
| Repair DB | `ldb repair --db=db/` | Corruption | ⚠️ Caution | RocksDB tools |
| Backup DB | `cp -r db/ db.backup` | Before changes | ✅ Safe | Manual |

### Emergency Commands

| Task | Command | When to Use | Impact | Evidence |
|------|---------|-------------|--------|----------|
| Kill server | `pkill gotham-server` | Hung process | 🚨 HIGH RISK | OS command |
| Clear database | `rm -rf db/` | Fresh start | 🚨 HIGH RISK (data loss) | — |
| Restore backup | `cp -r db.backup/ db/` | Recovery | ⚠️ Caution | [Database_Recovery.md](./Runbooks/Database_Recovery.md) |

**Command Safety Legend**:
- ✅ Safe — No side effects, read-only
- ⚠️ Caution — May modify state, review before running
- 🚨 HIGH RISK — Data loss possible, requires approval

---

## Sources

### Internal Code References

| Component | File | Purpose |
|-----------|------|---------|
| Server entry | [`gotham-server/src/main.rs`](../gotham-server/src/main.rs) | Application startup |
| Server routes | [`gotham-server/src/server.rs`](../gotham-server/src/server.rs) | Route registration |
| DB implementation | [`gotham-server/src/public_gotham.rs`](../gotham-server/src/public_gotham.rs) | RocksDB integration |
| Client library | [`gotham-client/src/lib.rs`](../gotham-client/src/lib.rs) | HTTP client wrapper |
| Key generation | [`gotham-client/src/ecdsa/keygen.rs`](../gotham-client/src/ecdsa/keygen.rs) | 2P-ECDSA keygen |
| Signing | [`gotham-client/src/ecdsa/sign.rs`](../gotham-client/src/ecdsa/sign.rs) | 2P-ECDSA signing |
| Recovery | [`gotham-client/src/ecdsa/recover.rs`](../gotham-client/src/ecdsa/recover.rs) | Key backup/recovery |
| Wallet CLI | [`demo-wallet/src/main.rs`](../demo-wallet/src/main.rs) | CLI application |

### External References

| Source | Type | URL |
|--------|------|-----|
| Lindell 2017 Paper | Academic | [ePrint 2017/552](https://eprint.iacr.org/2017/552) |
| two-party-ecdsa | Library | [GitHub](https://github.com/ZenGo-X/two-party-ecdsa) |
| gotham-engine | Library | [GitHub](https://github.com/ZenGo-X/gotham-engine) |
| Rocket Framework | Docs | [rocket.rs](https://rocket.rs/) |
| RocksDB | Docs | [rocksdb.org](https://rocksdb.org/) |
| BIP32 (HD Wallets) | Spec | [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) |
| BIP143 (SegWit Signing) | Spec | [BIP-0143](https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki) |
| EIP-155 (Replay Protection) | Spec | [EIP-155](https://eips.ethereum.org/EIPS/eip-155) |
| secp256k1 Curve | Spec | [SEC 2](https://www.secg.org/sec2-v2.pdf) |

---

## Acronyms

| Acronym | Expansion |
|---------|-----------|
| 2P-ECDSA | Two-Party Elliptic Curve Digital Signature Algorithm |
| MPC | Multi-Party Computation |
| HD | Hierarchical Deterministic |
| PDL | Paillier Discrete Log |
| ZK | Zero-Knowledge |
| UTXO | Unspent Transaction Output |
| RPC | Remote Procedure Call |
| FFI | Foreign Function Interface |
| JNI | Java Native Interface |
| CLI | Command Line Interface |
| API | Application Programming Interface |
| KV | Key-Value |
| DB | Database |
| TLS | Transport Layer Security |
