# Gotham City - Security Audit Report

| Field | Value |
|-------|-------|
| **Repository** | https://github.com/ZenGo-X/gotham-city |
| **Audit Date** | 2025-12-07 |
| **Audit Tier** | FULL (Comprehensive) |
| **Audit Scope** | ALL (Security + Code Quality) |
| **Status** | CRITICAL ISSUES IDENTIFIED - DO NOT USE IN PRODUCTION WITHOUT HARDENING |
| **Analyzed Revision (SHA)** | 7ae75229599f8e34881611c65ea8c0ce59231127 |
| **Working Tree Status** | Clean |
| **Tech Stack** | Rust (Rocket 0.5.0-rc.1, reqwest 0.9.5, RocksDB 0.21.0), two-party-ecdsa, gotham-engine, demo Bitcoin/Ethereum wallets |

## Quick Start

### For Executives & Decision Makers

- **Read**: [`Executive_Summary.md`](Executive_Summary.md) (5–10 min) for overall risk, key statistics, and prioritized next steps.

### For Security Teams

- **Read**:
  1. [`Executive_Summary.md`](Executive_Summary.md) – Overview and risk posture
  2. [`Findings.md`](Findings.md) – Detailed findings with code citations and CVSS
  3. [`Coverage_Report.md`](Coverage_Report.md) – Scope, coverage, and limitations
  4. [`Dependencies_Report.md`](Dependencies_Report.md) – Dependency posture and lifecycle
  5. [`Recommendations.md`](Recommendations.md) – Remediation roadmap and OWASP/CWE mapping

### For Development Teams

- **Read**:
  1. [`Recommendations.md`](Recommendations.md) – Implementation guide and priorities
  2. [`Findings.md`](Findings.md) – Code examples and concrete fixes (by F-ID)
  3. [`Dependencies_Report.md`](Dependencies_Report.md) – Library upgrade guidance and CI integration

### For Compliance & Audit

- **Read**:
  1. [`Coverage_Report.md`](Coverage_Report.md) – Audit scope, methodology, files reviewed
  2. [`Executive_Summary.md`](Executive_Summary.md) – Risk level and key statistics
  3. [`Recommendations.md`](Recommendations.md) – OWASP/CWE coverage and follow-up plan

## Audit Findings at a Glance

- **Total Findings**: 8
- **Critical**: 1 (must be addressed before any production exposure)
- **High**: 2 (address before handling real funds)
- **Medium**: 2 (robustness and safety improvements)
- **Low/Info**: 3 (defense-in-depth and hygiene)

**Most Critical Issues** (all correspond to F-IDs in `Findings.md`):

1. F-001 – Default allow-all authorization for signing operations in `gotham-server`, making all MPC operations effectively authorized when deployed without additional access control.
2. F-002 – Plaintext storage of server key shares in RocksDB without encryption at rest or OS-level hardening.
3. F-003 – Plaintext storage of client wallet and escrow secrets in JSON files on disk in the demo wallet.

## File Guide

| File | Purpose | Audience | Read Time |
|------|---------|----------|-----------|
| **Executive_Summary.md** | High-level overview, risk areas, remediation priority | Leadership, PM, Security | 5–10 min |
| **Findings.md** | Detailed findings with code locations, CVSS, and remediation | Security, Developers | 30–60 min |
| **Recommendations.md** | Phase-based remediation plan and compliance mapping | Developers, DevOps, Security | 20–40 min |
| **Architecture.md** | Threat model, system design, attack surface | Architects, Security | 20–40 min |
| **Coverage_Report.md** | Audit scope, coverage metrics, limitations | QA, Audit, Security | 10–20 min |
| **Dependencies_Report.md** | Dependency posture and upgrade strategy | Developers, DevOps | 10–20 min |
| **README.md** | This file; front page and navigation | Everyone | 5–10 min |

## Priority Actions (Summary)

- **Phase 1 (Immediate)**: Implement robust authorization in `Db::granted()` and ensure the signing API is not reachable from untrusted networks without strong authentication and policy checks (F-001).
- **Phase 1 (Immediate)**: Protect server-side key shares and client wallet/escrow material via encryption at rest and filesystem hardening (F-002, F-003).
- **Phase 2 (Short-Term)**: Improve robustness of FFI boundaries and demo-wallet transaction handling; standardize integer-based amount handling and structured error propagation (F-004, F-005).
- **Phase 3 (Short-Term / Ongoing)**: Introduce dependency auditing and CI checks (`cargo audit`, SAST), add structured audit logging for signing operations, and harden deployment configuration (F-006, F-007, F-008).

## Risk Assessment Summary

- **Current Risk Level**: HIGH (see `Executive_Summary.md`)
- **Exploit Effort**: Moderate – requires network access to the server plus knowledge of key identifiers; impact is severe (funds at risk) if used with real assets.
- **Key Drivers**:
  - Default allow-all authorization implementation in `gotham-server`.
  - Plaintext storage of long-lived ECDSA key shares on both server and client.
  - Lack of encryption-at-rest, rate limiting, and structured audit logging around signing operations.

## Next Audit

- **After Phase 1**: Re-run a focused security audit on authorization and key storage, including a configuration review for the production environment.
- **After Phase 2**: Validate FFI and wallet robustness improvements with targeted tests and negative test cases.
- **Long-Term Cadence**: Align with release cycles; at minimum, perform an annual security review and ad-hoc audits after major protocol/dependency upgrades.
