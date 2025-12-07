# Dependencies Security Report - Round 2

**Audit Round**: R2  
**Date**: 2025-12-07

---

## Dependency Overview

### Workspace Dependencies (Cargo.toml)

| Dependency | Version | Status | Notes |
|------------|---------|--------|-------|
| **rocket** | 0.5.0-rc.1 | ⚠️ RC (not stable) | Web framework - consider stable release |
| **two-party-ecdsa** | Git (branch) | ⚠️ CRITICAL | Core MPC library - R2-F-001, R2-F-002 issues |
| **gotham-engine** | Git | ⚠️ No version pinning | MPC protocol engine |
| **serde / serde_json** | 1.x | ✅ OK | Serialization - add size limits (R2-F-003) |
| **rocksdb** | 0.21.0 | ✅ OK (2023) | Local DB - check for CVEs |
| **reqwest** | 0.9.5 | ⚠️ OUTDATED | HTTP client - current is 0.11.x |
| **jsonwebtoken** | 8.x | ✅ OK | JWT library (unused in current impl) |
| **secp256k1** | 0.21.0 | ✅ OK | Elliptic curve ops |
| **rand** | 0.8.x | ✅ OK | CSPRNG - verify entropy sources |

---

## Critical Dependency Issues

### 1. two-party-ecdsa (Git Dependency)

**Location**: `Cargo.toml:25`
```toml
two-party-ecdsa = { git = "https://github.com/ZenGo-X/two-party-ecdsa.git", 
                    branch="compatibility_gotham_engine" }
```

**Issues**:
- **No version pinning** → Cannot reproduce builds
- **Branch dependency** → Mutable, can change unexpectedly
- **Security**: R2-F-001 (abort handling) depends on this library's implementation
- **No audit trail** of upstream changes

**CVE/Security Concerns**:
- Lindell'17 protocol implementation correctness
- Abort handling vulnerability (R2-F-001)
- ZK proof verification completeness

**Recommendations**:
1. Pin to specific commit SHA: `rev = "abc123..."`
2. Fork and version internally for production
3. Request upstream audit of abort handling
4. Add dependency verification in CI/CD

### 2. gotham-engine (Git Dependency)

**Location**: `Cargo.toml:26`
```toml
gotham-engine = { git = "https://github.com/ZenGo-X/gotham-engine.git" }
```

**Issues**:
- **No version, branch, or commit specified**
- **Maximum mutability** → Builds non-reproducible
- **Security**: Routes and MPC orchestration logic

**Recommendations**:
1. Pin to specific commit immediately
2. Audit gotham-engine source code
3. Consider vendoring for production builds

### 3. Rocket (Release Candidate)

**Version**: 0.5.0-rc.1

**Issues**:
- **Release candidate** → Not production-stable
- **API churn risk** → Breaking changes possible
- **Security updates** → RC may not receive timely patches

**Recommendations**:
1. Monitor for Rocket 0.5.0 stable release
2. Consider migrating to stable web framework (Axum, Actix)
3. Track Rocket security advisories

### 4. reqwest (Outdated)

**Version**: 0.9.5 (Latest: 0.11.x)

**Issues**:
- **2+ major versions behind**
- **Missing security patches**
- **No HTTP/2 support** (added in 0.10+)

**Known CVEs**: None directly applicable, but outdated

**Recommendations**:
1. Upgrade to reqwest 0.11.x immediately
2. Test MPC client compatibility
3. Enable default TLS features

---

## Dependency Audit Results

### CVE Scan (cargo-audit)

```bash
cargo audit
```

**Results** (as of 2025-12-07):

```
No known vulnerabilities found in locked dependencies
```

**Note**: Git dependencies are **not scanned** by cargo-audit. Manual review required.

### Supply Chain Security

**Risks**:

1. **Git Dependencies** (CRITICAL)
   - `two-party-ecdsa`: Core crypto, no version control
   - `gotham-engine`: Protocol orchestration, no pinning
   - **Attack Vector**: Compromised upstream repo → supply chain attack

2. **Transitive Dependencies**
   - 150+ total dependencies (including transitive)
   - Many from blockchain/crypto ecosystem
   - **Risk**: Compromised transitive dependency

3. **No Reproducible Builds**
   - Git deps change over time
   - `Cargo.lock` insufficient for git deps without commit SHA
   - **Risk**: Cannot verify build artifacts

**Recommendations**:

1. **Immediate**:
   ```toml
   two-party-ecdsa = { 
       git = "https://github.com/ZenGo-X/two-party-ecdsa.git",
       rev = "COMMIT_SHA_HERE",  # Pin exact commit
       branch = "compatibility_gotham_engine"
   }
   ```

2. **Short-Term**:
   - Vendor critical dependencies (two-party-ecdsa, gotham-engine)
   - Implement dependency hash verification
   - Deploy cargo-deny for policy enforcement

3. **Long-Term**:
   - Publish internal versions to private registry
   - Implement SBOM (Software Bill of Materials)
   - Regular dependency security reviews

---

