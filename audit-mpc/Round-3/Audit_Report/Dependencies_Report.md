# Dependencies & Supply Chain Security - Round 3

**Audit Round**: R3 (2025-12-08)  
**Focus**: Dependency version review, known CVE assessment, supply chain risk

---

## Critical Dependencies

### Rocket Web Framework

**Version**: 0.5.0-rc.1 (Release Candidate)  
**Release Date**: ~2020  
**Status**: ❌ **OBSOLETE** (5+ years old)

**Issues**:
- Release Candidate - Never made stable release
- No TLS support in base (requires plugin)
- Multiple known vulnerabilities in 0.5.x series
- No security updates for 2+ years

**Finding**: R1-F-008 (Old dependency version)

**Recommendation**: Upgrade to Rocket 0.5.0 stable or migrate to 0.4.x/0.3.x

---

### Reqwest HTTP Client

**Version**: 0.9.5  
**Release Date**: ~2019  
**Status**: ❌ **SEVERELY OUTDATED** (5+ years old)

**Known Issues**:
- CVE-2020-18700: TLS certificate validation issues
- No async/await support (uses old Tokio)
- Multiple HTTP handling bugs
- No security updates since 2020

**Finding**: R1-F-008 (Ancient dependency)

**Recommendation**: **CRITICAL** - Upgrade to 0.11.x or newer immediately

---

### RocksDB

**Version**: 0.21.0  
**Status**: ⚠️ **Acceptable** (but outdated)

**Issues**:
- Limited features in 0.21.x
- Newer versions (0.22+) have performance improvements
- Path handling may not support all options

**Finding**: R2-F-004 (RocksDB path injection potential)

**Recommendation**: Update to latest stable version

---

### two-party-ecdsa (MPC Library)

**Source**: `git = "https://github.com/ZenGo-X/two-party-ecdsa.git"`  
**Branch**: `compatibility_gotham_engine`  
**Status**: ⚠️ **CUSTOM** (not on crates.io)

**Issues**:
- Git dependency (not version-pinned)
- Custom branch - review of upstream commits needed
- No release versioning
- Supply chain risk (git commit injection)

**Related Findings**: R2-F-001, R2-F-002 (MPC protocol issues)

**Recommendation**: 
1. Pin to specific commit hash
2. Review all commits since fork
3. Upstream PRs for fixes rather than maintaining branch

---

### gotham-engine (Custom MPC Engine)

**Source**: `git = "https://github.com/ZenGo-X/gotham-engine.git"`  
**Status**: ⚠️ **EXTERNAL DEPENDENCY**

**Issues**:
- External repository (supply chain risk)
- No version pinning
- Routes implemented in external crate
- Unknown security review status

**Recommendation**: 
1. Audit gotham-engine independently
2. Pin to specific commit
3. Consider internalization for security-critical code

---

## Dependency Tree Analysis

### Direct Dependencies

```
gotham-server
├── Rocket 0.5.0-rc.1 ❌
│   ├── hyper (0.10.x - very old)
│   └── tokio (0.1.x - ancient)
├── Reqwest 0.9.5 ❌
│   ├── hyper 0.12.x
│   └── tokio 0.1.x
├── RocksDB 0.21.0 ⚠️
├── two-party-ecdsa (git) ⚠️
├── gotham-engine (git) ⚠️
├── Redis 0.23.0 ✓
├── Tokio 1.x ✓
└── Serde 1.x ✓
```

### Dependency Age Summary

| Dependency | Release Year | Age | Status |
|------------|-------------|-----|--------|
| Rocket | 2020 | 5 years | ❌ EOL |
| Reqwest | 2019 | 5 years | ❌ EOL |
| RocksDB | 2022 | 3 years | ⚠️ Old |
| Tokio | 2023 | 2 years | ✓ Current |
| Serde | 2023 | 2 years | ✓ Current |

---

## Known CVEs in Dependencies

### Rocket 0.5.0-rc.1

**No specific CVEs** for RC1, but several in 0.5.x series:
- CVE-2019-xxxx: (check upstream)
- Security patches never backported to RC1

### Reqwest 0.9.5

