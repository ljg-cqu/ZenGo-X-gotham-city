# Architecture Analysis - Round 2

**Audit Round**: R2  
**Date**: 2025-12-07

---

## System Architecture Overview

Gotham City implements a **client-server two-party ECDSA threshold signature system** based on [Lindell'17 protocol](https://eprint.iacr.org/2017/552).

### High-Level Design

```
┌─────────────┐                          ┌─────────────┐
│   Client    │                          │   Server    │
│  (Party 2)  │◄────────MPC Protocol────►│  (Party 1)  │
│             │      over HTTP/REST      │             │
└─────────────┘                          └─────────────┘
      │                                        │
      │  Holds: x₂ (secret share)              │  Holds: x₁, Paillier key
      │  Stores: MasterKey2                    │  Stores: Session state
      └─────────────────────────────────────────┘
               Combined: sk = x₁ + x₂
               Public Key: pk = g^sk
```

### Components

1. **gotham-client** (Rust library)
   - MPC protocol client-side logic (Party 2)
   - Key generation, signing, rotation, recovery
   - Mobile FFI bindings (iOS, Android)
   - HTTP client for server communication

2. **gotham-server** (Rocket web service)
   - MPC protocol server-side logic (Party 1)
   - RESTful API endpoints
   - RocksDB storage for session state
   - **No authentication** (R1-F-001)

3. **two-party-ecdsa** (Git dependency)
   - Core MPC cryptographic primitives
   - Lindell'17 protocol implementation
   - ZK proofs, Paillier homomorphic encryption
   - **Abort handling concerns** (R2-F-001)

4. **gotham-engine** (Git dependency)
   - Protocol orchestration and routing
   - MPC state machine
   - Database abstraction layer

---

## Attack Surface Analysis

### External Attack Surface

| Surface | Exposure | Risk | Findings |
|---------|----------|------|----------|
| **HTTP API** | 0.0.0.0:8000 | 🔴 CRITICAL | R1-F-001, R1-F-008 |
| **Mobile FFI** | Native interfaces | ⚠️ HIGH | R2-F-006 |
| **Database** | Local filesystem | ⚠️ MEDIUM | R1-F-004, R2-F-004 |

### API Endpoints (Attack Surface)

```
POST /ecdsa/keygen/first
POST /ecdsa/keygen/{id}/second
POST /ecdsa/keygen/{id}/third
POST /ecdsa/keygen/{id}/fourth
POST /ecdsa/keygen/{id}/chaincode/first
POST /ecdsa/keygen/{id}/chaincode/second
POST /ecdsa/sign/{id}/first
POST /ecdsa/sign/{id}/second
POST /ecdsa/rotate/{id}/*
```

**Current State**: All endpoints **unauthenticated** (R1-F-001)

### Data Flow Diagram

```
Mobile App (iOS/Android)
    │ FFI
    ↓
┌─────────────────────────────────┐
│     Gotham Client (Rust)        │
│  ┌──────────────────────────┐   │
│  │  MPC Party 2 Logic       │   │
│  │  - KeyGen, Sign, Rotate  │   │
│  │  - Holds: x₂ share       │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
            │ HTTP (Bearer Token sent, not validated)
            ↓
┌─────────────────────────────────┐
│    Gotham Server (Rocket)       │
│  ┌──────────────────────────┐   │
│  │  MPC Party 1 Logic       │   │
│  │  - Holds: x₁, Paillier   │   │
│  │  - No Auth (R1-F-001)    │   │
│  └──────────────────────────┘   │
│            ↓                     │
│  ┌──────────────────────────┐   │
│  │   RocksDB (local)        │   │
│  │  - Session state         │   │
│  │  - Partial signatures    │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

---

## MPC Protocol Flow

### Key Generation (Lindell'17)

```
Client (Party 2)                    Server (Party 1)
─────────────────────────────────────────────────────
1. Generate x₂, commit(Q₂)      →
                                    Generate x₁
                                    Paillier KeyGen
                                ←   Q₁, Paillier PK, ZK proof

2. Verify ZK proof
   Q = Q₁ · Q₂                  →   DLog proof of x₂

3. PDL protocol                 ↔   Paillier DLog verification
                                     ⚠️ Abort handling (R2-F-001)

4. Chain code generation        ↔   2PC for chain code

5. Store MasterKey2                 Store Party1 state
   - x₂, Paillier PK                - x₁, Paillier SK
   - Public key Q                   - Public key Q
```

**Security Properties**:
- ✅ Neither party learns the other's share
- ✅ Public key computed jointly
- ⚠️ **Abort handling not verified** (R2-F-001)

### Signing (Lindell'17)

```
Client (Party 2)                    Server (Party 1)
─────────────────────────────────────────────────────
1. Generate ephemeral k₂        →
                                    Generate ephemeral k₁
                                ←   R₁ commitment

2. Send R₂, ZK proof            →
                                    Compute R = R₁ + R₂
                                    Compute s₁ (partial sig)
                                ←   s₁, partial signature
                                     ⚠️ No message validation (R1-F-002)
                                     ⚠️ Abort handling (R2-F-001)

3. Compute s = s₁ + s₂
   Full signature (r, s)
```

**Vulnerabilities**:
- ❌ Server signs **without message validation** (R1-F-002: Blind signing)
- ⚠️ Malicious abort can leak key bits (R2-F-001)
- ❌ No transaction policy enforcement

### Key Rotation/Refresh

```
Client (Party 2)                    Server (Party 1)
─────────────────────────────────────────────────────
1. Generate Δx₂                 →
                                    Generate Δx₁
                                ←   Commitment to Δx₁
                                     ⚠️ Missing ZK proof (R2-F-002)

2. x₂' = x₂ + Δx₂               →   x₁' = x₁ + Δx₁
   
   ⚠️ NO VERIFICATION that:
      - Public key unchanged
      - Server didn't inject backdoor
      - ZK proof valid
```

**Critical Issue**: R2-F-002 - Missing ZK proof verification allows backdoor injection.

---

## Threat Model

### Adversary Capabilities

| Adversary Type | Capabilities | Mitigations | Status |
|----------------|--------------|-------------|--------|
| **Network Attacker** | Eavesdrop, MITM | TLS/HTTPS | ❌ Not deployed (R1-F-003) |
| **Malicious Client** | Craft malicious MPC messages | Input validation | ⚠️ Partial (R2-F-003) |
| **Malicious Server** | Abort attacks, backdoor injection | Abort detection, ZK verification | ❌ Missing (R2-F-001, R2-F-002) |
| **Unauthenticated User** | Access all endpoints | Authentication | ❌ None (R1-F-001) |

### Attack Scenarios

#### Scenario 1: Key Extraction via Malicious Abort (R2-F-001)

**Attacker**: Malicious server  
**Target**: Client's secret share x₂

```
for i in 0 to key_bits:
    Server initiates signing
    Server receives client commitment
    Server computes partial_info = f(commitment, x₁)
    if partial_info reveals bit i:
        Server ABORTS  ← Client thinks it's network error
        Server records bit i
    else:
        Server continues normally
        
After N signing attempts, server reconstructs x₂
```

**Impact**: Complete key extraction  
**Likelihood**: MEDIUM (requires MPC protocol knowledge)  
**Risk**: 🔴 CRITICAL

#### Scenario 2: Backdoor Injection via Key Refresh (R2-F-002)

**Attacker**: Malicious server  
**Target**: Client's future signatures

```
Client initiates key refresh
Server receives Δx₂ commitment
Server computes: Δx₁ = -KNOWN_VALUE + legitimate_delta
Server sends backdoored Δx₁ (no ZK proof to verify)
Client accepts without verification

Post-refresh: sk' = x₁' + x₂' = (x₁ + Δx₁) + (x₂ + Δx₂)
                             = sk + (Δx₁ + Δx₂)
                             = sk + (-KNOWN_VALUE + ...)
                             
Server now knows effective secret key offset
```

**Impact**: Server can forge signatures  
**Likelihood**: MEDIUM  
**Risk**: 🔴 CRITICAL

#### Scenario 3: Unauthenticated Asset Theft (R1-F-001 + R1-F-002)

**Attacker**: Any network user  
**Target**: All user funds

```
Attacker: curl -X POST http://victim:8000/ecdsa/keygen/first
Server: { "id": "abc123" }  ← No auth check

Attacker co-generates key with server

Attacker: curl -X POST http://victim:8000/ecdsa/sign/abc123/second \
          -d '{"message": "0xMALICIOUS_TX", ...}'
Server: Signs without validation (blind signing)

Attacker obtains valid signature, broadcasts transaction
```

**Impact**: Complete asset theft  
**Likelihood**: TRIVIAL  
**Risk**: 🔴 CRITICAL

---

## Security Architecture Assessment

### Authentication & Authorization

**Current**: ❌ **NONE**

```rust
// Current (gotham-server/src/public_gotham.rs:93)
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)  // Always returns true!
}
```

**Required**:

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client  │ ──JWT──►│ API GW   │────────►│  Server  │
└──────────┘         │ (Auth)   │         └──────────┘
                     │  - Verify│
                     │  - Rate  │
                     │    Limit │
                     │  - Log   │
                     └──────────┘
```

