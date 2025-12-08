# Recommendations (Round 2, Multi-Round View)

## Prioritized Roadmap (R1+R2)

| Priority | Finding ID | Title | Effort |
|----------|-----------|-------|--------|
| Immediate | R1-F-001 | Fix Authorization Bypass in `granted` | Low |
| Immediate | R1-F-002 | Encrypt Server Key Storage (RocksDB) | Medium |
| Immediate | R1-F-003 | Encrypt Client Wallet Storage (Bitcoin/Ethereum) | Medium |
| Immediate | R2-F-001 | Protect Escrow Secret Storage (Bitcoin Escrow) | Medium |
| Short-term | R1-F-004 | Enable TLS (HTTPS) for MPC Endpoints | Low |
| Short-term | R1-F-005 | Replace Panicking Error Handling and Add Basic Rate Limiting | Medium |
| Long-term | R1-F-006 | Modernize and Lock Down Dependencies | High |

## Immediate Actions

### 1. Fix Authorization Bypass (R1-F-001)

- **Goal**: Ensure only authorized principals can initiate keygen/sign operations for a given wallet.
- **Where**: `gotham-server/src/public_gotham.rs` (`granted`).
- **Actions**:
  - Replace `Ok(true)` with policy checks that bind `customer_id` and request context to stored ownership information.
  - Integrate with an authentication/authorization layer (JWTs, OAuth, or equivalent) and verify tokens before permitting signing.
  - Add tests in `gotham-server/src/tests.rs` that prove unauthorized users cannot sign.

### 2. Encrypt Server Key Storage (R1-F-002)

- **Goal**: Prevent offline attackers from reading Party 1 key shares in RocksDB.
- **Where**: `gotham-server/src/public_gotham.rs` insert/get paths.
- **Actions**:
  - Introduce an encryption layer for all RocksDB values containing MPC state.
  - Use authenticated encryption (e.g., AES-GCM) and manage keys via environment configuration, KMS, or HSM.
  - Ensure keys are not stored in the same database or on disk unprotected.

### 3. Encrypt Client Wallet Storage (R1-F-003)

- **Goal**: Prevent local attackers and malware from reading Party 2 key shares from wallet files.
- **Where**:
  - `demo-wallet/src/bitcoin/mod.rs` (`save_to`, `load_from` or equivalent routines).
  - `demo-wallet/src/ethereum/mod.rs` (`GothamWallet::save` / `GothamWallet::load`).
- **Actions**:
  - Introduce a user passphrase or OS keychain-based key to encrypt wallet JSON before writing to disk.
  - Use modern password-based key derivation (Argon2 or scrypt) with per-device salts.
  - Add migration code to detect legacy plaintext wallets and guide users through a one-time upgrade path.

### 4. Protect Escrow Secret Storage (R2-F-001)

- **Goal**: Ensure escrow secrets used for backup/recovery are not another plaintext single point of failure.
- **Where**: `demo-wallet/src/bitcoin/escrow.rs` (`Escrow::new`, `Escrow::load`).
- **Actions**:
  - Stop writing raw `(secret, public)` tuples directly to JSON files.
  - Introduce an abstraction (e.g., `EscrowKeyStore`) that can back escrow storage with:
    - OS keychain / secure enclave on mobile,
    - hardware tokens or HSMs in institutional deployments,
    - or at least encrypted blobs protected by a strong passphrase-derived key.
  - Separate escrow storage location and permissions from the main wallet file.
  - Document the intended custody model for escrow in README and operator docs.

## Short-Term Actions

### 5. Enable TLS for MPC Endpoints (R1-F-004)

- **Goal**: Prevent interception and tampering with MPC protocol messages.
- **Where**:
  - `gotham-server/Rocket.toml` and deployment tooling.
  - Demo wallet defaults (`demo-wallet/src/main.rs`) which currently use `http://127.0.0.1:8000`.
- **Actions**:
  - Configure Rocket for HTTPS in production or place a hardened TLS reverse proxy (Nginx, Envoy) in front.
  - Update default URLs and documentation to prefer HTTPS.
  - Add tests or scripts to validate TLS configuration as part of CI or deployment.

### 6. Improve Robustness and Add Basic Rate Limiting (R1-F-005)

- **Goal**: Reduce the risk of DoS from panics and abuse of signing endpoints.
- **Where**:
  - `gotham-server/src/public_gotham.rs` and client MPC flows (`gotham-client/src/ecdsa/*.rs`).
- **Actions**:
  - Replace `unwrap`/`expect` with error handling that returns appropriate HTTP status codes.
  - Log structured errors (without leaking secrets) and consider backoff for repeated failures.
  - Add basic rate limiting and request size limits at the web server or reverse proxy layer.

## Long-Term Actions

### 7. Modernize and Lock Down Dependencies (R1-F-006)

- **Goal**: Reduce exposure to known vulnerabilities and supply-chain risk.
- **Where**: Workspace `Cargo.toml` and crate-specific manifests.
- **Actions**:
  - Upgrade key crates (`rocket`, `reqwest`, `jsonwebtoken`, `secp256k1`) to supported versions.
  - Replace broad git dependencies (`two-party-ecdsa`, `gotham-engine`) with version-pinned crates or explicit commit hashes.
  - Integrate `cargo audit` (or similar) into CI and require clean reports before releases.

## Execution Phases

- **Phase 1 (Blocking Issues)**
  - Implement real authorization.
  - Encrypt all key and escrow storage (server, wallets, escrow files).

- **Phase 2 (Resilience and Transport Security)**
  - Enable TLS and harden error handling / rate limiting.

- **Phase 3 (Sustainability and Governance)**
  - Modernize dependencies and add continuous security monitoring.

Completion of Phase 1 and Phase 2, followed by a focused re-audit, should be considered a minimum requirement before any real-funds deployment.
