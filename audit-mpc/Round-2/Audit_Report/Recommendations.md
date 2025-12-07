# Recommendations - Round 2

**Last Updated**: 2025-12-07  
**Audit Round**: R2  
**Previous Rounds**: R1 (2025-12-07)

---

## Executive Recommendations

### Critical Priority: DO NOT DEPLOY

**This system MUST NOT be deployed to production** until ALL Phase 1 findings are remediated and verified through Round 3 audit.

---

## Remediation Roadmap (Updated for R2)

### Phase 1: IMMEDIATE (BLOCKING)

**Timeline**: 5-7 days (increased from R1 estimate of 3-5 days)  
**Estimated Effort**: 60-80 hours  
**Status**: ❌ NOT STARTED  
**Priority**: **CRITICAL - BLOCKS ALL DEPLOYMENT**

#### Must-Fix Items

**From Round 1 (Carry-Forward - UNRESOLVED)**:

1. **R1-F-001: Authentication & Authorization** [2-3 days]
   - Implement JWT validation middleware
   - Add request guards to all endpoints
   - Deploy identity provider integration
   - **Blocking**: System has zero access control

2. **R1-F-002: Transaction Validation** [3-5 days]
   - Implement policy engine for signing authorization
   - Add transaction simulation and display
   - Deploy spending limits and whitelists
   - **Blocking**: Blind signing enables asset theft

3. **R1-F-003: TLS/HTTPS Enforcement** [1 day]
   - Deploy valid TLS certificates
   - Force HTTPS for all connections
   - Disable HTTP fallback
   - **Blocking**: Cleartext protocol enables MITM attacks

4. **R1-F-005: Rate Limiting** [1-2 days]
   - Implement per-IP rate limiting
   - Add per-user operation limits
   - Deploy DDoS protection
   - **Blocking**: Service availability attacks trivial

5. **R1-F-004: Input Sanitization** [1 day]
   - Fix path traversal in database names
   - Validate all user inputs
   - Implement allowlist validation
   - **Blocking**: Path traversal enables file system access

**NEW from Round 2 (MPC Protocol Critical)**:

6. **R2-F-001: MPC Abort Handling** [3-5 days] 🆕
   - Implement malicious abort detection
   - Add abort rate limiting
   - Deploy abort proof verification
   - **Blocking**: Key extraction via abort attacks

7. **R2-F-002: Key Refresh ZK Proofs** [4-6 days] 🆕
   - Add zero-knowledge proof verification in rotation
   - Implement public key invariant checks
   - Deploy rotation audit logging
   - **Blocking**: Backdoor injection during key refresh

8. **R2-F-003: Deserialization Security** [2-3 days] 🆕
   - Add message size limits
   - Implement schema validation
   - Deploy message authentication
   - **Blocking**: DoS via malformed messages

#### Success Criteria

- [ ] All authentication endpoints protected with JWT validation
- [ ] Transaction validation policy engine deployed and tested
- [ ] TLS/HTTPS enforced on all connections
- [ ] Rate limiting active and configured
- [ ] Path traversal vulnerabilities patched
- [ ] MPC abort detection implemented and tested
- [ ] Key refresh ZK proof verification working
- [ ] Deserialization limits and validation deployed
- [ ] Integration tests passing for all remediation
- [ ] Security team sign-off on Phase 1 completion

**Estimated Completion**: 7-10 days with 2 full-time engineers

---

### Phase 2: SHORT-TERM (High Priority)

**Timeline**: 1-2 weeks after Phase 1  
**Estimated Effort**: 50-70 hours  
**Status**: Pending Phase 1 completion

#### High Severity Fixes

**From Round 1**:
- R1-F-006: Replace panics with error returns [2 days]
- R1-F-007: Fix database key collision [1 day]
- R1-F-008: Secure network binding [1 day]

**From Round 2**:
- R2-F-004: Enhanced path validation [1-2 days]
- R2-F-005: Session state management [3-4 days]
- R2-F-006: FFI boundary hardening [3-5 days]

#### Infrastructure & Operations

1. **Security Headers** (R1-F-009)
   - HSTS enforcement
   - CSP policies
   - X-Frame-Options
   - Security.txt deployment

2. **Audit Logging** (R1-F-013)
   - Comprehensive operation logging
   - Tamper-evident log storage
   - SIEM integration
   - Anomaly detection

3. **Monitoring & Alerting**
   - Real-time security monitoring
   - MPC protocol anomaly detection
   - Performance metrics
   - SLA tracking

#### Success Criteria

- [ ] All HIGH severity findings resolved
- [ ] Security headers deployed and verified
- [ ] Audit logging comprehensive and tested
- [ ] Monitoring dashboards operational
- [ ] Incident response procedures documented
- [ ] Penetration testing completed

**Estimated Completion**: 2-3 weeks total (including Phase 1)

---

### Phase 3: MEDIUM-TERM (Hardening)

**Timeline**: 1 month after Phase 1  
**Estimated Effort**: 80-100 hours

#### Medium Severity Fixes