### Cryptographic Architecture

**Key Material**:
- Client: x₂ (additive share), stored in mobile secure enclave (ideal)
- Server: x₁ (additive share), Paillier secret key
- Public Key: Q = g^(x₁+x₂) (derived, not stored)

**Cryptographic Libraries**:
- secp256k1: Elliptic curve operations (✅ well-audited)
- two-party-ecdsa: MPC protocol (⚠️ audit needed)
- Paillier: Homomorphic encryption (from two-party-ecdsa)

**Missing**:
- ❌ Cryptographic agility (R2-F-008)
- ❌ Post-quantum readiness
- ❌ Algorithm version negotiation

### Network Architecture

**Current**:

```
Client ──HTTP (plaintext)──► Server (0.0.0.0:8000)
                             No TLS, No Auth
```

**Required**:

```
Client ──HTTPS/TLS 1.3──► Load Balancer ──► Server (127.0.0.1)
          (mTLS ideal)       │
                             ├─ Rate Limiting
                             ├─ WAF
                             └─ DDoS Protection
```

### Data Storage Architecture

**Current**:

```
RocksDB (./db)
├── {customer_id}_{session_id}_{type}
│   - No encryption at rest
│   - Path traversal risk (R1-F-004, R2-F-004)
│   - No key collision handling (R1-F-007)
```

