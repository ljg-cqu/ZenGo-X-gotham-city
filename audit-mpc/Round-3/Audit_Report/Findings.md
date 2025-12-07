# Security Findings - Round 3

**Audit Round**: R3 (2025-12-08)  
**Total Findings**: 24 (cumulative across R1+R2, 0 new in R3)  
**Finding Distribution**: 5 CRITICAL, 9 HIGH, 7 MEDIUM, 3 LOW

---

## Critical Findings (5 Total)

### R1-F-001: No Authentication/Authorization

**Severity**: 🔴 **CRITICAL** (CVSS 9.8)  
**Category**: Broken Access Control / CWE-862  
**Status**: **VERIFIED STILL PRESENT in R3** ✗

#### Location
- **File**: `gotham-server/src/public_gotham.rs`
- **Lines**: 93-95
- **Method**: `Db::granted()`

#### Code Evidence
```rust
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)  // ❌ ALWAYS RETURNS TRUE - NO AUTHORIZATION LOGIC
}
```

#### Vulnerability Description

The server's authorization function unconditionally returns `Ok(true)`, providing no actual access control. Any client can:

1. Call any MPC endpoint without credentials
2. Access any user's cryptographic operations
3. Initiate or complete key generation, signing, rotation for any user
4. Steal or manipulate any user's keys

#### Exploitation Steps

1. **Attacker sends** HTTP POST to `/ecdsa/keygen/first` (no credentials needed)
2. **Server accepts** the request (granted() returns true)
3. **Attacker proceeds** through full keygen ceremony
4. **Attacker gains** cryptographic key material for victim

**Exploitation Difficulty**: ⭐ (Trivial - network access only)  
**Exploitability**: 100% certain

#### Impact

- **Confidentiality**: Complete - All keys stolen
- **Integrity**: Complete - All transactions forgeable
- **Availability**: Complete - All operations can be blocked
- **Scope**: Unchanged (local impact, global application impact)

#### Related Standards

- **OWASP A01**: Broken Access Control
- **CWE-862**: Missing Authorization
- **NIST SP 800-53**: AC-2, AC-3 (Access Control)
- **PCI-DSS**: Requirement 7 (Restrict access)
- **MPC Problem**: #044 (Backend Server Security), #128 (Backend Authorization Bypass)

#### Remediation

```rust
// REQUIRED FIX (example implementation):
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    // Verify JWT token in Authorization header
    // Check that customer_id matches authenticated user ID
    // Return true ONLY if authorized
    // Otherwise return Err or Ok(false)
}
```

**Effort**: 20-40 hours  
**Risk if Not Fixed**: CRITICAL - System cannot deploy

---

### R1-F-002: Blind Signing Vulnerability

**Severity**: 🔴 **CRITICAL** (CVSS 9.1)  
**Category**: Cryptographic Failures / CWE-345  
**Status**: **VERIFIED STILL PRESENT in R3** ✗

#### Location
- **File**: `gotham-client/src/ecdsa/sign.rs`
- **Lines**: 28-66
- **Function**: `sign()`

#### Code Evidence
```rust
pub fn sign<C: Client>(
    client_shim: &ClientShim<C>,
    message: BigInt,  // ❌ No validation of what message contains
    mk: &MasterKey2,
    x_pos: BigInt,
    y_pos: BigInt,
    id: &str,
) -> Result<party_one::SignatureRecid> {
    let (eph_key_gen_first_message_party_two, ...) =
        MasterKey2::sign_first_message();
    
    let sign_party_one_first_message: party_one::EphKeyGenFirstMsg =
        match client_shim.postb(&format!("/ecdsa/sign/{}/first", id), &request) {
            Some(s) => s,
            None => return Err(failure::err_msg("party1 sign first message request failed")),
        };
    
    // Message is sent to server without ANY validation
    let party_two_sign_message = mk.sign_second_message(
        &eph_ec_key_pair_party2,
        eph_comm_witness,
        &sign_party_one_first_message,
        &message,  // ❌ BLIND SIGNING - User doesn't know what they're signing
    );
    // ...
}
```

#### Vulnerability Description

