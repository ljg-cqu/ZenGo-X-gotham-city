# Executive Summary

| Field | Value |
|-------|-------|
| **Repository** | `/home/zealy/github/ljg-cqu/ZenGo-X-gotham-city` |
| **Audit Date** | 2025-12-08 |
| **Audit Period** | Not recorded (point-in-time static analysis) |
| **Tier** | FULL |
| **Audit Scope** | ALL |
| **Coverage Mode** | COMPREHENSIVE |
| **Analyzed Revision (SHA)** | 7e56c3bb7df5469aca3b233000b7adfaf17a8c00 |
| **Scope Base SHA** | N/A |
| **Scope Head SHA** | N/A |
| **Working Tree Status** | Dirty (uncommitted changes present) |
| **Audit Mode** | Draft / Pre-commit / WIP (Dirty Working Tree) |
| **Tech Stack** | Rust, Rocket, reqwest, RocksDB, ethers-rs, two-party-ecdsa, gotham-engine |
| **Tier Rationale** | ~55K LOC, MPC wallet server/client demo, cryptography and signing flows, explicit FULL tier requested |
| **Audit Round** | R2 |
| **Previous Rounds** | R1 (2025-12-08, MPC-focused, ProblemList FOCUSED) |
| **Baseline Findings** | 6 findings from previous rounds (R1: 6) |
| **New Findings (This Round)** | 1 finding (CRITICAL: 1, HIGH: 0, MEDIUM: 0, LOW: 0) |
| **Cumulative Findings** | 7 total findings across all rounds |

## Audit Results

| Severity | Count | Exploitable | Status |
|----------|-------|-------------|--------|
| CRITICAL | 4 | Yes (local/remote, depending on component) | Open |
| HIGH | 1 | Yes (network MitM / config) | Open |
| MEDIUM | 2 | — | Open |
| LOW | 0 | — | — |
| INFO | 0 | — | — |

**Total Findings (All Rounds)**: 7 (R1: 6 baseline, R2: 1 new)

- **New in R2**: 1 CRITICAL finding (insecure escrow secret storage in plaintext on disk, mapped to ProblemList 067).
- **Baseline from R1**: 3 CRITICAL, 1 HIGH, 2 MEDIUM findings (authorization bypass, plaintext key-share storage, HTTP only, DoS via panics, outdated dependencies) remain open and unmitigated.

## Overall Risk Assessment

**Risk Level**: CRITICAL

**Justification**:
- Server-side authorization remains effectively disabled for MPC transactions (`granted` always returns `true`).
- Both server and client continue to store long-lived key material and MPC key shares in plaintext on disk.
- Demo wallet escrow secrets are now also confirmed to be written unencrypted to JSON files, further expanding the blast radius of a host compromise.
- Network transport defaults to HTTP without TLS, and dependencies remain outdated, compounding risk.

Until all CRITICAL issues (R1-F-001, R1-F-002, R1-F-003, R2-F-001) are remediated, this codebase should not be used for real funds or production MPC custody.

## Key Statistics

- **Complexity**: Medium – small Rust workspace, but non-trivial MPC protocol integration and multi-asset wallet flows.
- **Attack Surface**:
  - HTTP Rocket server exposing ECDSA keygen/signing routes via `gotham-engine`.
  - Rust client library (`gotham-client`) used by mobile/CLI wallets over HTTP.
  - Demo Bitcoin and EVM wallets persisting MPC key shares and escrow material to local JSON files.
- **Auth Mechanisms**: Optional bearer token support in `ClientShim`; server-side `granted` policy hook currently implemented as unconditional allow.
- **Data Sensitivity**: MPC key shares for ECDSA keys controlling blockchain assets; escrow recovery material; wallet state.
- **Dependency Risk**: Medium – dependencies confirmed outdated in R1 (R1-F-006); no new dependency issues identified in R2 but previous risk remains.

## Definitions & Abbreviations

- **MPC**: Multi-Party Computation; here, two-party ECDSA signing (Lindell17-based) splitting a private key between client and server.
- **Key share**: One party’s secret share of the threshold private key.
- **Escrow secret**: Additional offline secret used to encrypt and later recover client key shares.
- **ProblemList**: External MPC-wallet problem library under `knowledge/Blockchain/Wallets/MPC/Problems` used as a focused risk catalog.
- **ProblemListMode = FOCUSED**: Only findings that map to one or more ProblemList entries are reported as F-IDs in this audit.
- **EssentialScope**: Security-critical code (entry points, MPC protocol glue, key storage, configuration) that must not be silently skipped.

## Coverage & Limitations

