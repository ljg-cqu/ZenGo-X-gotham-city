# Architecture Analysis - Round 4

**Audit Round**: R4  
**Analysis Type**: Two-Party MPC Wallet System  
**Protocol**: Lindell'17 ECDSA Threshold Signatures

---

## System Overview

Gotham City is a **two-party ECDSA signing system** that splits private key operations between:
- **Client** (party-two): Mobile/desktop wallet
- **Server** (party-one): Backend signing service

**Goal**: Enable cryptocurrency custody without single point of key compromise.

---

## High-Level Architecture

```
┌─────────────────┐                    ┌──────────────────┐
│                 │   HTTP/JSON        │                  │
│  Client (P2)    │◄──────────────────►│  Server (P1)     │
│  Mobile/Desktop │   MPC Protocol     │  Backend Service │
│                 │                    │                  │
└─────────────────┘                    └──────────────────┘
        │                                       │
        │                                       │
        ▼                                       ▼
   Key Share 2                             Key Share 1
   (Local Storage)                         (RocksDB)
```

**Security Model**: 
- Neither party has full private key
- Both parties required for signing
- Compromise of one party does NOT compromise key

**Current Reality**:
- ❌ No authentication (R1-F-001)
- ❌ Cleartext protocol (R1-F-003)
- ⚠️ MPC protocol vulnerabilities (R2-F-001, R2-F-002)

---

## Component Breakdown

### 1. Client (gotham-client)

**Purpose**: User-facing wallet application

**Key Files**:
- `lib.rs`: HTTP client, bearer token (unused)
- `ecdsa/keygen.rs`: Key generation protocol
- `ecdsa/sign.rs`: Signing protocol (R1-F-002, R2-F-001)
- `ecdsa/rotate.rs`: Key refresh (R2-F-002)
- `ecdsa/recover.rs`: Key recovery (FFI bindings)

**Security Concerns**:
- Blind signing (R1-F-002)
- No transaction validation
- FFI panics (R2-F-010)
- Abort handling (R2-F-001)

---

### 2. Server (gotham-server)

**Purpose**: Backend MPC co-signer

**Key Files**:
- `server.rs`: Rocket server initialization
- `public_gotham.rs`: Database interface (R1-F-001, R1-F-004)
- `main.rs`: Entry point

**Security Concerns**:
- No authentication (R1-F-001)
- Blind signing (R1-F-002)
- Public binding (R1-F-008)
- Path traversal (R1-F-004)
- No TLS (R1-F-003)

---

### 3. MPC Engine (gotham-engine)

**Purpose**: Route handlers for MPC protocol

**Endpoints**:
- `/ecdsa/keygen/first` - Key generation round 1
- `/ecdsa/keygen/second` - Key generation round 2
- `/ecdsa/keygen/third` - Key generation round 3
- `/ecdsa/keygen/fourth` - Key generation round 4
- `/ecdsa/sign/{id}/first` - Signing round 1
- `/ecdsa/sign/{id}/second` - Signing round 2

**Security Status**: All endpoints lack authentication (R1-F-001)

---

## MPC Protocol Flow

### Key Generation (4 Rounds)

```
Client (P2)                                    Server (P1)
    │                                               │
    │  POST /ecdsa/keygen/first                     │
    ├──────────────────────────────────────────────►│
    │  {commitments, proofs}                        │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {commitments, proofs}                        │
    │                                               │
    │  POST /ecdsa/keygen/second                    │
    ├──────────────────────────────────────────────►│
    │  {decommitments}                              │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {decommitments, paillier_pk}                 │
    │                                               │
    │  POST /ecdsa/keygen/third                     │
    ├──────────────────────────────────────────────►│
    │  {pdl_proof}                                  │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {pdl_proof}                                  │
    │                                               │
    │  POST /ecdsa/keygen/fourth                    │
    ├──────────────────────────────────────────────►│
    │  {pdl_decommit}                               │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {success}                                    │
    │                                               │
    ▼                                               ▼
 Store key                                    Store key
 share 2                                      share 1
```

**Vulnerabilities**: No authentication (R1-F-001), abort handling (R2-F-001)

---

### Signing (2 Rounds)

```
Client (P2)                                    Server (P1)
    │                                               │
    │  Load key share 2                             │  Load key share 1
    │                                               │
    │  POST /ecdsa/sign/{id}/first                  │
    ├──────────────────────────────────────────────►│
    │  {ephemeral_public, commitment}               │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {ephemeral_public, commitment}               │
    │                                               │
    │  POST /ecdsa/sign/{id}/second                 │
    ├──────────────────────────────────────────────►│
    │  {message, partial_sig}                       │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {partial_sig}                                │
    │                                               │
    ▼                                               │
Combine to                                         │
full signature                                     │
```

**Vulnerabilities**: Blind signing (R1-F-002), abort handling (R2-F-001), no auth (R1-F-001)

---

### Key Refresh/Rotation