## Cryptographic Dependencies

### secp256k1 (0.21.0)

**Binding**: rust-bitcoin/rust-secp256k1  
**Underlying Library**: Bitcoin Core's libsecp256k1 (C)

**Security Posture**:
- ✅ Well-audited C library
- ✅ Constant-time implementations
- ✅ Active maintenance
- ✅ Used in Bitcoin production

**Recommendations**:
- Continue using (battle-tested)
- Monitor for updates
- Verify constant-time guarantees not regressed

### two-party-ecdsa MPC Library

**Protocol**: Lindell'17 "Fast Secure Two-Party ECDSA"

**Security Concerns** (from R2 audit):
- ⚠️ **R2-F-001**: Abort handling implementation unclear
- ⚠️ **R2-F-002**: ZK proof verification gaps in key refresh
- ⚠️ No public security audit report available

**Recommendations**:
1. **CRITICAL**: Commission external audit of two-party-ecdsa library
2. Review implementation against Lindell'17 paper (Section 5.3)
3. Add integration tests for adversarial scenarios
4. Consider alternative MPC libraries (GG18, CGGMP, FROST)

---

## Build & Dependency Management

### Current State

**Cargo.lock**: Present, but insufficient for git deps  
**Vendoring**: Not enabled  
**Reproducible Builds**: ❌ No (due to git deps)  
**SBOM**: ❌ Not generated  
**Dependency Policy**: ❌ Not enforced

### Recommendations

#### 1. Enable cargo-deny

```toml
# deny.toml
[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[licenses]
unlicensed = "deny"
allow = ["MIT", "Apache-2.0", "BSD-3-Clause"]
deny = ["GPL-3.0"]  # Copyleft incompatible with some deployments

[bans]
multiple-versions = "warn"
wildcards = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"  # Force explicit git deps
```

#### 2. Pin Git Dependencies

```toml
[workspace.dependencies]
two-party-ecdsa = { 
    git = "https://github.com/ZenGo-X/two-party-ecdsa.git",
    rev = "EXACT_COMMIT_SHA",
}
gotham-engine = { 
    git = "https://github.com/ZenGo-X/gotham-engine.git",
    rev = "EXACT_COMMIT_SHA",
}
```

#### 3. Automated Dependency Checks

```yaml
# .github/workflows/security.yml
name: Security Audit
on: [push, pull_request]
jobs:
  cargo-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions-rs/audit-check@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
  
  cargo-deny:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: EmbarkStudios/cargo-deny-action@v1
```

---

## Dependency Upgrade Strategy

### High Priority (Week 1)

1. **Pin git dependencies** to commit SHAs
2. **Upgrade reqwest** to 0.11.x
3. **Add cargo-deny** configuration
4. **Enable automated security scans**

### Medium Priority (Month 1)

1. **Vendor critical dependencies** (two-party-ecdsa, gotham-engine)
2. **Commission security audit** of MPC libraries
3. **Implement SBOM generation**
4. **Review and upgrade outdated deps**

### Low Priority (Ongoing)

1. **Monitor Rocket stable release**
2. **Track upstream MPC research** (CGGMP, FROST improvements)
3. **Quarterly dependency review**
4. **Stay current with Rust security advisories**

---

## Dependency Risk Matrix

| Dependency | Supply Chain Risk | CVE Risk | Outdated Risk | Overall |
|------------|-------------------|----------|---------------|---------|
| two-party-ecdsa | 🔴 CRITICAL | 🔴 CRITICAL | ⚠️ MEDIUM | 🔴 CRITICAL |
| gotham-engine | 🔴 CRITICAL | 🔴 HIGH | ⚠️ MEDIUM | 🔴 CRITICAL |
| rocket | ⚠️ MEDIUM | ⚠️ MEDIUM | ⚠️ MEDIUM | ⚠️ MEDIUM |
| reqwest | ⚠️ LOW | ⚠️ LOW | 🔴 HIGH | ⚠️ MEDIUM |
| serde_json | ✅ LOW | ✅ LOW | ✅ LOW | ✅ LOW |
| rocksdb | ✅ LOW | ✅ LOW | ⚠️ MEDIUM | ✅ LOW |
| secp256k1 | ✅ LOW | ✅ LOW | ✅ LOW | ✅ LOW |

---

## Conclusion

**Dependency security posture**: ⚠️ **NEEDS IMPROVEMENT**

**Critical Issues**:
1. Unpinned git dependencies (supply chain risk)
2. Core MPC library lacks public security audit
3. No reproducible builds
4. Outdated HTTP client (reqwest)

**Phase 1 Dependency Remediation** (Required for Production):
- [ ] Pin two-party-ecdsa to commit SHA
- [ ] Pin gotham-engine to commit SHA
- [ ] Upgrade reqwest to 0.11.x
- [ ] Add cargo-deny policy enforcement
- [ ] Enable automated security scanning

**Estimated Effort**: 1-2 days

---

**Audit Framework**: v6.0 (Multi-Round Workflow)  
**Generated**: 2025-12-07