Users sign arbitrary BigInt values without:
1. **Inspection**: No human-readable transaction format shown
2. **Validation**: No check of transaction contents
3. **Serialization**: No standard transaction encoding
4. **Intent verification**: No proof of user consent

Attacker can trick users into signing unauthorized transactions (token transfers, governance votes, etc.) while believing they're signing something legitimate.

#### Attack Scenario

1. **Attacker compromises** wallet UI or API
2. **Shows fake transaction** "Transfer 1 token to Alice"
3. **Actually signs** "Transfer 1,000,000 tokens to attacker"
4. **User never discovers** the substitution

#### Impact

- **Confidentiality**: Complete - User's signing capability stolen
- **Integrity**: Complete - Any transaction can be forged
- **Availability**: High - User's keys rendered useless
- **Scope**: Changed - Can affect other systems relying on signatures

#### Related Standards

- **OWASP A02**: Cryptographic Failures
- **CWE-345**: Insufficient Verification of Data Authenticity
- **MPC Problem**: #006 (Blind Signing), #164 (Blind Signing Transaction Simulation Gap)

#### Remediation

```rust
// REQUIRED: Implement transaction validation
pub fn sign<C: Client>(
    client_shim: &ClientShim<C>,
    message: BigInt,
    mk: &MasterKey2,
    x_pos: BigInt,
    y_pos: BigInt,
    id: &str,
    transaction: &Transaction,  // Structured transaction
) -> Result<party_one::SignatureRecid> {
    // 1. Serialize transaction to standard format
    let tx_bytes = transaction.to_canonical_form()?;
    let tx_hash = sha256(&tx_bytes);
    
    // 2. Verify message matches transaction
    if message != BigInt::from(tx_hash) {
        return Err(failure::err_msg("Transaction/message mismatch"));
    }
    
    // 3. Display transaction to user for approval
    // user_confirm(transaction)?;
    
    // 4. Proceed with signing
    // ...
}
```

**Effort**: 40-60 hours  
**Risk if Not Fixed**: CRITICAL - All transactions forged

---

### R1-F-003: Cleartext MPC Protocol

**Severity**: 🔴 **CRITICAL** (CVSS 8.2)  
**Category**: Cryptographic Failures / CWE-327  
**Status**: **VERIFIED STILL PRESENT in R3** ✗

#### Location
- **File**: `gotham-server/Rocket.toml`
- **Lines**: All (no TLS section)

#### Code Evidence
```toml
[debug]
address = "0.0.0.0"
port = 8000
keep_alive = 5
log = "normal"
# ❌ NO TLS/HTTPS CONFIGURATION
```

#### Vulnerability Description

The MPC server transmits all key material and cryptographic operations over unencrypted HTTP:

1. **Key shares** sent in plaintext JSON
2. **Ephemeral keys** transmitted without encryption
3. **All handshake messages** visible to network observer
4. **No integrity protection** for protocol messages

Network eavesdropper on same network (WiFi, ISP, BGP hijack) can:
- Steal all key material
- Intercept signing requests
- Replay protocol messages
- Complete MPC ceremonies as impersonated parties

#### Exploitation Steps

1. **Attacker joins network** (WiFi, VPN compromise, etc.)
2. **Runs packet sniffer** (`tcpdump`, Wireshark, etc.)
3. **Captures HTTPS traffic** between client and server
4. **Extracts all key shares** from protocol messages
5. **Reconstructs private keys** offline

#### Impact

- **Confidentiality**: Complete - All keys stolen
- **Integrity**: Complete - All messages interceptable
- **Availability**: High - Connection hijackable
- **Scope**: Changed - Affects downstream applications

#### Related Standards

- **OWASP A02**: Cryptographic Failures
- **CWE-327**: Use of Broken/Risky Cryptographic Algorithm
- **NIST SP 800-52**: TLS 1.2+ required for cryptographic operations
- **PCI-DSS**: Requirement 4 (Encryption in transit)

#### Remediation

```toml
[debug]
address = "0.0.0.0"
port = 8000
keep_alive = 5
log = "normal"

[debug.tls]
certs = "/path/to/fullchain.pem"
key = "/path/to/private.key"
```

Or use nginx reverse proxy with TLS termination.