```
Client (P2)                                    Server (P1)
    │                                               │
    │  POST /ecdsa/rotate/first                     │
    ├──────────────────────────────────────────────►│
    │  {refresh_commitment}                         │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {refresh_commitment, zk_proof}               │
    │                                               │
    │  ⚠️ MISSING: ZK proof verification            │
    │                                               │
    │  POST /ecdsa/rotate/second                    │
    ├──────────────────────────────────────────────►│
    │  {new_share_commitment}                       │
    │                                               │
    │◄──────────────────────────────────────────────┤
    │  {new_share_commitment, pdl_proof}            │
    │                                               │
    │  ⚠️ MISSING: Public key invariant check       │
    │                                               │
    ▼                                               ▼
Update key                                    Update key
share 2                                       share 1
```

**Vulnerabilities**: Missing ZK proof verification (R2-F-002), backdoor injection risk

---

## Attack Surface

### 1. Network Attack Surface

**Exposed Interfaces**:
- HTTP server on `0.0.0.0:8000` (default)
- No authentication
- No TLS
- No rate limiting

**Attack Vectors**:
- Network eavesdropping (R1-F-003)
- MITM attacks (R1-F-003)
- DoS via unlimited requests (R1-F-005)
- Unauthorized key generation (R1-F-001)
- Unauthorized signing (R1-F-001)

**Risk Level**: **CRITICAL** ❌

---

### 2. Client Attack Surface

**Exposed Interfaces**:
- FFI for mobile (iOS/Android)
- HTTP API to server
- Local key storage

**Attack Vectors**:
- Phishing (blind signing, R1-F-002)
- Malicious transaction injection (R1-F-002)
- FFI panics (R2-F-010)
- Local key theft (platform-dependent)

**Risk Level**: **HIGH** ⚠️

---

### 3. Server Attack Surface

**Exposed Interfaces**:
- Public HTTP endpoints
- RocksDB file system
- Redis (unused)

**Attack Vectors**:
- Unauthorized access (R1-F-001)
- Path traversal (R1-F-004)
- Database key collision (R1-F-007)
- Service disruption via panic (R1-F-006)
- Insecure deserialization (R2-F-003)

**Risk Level**: **CRITICAL** ❌

---

### 4. MPC Protocol Attack Surface

**Exposed Interfaces**:
- Key generation ceremony
- Signing ceremony
- Key refresh ceremony

**Attack Vectors**:
- Malicious abort (R2-F-001)
- Backdoor injection during refresh (R2-F-002)
- Key extraction via timing (potential)
- Replay attacks (no session validation, R2-F-005)

**Risk Level**: **CRITICAL** ❌

---

## Data Flow Diagram

### Signing Flow (Current)

```
┌──────────┐
│   User   │
└────┬─────┘
     │
     │ 1. Initiate signing
     ▼
┌─────────────┐         2. HTTP POST           ┌──────────────┐
│   Client    │────────(no auth, cleartext)───►│    Server    │
│  (Party 2)  │                                 │  (Party 1)   │
│             │         3. Partial sig          │              │
│ Key Share 2 │◄───────(no validation)─────────┤ Key Share 1  │
└─────────────┘                                 └──────────────┘
     │
     │ 4. Combine sigs
     ▼
┌──────────────┐
│  Blockchain  │  ⚠️ Arbitrary transaction signed without validation
└──────────────┘
```

**Issues**:
- No auth at step 2 (R1-F-001)
- Cleartext at step 2 (R1-F-003)
- No validation at step 3 (R1-F-002)
- No abort detection (R2-F-001)

---

### Signing Flow (Recommended)

```
┌──────────┐
│   User   │
└────┬─────┘
     │
     │ 1. Initiate signing (review transaction details)
     ▼
┌─────────────┐         2. HTTPS POST + JWT     ┌──────────────┐
│   Client    │────────(authenticated + TLS)───►│    Server    │
│  (Party 2)  │                                 │  (Party 1)   │
│             │         3. Validated partial    │              │
│ Key Share 2 │◄───(policy check, whitelist)───┤ Key Share 1  │
└─────────────┘                                 └──────────────┘
     │
     │ 4. Combine sigs (after user confirmation)
     ▼
┌──────────────┐
│  Blockchain  │  ✅ Validated transaction only
└──────────────┘
```

**Fixes**:
- ✅ JWT auth (Phase 1)
- ✅ TLS encryption (Phase 1)
- ✅ Transaction validation (Phase 1)
- ✅ Abort handling (Phase 1)

---

## Security Architecture Issues

### 1. Zero Trust Boundaries

**Current**: No trust boundaries - open access

```
Internet ──► Server ──► Database
    (no auth)   (no validation)   (path traversal)
```

**Should Be**:
```
Internet ──[TLS]──► [Auth]──► Server ──[Validation]──► Database
```

**Findings**: R1-F-001, R1-F-003, R1-F-004

---

### 2. Defense in Depth