- R1-F-010: Error information disclosure [1 day]
- R1-F-011: Error handling improvements [3 days]
- R1-F-012: Environment variable configuration [1 day]
- R2-F-007: Integer overflow validation [1-2 days]
- R2-F-008: Cryptographic agility framework [5-7 days]
- R2-F-009: Session timeout handling [2-3 days]

#### Architectural Improvements

1. **Cryptographic Agility** (R2-F-008) 🆕
   - Abstract curve/algorithm layer
   - Multi-curve support (secp256r1, Ed25519)
   - Protocol version negotiation
   - Migration tooling

2. **Post-Quantum Readiness**
   - PQC algorithm survey
   - Hybrid signature research
   - Migration timeline planning
   - Key backup for PQ transition

3. **MPC Protocol Hardening**
   - Formal verification of critical paths
   - Property-based testing of MPC flows
   - Fuzzing of protocol message handling
   - Security proof review

4. **Key Lifecycle Management**
   - Automated key refresh scheduling
   - Key rotation procedures
   - Backup and recovery workflows
   - Disaster recovery testing

#### Success Criteria

- [ ] All MEDIUM/LOW findings resolved
- [ ] Cryptographic agility framework deployed
- [ ] PQC migration roadmap documented
- [ ] Formal verification reports for MPC
- [ ] Comprehensive test coverage (>80%)
- [ ] Security documentation complete

---

## Technical Deep-Dives

### 1. Authentication Implementation (R1-F-001)

**Architecture**:
```
Client → [JWT Bearer Token] → Rocket Request Guard → Route Handler
                                      ↓
                                JWT Validation
                                  - Signature check
                                  - Expiry check
                                  - Claims validation
```

**Implementation Steps**:

1. **JWT Infrastructure**:
```rust
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Validation};

#[derive(Serialize, Deserialize)]
struct Claims {
    sub: String,        // customer_id
    exp: usize,         // expiration
    iat: usize,         // issued at
    role: String,       // user | admin | service
    permissions: Vec<String>,
}
```

2. **Request Guard**:
```rust
#[rocket::async_trait]
impl<'r> FromRequest<'r> for AuthenticatedUser {
    async fn from_request(req: &'r Request<'_>) -> request::Outcome<Self, String> {
        // Extract and validate JWT
        // Return authenticated user or 401
    }
}
```

3. **Policy Engine**:
```rust
impl Db for PublicGotham {
    fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
        // Validate transaction against policy
        // - Spending limits
        // - Whitelists
        // - Approval requirements
    }
}
```

### 2. MPC Abort Handling (R2-F-001) 🆕

**Security Requirements**:
1. Distinguish network errors from malicious aborts
2. Implement abort-with-proof protocol
3. Rate-limit abort attempts per session
4. Alert on suspicious abort patterns

**Implementation**:
```rust
enum SigningOutcome {
    Success(Signature),
    NetworkError { retry_after: Duration },
    MaliciousAbort { proof: AbortProof, alert: SecurityAlert },
    TooManyRetries,
}

impl AbortDetector {
    fn analyze_failure(&self, context: &SigningContext) -> SigningOutcome {
        if is_network_timeout(context) {
            return SigningOutcome::NetworkError { 
                retry_after: exponential_backoff(context.attempt)
            };
        }
        
        if is_malicious_abort(context) {
            trigger_security_alert(SecurityAlert::MaliciousAbort {
                session: context.id,
                round: context.round,
                timestamp: Utc::now(),
            });
            return SigningOutcome::MaliciousAbort { ... };
        }
        
        // ... rest of abort analysis
    }
}
```

### 3. Key Refresh ZK Verification (R2-F-002) 🆕

**Verification Protocol**:

```rust
pub async fn rotate_with_verification(
    client_shim: &ClientShim<C>,
    master_key: &MasterKey2,
) -> Result<MasterKey2, RotationError> {
    // Step 1: Client commits to new share
    let (commitment, witness) = generate_refresh_commitment(master_key);
    
    // Step 2: Request server refresh
    let server_msg: ServerRefreshMsg = client_shim
        .postb("/ecdsa/rotate", &commitment).await?;
    
    // Step 3: CRITICAL - Verify server's ZK proof
    verify_refresh_zk_proof(
        &server_msg.new_share_commitment,
        &server_msg.zk_proof,
        &master_key.public_key,
    )?;
    
    // Step 4: Verify public key invariant
    let new_public = compute_combined_public(
        &witness.new_share,
        &server_msg.public_share,
    );
    
    assert_eq!(new_public, master_key.public_key, 
        "Key refresh MUST preserve public key");
    
    // Step 5: Finalize rotation
    Ok(finalize_rotation(master_key, witness, server_msg))
}
```

---

## Testing Strategy

### Unit Tests (Required for Phase 1)