**Required**:

```
Encrypted Storage
├── /secure/databases/{customer_id}/
│   ├── session_state (encrypted, AES-256-GCM)
│   ├── audit_log (append-only, tamper-evident)
│   └── access_control (per-customer isolation)
```

---

## Trust Boundaries

```
┌────────────────────────────────────────────────────┐
│ Untrusted Zone (Internet)                          │
│  - Any network user (R1-F-001)                     │
│  - MITM attackers (R1-F-003)                       │
└────────────────────────────────────────────────────┘
              │ ⚠️ No TLS, No Auth
              ↓
┌────────────────────────────────────────────────────┐
│ Server Trust Zone (gotham-server)                  │
│  - Holds: x₁, Paillier SK                          │
│  - Should be: High-trust, authenticated            │
│  - Reality: Open to internet (R1-F-008)            │
└────────────────────────────────────────────────────┘
              ↕ MPC Protocol
┌────────────────────────────────────────────────────┐
│ Client Trust Zone (gotham-client)                  │
│  - Holds: x₂                                       │
│  - Should be: Isolated, user-controlled            │
│  - Reality: Trust server responses (R2-F-001/002)  │
└────────────────────────────────────────────────────┘
```

**Key Issue**: Trust boundaries are **inverted** - client trusts unauthenticated server.

---

## Deployment Architecture

### Current Deployment

```
┌────────────────────┐
│  Single Server     │
│  - Rocket (HTTP)   │
│  - RocksDB (local) │
│  - No redundancy   │
│  - No monitoring   │
└────────────────────┘
      0.0.0.0:8000
```

**Risks**:
- Single point of failure
- No high availability
- No disaster recovery
- No security monitoring

### Recommended Deployment

```
                    ┌────────────┐
                    │ CloudFlare │ (DDoS)
                    └────────────┘
                          │
                    ┌────────────┐
                    │   AWS ALB  │ (TLS termination, auth)
                    └────────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
      ┌─────────┐                 ┌─────────┐
      │ Server1 │                 │ Server2 │ (HA)
      │ (HSM)   │                 │ (HSM)   │
      └─────────┘                 └─────────┘
            │                           │
            └─────────────┬─────────────┘
                          │
                    ┌──────────┐
                    │ DynamoDB │ (encrypted at rest)
                    └──────────┘
```

---

## Recommendations

### Immediate (Phase 1)

1. **Deploy TLS/HTTPS** (R1-F-003)
2. **Implement Authentication** (R1-F-001)
3. **Add Transaction Validation** (R1-F-002)
4. **Fix Abort Handling** (R2-F-001)
5. **Add Key Refresh ZK Verification** (R2-F-002)

### Short-Term (Phase 2)

1. Deploy behind load balancer with WAF
2. Implement rate limiting
3. Add comprehensive logging
4. Deploy security monitoring
5. Implement session management (R2-F-005)

### Long-Term (Phase 3)

1. HSM integration for server key shares
2. Multi-region deployment
3. Disaster recovery procedures
4. Cryptographic agility framework (R2-F-008)
5. Post-quantum migration planning

---

**Audit Framework**: v6.0 (Multi-Round Workflow)  
**Generated**: 2025-12-07
