# Audit Coverage & Scope Report

| Field | Value |
|-------|-------|
| **Audit Tier** | FULL |
| **Audit Duration** | ~short, partial run (single session) |
| **Code Base Size** | ~38.8K LOC (workspace) |
| **Files Analyzed** | Small subset of gotham-server + top-level manifests |
| **Coverage Level** | Narrow slice of server and config only |
| **EssentialScope Coverage** | Incomplete |
| **Coverage Mode** | COMPREHENSIVE (coverage incomplete) |

## Coverage by Category

### 1. Configuration Files

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-server/Rocket.toml` | Reviewed | F-001 (debug binding on 0.0.0.0) |
| `gotham-server/Settings.toml` | Reviewed | None (usage observed via include + env) |
| `Cargo.toml` (workspace root) | Reviewed (summary) | None (structural observation only) |
| `gotham-server/Cargo.toml` | Reviewed (summary) | None (dependency notes only) |

**Findings Summary**: 1 LOW (F-001) in configuration.

---

### 2. Server Logic (gotham-server)

| File | Status | Issues Found |
|------|--------|--------------|
| `gotham-server/src/main.rs` | Reviewed | None |
| `gotham-server/src/server.rs` | Reviewed | None (routes/handlers wiring only) |
| `gotham-server/src/public_gotham.rs` | Reviewed | F-002 (authorization stub), F-003 (RocksDB hardening gap) |
| `gotham-server/src/tests.rs` | Reviewed (keygen/sign flow only) | None (used to discover routes and flows) |

**Findings Summary**: 1 INFO (F-002), 1 LOW (F-003) in server-side logic.

---

### 3. Other Areas

| Area | Status | Notes |
|------|--------|-------|
| `gotham-client` crate | Not reviewed | CLI wallet code and its security properties were not inspected in this run. |
| `demo-wallet` crate | Not reviewed | Example/demo wallet not inspected. |
| `integration-tests` (beyond `gotham-server/src/tests.rs`) | Not reviewed | Only server-side tests file inspected. |
| External MPC engine crates (`two-party-ecdsa`, `gotham-engine`) | Not reviewed | Treated as external dependencies requiring their own audits. |

**Findings Summary**: No findings recorded because these areas were not reviewed.

## ProblemList Coverage (ProblemList provided)

| Problem ID / Reference | Title | Status | Related Findings (F-IDs) | Mapped Files/Targets | Notes |
|------------------------|-------|--------|---------------------------|----------------------|-------|
| `Blockchain/Wallets/MPC/Problems/*.md` | MPC Wallet Problem Library (01–~395) | Not evaluated | — | — | This run did not attempt per-problem mapping. All entries remain `Not evaluated`. Future audits should select a prioritized subset and evaluate them against concrete code paths. |

## Files NOT Deeply Analyzed

### Intentional (Out of Scope for This Run)

- External MPC engine crates: `two-party-ecdsa`, `gotham-engine` (external repositories).

### Time-Limited (Would Require More Time)

- `gotham-client/**`
- `demo-wallet/**`
- `integration-tests/**` (except `gotham-server/src/tests.rs`)
- Any additional modules or configuration files beyond those listed as Reviewed above.

Because these are in-scope for a realistic deployment but unreviewed here, this audit is labeled **"Comprehensive coverage incomplete"** and **"Essential coverage incomplete"**.

## Files Reviewed (Detailed List)

- `Cargo.toml` (workspace root)
- `README.md` (workspace root)
- `gotham-server/Cargo.toml`
- `gotham-server/Rocket.toml`
- `gotham-server/Settings.toml`
- `gotham-server/src/main.rs`
- `gotham-server/src/server.rs`
- `gotham-server/src/public_gotham.rs`
- `gotham-server/src/tests.rs`

## Audit Quality Notes

- Coverage claims above are based solely on the files explicitly read during this run.
- No dynamic analysis or dependency/CVE scanning was performed; those activities should be part of a future, more complete audit.