- **Pass Coverage**: R2 conceptually followed Passes 0–4A and 5 from the prompt, but as a targeted follow-up on top of R1’s FULL, COMPREHENSIVE sweep.
- **Scope Definition**: `AuditScope = ALL` with in-scope code limited to this workspace (`gotham-server`, `gotham-client`, `demo-wallet`, `integration-tests`). External git dependencies `gotham-engine` and `two-party-ecdsa` remain out-of-scope for line-level auditing; only their usage in this repo was examined.
- **Working Tree Status & Audit Mode**: Git working tree is **Dirty** (uncommitted changes under `audit-mpc-focused/Round-1`). This R2 report is therefore a **Draft / Pre-commit / WIP (Dirty Working Tree)** and is not itself a canonical compliance artifact, even though cumulative coverage remains COMPREHENSIVE when combined with R1.
- **Summary Mode**: R2 ran in summary/targeted mode over previously covered files, focusing on ProblemList-mapped storage flows (escrow, wallets) and confirming R1 findings; no new code areas were left unreviewed.
- **File & Target Coverage (Multi-Round)**:
  - **Previous Rounds (R1)**: ~12 core source files fully reviewed across `gotham-server`, `gotham-client`, and `demo-wallet`, with coverage reported as 100% of EssentialScope and COMPREHENSIVE for in-scope code.
  - **Current Round (R2, Incremental)**: Targeted re-review of key storage and MPC integration files:
    - `demo-wallet/src/bitcoin/escrow.rs`
    - `demo-wallet/src/ethereum/mod.rs`
    - `gotham-client/src/ecdsa/{keygen.rs,sign.rs,recover.rs}`
  - **Cumulative (All Rounds)**: No new in-scope files were added; cumulative coverage of EssentialScope and in-scope code remains 100% (COMPREHENSIVE) when combining R1 + R2.
- **EssentialScope Coverage**: All EssentialScope entries identified in R1 remain fully reviewed across R1+R2. This round deepened analysis of client/escrow storage but did not uncover additional EssentialScope gaps.
- **Signals Detected**:
  - `⟨Auth⟩`: Presence of an authorization hook (`granted`) with insecure implementation (R1-F-001).
  - `⟨API⟩`: HTTP-based MPC API exposed by Rocket server and consumed by `gotham-client`.
  - `⟨Infra⟩`: File-based RocksDB storage; local filesystem wallet and escrow JSON files.
- **Not in Scope**:
  - The internal implementation of `gotham-engine` and `two-party-ecdsa` git dependencies (only their public APIs and usage patterns were reviewed).
  - Runtime/production infrastructure (containers, Kubernetes, cloud accounts, network appliances) beyond what is implied by `Rocket.toml` and local settings.

## ProblemList Coverage (ProblemListMode = FOCUSED)

- **ProblemList Source**: `knowledge/Blockchain/Wallets/MPC/Problems` (full MPC wallet problem catalog). R2, like R1, is a **ProblemList-focused check**: only findings that clearly map to one or more ProblemList entries are promoted to F-IDs.
- **Mode**: `{ProblemListMode} = FOCUSED` – this audit intentionally **does not** report vulnerability types outside the supplied ProblemList, even if present. Non-ProblemList issues may exist; they are out of reporting scope by explicit configuration.

**Problem Evaluation Summary (Multi-Round)**

| Problem ID | Title | Status (R1+R2) | Related Findings (All Rounds) | Notes |
|-----------|-------|----------------|--------------------------------|-------|
| 044 | Backend Server Infrastructure Operational Security | Confirmed | R1-F-001, R1-F-004 | Authorization bypass and HTTP-only Rocket configuration remain present; no new mitigations observed in R2. |
| 067 | MPC Key Shard Persistence and Encrypted Storage Design Risks | Confirmed (expanded) | R1-F-002, R1-F-003, R2-F-001 | R1 covered RocksDB plaintext and wallet JSON plaintext; R2 adds escrow secret JSON plaintext storage (`escrow-bitcoin.json`), further increasing key exposure risk. |
| 086 | API Rate Limiting & DDoS Protection for MPC Signing Services | Confirmed | R1-F-005 | R2 re-confirms pervasive use of `unwrap`/`expect` and lack of defensive throttling; no new mitigations identified. |
| 039 | Cryptographic Library Supply Chain Attacks | Confirmed | R1-F-006 | Dependencies remain outdated; no upgrade or SBOM/CI hardening detected in R2. |
| 001–038, 040–043, 045–066, 068–442 | Other MPC risks | Not evaluated in detail | — | Still out of scope for this focused check; coverage has not been expanded to these problem IDs in R2.

When interpreting this report, treat all coverage and risk statements as **ProblemList-scoped**: they apply only to the MPC wallet risks modeled by the ProblemList entries above and do not imply absence of other vulnerability categories.

## Next Steps

1. **Immediate** (Before any production or real-funds usage):
   - Implement real authorization policy in `granted` and ensure all signing and keygen flows enforce it (R1-F-001).
   - Encrypt all key material at rest (R1-F-002, R1-F-003, R2-F-001): RocksDB values, wallet files, and escrow secrets.
   - Ensure HTTPS/TLS is enforced for all MPC traffic (R1-F-004) or terminate TLS in a hardened reverse proxy.
2. **Short-term** (Next iteration):
   - Replace `unwrap`/`expect` panics with structured error handling and backoff, and introduce basic rate limiting for signing/keygen APIs (R1-F-005).
   - Upgrade and audit dependencies; integrate `cargo audit` or equivalent into CI (R1-F-006).
3. **Long-term**:
   - Revisit MPC wallet architecture against broader ProblemList entries (beyond 039/044/067/086), and consider dynamic testing and formal verification for protocol-level properties.
