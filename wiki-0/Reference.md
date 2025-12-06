# Reference

## Glossary

| Term | Definition |
|------|------------|
| **2P-ECDSA** | Two-Party Elliptic Curve Digital Signature Algorithm; a threshold signature scheme where two parties collaborate to produce a valid ECDSA signature without either learning the full private key |
| **Party1** | The server-side participant in 2P-ECDSA; holds one share of the private key and Paillier encryption keys |
| **Party2** | The client-side participant in 2P-ECDSA; holds one share of the private key and initiates protocol rounds |
| **MasterKey2** | The client's master key structure containing the private share, public key component, and chain code for HD derivation |
| **PrivateShare** | Client-side structure containing the session ID and MasterKey2; required for all signing operations |
| **PDL Proof** | Proof of Discrete Log; a zero-knowledge proof used to verify that Paillier ciphertexts encode the correct elliptic curve point |
| **Paillier** | A partially homomorphic encryption scheme used in the 2P-ECDSA protocol for secure computation |
| **Chain Code** | A 256-bit value used in HD key derivation to generate child keys deterministically |
| **HD Derivation** | Hierarchical Deterministic key derivation (BIP32-style); generates child keys from a master key |
| **secp256k1** | The elliptic curve used by Bitcoin and Ethereum; defines the mathematical group for ECDSA |
| **SignatureRecid** | ECDSA signature with recovery ID; contains (r, s, recid) allowing public key recovery |
| **Ephemeral Key** | A temporary key pair generated fresh for each signing operation; ensures signature uniqueness |
| **gotham-engine** | A separate crate providing Rocket route handlers for the MPC protocol |
| **Electrum** | A lightweight Bitcoin protocol for querying UTXOs without running a full node |
| **P2WPKH** | Pay-to-Witness-Public-Key-Hash; a SegWit Bitcoin address type |

---

## Common Pitfalls

| Pitfall | Symptom | Solution | Evidence |
|---------|---------|----------|----------|
| Server not running | Connection refused on keygen/sign | Start server: `cd gotham-server && cargo run` | — |
| Wrong session ID | HTTP 400 on signing | Use `private_share.id` from keygen result | [`sign.rs:34`](../gotham-client/src/ecdsa/sign.rs#L34) |
| DB name with special chars | Server panic on startup | Use alphanumeric db_name only | [`public_gotham.rs:38-40`](../gotham-server/src/public_gotham.rs#L38-L40) |
| Stale wallet file | Signing fails after server reset | Re-run keygen if server DB was cleared | — |
| Missing auth token | Unauthorized (if auth implemented) | Set auth_token in ClientShim | [`lib.rs:22`](../gotham-client/src/lib.rs#L22) |
| HD path confusion | Wrong address derived | Use consistent (x_pos, y_pos) for each address | [`keygen.rs:108`](../gotham-client/src/ecdsa/keygen.rs#L108) |
| Message hash format | Invalid signature | Pass transaction hash as BigInt, not raw bytes | [`sign.rs:30`](../gotham-client/src/ecdsa/sign.rs#L30) |

### Debugging Tips

- **Enable logging**: `RUST_LOG=debug cargo run` to see request timing
- **Check wallet JSON**: Verify `private_share.id` matches server session
- **Test connectivity**: `curl http://localhost:8000/` should return 404 (not connection refused)
- **Verify keygen completed**: All 6 rounds (4 keygen + 2 chaincode) must succeed
- **Check DB persistence**: Server DB survives restarts; client wallet.json must be preserved

---

## Sources

### Internal

| Path | Purpose |
|------|---------|
| [`gotham-server/src/`](../gotham-server/src/) | Server implementation (Party1) |
| [`gotham-client/src/`](../gotham-client/src/) | Client library (Party2) |
| [`gotham-client/src/ecdsa/`](../gotham-client/src/ecdsa/) | ECDSA keygen/sign/recover modules |
| [`demo-wallet/src/bitcoin/`](../demo-wallet/src/bitcoin/) | Bitcoin wallet integration |
| [`demo-wallet/src/ethereum/`](../demo-wallet/src/ethereum/) | Ethereum wallet integration |
| [`integration-tests/tests/`](../integration-tests/tests/) | E2E protocol tests |
| [`white-paper/`](../white-paper/) | Academic documentation |

### External (Tier 1-2: Official Docs, Specifications)

| Source | Relevance |
|--------|-----------|
| [Lindell, Y. (2017). Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552) | Core 2P-ECDSA protocol specification |
| [BIP32: Hierarchical Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) | HD key derivation standard |
| [secp256k1 curve parameters](https://www.secg.org/sec2-v2.pdf) | Elliptic curve specification |
| [Rocket Framework Documentation](https://rocket.rs/v0.5-rc/) | HTTP server framework |
| [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki) | Embedded database documentation |

### ZenGo-X Dependencies

| Repository | Purpose |
|------------|---------|
| [ZenGo-X/two-party-ecdsa](https://github.com/ZenGo-X/two-party-ecdsa) | Core 2P-ECDSA protocol implementation |
| [ZenGo-X/gotham-engine](https://github.com/ZenGo-X/gotham-engine) | Server-side MPC route handlers |
| [ZenGo-X/curv](https://github.com/ZenGo-X/curv) | Elliptic curve utilities |
| [ZenGo-X/centipede](https://github.com/ZenGo-X/centipede) | Verifiable encryption for backup |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Current | 2025-12-05 | Wiki generated |

See [`CHANGELOG.md`](../CHANGELOG.md) for project version history.
