# Gotham City - Security Audit Report

| Field | Value |
|-------|-------|
| **Repository** | https://github.com/ZenGo-X/gotham-city |
| **Audit Date** | 2025-12-07 |
| **Audit Tier** | FULL |
| **Audit Scope** | ALL (Security + Code Quality) |
| **Coverage Mode** | COMPREHENSIVE (target **not achieved** – _"Comprehensive coverage incomplete"_) |
| **Status** | ⚠️ CRITICAL ISSUES IDENTIFIED – DO NOT USE DEFAULT CONFIGURATION IN PRODUCTION |
| **Analyzed Revision (SHA)** | 81e1110fe95fdd3efd60dace2793e4549f3dfd2c |
| **Working Tree Status** | Dirty (uncommitted changes present) |
| **Tech Stack** | Rust, Rocket 0.5.0-rc.1, RocksDB, secp256k1, two-party-ecdsa, ethers, electrumx-client |

## Quick Start

### For Executives & Decision Makers

- **Read**: [`Executive_Summary.md`](Executive_Summary.md) (**5–10 min**) for overall risk, key statistics, and remediation priorities.

### For Security Teams

- **Read**:
  1. [`Executive_Summary.md`](Executive_Summary.md) – Overview & risk posture.
  2. [`Findings.md`](Findings.md) – Detailed findings with code citations.
  3. [`Coverage_Report.md`](Coverage_Report.md) – Scope, coverage, and limitations (including "Comprehensive coverage incomplete").
  4. [`Dependencies_Report.md`](Dependencies_Report.md) – Dependency posture and recommendations.
  5. [`Recommendations.md`](Recommendations.md) – Remediation roadmap & OWASP/CWE mapping.

### For Development Teams

- **Read**:
  1. [`Recommendations.md`](Recommendations.md) – Implementation guide & priorities.
  2. [`Findings.md`](Findings.md) – Code examples and concrete fixes (by F-ID).
  3. [`Architecture.md`](Architecture.md) – Attack surface and data-flow context.

### For Compliance & Audit

- **Read**:
  1. [`Coverage_Report.md`](Coverage_Report.md) – Audit scope, methodology, files reviewed.
  2. [`Executive_Summary.md`](Executive_Summary.md) – Risk level & key statistics.
  3. [`Recommendations.md`](Recommendations.md) – OWASP/CWE mappings & follow-up plan.

## Audit Findings at a Glance

- **Total Findings**: 5
- **Critical**: 1 (F-001 – missing authorization on signing/keygen endpoints).
- **High**: 2 (F-002, F-003 – plaintext server and client key-share storage).
- **Medium**: 0
- **Low/Info**: 2 (F-004, F-005 – dependency/DoS hygiene gaps).

**Most Critical Issues**:

1. **Missing server-side authorization** – F-001: `Db::granted` always returns `true`, so any caller able to reach the HTTP API can drive keygen/signing flows.
2. **Plaintext server key-share storage** – F-002: RocksDB stores sensitive MPC state and key shares unencrypted on disk.
3. **Plaintext wallet and escrow secrets** – F-003: Demo wallet writes key material and escrow secrets as JSON files without encryption.

## File Guide

| File | Purpose | Audience | Read Time |
|------|---------|----------|-----------|
| **Executive_Summary.md** | High-level overview, risk areas, remediation priority | Leadership, PM, Security | 5–10 min |
| **Findings.md** | Detailed findings with code locations and fixes | Security, Devs | 30–60 min |
| **Recommendations.md** | Phase-based remediation plan with compliance mapping | Devs, DevOps, Security | 20–40 min |
| **Architecture.md** | System boundaries, containers, attack surface diagrams | Architects, Security | 20–40 min |
| **Coverage_Report.md** | Audit scope, coverage metrics, limitations | QA, Audit, Security | 10–20 min |
| **Dependencies_Report.md** | Dependency posture & update strategy | Devs, DevOps | 10–20 min |
| **README.md** | This file (navigation & summary) | Everyone | 5–10 min |

## Priority Actions (Summary)

- **Phase 1 (Immediate)**:
  - Enforce authentication and authorization on all `/ecdsa/*` endpoints (F-001).
  - Protect server and client key material at rest (F-002, F-003).
- **Phase 2 (Short-term)**:
  - Introduce rate limiting and monitoring for ECDSA endpoints (F-005).
  - Integrate `cargo audit` or equivalent dependency scanning in CI (F-004).
- **Phase 3 (Ongoing)**:
  - Expand coverage to remaining modules and external integrations; schedule periodic security reviews.

## Risk Assessment Summary

- **Current Risk Level**: CRITICAL (driven primarily by F-001 and the combination of F-002/F-003).
- **Exploit Effort**: TRIVIAL for F-001 in default deployments exposed to untrusted networks; MODERATE for F-002/F-003 (requires host or filesystem compromise).
- **Key Drivers**:
  - Default allow-all authorization hook in `PublicGotham`.
  - Plaintext storage of long-lived key material on both server and client.
  - Lack of automated dependency auditing and rate limiting.

## Next Audit

- **After Phase 1**: Re-audit Gotham server and demo wallet once F-001–F-003 are remediated and deployed on a clean working tree.
- **After Phase 2**: Validate operational hardening (rate limiting, monitoring, dependency scanning) and revisit threat model based on deployment realities.
- **Long-Term Cadence**: Annual or semi-annual code audit plus continuous dependency and configuration monitoring.

> **Integrity Rule**: All numbers, severities, and F-ID references in this `README.md` are derived from `Executive_Summary.md`, `Findings.md`, `Coverage_Report.md`, and `Dependencies_Report.md`. No new findings are introduced here.
