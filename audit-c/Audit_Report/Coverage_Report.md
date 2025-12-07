# Audit Coverage & Scope Report

| Field | Value |
|-------|-------|
| **Audit Tier** | FULL |
| **Audit Duration** | ~2.0 hours (static review) |
| **Code Base Size** | ~36K LOC (workspace total) |
| **Files Analyzed** | ~20 key source and config files |
| **Coverage Level** | Security-critical paths prioritized; non-essential modules partially/unreviewed |
| **EssentialScope Coverage** | Partially reviewed – see below |
| **Coverage Mode** | COMPREHENSIVE (**Comprehensive coverage incomplete**) |

## Coverage by Category

### 1. Configuration & Secrets

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-server/Settings.toml` | Reviewed | Context for F-002 (no encryption-at-rest), AWS placeholders. |
| `gotham-server/Rocket.toml` | Reviewed | Listens on `0.0.0.0:8000`; TLS delegated to deployment. |
| `gotham-client/Settings.toml` | Reviewed | Default endpoint `http://localhost:8000` (dev). |

**Summary**: Input to F-002/F-003.

### 2. Authentication & Authorization

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-server/src/public_gotham.rs` | Reviewed | `Db::granted` always `Ok(true)` (F-001). |
| `gotham-server/src/server.rs` | Reviewed | Mounts `/ecdsa/*` routes without extra guards. |
| `gotham-client/src/lib.rs` | Reviewed | Client supports bearer token; server does not enforce by default. |

**Summary**: F-001 (CRITICAL broken access control).

### 3. 2P-ECDSA Orchestration & Tests

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-client/src/ecdsa/keygen.rs` | Reviewed | Protocol orchestration; no additional issues in reviewed slice. |
| `gotham-client/src/ecdsa/recover.rs` | Reviewed | FFI helpers and recovery; no new issues beyond storage concerns. |
| `integration-tests/tests/ecdsa.rs` | Reviewed | End-to-end keygen/signing flow; used as attack-surface reference. |

**Summary**: No extra findings beyond F-001–F-003; cryptographic correctness left to external crates.

### 4. Wallet Flows (Bitcoin & EVM)

| File | Status | Issues Found |
|------|--------|--------------|
| `demo-wallet/src/main.rs` | Reviewed | CLI + config handling understood. |
| `demo-wallet/src/bitcoin/mod.rs` | Reviewed | Plaintext wallet and backup storage (F-003). |
| `demo-wallet/src/bitcoin/escrow.rs` | Reviewed | Plaintext escrow secret storage (F-003). |
| `demo-wallet/src/bitcoin/commands.rs` | Reviewed | Electrum integration; printing/exit-based error handling. |
| `demo-wallet/src/ethereum/commands.rs` | Reviewed | EVM wallet control, `no_mpc` flag. |

**Summary**: F-003 (HIGH data-security risk on client-side secrets).

## Files NOT Deeply Analyzed

### Intentional (Out of Scope for Code Review)

- External crates: `two-party-ecdsa`, `gotham-engine`, `ethers`, `electrumx_client` – treated as black boxes.
- Most wiki content beyond `Front_Page.md` and `Security.md` – used for context only.

### Time/Context-Limited (In-Scope but Not Reviewed or Only Sampled)

- Additional Gotham client ECDSA modules:
  - `gotham-client/src/ecdsa/sign.rs`
  - `gotham-client/src/ecdsa/rotate.rs`
  - `gotham-client/src/ecdsa/types.rs`
- Other benches/tests:
  - `gotham-server/benches/*.rs` and any extra tests beyond `src/tests.rs`.

Because these in-scope files were not reviewed and `CoverageMode = COMPREHENSIVE`, this audit is explicitly marked **"Comprehensive coverage incomplete"**.

## Limitations & Caveats

- Static review only; no live environment, fuzzing, or dynamic testing.
- No automated dependency CVE scan executed as part of this run; dependency risk discussed qualitatively.
- Audit was performed on a dirty working tree; for compliance, rerun on a clean commit.

## Files Reviewed (List)

- Workspace / docs: `Cargo.toml`, `README.md`, `wiki/Front_Page.md`, `wiki/Security.md`
- Gotham server: `src/main.rs`, `src/lib.rs`, `src/server.rs`, `src/public_gotham.rs`, `src/tests.rs`, `Rocket.toml`, `Settings.toml`
- Gotham client: `src/lib.rs`, `src/ecdsa/keygen.rs`, `src/ecdsa/recover.rs`, `Settings.toml`
- Demo wallet: `src/main.rs`, `src/bitcoin/mod.rs`, `src/bitcoin/escrow.rs`, `src/bitcoin/commands.rs`, `src/ethereum/commands.rs`
- Integration tests: `tests/ecdsa.rs`