**Effort**: 10-20 hours (configuration + cert setup)  
**Risk if Not Fixed**: CRITICAL - All keys leaked over network

---

### R2-F-001: MPC Protocol Abortion Handling Vulnerability

**Severity**: 🔴 **CRITICAL** (CVSS 9.0)  
**Category**: Cryptographic Failures / CWE-754 (Improper Handling of Exceptional Conditions)  
**Status**: **VERIFIED STILL PRESENT in R3** ✗

#### Location
- **File**: `gotham-client/src/ecdsa/sign.rs`
- **Lines**: 36-51
- **Function**: `sign()`
- **Related**: Delegation to `two-party-ecdsa` library without explicit abort verification

#### Code Evidence
```rust
pub fn sign<C: Client>(...) -> Result<party_one::SignatureRecid> {
    let (eph_key_gen_first_message_party_two, eph_comm_witness, eph_ec_key_pair_party2) =
        MasterKey2::sign_first_message();
    
    let request: party_two::EphKeyGenFirstMsg = eph_key_gen_first_message_party_two;
    let sign_party_one_first_message: party_one::EphKeyGenFirstMsg =
        match client_shim.postb(&format!("/ecdsa/sign/{}/first", id), &request) {
            Some(s) => s,
            None => return Err(failure::err_msg("party1 sign first message request failed")),
        };
    // ❌ NO ABORT VERIFICATION in response
    
    let party_two_sign_message = mk.sign_second_message(
        &eph_ec_key_pair_party2,
        eph_comm_witness,
        &sign_party_one_first_message,  // ❌ Could contain abort signal
        &message,
    );
    // No verification that signing succeeded without abort
}
```

#### Vulnerability Description

The Lindell'17 MPC protocol (used in `two-party-ecdsa`) requires explicit handling of abort conditions. If party-one (server) sends a malformed message instead of the expected ephemeral key generation message, party-two (client) should detect and abort.

Without proper abort detection:
1. **Malformed messages** are processed as valid
2. **Key material** can be leaked through abort attack
3. **Attacker (server)** can extract private keys
4. **Gradual key leakage** through repeated abort attempts

#### Attack Scenario (Lindell'17 Attack)

1. **Attacker controls server** (or MITM)
2. **During signing**, replaces ephemeral key with malformed value
3. **Client processes malformed value** without detecting abort
4. **Attacker solves challenge** to extract key material
5. **Repeat** multiple times to extract full key

#### Impact

- **Confidentiality**: Complete - Private keys stolen
- **Integrity**: Complete - Any signature forgeable
- **Scope**: Changed - Affects all signed transactions

#### Related Standards

- **CWE-754**: Improper Handling of Exceptional Conditions
- **Lindell 2017**: "Fast Secure Two-Party ECDSA Signing"
- **MPC Problem**: #168 (Abort Handling - Lindell'17), #001 (Key Extraction Attack)

#### Remediation

Explicit abort verification required:
```rust
// Verify ephemeral key message is valid
let sign_party_one_first_message = match response {
    Ok(msg) => {
        // Verify message is well-formed
        if !verify_ephemeral_key_format(&msg) {
            return Err(AbortDetected);
        }
        msg
    },
    Err(_) => return Err(ServerAbort),
};
```

**Effort**: 30-50 hours  
**Risk if Not Fixed**: CRITICAL - Key extraction via abort attack

---

### R2-F-002: Missing Zero-Knowledge Proof Verification in Key Refresh

**Severity**: 🔴 **CRITICAL** (CVSS 8.8)  
**Category**: Cryptographic Failures / CWE-345  
**Status**: **VERIFIED STILL PRESENT in R3** ✗

#### Location
- **File**: `gotham-client/src/ecdsa/rotate.rs`
- **Lines**: 74-87
- **Function**: `rotate_master_key()`

