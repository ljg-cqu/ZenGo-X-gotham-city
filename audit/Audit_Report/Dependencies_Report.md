# Dependency Vulnerability & Lifecycle Analysis

| Field | Value |
|-------|-------|
| **Analysis Date** | 2025-12-07 |
| **Total Direct Workspace Dependencies** | ~10 (shared via root `Cargo.toml`) |
| **Estimated Transitive Dependencies** | 50+ (including cryptography, networking, and async runtime crates) |
| **Overall Dependency Risk** | MEDIUM (demo code with security-sensitive dependencies but no automated audit pipeline) |

## Critical & High-Risk Dependencies

This section highlights dependencies on the security-critical path and summarizes their role and current risk posture. All referenced risks are backed by findings in `Findings.md`.

### 1. `rocksdb` 0.21.0

- **Status**: Embedded key-value store used by `gotham-server` to persist Party1 key shares.
- **Known Issues**:
  - Snyk advisory `SNYK-RUST-ROCKSDB-2980271` (Out-of-bounds Read) affects versions `<0.19.0` with CVSS 5.9; the currently used version 0.21.0 is **not** in the vulnerable range.
- **Related Findings**: F-002 (plaintext key shares at rest – data security issue independent of specific CVEs).
- **Risk Assessment**:
  - Primary risk arises from application use (plaintext secret storage) rather than from a known library vulnerability.
  - A host compromise or filesystem read access is sufficient to extract key shares.
- **Mitigation / Upgrade Plan**:
  - Keep `rocksdb` at a maintained version and re-run `cargo audit` after upgrades.
  - Implement encryption-at-rest for values written to RocksDB (see F-002 remediation).

### 2. `reqwest` 0.9.5

- **Status**: HTTP client used by `gotham-client` via `ClientShim` for MPC protocol messages.
- **Known Issues**:
  - This is an older, pre-1.0 line; while no specific CVEs were confirmed in this audit window, the age of the dependency increases the risk of unpatched issues and weaker defaults.
- **Related Findings**: F-007 (lack of automated dependency risk management).
- **Risk Assessment**:
  - Any TLS or HTTP parsing vulnerabilities in this version would directly affect the MPC protocol channel between client and server.
- **Mitigation / Upgrade Plan**:
  - Plan migration to a current `reqwest` release and re-test the client library.
  - Use `cargo audit` and upstream advisories to validate the chosen version.

### 3. `rocket` 0.5.0-rc.1

- **Status**: Pre-1.0 web framework used by `gotham-server` for HTTP routing.
- **Known Issues**:
  - No specific CVEs confirmed in this audit window; risk is primarily around pre-1.0 API stability and configuration.
- **Related Findings**: F-006 (debug profile binding to `0.0.0.0`), F-007.
- **Risk Assessment**:
  - Misconfiguration (for example, binding debug profile on public interfaces without TLS) can expose MPC APIs to untrusted clients.
- **Mitigation / Upgrade Plan**:
  - Track Rocket’s stable releases and migration guides, and move to a supported release once available.
  - Ensure production deployments use hardened configuration (TLS, restricted interfaces, reverse proxy).

### 4. `two-party-ecdsa` (Git dependency) and `gotham-engine`

- **Status**: Cryptographic and protocol glue crates pulled from GitHub (`two-party-ecdsa` branch `compatibility_gotham_engine`, and `gotham-engine`).
- **Known Issues**:
  - Not assessed in this audit; these crates are treated as external, domain-specific dependencies.
- **Related Findings**: F-007.
- **Risk Assessment**:
  - Any vulnerability here would directly affect key generation and signing correctness.
- **Mitigation / Upgrade Plan**:
  - Pin to reviewed tags or commit hashes instead of tracking moving branches.
  - Periodically review upstream repositories for security advisories and updates.

---

## Transitive Dependencies (High-Risk Branches)

This audit did not enumerate all transitive dependencies. Instead, it focused on confirming that key high-risk crates (`rocksdb`, `reqwest`, `rocket`, `secp256k1`, `serde`, `jsonwebtoken`) are used in straightforward, documented ways.

For future audits:

| Package | Severity | Issue | Remediation | Finding(s) |
|---------|----------|-------|------------|------------|
| `{dep}` | {LOW/MEDIUM/HIGH} | {To be populated by `cargo audit`} | {Upgrade/remove} | {Map to F-IDs in follow-up report} |

---

## Dependency Audit Output (Summary)

At the time of this report:

- **Critical Vulnerabilities**: 0 confirmed for the specific versions in use.
- **High Vulnerabilities**: 0 confirmed.
- **Medium Vulnerabilities**: 1 advisory noted for `rocksdb` (<0.19.0) but not applicable to 0.21.0.
- **Low Vulnerabilities**: Not enumerated.

All future vulnerability claims should be backed by explicit tool output (`cargo audit`, Snyk, etc.) and verified against actual imports and code paths.

---

## Problematic Dependency Patterns

- Use of older, pre-1.0 HTTP client (`reqwest` 0.9.5) on a critical path.
- Git-based dependencies (`two-party-ecdsa`, `gotham-engine`) tracking branches instead of immutable tags.
- Lack of documented or automated dependency update policy.

Provide code or config excerpts where helpful, but treat `Findings.md` as the primary source for vulnerability details and remediation (see F-002 and F-007).

---

## Version Management & Patch Strategy

### Immediate Actions (High Priority)

- Run `cargo audit` across the workspace and triage any reported vulnerabilities.
- For each advisory, confirm reachability (imported and used) and map to future F-IDs if needed.

### Short-term (Next Sprint)

- Migrate `reqwest` to a supported, current version and re-test MPC flows.
- Review and, if appropriate, upgrade `rocket` and related ecosystem crates once a stable line is available.

### Long-term (Quarterly)

- Establish a quarterly dependency review and upgrade window.
- Document supported versions of core dependencies and required minimum versions for downstream integrators.

---

## Security Scanning Tools & CI/CD Integration

- Integrate `cargo audit` into CI to catch newly published advisories.
- Consider adding SAST tools suitable for Rust (for example, Clippy with security-focused lints, or third-party scanners) to detect unsafe patterns in FFI, error handling, and configuration.
- Ensure CI logs do not expose secrets or key material.

---

## Next Dependency Audit

- **Recommended Date**: Within 3–6 months, or immediately after any major dependency upgrade.
- **Triggering Events**: Major framework/library releases, high-profile CVEs affecting Rust crates, or significant protocol changes.

All vulnerabilities and risk assessments in this report are backed by the findings listed in `Findings.md` and code/config citations from the main audit. Future audits should continue to use F-IDs for traceability.
