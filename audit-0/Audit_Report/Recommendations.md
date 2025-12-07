# Remediation Roadmap & Compliance

## Priority-Based Action Plan

| Priority | Action | Finding(s) | Category | Effort | Timeline |
|----------|--------|-----------|----------|--------|----------|
| **Immediate** | Implement robust authorization logic in `Db::granted()` and ensure `gotham-server` is not exposed to untrusted networks without strong auth | F-001 | Auth/AuthZ | 1–2 days | This week |
| **Immediate** | Protect server-side key shares in RocksDB with encryption-at-rest and hardened filesystem permissions | F-002 | Data Security | 3–5 days | This sprint |
| **Immediate** | Protect wallet JSON and escrow secrets on client devices with encryption and secure storage mechanisms | F-003 | Data Security | 3–5 days | This sprint |
| **Short-term** | Refine FFI boundaries to avoid panics on malformed input and return structured error codes/strings | F-004 | Robustness | 2–3 days | Next sprint |
| **Short-term** | Replace floating-point amount handling with fixed-point/integer types for BTC/ETH transfers | F-005 | Code Quality | 2–3 days | Next sprint |
| **Short-term** | Introduce `cargo audit` and basic dependency scanning into CI pipelines | F-007 | Dependencies | 1–2 days | Next sprint |
| **Long-term** | Add structured audit logging and monitoring around keygen/sign operations and RocksDB access | F-008 | Observability | 3–5 days | Q1 next year |

## Remediation Details

### Immediate Actions

#### Action: Enforce Authorization Policy in `Db::granted()`

- **Finding**: F-001
- **Effort**: 1–2 days
- **Owner**: Backend / Security
- **Steps**:
  1. Design an authorization model for MPC operations (for example, per-customer policies, whitelists, velocity limits).
  2. Implement policy evaluation in `PublicGotham::granted()` using authenticated context from `gotham-engine`.
  3. Add unit tests to cover allow/deny scenarios and error handling.
  4. Add structured logs for each authorization decision.

#### Action: Encrypt Server Key Shares at Rest

- **Finding**: F-002
- **Effort**: 3–5 days
- **Owner**: Backend / DevOps
- **Steps**:
  1. Decide on an encryption strategy (file-system encryption vs. application-level encryption).
  2. If using application-level encryption, introduce a key management mechanism (HSM, KMS, or OS keystore) and wrap RocksDB writes/reads with encryption/decryption.
  3. Restrict filesystem permissions for the DB directory.
  4. Add operational runbooks for key rotation and backup.

#### Action: Encrypt Wallet and Escrow Files on Client Devices

- **Finding**: F-003
- **Effort**: 3–5 days
- **Owner**: Wallet / Client
- **Steps**:
  1. Introduce an abstraction for secure storage that can wrap `fs::write`/`fs::read` with encryption and platform-specific key management.
  2. Migrate demo wallet code to use this abstraction.
  3. Provide guidance for mobile integrators (iOS/Android) to use OS key stores.

### Short-Term Actions

#### Action: Harden FFI Boundaries

- **Finding**: F-004
- **Effort**: 2–3 days
- **Owner**: Client Library
- **Steps**:
  1. Replace `panic!` calls in FFI helpers with error-returning paths.
  2. Provide explicit "free" functions for FFI-allocated strings.
  3. Add tests that pass malformed pointers/strings (under controlled conditions) to validate error behavior.

#### Action: Use Integer/Fix-Point Amount Types

- **Finding**: F-005
- **Effort**: 2–3 days
- **Owner**: Wallet
- **Steps**:
  1. Replace `f32`/`f64` amount fields with integer/fixed-point types (satoshis, wei, or decimals).
  2. Update CLI help text and examples accordingly.
  3. Add tests for edge-case amounts and rounding behavior.

### Long-Term Strategic Improvements

- Standardize on a hardened deployment pattern for `gotham-server` (behind a reverse proxy, with TLS, rate limiting, and access control).
- Integrate a continuous security pipeline (SAST, DAST where applicable, and recurring dependency audits).
- Expand logging and observability for signing-related events and add playbooks for incident response.

---

## Compliance Mapping

### OWASP Top 10 2021 Coverage

| Rank | Category | Findings | Status |
|------|----------|----------|--------|
| A01 | Broken Access Control | F-001 | Open |
| A02 | Cryptographic Failures | F-002, F-003 | Open |
| A03 | Injection | — | Not Observed (no unvalidated SQL/command injection identified in reviewed code) |
| A04 | Insecure Design | F-002, F-003 | Open |
| A05 | Security Misconfiguration | F-006 | Open |
| A06 | Vulnerable and Outdated Components | F-007 | Open |
| A07 | Identification and Authentication Failures | — | Not Assessed (auth integrated via external providers not implemented here) |
| A08 | Software and Data Integrity Failures | — | Not Observed in reviewed code |
| A09 | Security Logging and Monitoring Failures | F-008 | Open |
| A10 | Server-Side Request Forgery (SSRF) | — | Not Observed in reviewed code |

### CWE Top 25 Coverage

| CWE | Title | Findings | Status |
|-----|-------|----------|--------|
| CWE-285 | Improper Authorization | F-001 | Open |
| CWE-312 | Cleartext Storage of Sensitive Information | F-002, F-003 | Open |
| CWE-248 | Uncaught Exception | F-004 | Open |
| CWE-190 | Integer Overflow or Wraparound (related) | F-005 | Open |
| CWE-1104 | Use of Unmaintained Third Party Components | F-007 | Open |
| CWE-778 | Insufficient Logging | F-008 | Open |

---

## Measurement & Follow-up

- **Target Remediation Date (Phase 1)**: 2026-01-31
- **Follow-up Audit**: Schedule a focused review after F-001–F-003 are fully remediated and deployed.
- **Responsible Parties**:
  - Backend/Security for server-side authorization and key storage.
  - Wallet/Client teams for local key and escrow protection.
  - DevOps for deployment hardening and CI integration.

- **Escalation Path**:
  - Security engineering lead for unresolved CRITICAL/HIGH issues.
  - Product owners for trade-offs involving user experience and rollout planning.
