# Remediation Roadmap & Compliance

## Priority-Based Action Plan

| Priority | Action | Finding(s) | Category | Effort | Timeline |
|----------|--------|-----------|----------|--------|----------|
| **Immediate** | Implement strict authorization for all ECDSA endpoints | F-001 | Auth/AuthZ | 1–2 days | This week |
| **Immediate** | Encrypt server key shares at rest or migrate to HSM-backed storage | F-002 | Data Security | 3–5 days | This sprint |
| **Short-term** | Encrypt wallet and escrow secrets or migrate them to secure OS storage | F-003 | Data Security | 3–5 days | Next sprint |
| **Short-term** | Integrate `cargo audit` (or equivalent) into CI for all workspace members | F-004 | Dependency | 0.5–1 day | Next sprint |
| **Short-term** | Add basic rate limiting and monitoring for keygen/signing routes | F-005 | Availability | 1–2 days | Next sprint |

## Remediation Details

### Immediate Actions

#### Action: Implement strict authorization for `/ecdsa/*` endpoints

- **Findings**: F-001
- **Effort**: 1–2 days (design + implementation + tests)
- **Owner**: Gotham server maintainers

**Steps** (high level):

1. **Define an authorization model**:
   - Identify principals (end-users, custodians, services).
   - Define policies for keygen (who can own keys) and signing (who can sign what, when, and for how much).
2. **Integrate authentication**:
   - Choose an identity mechanism (JWT, mTLS, API keys) consistent with the deployment environment.
   - Ensure tokens/identities are validated on every `/ecdsa/keygen/*` and `/ecdsa/sign/*` request.
3. **Implement `Db::granted` as a deny-by-default gate**:
   - Replace `Ok(true)` with logic that:
     - Extracts principal identity from request context.
     - Evaluates policy (customer ID, transaction parameters, rate limits).
     - Returns `Ok(true)` only when explicitly authorized.
4. **Add tests**:
   - Positive tests: valid identities and policies must pass.
   - Negative tests: unauthenticated and unauthorized calls must fail.
5. **Deployment guardrail**:
   - Ensure production builds cannot accidentally ship with the default `Ok(true)` implementation.

#### Action: Encrypt server key shares at rest

- **Findings**: F-002
- **Effort**: 3–5 days
- **Owner**: Gotham server maintainers

**Steps** (sketch):

1. Introduce an abstraction (for example, `KeyShareStore`) in front of RocksDB.
2. Select an encryption mechanism (KMS-managed key, HSM, or OS keyring) for data-at-rest encryption.
3. Implement transparent encryption/decryption:
   - Before `put`: encrypt serialized value with a data key.
   - After `get`: decrypt bytes before deserialization.
4. Ensure encryption keys are never stored alongside encrypted RocksDB files.
5. Implement migration tooling to re-encrypt existing deployments.

### Short-term Actions

#### Action: Protect wallet and escrow secrets on client

- **Findings**: F-003
- **Effort**: 3–5 days
- **Owner**: Demo wallet maintainers / client application teams

**Steps**:

1. Identify all wallet, backup, and escrow file locations (for example, `wallet.json`, `escrow-bitcoin.json`, backups).
2. Choose storage strategy per platform:
   - Desktop/server: encrypted files or secure filesystem paths.
   - Mobile: OS keychain / keystore.
3. For file-based storage, encrypt data before writing and decrypt on load; protect keys with passwords or device-bound secrets.
4. Harden file permissions and document safe backup procedures.

#### Action: Automate dependency auditing

- **Findings**: F-004
- **Effort**: 0.5–1 day
- **Owner**: Repository maintainers / DevOps

**Steps**:

1. Add a CI job (for example, GitHub Actions) that runs `cargo audit` for all workspace members.
2. Configure alerts for HIGH/CRITICAL vulnerabilities.
3. Define a policy for handling new findings (patch window, risk acceptance, or mitigation).

#### Action: Add rate limiting for keygen/sign endpoints

- **Findings**: F-005
- **Effort**: 1–2 days
- **Owner**: Gotham server maintainers / Infrastructure team

**Steps**:

1. Decide on rate-limiting strategy (per IP, per authenticated principal, or both).
2. Implement rate limits at the ingress layer (reverse proxy, API gateway) or within Rocket handlers.
3. Monitor request rates and error codes; tune thresholds based on observed load.

## Compliance Mapping

### OWASP Top 10 2021 Coverage

| Rank | Category | Findings | Status |
|------|----------|----------|--------|
| A01 | Broken Access Control | F-001 | Open |
| A02 | Cryptographic Failures | F-002, F-003 | Open |
| A03 | Injection | — | Not observed in reviewed slices |
| A04 | Insecure Design | F-001, F-002, F-003 | Open |
| A05 | Security Misconfiguration | F-004, F-005 | Open |
| A06 | Vulnerable Components | — | Not evaluated (no automated CVE scan) |
| A07 | Identification & Authentication Failures | F-001 | Open |
| A08 | Software and Data Integrity Failures | — | Not in scope |
| A09 | Security Logging & Monitoring Failures | — | Not evaluated |
| A10 | Server-Side Request Forgery (SSRF) | — | Not observed in reviewed slices |

### CWE Top 25 Coverage

| CWE | Title | Findings | Status |
|-----|-------|----------|--------|
| CWE-306 | Missing Authentication for Critical Function | F-001 | Open |
| CWE-311 | Missing Encryption of Sensitive Data | F-002 | Open |
| CWE-922 | Insecure Storage of Sensitive Information in a File or Directory | F-003 | Open |

## Measurement & Follow-up

- **Target Remediation Date (Phase 1: Critical & High)**: 2026-01-15 (suggested).
- **Follow-up Audit**: After F-001–F-003 are remediated and deployed to a clean working tree.
- **Responsible Parties**:
  - Security owner: Gotham City security lead.
  - Implementation owners: Gotham server/client and demo-wallet maintainers.
- **Escalation Path**: If remediation timelines slip or new critical findings emerge, escalate to engineering leadership and security steering committee.
