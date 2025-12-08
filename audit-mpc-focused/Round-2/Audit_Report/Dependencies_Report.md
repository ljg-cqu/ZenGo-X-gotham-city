# Dependencies Report (Round 2)

## Summary

Round 2 did not introduce any new dependency analysis beyond Round 1. The git status shows no changes to `Cargo.toml` files since the previous audit; therefore, all dependency-related risks from R1 remain valid and unresolved.

## Key Dependencies and Risk (Carried from R1)

| Crate | Version | Status | Risk (R1+R2) |
|-------|---------|--------|-------------|
| `rocket` | 0.5.0-rc.1 | Outdated release candidate | Medium (stability and potential security issues) |
| `reqwest` | 0.9.5 | Very outdated | Medium (missing modern TLS and security features) |
| `jsonwebtoken` | 8 | Outdated | Medium (older JWT implementation; should track current best practices) |
| `secp256k1` | 0.21.0 | Outdated | Low/Medium (crypto libraries should be kept current) |
| `two-party-ecdsa` | git | Unversioned git dependency | High (core MPC logic in external repo, not version-pinned) |
| `gotham-engine` | git | Unversioned git dependency | High (core routing/logic in external repo, not version-pinned) |

## Observations (R2)

- No dependency upgrades or structural changes were detected between R1 and R2; the same crates and versions are in use.
- The MPC and wallet logic still depend on git-based crates for critical functionality.
- The project still lacks an explicit SBOM or automated vulnerability scanning pipeline.

## Recommendations (Unchanged from R1)

1. **Upgrade to Supported Versions**
   - Move from `rocket` 0.5.0-rc.1 to a stable release.
   - Upgrade `reqwest` to the current supported series (requires async refactoring in `gotham-client`).
   - Bring `jsonwebtoken` and `secp256k1` up to date with their current stable versions.

2. **Reduce Git Dependency Risk**
   - Replace `two-party-ecdsa` and `gotham-engine` git dependencies with published crates or explicit commit hashes.
   - Maintain a documented process for tracking upstream changes, including security advisories.

3. **Automate Dependency Security Checks**
   - Integrate `cargo audit` (or equivalent) into CI and treat failing audits as build breakers.
   - Generate and maintain an SBOM for the project to aid incident response.

Until these actions are taken, the project remains exposed to the same dependency and supply-chain risks identified in Round 1.