**Current Layers**:
- ❌ Network (no TLS)
- ❌ Authentication (none)
- ❌ Authorization (stub returns true)
- ⚠️ Input validation (partial)
- ✅ Cryptography (strong, Lindell'17)
- ❌ Monitoring (none)

**Status**: **1 of 6 layers functional** - Fails defense in depth principle

---

### 3. Separation of Concerns

**Current**:
- ✅ Client/server separation (architectural)
- ❌ No auth layer separation
- ❌ No policy layer separation
- ⚠️ MPC protocol delegation (to two-party-ecdsa)

**Issues**: Monolithic auth/policy logic (all in `granted()` function)

---

## Deployment Architecture (Assumed)

### Current (Insecure)

```
┌──────────────┐
│   Internet   │
└──────┬───────┘
       │
       │ HTTP :8000 (no TLS)
       ▼
┌──────────────┐
│    Server    │
│  0.0.0.0:8000│  ⚠️ Public binding
└──────────────┘
       │
       ▼
┌──────────────┐
│   RocksDB    │
│  (local FS)  │  ⚠️ Path traversal
└──────────────┘
```

**Issues**: Public exposure, no TLS, no firewall

---

### Recommended (Secure)

```
┌──────────────┐
│   Internet   │
└──────┬───────┘
       │
       │ HTTPS :443
       ▼
┌──────────────┐
│    Nginx     │  (TLS termination)
│  Reverse Proxy│  (Rate limiting)
└──────┬───────┘
       │
       │ HTTP :8000 (localhost only)
       ▼
┌──────────────┐
│    Server    │
│  127.0.0.1:8000│  (Internal binding)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   RocksDB    │
│  (validated   │  (Restricted permissions)
│   paths only) │
└──────────────┘
```

**Improvements**: TLS, rate limiting, localhost binding, validated paths

---

## Threat Model

### Assets

1. **Private Key Shares** (CRITICAL)
   - Client key share (party 2)
   - Server key share (party 1)

2. **MPC Protocol Messages** (HIGH)
   - Ephemeral keys
   - Commitments/decommitments
   - Proofs

3. **Transaction Data** (MEDIUM)
   - Signed transactions
   - Unsigned transaction requests

4. **Service Availability** (MEDIUM)
   - Signing service uptime
   - Key generation service

---

### Threat Actors

1. **Network Attacker** (CRITICAL)
   - Capability: Eavesdrop, MITM
   - Goal: Steal key material
   - Findings: R1-F-003 (cleartext protocol)

2. **Unauthenticated Attacker** (CRITICAL)
   - Capability: Network access
   - Goal: Trigger unauthorized operations
   - Findings: R1-F-001 (no auth)

3. **Malicious Client** (HIGH)
   - Capability: Control client software
   - Goal: Trick server into signing malicious transactions
   - Findings: R1-F-002 (blind signing)

4. **Malicious Server** (HIGH)
   - Capability: Control server
   - Goal: Extract client key share
   - Findings: R2-F-001 (abort handling), R2-F-002 (key refresh)

5. **Insider** (MEDIUM)
   - Capability: Access to server infrastructure
   - Goal: Steal server key share, audit logs
   - Mitigation: Requires additional controls (out of scope)

---

## Cryptographic Architecture

### Primitives

- **Elliptic Curve**: secp256k1 (Bitcoin/Ethereum standard)
- **Hash**: SHA-256
- **Encryption**: Paillier (for MPC)
- **Proofs**: Zero-knowledge (DLog, PDL)
- **Signature**: ECDSA (threshold, 2-of-2)

**Status**: ✅ Cryptographic primitives are sound (assuming two-party-ecdsa correctness)

---

### Key Hierarchy

```
Master Key (never exists in full)
    ├── Party 1 Share (server)
    └── Party 2 Share (client)
          ├── BIP32 Child Keys
          │     ├── m/0 (first derived)
          │     ├── m/1 (second derived)
          │     └── ...
          └── Chain Codes (for derivation)
```

**Issues**: Integer overflow in child derivation (R2-F-007)

---

## Architectural Recommendations

### Immediate (Phase 1)

1. **Add Authentication Layer**
   - JWT validation middleware
   - Token refresh mechanism
   - Session management

2. **Add TLS/HTTPS**
   - TLS 1.3 configuration
   - Valid certificates
   - HSTS enforcement

3. **Add Transaction Validation Layer**
   - Structured transaction format
   - Policy engine
   - Spending limits

4. **Harden MPC Protocol**
   - Abort detection
   - ZK proof verification
   - Replay prevention

---

### Long-Term (Phase 3)

1. **Microservices Architecture**
   - Separate auth service
   - Separate policy service
   - Separate MPC service
   - API gateway

2. **Enhanced Monitoring**
   - Prometheus metrics
   - Grafana dashboards
   - Alert rules
   - Audit logging

3. **High Availability**
   - Multi-region deployment
   - Load balancing
   - Failover mechanisms
   - Disaster recovery

4. **Advanced Security**
   - HSM integration
   - Formal verification
   - Penetration testing
   - Bug bounty program

---

## Summary

**Architecture**: Two-party MPC wallet (client + server)  
**Protocol**: Lindell'17 ECDSA threshold signatures  
**Status**: **INSECURE** - Missing foundational security layers  
**Primary Issues**: No auth, no TLS, blind signing, MPC vulnerabilities

**Recommendation**: Implement Phase 1 security architecture before ANY deployment.

---

**Report Generated**: 2025-12-08 00:44 UTC+8  
**Architecture Status**: INSECURE  
**Action Required**: Phase 1 remediation
