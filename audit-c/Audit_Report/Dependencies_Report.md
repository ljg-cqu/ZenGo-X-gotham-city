# Dependency Vulnerability & Lifecycle Analysis

| Field | Value |
|-------|-------|
| **Analysis Date** | 2025-12-07 |
| **Total Direct Workspace Dependencies** | ~15 (workspace + per-crate) |
| **Estimated Transitive Dependencies** | 50+ |
| **Overall Dependency Risk** | MEDIUM (qualitative – no automated scan run) |

## Critical & High-Risk Dependencies (Qualitative)

The workspace depends on several security-sensitive crates:

- **`rocket` 0.5.0-rc.1** – pre-release web framework; stay current with security advisories and consider upgrading to a stable release when available.
- **`secp256k1` 0.21.0** – core ECC implementation; widely used but must be kept up to date.
- **`two-party-ecdsa` (Git)** and **`gotham-engine` (Git)** – implement the 2P-ECDSA protocol; protocol correctness and constant-time behavior are critical but were not re-audited here.
- **`reqwest` 0.9.5** – HTTP client used by `gotham-client`; relatively old version, may accumulate CVEs over time.
- **`redis`**, **`rocksdb`**, **`tokio`** – infrastructure/runtime crates used on the server.

Each of these should be monitored for security advisories and updated as needed. This audit did **not** map specific CVEs to these packages.

## Dependency Audit Output (Summary)

- **Critical Vulnerabilities**: Not assessed (no `cargo audit` run in this audit).
- **High Vulnerabilities**: Not assessed.
- **Medium Vulnerabilities**: Not assessed.

Instead of listing CVEs without verification, this audit records **F-004 (INFO)** in `Findings.md` to capture the absence of automated dependency scanning and recommends integrating a tool such as `cargo audit` into CI.

## Problematic Dependency Patterns (Observed)

- Multiple security-sensitive dependencies (crypto, networking, databases) without automated monitoring.
- Git-based dependencies (`two-party-ecdsa`, `gotham-engine`) pinned by branch rather than immutable version tags; this can make reproducible builds and vulnerability triage harder.

## Version Management & Patch Strategy (Recommended)

### Immediate

- Add a CI job that runs `cargo audit` against all workspace members and fails on new HIGH/CRITICAL vulnerabilities unless explicitly waived.
- Record and track any discovered issues, mapping them to future F-IDs in `Findings.md`.

### Short-term

- Review Git-based dependencies and, where possible, pin to specific commits or tags.
- Document an upgrade policy for foundational crates (Rocket, secp256k1, two-party-ecdsa, ethers).

### Long-term

- Establish a quarterly dependency review cycle that:
  - Reviews `cargo audit` output.
  - Tests major-version upgrades in staging.
  - Updates threat models for new cryptographic or protocol-level findings in upstream libraries.

## Security Scanning Tools & CI/CD Integration (Recommended)

- **Tooling**: `cargo audit`, GitHub Dependabot (or similar), and optional SBOM tooling.
- **Integration**:
  - Run `cargo audit` on every PR and nightly against the default branch.
  - Use Dependabot (or equivalent) to propose version bumps and surface known CVEs.

## Next Dependency Audit

- **Recommended Date**: 2026-01-15 (aligned with remediation follow-up).
- **Triggering Events**: New major releases of core crypto crates, high-profile CVEs affecting Rocket, secp256k1, or two-party-ecdsa, and any significant architectural changes.

> All dependency risk statements in this file are qualitative and must be backed by future, explicit F-IDs and code/config citations if individual CVEs are to be tracked as findings.
