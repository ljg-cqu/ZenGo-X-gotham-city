# gotham-city - Security Audit Report

| Field | Value |
|-------|-------|
| **Repository** | https://github.com/ZenGo-X/gotham-city |
| **Audit Date** | 2025-12-07 |
| **Audit Tier** | FULL |
| **Audit Scope** | ALL |
| **Coverage Mode** | COMPREHENSIVE (coverage incomplete) |
| **Status** | Partial code audit – EssentialScope and comprehensive coverage incomplete; not suitable as a standalone production go/no-go decision |
| **Analyzed Revision (SHA)** | 5c9787dbf451771d71645994b278ebe78bc243eb |
| **Working Tree Status** | Clean |
| **Tech Stack** | Rust workspace (gotham-server, gotham-client, demo-wallet, integration-tests), Rocket 0.5.0-rc.1, two-party-ecdsa engine, gotham-engine, RocksDB, Redis (optional) |

## Quick Start

### For Executives & Decision Makers

- **Read**: `Executive_Summary.md` (5–10 min) for overall risk posture, key statistics, and coverage limitations.

### For Security Teams

- **Read**:
  1. `Executive_Summary.md` – Overview, risk posture, coverage & limitations, ProblemList summary
  2. `Findings.md` – Current findings with code citations
  3. `Coverage_Report.md` – Scope, coverage metrics, and explicit gaps (including "Comprehensive coverage incomplete")
  4. `Dependencies_Report.md` – Dependency footprint and high-level risk notes
  5. `Recommendations.md` – Remediation roadmap and follow-up actions

### For Development Teams

- **Read**:
  1. `Recommendations.md` – Implementation-oriented remediation guidance and priorities
  2. `Findings.md` – Code examples and concrete improvement items (by F-ID)
  3. `Dependencies_Report.md` – Dependency hygiene and future hardening steps

### For Compliance & Audit

- **Read**:
  1. `Coverage_Report.md` – Scope, methodology, and which files were / were not reviewed
  2. `Executive_Summary.md` – Risk level, statistics, and coverage statements
  3. `Recommendations.md` – High-level remediation and standards mapping (OWASP/CWE and general secure development guidance)

## Audit Findings at a Glance

- **Total Findings**: 3 (all LOW/INFO; this does **not** imply the absence of higher-severity issues due to incomplete coverage)
- **Critical**: 0
- **High**: 0
- **Medium**: 0
- **Low**: 2
- **Info**: 1

**Most Visible Issues** (all must correspond to F-IDs in `Findings.md`):

1. Configuration – Rocket debug configuration binds on `0.0.0.0` without a clearly separated production profile (`gotham-server/Rocket.toml`) – F-001 (LOW).
2. Business Logic – Transaction authorization hook `granted` is implemented as a stub that always returns `true` (`gotham-server/src/public_gotham.rs`) – F-002 (INFO: sample implementation placeholder; real deployments must replace it with policy logic).
3. Data Security – Default RocksDB-backed `PublicGotham` database stores MPC-related state in plaintext on local disk without documented encryption or retention controls – F-003 (LOW).

## File Guide

| File | Purpose | Audience | Read Time |
|------|---------|----------|-----------|
| **Executive_Summary.md** | High-level overview, key statistics, coverage & limitations, ProblemList status | Leadership, PM, Security, Compliance | 5–10 min |
| **Findings.md** | Detailed findings with code locations and suggested improvements | Security, Devs | 15–30 min (current run has 3 findings) |
| **Recommendations.md** | Phase-based remediation plan and future hardening work | Devs, DevOps, Security | 15–30 min |
| **Architecture.md** | System design, MPC flows, and attack surface | Architects, Security | 15–25 min |
| **Coverage_Report.md** | Audit scope, coverage metrics, explicit gaps, ProblemList coverage summary | QA, Audit, Security | 10–20 min |
| **Dependencies_Report.md** | Dependency footprint and lifecycle/hardening strategy | Devs, DevOps | 10–20 min |
| **README.md** | This file | Everyone | 5–10 min |

## Priority Actions (Summary)

- **Phase 0 – Treat this as a partial audit only**
  - Do not treat this run as a complete COMPREHENSIVE audit. EssentialScope and full in-scope coverage are both incomplete.
  - Schedule a follow-on audit that systematically expands coverage across `gotham-client`, `demo-wallet`, `integration-tests`, and the MPC engine dependencies.

- **Phase 1 – Hardening before any production consideration**
  - Replace the stub `granted` authorization hook with real transaction-approval policy logic and tests (F-002).
  - Introduce an at-rest protection and retention strategy for RocksDB data that may contain MPC session/state information, or migrate to a hardened store (F-003).
  - Define and document a clear separation between local/test and production Rocket configurations (binding, logging, TLS termination, reverse proxy expectations) (F-001).

- **Phase 2 – Broader security program and dependency hygiene**
  - Add automated dependency and RustSec scanning for all workspace crates (see `Dependencies_Report.md`).
  - Extend tests and monitoring to cover authorization flows, MPC engine integration, and error handling across the full stack.

## Risk Assessment Summary

- **Current Risk Level**: MEDIUM (provisional; coverage is incomplete and this rating cannot rule out undiscovered HIGH/CRITICAL issues).
- **Exploit Effort**: Not fully assessed; only a narrow slice of the codebase was reviewed.
- **Key Drivers**:
  - MPC/threshold signing domain and cryptographic dependencies (`two-party-ecdsa`, `gotham-engine`) are inherently high-impact and currently unaudited in this run.
  - Server exposes a public HTTP API and persists MPC-related state in RocksDB by default.
  - Transaction authorization logic is intentionally stubbed out in `PublicGotham` and must be replaced before any production use.

## Next Audit

- **After Phase 1**: Perform a targeted follow-up code audit focusing on the new authorization logic, RocksDB hardening (or alternative store), and any new production configuration profiles.
- **After Phase 2**: Perform a broader FULL audit with explicit COMPREHENSIVE-coverage goals across all workspace crates, and a separate, in-depth review of the external MPC engine dependencies.
- **Long-Term Cadence**: Re-audit at least annually or on major engine/cryptographic library upgrades, with additional focused reviews after significant architectural changes in MPC flows or external integrations.
