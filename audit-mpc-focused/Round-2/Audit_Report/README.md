# Gotham City - Security Audit Report (Round 2)

| Field | Value |
|-------|-------|
| **Repository** | `/home/zealy/github/ljg-cqu/ZenGo-X-gotham-city` |
| **Audit Tier** | FULL |
| **Audit Scope** | ALL |
| **Coverage Mode** | COMPREHENSIVE (multi-round, with R1) |
| **Status** | Critical issues identified; not safe for production use |
| **Analyzed Revision (SHA)** | 7e56c3bb7df5469aca3b233000b7adfaf17a8c00 |
| **Working Tree Status** | Dirty (audit artifacts under `audit-mpc-focused`) |
| **Audit Round** | R2 (ProblemList-focused follow-up) |
| **New Findings (R2)** | 1 (CRITICAL: 1, HIGH: 0, MEDIUM: 0) |
| **Total Findings (R1+R2)** | 7 |

This Round 2 report should be read **together with** the Round 1 audit under `audit-mpc-focused/Round-1/Audit_Report`. Round 2 only introduces one new CRITICAL finding and reconfirms the status of all existing issues.

## Quick Start

### For Executives and Decision Makers

- Read `Executive_Summary.md` (5–10 minutes) for overall risk, key statistics, and the combined R1+R2 picture.

### For Security Teams

- Read, in order:
  1. `Executive_Summary.md` – Overview, risk posture, and ProblemList coverage.
  2. `Findings.md` – New Round 2 finding and baseline status of R1 findings.
  3. `Coverage_Report.md` – What Round 2 re-reviewed and how it composes with R1 coverage.
  4. `Dependencies_Report.md` – Dependency status (unchanged since R1).
  5. `Recommendations.md` – Prioritized remediation roadmap across R1 and R2.

### For Development Teams

- Focus on:
  1. `Recommendations.md` – Implementation guidance and priorities across findings.
  2. `Findings.md` – Code locations, risk rationales, and suggested remediation patterns.
  3. `Architecture.md` – How MPC, storage, and escrow components fit together.

### For Compliance and Internal Audit

- Focus on:
  1. `Coverage_Report.md` – Scope, methodology, and multi-round coverage.
  2. `Executive_Summary.md` – Consolidated risk view and ProblemList alignment.
  3. `Recommendations.md` – Mapping to phases and policy controls.

## Audit Findings at a Glance (Multi-Round)

- **Total Findings**: 7
- **Critical**: 4 (R1: 3, R2: 1)
- **High**: 1 (R1)
- **Medium**: 2 (R1)
- **Low/Info**: 0

Most critical issues remain the same as in R1, with an expanded blast radius for key-storage risk:

1. **Authorization bypass** in the server `granted` hook (R1-F-001).
2. **Plaintext key share storage** on the server (R1-F-002) and in client wallets (R1-F-003).
3. **New in R2**: Plaintext escrow secret storage in `demo-wallet/src/bitcoin/escrow.rs` (R2-F-001), which further undermines key-separation assumptions.

## File Guide (Round 2)

| File | Purpose | Audience | Read Time |
|------|---------|----------|-----------|
| `Executive_Summary.md` | High-level overview, risk areas, multi-round ProblemList coverage | Leadership, PM, Security | 5–10 min |
| `Findings.md` | New R2 finding plus baseline R1 status, with code locations | Security, Devs | 15–30 min |
| `Recommendations.md` | Phase-based remediation plan across R1+R2 | Devs, DevOps, Security | 20–40 min |
| `Architecture.md` | System design, trust boundaries, R2 key-storage/escrow notes | Architects, Security | 15–30 min |
| `Coverage_Report.md` | Round 2 scope, files re-reviewed, and multi-round coverage | QA, Audit, Security | 10–20 min |
| `Dependencies_Report.md` | Dependency inventory and risk (carried forward from R1) | Devs, DevOps | 10–20 min |
| `README.md` | This file | Everyone | 5–10 min |

## Priority Actions (Combined R1+R2)

1. **Immediate**
   - Implement real authorization checks in `gotham-server` (`granted`) and enforce them on all keygen/signing paths.
   - Encrypt all key material at rest (server RocksDB, client wallet files, escrow secrets).
   - Do not use this codebase with real funds until all CRITICAL issues are mitigated and retested.

2. **Short-Term**
   - Enable TLS for all MPC traffic or front the service with a hardened TLS terminator.
   - Replace `unwrap`/`expect` patterns that can panic with structured error handling and basic rate limiting.

3. **Long-Term**
   - Modernize dependencies and add automated checks (`cargo audit`, SBOM generation).
   - Expand coverage to additional ProblemList entries beyond those targeted in this focused audit.

## Risk Assessment Summary

- **Current Risk Level**: Critical (unchanged from R1).
- **Exploit Effort**: Low to moderate. Many issues (authorization bypass, plaintext storage) are easy to exploit once an attacker has network or local access.
- **Key Drivers**: Unconditional authorization allow, pervasive plaintext storage of long-lived secrets (including escrow), and cleartext HTTP transport.

## Relationship to Round 1

- Round 1 performed the initial FULL, COMPREHENSIVE sweep and identified 6 findings.
- Round 2 is a **ProblemList-focused follow-up** that:
  - Confirms all R1 findings remain present and unmitigated.
  - Adds one new CRITICAL finding (R2-F-001) expanding ProblemList 067 coverage to escrow secrets.
  - Does not introduce new code areas; it refines analysis of key-storage and escrow flows.

For a complete understanding of system risk, read both Round 1 and Round 2 reports together.
