# Audit Report: ZenGo-X Gotham City

**Repository**: `ZenGo-X-gotham-city`  
**Audit Scope**: ALL (ProblemList-Focused)  
**ProblemList Mode**: FOCUSED  
**Date**: 2025-12-08  
**Version**: 1.0  

---

## 1. Executive Summary

This audit was performed as a **ProblemList-focused check** on the `ZenGo-X-gotham-city` repository. The primary objective was to evaluate the codebase against a specific set of known MPC wallet vulnerabilities defined in the provided ProblemList.

**Important Limitation**: This is **not** a comprehensive security audit. Only findings that map to specific ProblemList entries are reported. Other potential vulnerabilities outside the selected ProblemList entries have not been evaluated or reported.

### 1.1 Key Findings

The audit identified **4** critical/high-severity issues mapping to the ProblemList:

1.  **Blind Signing Vulnerability (CRITICAL)**: The CLI client signs transactions without displaying the decoded transaction details (amount, recipient, contract data) to the user for verification.
2.  **Key Shard Persistence Risk (CRITICAL)**: The server stores MPC key shares in a local RocksDB database without application-level encryption, relying solely on the underlying filesystem security.
3.  **Missing API Rate Limiting (HIGH)**: The server lacks rate limiting mechanisms, making it vulnerable to Denial of Service (DoS) attacks, which is critical for computationally expensive MPC operations.
4.  **Weak/Opaque Randomness Usage (HIGH)**: The codebase relies on implicit randomness sources within dependencies without explicit cryptographic strength guarantees or auditability for key generation and signing nonces.

### 1.2 Recommendations

1.  **Implement Transaction Decoding**: Update the client to decode and display all transaction fields (especially ERC-20 `data` payloads) in a human-readable format, requiring explicit user confirmation before signing.
2.  **Encrypt Key Shares**: Implement strong application-level encryption for all key shares stored in RocksDB, preferably using keys derived from a hardware security module (HSM) or a secure enclave (e.g., AWS KMS).
3.  **Add Rate Limiting**: Implement rate limiting middleware on the server to protect against DoS attacks, configured to handle the specific throughput requirements of MPC protocols.
4.  **Explicit RNG Management**: Refactor key generation and signing logic to use explicit, cryptographically secure random number generators (CSPRNG) and ensure they are properly seeded.

---

## 2. Audit Scope & Methodology

### 2.1 In-Scope Components

-   **gotham-client**: Rust library and CLI wallet.
-   **gotham-server**: Rust REST API server.
-   **demo-wallet**: CLI entry point.

### 2.2 Methodology

The audit followed a **FOCUSED** approach based on the provided ProblemList. The following specific problems were selected for evaluation:

-   `006_Blind_Signing_Vulnerability.md`
-   `053_Weak_Randomness_and_Entropy_in_MPC_Key_Generation_and_Signing.md`
-   `067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md`
-   `086_API_Rate_Limiting_DDoS_Protection_MPC_Signing_Services.md`

Code analysis was performed to verify the presence or absence of mitigations for these specific threats.

---

## 3. Findings Summary

| ID | Severity | Title | Problem Map |
|----|----------|-------|-------------|
| F-001 | CRITICAL | Blind Signing in CLI Wallet | 006 |
| F-002 | CRITICAL | Unencrypted Key Share Storage | 067 |
| F-003 | HIGH | Missing API Rate Limiting | 086 |
| F-004 | HIGH | Implicit/Opaque Randomness Usage | 053 |

See `Findings.md` for detailed descriptions.

---

## 4. Coverage & Limitations

### 4.1 Coverage

-   **Files Reviewed**: `gotham-client/src/ecdsa/*`, `gotham-server/src/public_gotham.rs`, `gotham-server/src/server.rs`, `demo-wallet/src/ethereum/*`.
-   **ProblemList Coverage**: 4 problems evaluated out of 436+.

### 4.2 Limitations

-   **Focused Scope**: Only the selected ProblemList entries were evaluated.
-   **Dependency Analysis**: External dependencies (`two-party-ecdsa`, `gotham-engine`) were not audited in depth, only their usage was reviewed.
-   **Dynamic Analysis**: No dynamic testing or fuzzing was performed.
