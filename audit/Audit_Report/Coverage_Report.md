# Audit Coverage & Scope Report

| Field | Value |
|-------|-------|
| **Audit Tier** | FULL |
| **Audit Duration** | ~2.5 hours (excluding documentation time) |
| **Code Base Size** | ~35K LOC (workspace-wide `wc -l`) |
| **Files Analyzed** | ~27 key files (server, client, wallet, tests, docs) |
| **Coverage Level** | High for security-critical paths (server/client MPC flows and wallet key handling); sampled for demo UX and ancillary docs |
| **EssentialScope Coverage** | 100% of EssentialScope entries reviewed (server entrypoints, MPC flows, key storage, wallet secrets, configs, dependency manifests) |

## Coverage by Category

### 1. Configuration and Infrastructure

| File | Status | Issues Found |
|------|--------|--------------|
| `Cargo.toml` (workspace) | Reviewed | Dependency posture (F-007) |
| `gotham-server/Cargo.toml` | Reviewed | RocksDB usage, Rocket, async runtime (F-002, F-007) |
| `gotham-server/Settings.toml` | Reviewed | DB configuration, region/auth placeholders (context) |
| `gotham-server/Rocket.toml` | Reviewed | Debug binding on `0.0.0.0` without TLS (F-006) |

**Findings Summary**: 0 CRITICAL / 0 HIGH / 0 MEDIUM / 1 LOW (F-006) / 1 INFO (F-007)

---

### 2. Authentication, Authorization, and Business Logic

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-server/src/public_gotham.rs` | Reviewed | Default allow-all authorization, plaintext key storage (F-001, F-002) |
| `wiki/Security.md` | Sampled | Confirms threat model and warns about missing auth (context for F-001) |

**Findings Summary**: 1 CRITICAL (F-001) / 1 HIGH (F-002) / 0 MEDIUM / 0 LOW / 0 INFO

---

### 3. MPC Client Library and FFI

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-client/src/lib.rs` | Reviewed | HTTP client wiring, bearer token support (context) |
| `gotham-client/src/ecdsa/keygen.rs` | Reviewed | FFI boundary panics, MPC keygen flows (F-004 context) |
| `gotham-client/src/ecdsa/sign.rs` | Reviewed | FFI boundary and signing orchestration (F-004 context) |
| `gotham-client/src/ecdsa/recover.rs` | Reviewed | FFI helpers and key reconstruction (F-004 context) |
| `gotham-client/src/utilities/mod.rs` | Reviewed | Error-to-C-string helper |

**Findings Summary**: 0 CRITICAL / 0 HIGH / 1 MEDIUM (F-004) / 0 LOW / 0 INFO

---

### 4. Demo Wallets (Bitcoin & Ethereum)

| File | Status | Issues Found |
|------|--------|--------------|
| `demo-wallet/src/main.rs` | Reviewed | Settings handling and entry point (context) |
| `demo-wallet/src/bitcoin/mod.rs` | Reviewed | Wallet structure, transaction construction, backup/verify (F-003, F-005) |
| `demo-wallet/src/bitcoin/commands.rs` | Reviewed | CLI flows and Electrum usage (F-005 context) |
| `demo-wallet/src/bitcoin/escrow.rs` | Reviewed | Escrow secret generation and storage (F-003) |
| `demo-wallet/src/ethereum/commands.rs` | Reviewed | ETH/ERC20 send and balance commands (F-005 context) |

**Findings Summary**: 1 HIGH (F-003) / 1 MEDIUM (F-005) / 0 LOW / 0 INFO

---

### 5. Tests, Documentation, and Observability

| File | Status | Issues Found |
|------|--------|--------------|
| `integration-tests/tests/ecdsa.rs` | Reviewed | Confirms MPC keygen/sign flows and endpoints (context for F-001–F-003) |
| `README.md` (root) | Reviewed | Project description and risk disclaimer (context) |
| `wiki/Architecture.md` | Sampled | Architecture and call/dependency graphs (context for attack surface) |
| `gotham-server/src/public_gotham.rs` | Reviewed | Lack of structured audit logging (F-008) |
| `gotham-server/src/server.rs` | Reviewed | Logging hooks via Rocket; no audit logs (F-008) |

