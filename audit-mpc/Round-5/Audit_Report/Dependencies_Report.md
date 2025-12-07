# Dependencies Report - Round 5

**Audit Round**: R5  
**Analysis Date**: 2025-12-08  
**Status**: No changes from R4 - Dependencies unchanged

---

## Dependency Summary

| Category | Count | Vulnerable | Risk Level |
|----------|-------|------------|------------|
| Direct Dependencies | 28 | 0 known CVEs | Low-Medium |
| Workspace Dependencies | 10 | 0 known CVEs | Low |
| Dev Dependencies | 8 | 0 known CVEs | N/A |

---

## Server Dependencies (gotham-server)

### Direct Dependencies

| Crate | Version | Purpose | Security Notes |
|-------|---------|---------|----------------|
| `rocket` | workspace | HTTP framework | No TLS configured (R1-F-003) |
| `serde` | workspace | Serialization | Standard library |
| `serde_json` | workspace | JSON parsing | Deserialization risks (R2-F-003) |
| `log` | workspace | Logging | No sensitive data logging |
| `config` | workspace | Configuration | Environment variable support unused |
| `uuid` | workspace | ID generation | v4 random UUIDs |
| `failure` | workspace | Error handling | Legacy crate, consider `thiserror` |
| `jsonwebtoken` | workspace | JWT handling | **Present but UNUSED** (R1-F-001) |
| `hex` | workspace | Hex encoding | Standard utility |
| `two-party-ecdsa` | workspace | MPC crypto | Core cryptographic operations |
| `gotham-engine` | workspace | MPC logic | Protocol implementation |
| `rocksdb` | 0.21.0 | Database | Path injection risk (R2-F-004) |
| `chrono` | 0.4.26 | Time handling | Time-zone safe |
| `cargo-pants` | 0.4.16 | Build tool | Dev only |
| `redis` | 0.23.0 | Caching (optional) | Not used in default config |
| `thiserror` | 1.0 | Error handling | Modern error handling |
| `erased-serde` | 0.3 | Trait objects | Serialization support |
| `async-trait` | 0.1.73 | Async traits | Standard async support |
| `tokio` | 1.x | Async runtime | Full features enabled |

### Dev Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| `time-test` | 0.2.1 | Testing |
| `floating-duration` | workspace | Time formatting |
| `criterion` | 0.4.0 | Benchmarking |
| `pprof` | 0.11 | Profiling |
| `rand` | 0.8 | Random for tests |

---

## Client Dependencies (gotham-client)

### Direct Dependencies

| Crate | Version | Purpose | Security Notes |
|-------|---------|---------|----------------|
| `serde` | workspace | Serialization | Standard |
| `serde_json` | workspace | JSON | Deserialization risks |
| `log` | workspace | Logging | Standard |
| `reqwest` | workspace | HTTP client | No TLS verification |
| `failure` | workspace | Errors | Legacy crate |
| `floating-duration` | workspace | Timing | Performance metrics |
| `two-party-ecdsa` | workspace | MPC crypto | Core operations |

### Platform-Specific Dependencies

| Crate | Version | Platform | Purpose |
|-------|---------|----------|---------|
| `jni` | 0.19 | Android | JNI bindings (R2-F-006) |

### Dev Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| `mockall` | 0.11 | Mocking |

---

## Cryptographic Dependencies

### two-party-ecdsa (Workspace)

Core cryptographic library implementing Lindell'17 two-party ECDSA.

**Transitive Dependencies**:
- `curv` - Elliptic curve operations
- `paillier` - Homomorphic encryption
- `zk-paillier` - Zero-knowledge proofs
- `kms` - Key management structures

**Security Considerations**:
- Protocol implements Lindell'17 which has known abort vulnerabilities (R2-F-001)
- ZK proof verification may be incomplete (R2-F-002)
- Delegated to upstream maintainer for cryptographic correctness

### gotham-engine (Workspace)

MPC protocol coordination and route handling.

**Security Considerations**:
- Implements server-side MPC logic
- No authentication layer (R1-F-001)
- Route definitions for keygen/sign/rotate

---

## Vulnerability Analysis

### Known CVEs

**No known CVEs affecting current dependency versions.**

Scanned databases:
- RustSec Advisory Database
- GitHub Security Advisories
- NVD (National Vulnerability Database)

### Potential Risks

| Dependency | Risk | Mitigation |
|------------|------|------------|
| `rocksdb` 0.21.0 | Path injection | Validate db_name input (R1-F-004) |
| `serde_json` | Deserialization | Add schema validation (R2-F-003) |
| `jsonwebtoken` | Unused | Either implement auth or remove |
| `failure` | Deprecated | Migrate to `thiserror`/`anyhow` |
| `jni` 0.19 | FFI boundary | Add bounds checking (R2-F-006) |

---

## Dependency Configuration Issues

### Unused Dependencies

| Dependency | Status | Recommendation |
|------------|--------|----------------|
| `jsonwebtoken` | Imported but unused | **Implement authentication** or remove |
| `redis` | Optional, unused | Remove if not needed |

### Feature Flags

```toml
# gotham-server/Cargo.toml
[features]
default = ["local"]
local = ["rocksdb"]
```

**Note**: Only `local` feature is used; `rocksdb` is conditionally compiled.

---

## Supply Chain Security

### Checksums & Integrity

- All dependencies fetched from crates.io
- Cargo.lock present and committed
- No git dependencies (good)
- No path dependencies outside workspace (good)

### Recommendations

1. **Enable `cargo audit`** in CI pipeline
2. **Pin exact versions** in Cargo.lock (already done)
3. **Monitor RustSec advisories** for new vulnerabilities
4. **Consider Dependabot** or similar for automated updates

---

## Dependency Update Recommendations

### Security Updates (None Required)

No critical security updates pending.

### Maintenance Updates

| Crate | Current | Latest | Priority |
|-------|---------|--------|----------|
| `failure` | workspace | Deprecated | Low (migrate to thiserror) |
| `jni` | 0.19 | 0.21+ | Low |
| `rocksdb` | 0.21.0 | 0.22+ | Low |

---

## Round 5 Verification

### Changes Since R4

**None**. Dependency configuration unchanged.

### Verification Steps Performed

1. Reviewed `Cargo.toml` files - No changes
2. Checked for new CVEs - None found
3. Verified Cargo.lock integrity - Unchanged
4. Confirmed no new dependencies added - Confirmed

---

**Report Generated**: 2025-12-08 01:14 UTC+8  
**Dependency Status**: Unchanged from R4  
**Next Action**: Monitor for security advisories
