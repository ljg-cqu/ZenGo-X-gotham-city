# Dependency Vulnerability & Lifecycle Analysis

| Field | Value |
|-------|-------|
| **Analysis Date** | 2025-12-07 |
| **Total Direct Dependencies** | Not fully enumerated (workspace-level observation only) |
| **Estimated Transitive Dependencies** | Not assessed |
| **Overall Dependency Risk** | Unknown – no automated CVE/RustSec scan performed in this run |

## Critical & High-Risk Dependencies (Qualitative)

This partial audit did **not** run automated tools such as `cargo audit` or RustSec scanners. Instead, it notes a few security-relevant dependencies from manifest inspection:

- `rocket` (0.5.0-rc.1) – web framework used by gotham-server.
- `rocksdb` – local key-value store used for MPC-related state.
- `redis` (with `cluster` feature) – optional integration for clustered Redis.
- `serde`, `serde_json` – serialization for protocol messages and stored values.
- `jsonwebtoken`, `config`, `uuid`, `reqwest`, and others – present in workspace manifests but not analyzed in detail.
- Git-based cryptographic/MPC engine dependencies:
  - `two-party-ecdsa`
  - `gotham-engine`

No specific CVEs, CVSS scores, or F-IDs are attached to dependencies in this run. All serious dependency issues, if any, remain **unknown** until a proper scan and analysis are completed.

## Recommended Next Steps for Dependencies

1. **Introduce RustSec / CVE Scanning**
   - Add a CI step (for example, `cargo audit` or equivalent) for each workspace crate.
   - For any HIGH/CRITICAL issues found, map them to usage sites in code and promote them to F-IDs in `Findings.md`.

2. **Track Engine Dependencies Explicitly**
   - Treat `two-party-ecdsa` and `gotham-engine` as critical dependencies and ensure they are covered by separate audits or trusted third-party reviews.
   - Document supported versions and any known security advisories affecting those crates.

3. **Rationalize and Minimize Dependencies**
   - Confirm that all declared crates in `Cargo.toml` files are actually required.
   - Remove unused dependencies to reduce attack surface and maintenance burden.

4. **Integrate Dependency Checks into Release Workflow**
   - Make dependency scans part of the release checklist.
   - Record a short summary and any new dependency-related F-IDs in future runs’ `Dependencies_Report.md` and `Findings.md`.

> In this run, no dependency-related F-IDs were created. Future audits should revisit this file after automated scanning and targeted code review of dependency usage.
