# Audit Coverage and Scope Report (Round 2)

| Field | Value |
|-------|-------|
| **Audit Tier** | FULL (incremental, building on R1) |
| **Code Base Size** | ~55K LOC (from R1; repo unchanged) |
| **Coverage Level (R2 only)** | Targeted review of key storage and escrow components |
| **EssentialScope Coverage (R2)** | Escrow and Ethereum wallet storage paths re-reviewed |
| **Coverage Mode** | COMPREHENSIVE (multi-round, with R1) |
| **Audit Round** | R2 |

Round 2 is a **focused follow-up** on top of the Round 1 FULL, COMPREHENSIVE audit. It does not attempt to re-scan the entire codebase; instead it deepens coverage of ProblemList 067 by reviewing escrow and Ethereum wallet storage paths.

## Coverage by Category (R2 Focus)

### 1. Server Logic

| File | Status (R2) | Notes |
|------|-------------|-------|
| `gotham-server/src/public_gotham.rs` | Spot-checked | R1 server findings (authorization bypass, plaintext RocksDB, DoS via panics) confirmed still present; no new issues beyond R1-F-001, R1-F-002, R1-F-005 recorded in R2. |

### 2. Client Logic

| File | Status (R2) | Notes |
|------|-------------|-------|
| `gotham-client/src/ecdsa/keygen.rs` | Spot-checked | Confirms continued use of `unwrap`/`expect` patterns; no new distinct findings beyond R1-F-005. |
| `gotham-client/src/ecdsa/sign.rs` | Spot-checked | Same DoS risk pattern as R1; still present. |
| `gotham-client/src/ecdsa/recover.rs` | Reviewed (targeted) | Recovery and key reconstruction logic reviewed to understand interaction with escrow; no new ProblemList-mapped issues beyond key-storage concerns captured in R2-F-001. |

### 3. Wallet Logic

| File | Status (R2) | Notes |
|------|-------------|-------|
| `demo-wallet/src/bitcoin/mod.rs` | Reviewed (targeted) | Confirms plaintext wallet storage (R1-F-003) remains; no new mitigation. |
| `demo-wallet/src/ethereum/mod.rs` | Reviewed (targeted) | Confirms `GothamWallet::save` persists the MPC private share via `serde_json` without encryption; treated as part of R1-F-003 scope. |

### 4. Escrow and Recovery Logic (New R2 Emphasis)

| File | Status (R2) | Issues Found |
|------|-------------|-------------|
| `demo-wallet/src/bitcoin/escrow.rs` | Reviewed (full) | New CRITICAL finding R2-F-001 (plaintext escrow secret storage). |

## Multi-Round Coverage Summary

### Round 1 (Baseline)

- Files analyzed: 12+ core logic files across configuration, server, client, and wallet modules.
- EssentialScope coverage: 100% for the server, client ECDSA flows, core wallet logic, and key configuration files.
- Coverage mode: COMPREHENSIVE for the in-scope code.

### Round 2 (Incremental)

- Expands and refines coverage for ProblemList 067 by:
  - Reviewing Bitcoin escrow secret handling in `demo-wallet/src/bitcoin/escrow.rs`.
  - Confirming Ethereum wallet storage behavior in `demo-wallet/src/ethereum/mod.rs`.
  - Re-checking ECDSA recover and wallet reconstruction flows for assumptions around persisted secrets.
- Confirms that all R1 findings remain open and unmitigated.

### Cumulative (R1 + R2)

- EssentialScope as defined in R1 (server entry points, MPC protocol glue, key storage, configuration) remains fully covered.
- Round 2 adds explicit coverage of escrow secret storage as an extension of the key-persistence threat model.
- No new in-scope directories or components were added in R2; instead, coverage within the existing wallet and escrow modules was deepened.

## ProblemList Coverage (ProblemListMode = FOCUSED)

The audit continues to operate in **FOCUSED** mode, constrained by the Blockchain MPC Wallet ProblemList.

| Problem ID | Title | Coverage Status (R1+R2) | Related Findings |
|-----------|-------|--------------------------|------------------|
| 044 | Backend Server Infrastructure Operational Security | Confirmed (R1-only) | R1-F-001, R1-F-004 |
| 067 | MPC Key Shard Persistence and Encrypted Storage Design Risks | Confirmed and expanded (R1+R2) | R1-F-002, R1-F-003, R2-F-001 |
| 086 | API Rate Limiting and DDoS Protection for MPC Signing Services | Confirmed (R1-only) | R1-F-005 |
| 039 | Cryptographic Library Supply Chain Attacks | Confirmed (R1-only) | R1-F-006 |
| 001–038, 040–043, 045–066, 068–442 | Other MPC Wallet Risks | Not evaluated in detail | – |

Round 2 does **not** expand the ProblemList beyond the entries already touched in R1; it instead strengthens coverage of Problem 067 by including escrow secret storage.

## Limitations (R2)

- **Scope**: R2 is intentionally incremental and does not re-audit every file from R1.
- **Dynamic Testing**: As in R1, R2 remains focused on static review; no fuzzing, black-box, or runtime analysis was added in this round.
- **External Dependencies**: `gotham-engine` and `two-party-ecdsa` remain out of scope for line-level review; only their integration points in this repository were considered.
- **ProblemList Focus**: Findings that do not clearly map to the supplied ProblemList remain out of reporting scope, even if suspected during review.