```rust
#[cfg(test)]
mod security_tests {
    #[test]
    fn test_unauthenticated_request_blocked() {
        // Verify R1-F-001 remediation
    }
    
    #[test]
    fn test_blind_signing_prevented() {
        // Verify R1-F-002 remediation
    }
    
    #[test]
    fn test_malicious_abort_detected() {
        // Verify R2-F-001 remediation
    }
    
    #[test]
    fn test_key_refresh_zk_verification() {
        // Verify R2-F-002 remediation
    }
}
```

### Integration Tests

```rust
#[tokio::test]
async fn test_authenticated_signing_flow() {
    // End-to-end with authentication
    let client = authenticated_client(TEST_JWT);
    let signature = client.sign(test_message).await.unwrap();
    assert!(verify_signature(&signature));
}

#[tokio::test]
async fn test_signing_with_abort_attack() {
    let malicious_server = MaliciousServer::new();
    let client = test_client(malicious_server);
    
    let result = client.sign(message).await;
    assert_matches!(result, Err(SigningError::MaliciousAbort { .. }));
}
```

### Security Tests

- Penetration testing of authentication
- Fuzzing of MPC message handlers
- Property-based testing of abort detection
- Load testing with rate limiting
- TLS/certificate validation testing

---

## MPC-Specific Recommendations

### 1. Protocol Security Audit

Engage MPC cryptography experts to review:
- Lindell'17 implementation correctness
- Abort handling security proofs
- ZK proof verification completeness
- Key refresh protocol soundness

**Recommended Auditors**:
- Kudelski Security (MPC specialists)
- NCC Group (crypto team)
- Trail of Bits (protocol analysis)

### 2. Formal Verification

Apply formal methods to critical components:
- MPC state machine correctness
- ZK proof verification
- Key refresh invariants
- Session management safety

**Tools**: F*, Coq, TLA+, Verifpal

### 3. Cryptographic Roadmap

**2025**:
- [ ] Support secp256r1 (NIST P-256)
- [ ] Support Ed25519
- [ ] Implement FROST threshold signatures

**2026-2027**:
- [ ] Hybrid classical/PQ signatures
- [ ] PQC research and testing

**2028-2030**:
- [ ] Full PQC migration (NIST standards)
- [ ] FIPS 140-3 compliance

---

## Organizational Recommendations

### 1. Security Culture

- Establish security champions program
- Regular security training for developers
- Secure coding standards enforcement
- Threat modeling for new features

### 2. Development Process

- Pre-commit security scans (SAST)
- PR security review checklist
- Automated security testing in CI/CD
- Quarterly security audits

### 3. Incident Response

- Document IR procedures
- Establish on-call rotation
- Deploy security monitoring
- Conduct IR drills

### 4. Compliance & Governance

- OWASP ASVS level 2 compliance
- ISO 27001 alignment
- SOC 2 Type II preparation
- Regular third-party audits

---

## Next Audit Schedule

### Round 3 (Post-Phase 1)

**Trigger**: After Phase 1 remediation complete  
**Timeline**: Target 2 weeks post-fixes  
**Scope**: Verification of R1 + R2 CRITICAL/HIGH remediations  
**Expected Outcomes**:
- Verify authentication implementation
- Verify MPC protocol hardening
- Test Phase 1 fixes under adversarial conditions
- Identify any regression or new issues

### Round 4 (Post-Phase 2)

**Trigger**: After Phase 2 complete  
**Timeline**: 4-6 weeks post-Phase 1  
**Scope**: Full system re-assessment + operational security  

---

## Budget & Resource Estimates

### Phase 1: $80K - $120K
- 2 senior engineers × 2 weeks
- Security consultant review
- Penetration testing
- Integration testing infrastructure

### Phase 2: $60K - $90K
- 1.5 engineers × 2 weeks
- Monitoring infrastructure
- Logging/SIEM setup

### Phase 3: $100K - $150K
- 2 engineers × 3-4 weeks
- Formal verification consulting
- Cryptographic audit (external)
- PQC research

### External Audits
- Round 3: $25K - $40K
- Round 4: $35K - $50K
- Annual: $50K - $75K

**Total First-Year Security Investment**: $350K - $525K

---

## Success Metrics

### Security KPIs

- Zero authentication bypasses in production
- 100% of MPC sessions authenticated
- < 0.1% false positive abort detection rate
- Mean time to detect (MTTD) < 5 minutes
- Mean time to respond (MTTR) < 30 minutes

### Quality KPIs

- Test coverage > 80%
- Zero CRITICAL/HIGH findings in Round 3
- Security review turnaround < 48 hours
- Incident response drill success rate > 90%

---

## Conclusion

The combination of **13 unresolved Round 1 findings** plus **11 new Round 2 findings** requires a comprehensive, phased remediation approach. Phase 1 is **MANDATORY and BLOCKING** for any deployment.

**The audit team recommends**:
1. ✅ Immediate commitment to Phase 1 remediation
2. ✅ Dedicated security engineering resources
3. ✅ External MPC cryptography review
4. ✅ Round 3 audit after Phase 1 completion
5. ❌ **DO NOT deploy until all CRITICAL findings resolved**

---

**Audit Framework**: v6.0 (Multi-Round Workflow)  
**Generated**: 2025-12-07
