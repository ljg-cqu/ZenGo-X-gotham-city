# Executive Summary

| Field | Value |
|-------|-------|
| **Repository** | https://github.com/ZenGo-X/gotham-city |
| **Audit Date** | 2025-12-07 |
| **Audit Period** | 2025-12-07 to 2025-12-07 |
| **Tier** | FULL |
| **Audit Scope** | ALL (Security + Code Quality) |
| **Tech Stack** | Rust (Rocket 0.5.0-rc.1, reqwest 0.9.5, RocksDB 0.21.0), two-party-ecdsa, gotham-engine, demo Bitcoin/Ethereum wallets |
| **Tier Rationale** | ~35K LOC workspace with public HTTP server, MPC key management, and demo wallets interacting with external networks (Bitcoin/Ethereum). User explicitly requested FULL tier. |

## Audit Results

| Severity | Count | Exploitable | Status |
|----------|-------|-------------|--------|
| CRITICAL | 1 | Yes (under default configuration) | Open |
| HIGH | 2 | Yes (under realistic attacker models) | Open |
| MEDIUM | 2 | N/A | Open |
| LOW | 1 | N/A | Open |
| INFO | 2 | N/A | Open |

**Total Findings**: 8

## Overall Risk Assessment

**Risk Level**: HIGH

**Justification** (2-3 sentences):
The default `Db::granted()` implementation in `gotham-server` unconditionally authorizes all signing operations when used as-is, which can allow unauthorized signing if the surrounding deployment fails to provide compensating controls. In addition, Party1 server key shares and demo wallet key material are stored unencrypted on disk, and several dependencies (notably `rocksdb` 0.21.0 and `reqwest` 0.9.5) are outdated and missing modern security hardening. While the cryptographic protocol itself relies on well-studied primitives, the surrounding authorization and key-management posture requires hardening before any production use.

## Key Statistics

- **Complexity**: Medium — 4-workspace Rust project (server, client library, CLI wallet, integration tests) with external cryptographic engines and blockchains.
- **Attack Surface**: 1 public HTTP service (`gotham-server`) with multiple ECDSA keygen/signing endpoints; 1 CLI wallet that talks to external Electrum and Ethereum JSON-RPC endpoints.
- **Auth Mechanisms**: Optional bearer token support on the client; server-side authorization hook present but default implementation always grants.
- **Data Sensitivity**: High — long-lived ECDSA key shares and derived keys controlling Bitcoin/Ethereum funds.
- **Dependency Risk**: At least 2 high-risk, out-of-date dependencies on critical paths (`rocksdb` 0.21.0; `reqwest` 0.9.5) plus transitive cryptographic and networking stacks.

## Definitions & Abbreviations

- **2P-ECDSA**: Two-party Elliptic Curve Digital Signature Algorithm. A signing scheme where two parties jointly generate keys and produce signatures without ever reconstructing the full private key in one place.
- **MPC**: Multi-Party Computation. Cryptographic techniques that allow several parties to jointly compute functions over their inputs while keeping those inputs private.
- **Party1 / Party2**: Roles in the 2P-ECDSA protocol. In this project, `gotham-server` plays Party1 (server share) and `gotham-client` / wallet plays Party2 (client share).
- **Key Share**: A fragment of a private key held by one party in an MPC protocol. Alone it cannot sign, but combined with the other share it yields full signing power.
- **FFI**: Foreign Function Interface. The boundary where Rust exposes `extern "C"` and JNI functions to be called from iOS/Android or other native code.
- **RocksDB**: Embedded key-value store used here to persist Party1 key shares on the server.
- **CVSS**: Common Vulnerability Scoring System; standardized scoring for vulnerability severity.
- **CWE**: Common Weakness Enumeration; taxonomy of software weakness types (e.g., CWE-285 Improper Authorization).
- **F-ID**: Finding Identifier (e.g., F-001) used to uniquely label each issue across this audit.
- **HTTPS**: HTTP over TLS; required for secure transport of MPC messages between client and server.

## Coverage & Limitations

- **Pass Coverage**: Pre-Pass; Pass 0 (secrets scan on configs and manifests); Pass 1 (recon & attack surface on `README.md`, `wiki/Architecture.md`, `wiki/Security.md`); Pass 2 (injection & input handling focused on FFI and HTTP client/server paths); Pass 3 (auth/authz and business logic, focused on `public_gotham.rs` and demo wallet flows); Pass 4A (secrets & dependencies across `Cargo.toml` files and RocksDB usage); Pass 4B (data security around key-share storage and wallet backups); Pass 4C (infrastructure/configuration using `Rocket.toml` and Settings files); Pass 5 (synthesis & reporting). Passes 4B and 4C ran in focused mode due to limited infrastructure artifacts.
- **Summary Mode**: Some low-risk/demo-only areas (e.g., portions of `demo-wallet` command UX and non-critical tests) were reviewed in summary mode once CRITICAL/HIGH issues were identified.
- **Signals Detected**: `⟨Auth⟩` (authorization hook and optional bearer token), `⟨API⟩` (HTTP/JSON API between client and server), `⟨Infra⟩` (Rocket configuration, RocksDB usage, external Electrum/Ethereum RPC).
- **Known Gaps**: The external crates `gotham-engine` and `two-party-ecdsa` were treated as black boxes; this audit focuses on how they are integrated rather than re-auditing their internal cryptographic correctness. Dynamic analysis (live network traffic capture, TLS configuration validation, and runtime side-channel analysis) was out of scope. Some ancillary documentation and wiki content was used as context but not fully audited.

## Next Steps

1. Address F-001 (default allow-all authorization) before any production exposure of the signing API.
2. Plan and implement encryption-at-rest and access-control hardening for server key-share storage (F-002) and client-side key material (F-003).
3. Upgrade and re‑audit critical dependencies (`rocksdb`, `reqwest`, and transitive cryptographic/network stacks), using `cargo audit` and CI enforcement (F-006, F-007).
4. Add targeted robustness improvements to FFI boundaries to avoid process aborts and to limit error-detail propagation (F-004, F-005).
5. Schedule a follow-up audit after these items are remediated to validate fixes and reassess overall risk level.
