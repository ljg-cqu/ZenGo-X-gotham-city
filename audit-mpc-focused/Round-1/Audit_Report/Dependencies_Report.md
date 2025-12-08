# Dependencies Report

## Summary
The project relies on several outdated dependencies, which poses a security risk.

## Critical Dependencies

| Crate | Version | Status | Risk |
|-------|---------|--------|------|
| `rocket` | 0.5.0-rc.1 | Outdated | Medium (RC version, potential stability/security issues) |
| `reqwest` | 0.9.5 | Very Outdated | Medium (Missing modern security features) |
| `jsonwebtoken` | 8 | Outdated | Medium (Known vulnerabilities in older JWT libs) |
| `secp256k1` | 0.21.0 | Outdated | Low/Medium (Crypto lib should be kept up to date) |
| `two-party-ecdsa` | git | Unverified | High (Git dependency, no version pinning, hard to audit) |
| `gotham-engine` | git | Unverified | High (Git dependency, core logic hidden) |

## Recommendations
1.  **Update to Stable**: Move from `rocket` RC to stable 0.5.
2.  **Modernize**: Update `reqwest` to 0.11+ (requires async refactoring).
3.  **Pin Versions**: Avoid git dependencies in production; publish crates or use specific commit hashes.