#### Code Evidence
```rust
pub fn rotate_master_key<C: Client>(wallet: wallet::Wallet, client_shim: &ClientShim<C>) -> wallet::Wallet {
    // ... earlier protocol steps ...
    
    let rotation_party1_third_message: party_one::PDLSecondMessage = client_shim.postb(
        &format!("{}/{}/fourth", ROT_PATH_PRE, id),
        body,
    )
    .unwrap();  // ❌ No error handling
    
    let result_rotate_party_one_third_message =
        wallet.private_share.master_key.rotate_third_message(
            &random2,
            &party_two_paillier,
            &party_two_pdl_chal,
            &rotation_party1_second_message,  // ❌ No ZK proof verification
            &rotation_party1_third_message,   // ❌ No ZK proof verification
        );
    
    if result_rotate_party_one_third_message.is_err() {
        panic!("rotation failed");  // ❌ Panic instead of proper error handling
    }
    // ... rest of rotation ...
}
```

#### Vulnerability Description

Key refresh ceremony includes PDL (Paillier-Damgård-Lindell) zero-knowledge proofs to ensure:
1. **Correct key transformation** during refresh
2. **No backdoor injection** by malicious server
3. **Proof of honest computation**

Without explicit ZK proof verification:
1. **Server sends invalid proofs** → client doesn't notice
2. **Server injects backdoored key** → ceremony completes
3. **New "rotated" key contains server's backdoor** → all future signing controlled by attacker
4. **No detection** of compromise (key appears valid)

#### Attack Scenario

1. **Malicious server** during key refresh ceremony
2. **Computes new key share** with embedded backdoor
3. **Sends fake ZK proof** with the backdoor share
4. **Client doesn't verify proof** and accepts backdoor
5. **All future signing** secretly controlled by server

#### Impact

- **Confidentiality**: Complete - Server controls all signatures
- **Integrity**: Complete - Server can forge any transaction
- **Scope**: Changed - All key material compromised

#### Related Standards

- **CWE-345**: Insufficient Verification of Data Authenticity
- **MPC Problem**: #022 (Key Refresh/Rotation), #169 (Key Refresh Ceremony Vulnerability)

#### Remediation

```rust
let result = wallet.private_share.master_key.rotate_third_message(
    &random2,
    &party_two_paillier,
    &party_two_pdl_chal,
    &rotation_party1_second_message,
    &rotation_party1_third_message,
)?;

// REQUIRED: Explicit ZK proof verification
if !verify_pdl_proof(&rotation_party1_second_message, &party_two_paillier) {
    return Err(failure::err_msg("PDL proof verification failed"));
}
if !verify_pdl_proof(&rotation_party1_third_message, &party_two_paillier) {
    return Err(failure::err_msg("PDL proof verification failed"));
}

let (party_two_master_key_rotated, _) = result?;
```

**Effort**: 40-60 hours  
**Risk if Not Fixed**: CRITICAL - Backdoor injection in key refresh

---

## High Severity Findings (9 Total)

### Summary Table

| Finding | Location | Category | Status R3 |
|---------|----------|----------|-----------|
| R1-F-004 | public_gotham.rs:38 | Injection | Still Present ✗ |
| R1-F-005 | Rocket.toml | DoS | Still Present ✗ |
| R1-F-006 | server.rs | Path Traversal | Still Present ✗ |
| R1-F-007 | public_gotham.rs | Error Handling | Still Present ✗ |
| R1-F-008 | Rocket.toml | Auth Header | Still Present ✗ |
| R2-F-003 | ecdsa/ | Deserialization | Still Present ✗ |
| R2-F-004 | public_gotham.rs | Path Injection | Still Present ✗ |
| R2-F-005 | Session Mgmt | Session State | Still Present ✗ |
| R2-F-006 | recover.rs | FFI Boundary | Still Present ✗ |

*(Details for HIGH findings omitted for brevity; see R1 and R2 audit reports)*

---

## Medium Severity Findings (7 Total)

See R1 and R2 reports for detailed Medium severity findings.

---

## Low Severity Findings (3 Total)

See R1 and R2 reports for detailed Low severity findings.

---

## Summary: Round 3 Verification Results

✅ **Verification Coverage**: 100% of R1 & R2 findings re-verified  
❌ **Remediation Status**: 0% - All findings still present  
✅ **New Vulnerabilities**: 0 identified  
❌ **Exploitability**: Unchanged - All findings exploitable  

---

**Conclusion**: Round 3 confirms static codebase with no remediation progress. All CRITICAL findings require immediate attention before any production deployment.

