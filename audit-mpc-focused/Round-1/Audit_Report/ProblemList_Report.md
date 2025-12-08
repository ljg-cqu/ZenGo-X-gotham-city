# ProblemList Report

**Repository**: `ZenGo-X-gotham-city`  
**Date**: 2025-12-08  

---

## Mapped Problems

### 006: Blind Signing Vulnerability
-   **Status**: Confirmed
-   **Finding**: F-001 (Blind Signing in CLI Wallet)
-   **Notes**: The CLI wallet does not decode or display transaction details before signing, leaving users vulnerable to malicious transaction payloads.

### 053: Weak Randomness and Entropy in MPC Key Generation and Signing
-   **Status**: Confirmed
-   **Finding**: F-004 (Implicit/Opaque Randomness Usage)
-   **Notes**: Randomness is handled implicitly by dependencies without explicit control or auditability in the application layer.

### 067: MPC Key Shard Persistence and Encrypted Storage Design Risks
-   **Status**: Confirmed
-   **Finding**: F-002 (Unencrypted Key Share Storage)
-   **Notes**: Key shares are stored in plaintext (JSON) in RocksDB.

### 086: API Rate Limiting, DDoS Protection, MPC Signing Services
-   **Status**: Confirmed
-   **Finding**: F-003 (Missing API Rate Limiting)
-   **Notes**: No rate limiting middleware is implemented in the server.

---

## Unmapped Problems

All other problems in the ProblemList were **NOT EVALUATED** in this focused audit.
