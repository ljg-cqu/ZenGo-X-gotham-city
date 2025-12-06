# Dependencies

## Workspace Dependencies

The project uses a Cargo workspace with shared dependencies defined in the root `Cargo.toml`:

| Priority | Package | Version | Purpose | License | Evidence |
|----------|---------|---------|---------|---------|----------|
| P1 | **two-party-ecdsa** | Git (ZenGo-X) | Core 2P-ECDSA cryptographic protocol | GPL-3.0 | [`Cargo.toml:25`](../Cargo.toml#L25) |
| P1 | **gotham-engine** | Git (ZenGo-X) | Server-side MPC route handlers | GPL-3.0 | [`Cargo.toml:26`](../Cargo.toml#L26) |
| P1 | **rocket** | 0.5.0-rc.1 | Async HTTP server framework | MIT/Apache-2.0 | [`Cargo.toml:18`](../Cargo.toml#L18) |
| P1 | **secp256k1** | 0.21.0 | Elliptic curve operations | CC0-1.0 | [`Cargo.toml:23`](../Cargo.toml#L23) |
| P2 | **serde** | 1.x | JSON serialization/deserialization | MIT/Apache-2.0 | [`Cargo.toml:12`](../Cargo.toml#L12) |
| P2 | **serde_json** | 1.x | JSON parsing | MIT/Apache-2.0 | [`Cargo.toml:13`](../Cargo.toml#L13) |
| P2 | **reqwest** | 0.9.5 | HTTP client | MIT/Apache-2.0 | [`Cargo.toml:15`](../Cargo.toml#L15) |
| P2 | **jsonwebtoken** | 8 | JWT handling | MIT | [`Cargo.toml:21`](../Cargo.toml#L21) |
| P3 | **uuid** | 0.7 | Session ID generation | MIT/Apache-2.0 | [`Cargo.toml:20`](../Cargo.toml#L20) |
| P3 | **config** | 0.9.2 | Configuration file parsing | MIT/Apache-2.0 | [`Cargo.toml:19`](../Cargo.toml#L19) |
| P3 | **log** | 0.4 | Logging facade | MIT/Apache-2.0 | [`Cargo.toml:14`](../Cargo.toml#L14) |
| P3 | **failure** | 0.1 | Error handling | MIT/Apache-2.0 | [`Cargo.toml:16`](../Cargo.toml#L16) |
| P3 | **floating-duration** | 0.1.2 | Duration formatting | MIT | [`Cargo.toml:17`](../Cargo.toml#L17) |
| P3 | **hex** | 0.4 | Hex encoding | MIT/Apache-2.0 | [`Cargo.toml:22`](../Cargo.toml#L22) |
| P3 | **rand** | 0.8 | Random number generation | MIT/Apache-2.0 | [`Cargo.toml:24`](../Cargo.toml#L24) |

---

## Dependency Health Matrix

| Priority | Package | Version | Last Release | Maintainer Activity | Health Score | Action |
|----------|---------|---------|--------------|---------------------|--------------|--------|
| P1 | two-party-ecdsa | Git | Active | ZenGo-X team | 🟡 Medium | Pin to specific commit |
| P1 | gotham-engine | Git | Active | ZenGo-X team | 🟡 Medium | Pin to specific commit |
| P1 | rocket | 0.5.0-rc.1 | 2022 | Active community | 🟡 Medium | Update when 0.5 stable |
| P1 | secp256k1 | 0.21.0 | 2022 | Bitcoin community | 🟢 Healthy | Consider update |
| P2 | reqwest | 0.9.5 | 2019 | Active | 🔴 Outdated | **Upgrade to 0.11+** |
| P2 | serde | 1.x | Active | dtolnay | 🟢 Healthy | Auto-update minor |
| P3 | failure | 0.1 | 2019 | Deprecated | 🔴 Deprecated | **Migrate to anyhow/thiserror** |
| P3 | uuid | 0.7 | 2019 | Active | 🟡 Outdated | Update to 1.x |
| P3 | config | 0.9.2 | 2019 | Active | 🟡 Outdated | Update to 0.13+ |

### Health Score Criteria

| Score | Last Release | Open CVEs | Maintainer Response | Bus Factor |
|-------|--------------|-----------|---------------------|------------|
| 🟢 Healthy | <1 year | 0 | <7 days | >3 maintainers |
| 🟡 Medium | 1-2 years | 0 | <30 days | 1-3 maintainers |
| 🔴 Outdated | >2 years | Any | >30 days or deprecated | 1 maintainer |

---

## Critical Dependencies

### two-party-ecdsa

**Repository**: https://github.com/ZenGo-X/two-party-ecdsa  
**Branch**: `compatibility_gotham_engine`

This is the core cryptographic library implementing Lindell's 2P-ECDSA protocol.

| Aspect | Value |
|--------|-------|
| Security Level | 128-bit (secp256k1 + 2048-bit Paillier) |
| Protocol | Lindell 2017 (Crypto17) |
| Dependencies | curv-kzen, paillier-zk |
| Audit Status | Review recommended |

**Risk**: Git dependency without version pinning. Upstream changes could break protocol compatibility.

**Mitigation**: Pin to specific commit SHA in production.

### gotham-engine

**Repository**: https://github.com/ZenGo-X/gotham-engine

Server-side MPC route handlers and trait definitions.

| Aspect | Value |
|--------|-------|
| Routes | 8 endpoints for keygen/sign |
| Traits | `Db`, `KeyGen`, `Sign` |
| Dependencies | two-party-ecdsa, rocket |

### rocket

**Version**: 0.5.0-rc.1 (Release Candidate)

| Feature | Status |
|---------|--------|
| Async | Enabled (tokio runtime) |
| JSON | Enabled via feature flag |
| TLS | Not enabled (reverse proxy recommended) |

**Risk**: RC version; API may change before stable release.

---

## Dependency Risk Categories

| Category | Packages | Monitoring | Update Policy | Evidence |
|----------|----------|------------|---------------|----------|
| **Cryptographic** | two-party-ecdsa, secp256k1 | Security advisories | Manual review | Security-critical |
| **Network** | reqwest, rocket | CVE databases | Auto-update patch | [`Cargo.toml:15,18`](../Cargo.toml#L15) |
| **Serialization** | serde, serde_json | CVE databases | Auto-update minor | [`Cargo.toml:12-13`](../Cargo.toml#L12-L13) |
| **Utility** | uuid, config, log | Dependabot | Auto-update | Low risk |

---

## License Compatibility

| License | Compatible | Packages | Obligation |
|---------|------------|----------|------------|
| GPL-3.0 | ⚠️ Copyleft | two-party-ecdsa, gotham-engine, gotham-city | Source disclosure required |
| MIT | ✅ Permissive | serde, reqwest, jsonwebtoken, failure | Attribution only |
| Apache-2.0 | ✅ Permissive | serde, reqwest, uuid | Attribution + patent grant |
| CC0-1.0 | ✅ Public Domain | secp256k1 | None |

**Note**: The project is GPL-3.0 licensed due to cryptographic dependencies. All derived works must also be GPL-3.0.

---

## Dependency Update Automation

| Tool | Config | Schedule | Auto-merge | Evidence |
|------|--------|----------|------------|----------|
| Dependabot | Not configured | N/A | N/A | — |
| cargo-audit | Manual | Ad-hoc | N/A | `cargo audit` |

### Recommended Setup

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    ignore:
      - dependency-name: "two-party-ecdsa"
      - dependency-name: "gotham-engine"
```

---

## Security Audit Commands

```bash
# Check for known vulnerabilities
cargo audit

# Update Cargo.lock with security patches
cargo update

# Verify dependency tree
cargo tree --duplicates

# Check outdated dependencies
cargo outdated
```

---

## Upgrade Recommendations

| Priority | Package | Current | Target | Breaking Changes | Effort |
|----------|---------|---------|--------|------------------|--------|
| P1 | reqwest | 0.9.5 | 0.11+ | Async API changes | High |
| P1 | failure | 0.1 | anyhow 1.x | Error type changes | Medium |
| P2 | uuid | 0.7 | 1.x | Minor API changes | Low |
| P2 | config | 0.9.2 | 0.13+ | Builder pattern changes | Medium |
| P3 | rocket | 0.5.0-rc.1 | 0.5 stable | None expected | Low (wait for release) |