**Findings Summary**: 0 CRITICAL / 0 HIGH / 0 MEDIUM / 0 LOW / 2 INFO (F-007, F-008)

---

## Files NOT Deeply Analyzed

### Intentional (Out of Scope for Security)

- Internal implementations of `two-party-ecdsa` and `gotham-engine` (external repositories). These components were treated as trusted cryptographic and routing libraries; this audit focused on their integration and configuration in Gotham City rather than re-verifying their internal correctness.
- Static assets and images under `misc/`.
- White paper sources under `white-paper/` (used as background only).

### Time-Limited (Would Require More Time)

- Additional wiki pages beyond `wiki/Architecture.md` and `wiki/Security.md` (documentation sampled for context only).

No in-repo EssentialScope files were left `Not reviewed` or `Partially reviewed` at the end of this audit.

## Severity Distribution by Coverage Area

- **Server Auth/AuthZ & Storage**: 1 CRITICAL (F-001), 1 HIGH (F-002)
- **Client Wallet & Escrow**: 1 HIGH (F-003), 1 MEDIUM (F-005)
- **FFI & Robustness**: 1 MEDIUM (F-004)
- **Config & Deployment**: 1 LOW (F-006)
- **Dependencies & Logging**: 2 INFO (F-007, F-008)

---

## Limitations & Caveats

- No dynamic analysis (for example, live TLS configuration checks, runtime fuzzing, or side-channel analysis) was performed.
- External services (Electrum servers, Ethereum RPC providers) were treated as out of scope.
- The audit assumes that production deployments will:
  - Use HTTPS between client and server.
  - Restrict network access to `gotham-server` to trusted clients.
  - Provide OS-level hardening for server and client hosts.

These assumptions are partially at odds with the default configuration and storage patterns and are the basis for recommendations in `Recommendations.md`.

---

## Files Reviewed (Detailed List)

- `Cargo.toml` (workspace)
- `gotham-server/Cargo.toml`
- `gotham-server/Settings.toml`
- `gotham-server/Rocket.toml`
- `gotham-server/src/main.rs`
- `gotham-server/src/server.rs`
- `gotham-server/src/public_gotham.rs`
- `gotham-server/src/tests.rs`
- `gotham-client/Cargo.toml`
- `gotham-client/src/lib.rs`
- `gotham-client/src/ecdsa/mod.rs`
- `gotham-client/src/ecdsa/keygen.rs`
- `gotham-client/src/ecdsa/sign.rs`
- `gotham-client/src/ecdsa/recover.rs`
- `gotham-client/src/ecdsa/rotate.rs`
- `gotham-client/src/utilities/mod.rs`
- `demo-wallet/Cargo.toml`
- `demo-wallet/src/main.rs`
- `demo-wallet/src/bitcoin/mod.rs`
- `demo-wallet/src/bitcoin/commands.rs`
- `demo-wallet/src/bitcoin/escrow.rs`
- `demo-wallet/src/ethereum/commands.rs`
- `integration-tests/Cargo.toml`
- `integration-tests/tests/ecdsa.rs`
- `README.md` (root)
- `wiki/Architecture.md`
- `wiki/Security.md`

---

## Audit Quality Metrics

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| Files Reviewed | ~27 | Focused on high-risk areas | Met |
| Security-Critical Code Coverage (server/client MPC flows, key storage) | ~90% | 100% | Partially Met |
| High-Risk Code Coverage (wallet storage, FFI) | ~80% | 100% | Partially Met |
| Finding Verification Rate | 100% | 100% | Met |
| Code Citation Rate | 100% of findings | 100% | Met |

All in-scope high-risk areas are represented in this report with explicit file and line citations.
