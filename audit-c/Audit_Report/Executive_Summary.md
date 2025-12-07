# Executive Summary

| Field | Value |
|-------|-------|
| **Repository** | https://github.com/ZenGo-X/gotham-city |
| **Audit Date** | 2025-12-07 |
| **Audit Period** | 2025-12-07 to 2025-12-07 |
| **Tier** | FULL |
| **Audit Scope** | ALL (Security + Code Quality) |
| **Coverage Mode** | COMPREHENSIVE (target **not achieved** – _"Comprehensive coverage incomplete"_) |
| **Analyzed Revision (SHA)** | 81e1110fe95fdd3efd60dace2793e4549f3dfd2c |
| **Scope Base SHA** (SPECIFIED+change-based only) | N/A |
| **Scope Head SHA** (SPECIFIED+change-based only) | N/A |
| **Working Tree Status** | Dirty (uncommitted changes present) |
| **Audit Mode** | Draft / Pre-commit / WIP (Dirty Working Tree) |
| **Tech Stack** | Rust, Rocket 0.5.0-rc.1, RocksDB, secp256k1, two-party-ecdsa, ethers, electrumx-client |
| **Tier Rationale** | ~36K LOC Rust workspace; user-facing cryptographic service with HTTP API and MPC wallet flows; security-critical key management and signing. |

## Audit Results

| Severity | Count | Exploitable | Status |
|----------|-------|-------------|--------|
| CRITICAL | 1 | Yes (if server exposed without strong network controls) | Open |
| HIGH | 2 | Yes (with host/filesystem compromise) | Open |
| MEDIUM | 0 | — | — |
| LOW | 0 | — | — |
| INFO | 2 | N/A | Open |

**Total Findings**: 5

## Overall Risk Assessment

**Risk Level**: CRITICAL

**Justification**: The default Gotham server implementation exposes ECDSA keygen/signing APIs with **no built-in authorization** (`Db::granted` always returns `true`), which can allow unauthorized signing if the service is reachable from untrusted networks. In addition, both server and client store long-lived key shares and escrow secrets as plaintext on disk. While the repository is labeled as research / demo software, these defaults are dangerous if deployed without additional hardening.

## Key Statistics

- **Complexity**: Medium – single Rust workspace with 4 members (server, client, demo wallet, integration tests) and several external cryptographic crates.
- **Attack Surface**: 1 primary HTTP service (Rocket server) with multiple `/ecdsa/*` endpoints, plus CLI tools that talk to Electrum and Ethereum RPC endpoints.
- **Auth Mechanisms**: Optional bearer-token support on the client side; default server-side authorization always grants.
- **Data Sensitivity**: High – Bitcoin/EVM private key shares, escrow secrets, and transaction signatures.
- **Dependency Risk**: Non-trivial – multiple security-sensitive crates; no automated dependency scanning present.

## Definitions & Abbreviations

- **2P-ECDSA** – Two-Party Elliptic Curve Digital Signature Algorithm; private key split between client and server.
- **MPC** – Multi-Party Computation; here used for threshold key generation and signing.
- **F-ID** – Finding identifier, e.g., `F-001`.
- **KEV** – Known Exploited Vulnerabilities catalog (CISA).
- **CWE** – Common Weakness Enumeration; taxonomy of software weaknesses.
- **CVSS** – Common Vulnerability Scoring System; used to quantify severity.
- **DoS** – Denial of Service.
- **HSM** – Hardware Security Module.
- **RPC** – Remote Procedure Call; Ethereum JSON-RPC endpoints in this project.

## Coverage & Limitations

- **Pass Coverage**: Pre-Pass, 0 (Secrets Fast-Fail – limited manual scan), 1 (Recon & Attack Surface), 2 (Injection – focused on ECDSA HTTP surfaces), 3 (Auth/AuthZ & Business Logic), 4A (Secrets & Dependencies – selected configs), 5 (Synthesis & Reporting). Optional 4B/4C (Data Security & Infrastructure) were not run as distinct passes; data-security observations were derived from the reviewed files instead.
- **Scope Definition**: Workspace-level audit across `gotham-server`, `gotham-client`, `demo-wallet`, and `integration-tests`. No change-based `{ScopeSpec}` was provided; scope is the current repository state at the analyzed revision.
- **Working Tree Status & Audit Mode**: `git status --porcelain` reported untracked audit-related directories (for example, `audit/`). The audit is therefore classified as **Draft / Pre-commit / WIP (Dirty Working Tree)** and must not be treated as a canonical compliance artifact.
- **Summary Mode**: Several passes (notably Pass 2 and Pass 4A) were executed in summary mode beyond key files to respect context limits. Not all in-scope files were read.
- **File & Target Coverage**: Core coverage included:
  - `gotham-server`: `src/main.rs`, `src/lib.rs`, `src/server.rs`, `src/public_gotham.rs`, `src/tests.rs`, `Rocket.toml`, `Settings.toml`.
  - `gotham-client`: `src/lib.rs`, `src/ecdsa/keygen.rs`, `src/ecdsa/recover.rs`, `Settings.toml`.
  - `demo-wallet`: `src/main.rs`, `src/bitcoin/mod.rs`, `src/bitcoin/escrow.rs`, `src/bitcoin/commands.rs`, `src/ethereum/commands.rs`.
  - `integration-tests`: `tests/ecdsa.rs`.
  - Workspace configuration and docs: root `Cargo.toml`, `README.md`, `wiki/Front_Page.md`, `wiki/Security.md`.
- **EssentialScope Coverage**: Essential entry points (HTTP routes, key storage, key backup, and signing flows) received focused review. However, several in-scope files (for example, additional ECDSA modules and utilities) were not read. As a result, EssentialScope coverage cannot be claimed complete with full confidence.
- **Comprehensive Coverage**: Because `CoverageMode = COMPREHENSIVE` and not all in-scope files/targets were reviewed, this audit is explicitly labeled **"Comprehensive coverage incomplete"**. Some non-essential and potentially essential code paths remain `Not reviewed`.
- **Signals Detected**: `⟨Auth⟩` and `⟨API⟩` signals present (HTTP API with optional bearer tokens); `⟨Infra⟩` signals (RocksDB, external RPCs) present but treated qualitatively.
- **Not in Scope**: External crates (`two-party-ecdsa`, `gotham-engine`, `ethers`, `electrumx_client`) were treated as black boxes; only call sites in this repository were considered.

## Next Steps

1. **Implement real authorization for all ECDSA endpoints (F-001)** and ensure the default build fails closed rather than granting all requests.
2. **Protect key material at rest (F-002, F-003)** by encrypting server RocksDB data and client wallet/escrow files or moving them into secure storage.
3. Add basic rate limiting and monitoring around keygen/signing routes (F-005) and integrate dependency scanning (F-004).
4. Schedule a follow-up audit after remediations and once the working tree is clean, with the goal of improving coverage towards full COMPREHENSIVE mode.