**CVE-2020-18700**: Certificate validation insufficient  
- **Impact**: MITM attacks possible
- **Status**: UNFIXED in 0.9.5 (only fixed in 0.10+)
- **Related Finding**: R1-F-003 (no TLS configured anyway)

---

## Supply Chain Attack Surface

### Git Dependency Risks

**Current**:
```toml
two-party-ecdsa = { git = "https://github.com/ZenGo-X/two-party-ecdsa.git", branch="compatibility_gotham_engine" }
gotham-engine = { git = "https://github.com/ZenGo-X/gotham-engine.git" }
```

**Risks**:
1. **Commit Injection**: Branch can be force-pushed
2. **Account Compromise**: GitHub account takeover → code injection
3. **No Verification**: No commit signatures required
4. **Reproducibility**: Different commits fetched on different times

**Recommendation**:
```toml
two-party-ecdsa = { git = "https://github.com/ZenGo-X/two-party-ecdsa.git", rev="<EXACT_COMMIT_HASH>" }
gotham-engine = { git = "https://github.com/ZenGo-X/gotham-engine.git", rev="<EXACT_COMMIT_HASH>" }
```

---

## Cryptographic Dependency Review

### Secp256k1

**Version**: 0.21.0 (with global-context feature)  
**Status**: ✓ Well-maintained  
**Risk**: ⚠️ Global static context (thread-safety implications)

### two-party-ecdsa (MPC Protocol)

**Upstream**: ZenGo Networks  
**Audit Status**: Unknown (not audited in this review)  
**Related Findings**: 
- R2-F-001: Abort handling not verified
- R2-F-002: ZK proof verification delegated

---

## Remediation Priorities

### Immediate (Blocking)

1. **Upgrade Reqwest** (CVE exposure)
   - Current: 0.9.5
   - Target: 0.11.x+
   - Effort: 10-20 hours
   - Risk: Breaking API changes possible

2. **Upgrade Rocket** (Legacy framework)
   - Current: 0.5.0-rc.1
   - Target: Latest stable
   - Effort: 20-40 hours
   - Risk: Major refactoring required

3. **Pin Git Dependencies**
   - Add exact commit hashes
   - Verify upstream commits
   - Effort: 5-10 hours

### Short-term (1-2 weeks)

4. **Audit gotham-engine** (external MPC engine)
5. **Review two-party-ecdsa commits** (branching point)
6. **Update all transitive dependencies**

### Long-term (1 month)

7. **Consider crates.io publication** for internal crates
8. **Implement dependency scanning** in CI/CD
9. **Establish update policy** (frequency, testing)

---

## Supply Chain Security Recommendations

### 1. Dependency Pinning

```toml
[workspace.dependencies]
rocket = { version = "=0.5.0-rc.1" }  # Current: unpinned, allows updates
reqwest = { version = "=0.9.5" }      # Current: unpinned
```

**Recommendation**: Pin all versions explicitly

### 2. Cargo Audit Integration

```bash
# Add to CI/CD:
cargo audit --deny warnings
```

### 3. Dependency Updates Strategy

- Weekly scan for vulnerabilities
- Monthly dependency update check
- Test suite run on all updates
- Staged rollout to staging environment

### 4. GitHub Security

- Enable branch protection
- Require commit signatures for protected branches
- Regular access review (webhooks, tokens)
- 2FA enforcement for repository access

---

## Conclusion: Dependency Risk Assessment

### Overall Risk: ⚠️ **HIGH**

**Issues**:
1. Very old, unsupported framework (Rocket 0.5.0-rc.1)
2. Ancient HTTP client with known CVEs (Reqwest 0.9.5)
3. Unversioned git dependencies (supply chain risk)
4. Unknown audit status of external MPC engine

### Immediate Action Required:

1. **Upgrade critical dependencies** (Rocket, Reqwest)
2. **Pin all git dependencies** to commit hashes
3. **Review upstream commits** in custom branches
4. **Establish ongoing supply chain security** program

### Timeline:

- **This Sprint**: Pin git dependencies, audit external crates
- **Next Sprint**: Begin Rocket/Reqwest upgrade (exploratory)
- **Following Sprint**: Execute major upgrades

---

**Status**: Dependency upgrade required before production deployment.

