# Architecture & Attack Surface Analysis - Round 3

**Audit Round**: R3 (2025-12-08)  
**Scope**: Full system architecture mapping and attack surface identification

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Gotham City MPC Wallet                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐                                            │
│  │   Clients    │                                            │
│  ├──────────────┤                                            │
│  │ - iOS FFI    │                                            │
│  │ - Android    │  ← FFI Bindings (R2-F-006 vulnerable)     │
│  │ - Web SDK    │                                            │
│  └────────┬─────┘                                            │
│           │                                                  │
│           ├─ INSECURE ─────┐                                │
│           │   HTTP         │ (R1-F-003 - Cleartext)         │
│           │   (No TLS)      │                                │
│           │                 ▼                                │
│  ┌────────────────────────────────────┐                     │
│  │    Rocket Server (No Auth)          │                    │
│  ├────────────────────────────────────┤                    │
│  │ Endpoints (R1-F-001: No AuthZ)      │                    │
│  │ - /ecdsa/keygen/first               │                    │
│  │ - /ecdsa/keygen/second              │                    │
│  │ - /ecdsa/keygen/third               │                    │
│  │ - /ecdsa/keygen/fourth              │                    │
│  │ - /ecdsa/sign/first                 │                    │
│  │ - /ecdsa/sign/second                │                    │
│  │ - /ecdsa/rotate/*                   │                    │
│  │ - /ecdsa/recover/*                  │                    │
│  └────────┬─────────────────────────────┘                   │
│           │                                                  │
│           ▼                                                  │
│  ┌────────────────────────────────────┐                     │
│  │  PublicGotham (Db Trait)            │                    │
│  ├────────────────────────────────────┤                    │
│  │ - granted() → Ok(true) ALWAYS       │ R1-F-001          │
│  │ - insert()  → RocksDB               │                    │
│  │ - get()     → RocksDB               │                    │
│  │ - has_active_share()                │                    │
│  └────────┬─────────────────────────────┘                   │
│           │                                                  │
│           ▼                                                  │
│  ┌────────────────────────────────────┐                     │
│  │    RocksDB (Local Database)         │                    │
│  │    Location: ./{db_name}/           │                    │
│  │    Keys: {user_id}_{id}_{component} │                    │
│  └────────────────────────────────────┘                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Network Exposure & Attack Surface

### Exposed Endpoints

| Endpoint | Method | Parameters | Auth | Finding |
|----------|--------|-----------|------|---------|
| `/ecdsa/keygen/first` | POST | - | ❌ None | R1-F-001 |
| `/ecdsa/keygen/{id}/second` | POST | `id` | ❌ None | R1-F-001 |
| `/ecdsa/keygen/{id}/third` | POST | `id` | ❌ None | R1-F-001 |
| `/ecdsa/keygen/{id}/fourth` | POST | `id` | ❌ None | R1-F-001 |
| `/ecdsa/sign/{id}/first` | POST | `id`, `message` | ❌ None | R1-F-001, R1-F-002 |
| `/ecdsa/sign/{id}/second` | POST | `id`, `message` | ❌ None | R1-F-001, R1-F-002 |
| `/ecdsa/rotate/{id}/*` | POST | `id` | ❌ None | R1-F-001, R2-F-002 |

**Finding**: All endpoints accessible without authentication (R1-F-001)

### Network Layer Issues

| Issue | Location | Finding | Severity |
|-------|----------|---------|----------|
| **No TLS** | Rocket.toml | R1-F-003 | CRITICAL |
| **No Auth** | public_gotham.rs | R1-F-001 | CRITICAL |
| **No Rate Limiting** | Rocket config | R1-F-005 | HIGH |
| **No CORS** | Rocket routes | R1-F-006 | HIGH |

---

## Cryptographic Data Flow

### MPC Protocol Flow (2-Party ECDSA)

```
Party 1 (Server)             Party 2 (Client)
────────────────             ───────────────

                    Keygen Round 1
              ──────────────────────────►
                 (Commit, Salt)
              ◄──────────────────────────
                    Keygen Round 2-4
              ◄───────────────────────────►
                 (Key Shares Exchanged)
         
         Master Key Established
         {MK1, MK2}
         
                    Sign Round 1-2
              ◄───────────────────────────►
              Message: BigInt (R1-F-002)
              ❌ No transaction validation
         
         Signature Generated
         {sig_r, sig_s, recovery_id}
         
                    Rotation (R2-F-002)
              ◄───────────────────────────►
              PDL Proofs Exchanged
              ❌ No ZK verification
         
         New Master Key Established
```

### Data Flow Vulnerabilities

1. **R1-F-003**: All data in plaintext HTTP
2. **R1-F-002**: Message/transaction mismatch possible
3. **R2-F-001**: No abort detection in signing
4. **R2-F-002**: No ZK proof verification in rotation
5. **R2-F-006**: FFI boundary unsafe (Android/iOS)

---

## Threat Modeling

### Attacker Profiles

#### 1. Network Eavesdropper (Easy)
- **Capability**: Monitor HTTP traffic
- **Exploitation**: Capture all key shares in plaintext
- **Related Findings**: R1-F-003
- **Effort**: Low (packet sniffer)

#### 2. Unauthenticated Client (Trivial)
- **Capability**: Send HTTP requests
- **Exploitation**: Initiate keygen/signing for any user
- **Related Findings**: R1-F-001, R1-F-002
- **Effort**: Very Low (curl command)

#### 3. Server Compromise (Medium)
- **Capability**: Run malicious server code
- **Exploitation**: Inject backdoors during key refresh
- **Related Findings**: R2-F-002, R2-F-001
- **Effort**: Medium (requires code injection)

#### 4. MPC Protocol Attacker (Medium)
- **Capability**: Understands Lindell'17 protocol
- **Exploitation**: Extract keys via abort attacks
- **Related Findings**: R2-F-001
- **Effort**: Medium (protocol knowledge required)

---

## FFI Boundary Analysis (R2-F-006)

### Unsafe FFI Functions

```rust
#[no_mangle]
pub extern "C" fn Java_com_zengo_components_kms_gotham_ECDSA_decryptPartyOneMasterKey(
    env: JNIEnv,
    _class: JClass,
    c_private_key: JString,  // ❌ Potential buffer overflow
    c_auth_token: JString,   // ❌ No token validation
) -> jstring {
    // ...
}
```

**Vulnerabilities**:
1. No input validation from Java
2. No bounds checking on strings
3. Panics instead of proper error handling
4. No protection against exception escaping

---

## Configuration Review

### Rocket.toml (R1-F-003, R1-F-008)

```toml
[debug]
address = "0.0.0.0"      # ❌ Binds to all interfaces
port = 8000
keep_alive = 5           # ❌ Long-lived connections
log = "normal"           # ⚠️ Minimal logging
# ❌ NO TLS CONFIGURATION
# ❌ NO CORS CONFIGURATION
# ❌ NO RATE LIMITING
```

### Settings.toml (R1-F-012)

```toml
db = "local"
region = ""              # ❌ Empty - no default
pool_id = ""             # ❌ Empty - no default
issuer = ""              # ❌ Empty - no default
audience = ""            # ❌ Empty - no default
```

---

## Dependency Attack Surface

### Critical Dependencies

| Dependency | Version | Risk | Finding |
|------------|---------|------|---------|
| Rocket | 0.5.0-rc.1 | HIGH | R1-F-008 (old) |
| Reqwest | 0.9.5 | CRITICAL | R1-F-008 (ancient) |
| RocksDB | 0.21.0 | MEDIUM | R2-F-004 (path injection) |
| two-party-ecdsa | Git branch | HIGH | R2-F-001, R2-F-002 |
| gotham-engine | Git branch | UNKNOWN | Upstream risk |

**Finding**: Very old dependency versions with known CVEs

---

## Attack Chains

### Chain 1: Complete Key Extraction (3 steps)

```
1. Network Eavesdropping (R1-F-003)
   ├─ Attacker joins WiFi
   ├─ Intercepts HTTP traffic
   └─ Steals key shares

2. Authentication Bypass (R1-F-001)
   ├─ No credentials required
   ├─ Initiate own keygen
   └─ Extract keys directly

3. Result: Complete key compromise
```

### Chain 2: Transaction Forgery (2 steps)

```
1. Blind Signing (R1-F-002)
   ├─ Attacker controls client UI
   ├─ Shows fake transaction
   └─ User signs attacker's data

2. Result: Unauthorized transaction
```

### Chain 3: Backdoor Injection (Key Refresh)

```
1. Server Compromise (R2-F-002)
   ├─ Attacker controls server
   ├─ Key refresh ceremony
   └─ Injects backdoored key

2. Missing ZK Verification
   ├─ Client doesn't verify proofs
   ├─ Accepts backdoored key
   └─ Result: Server controls all future signing
```

---

## Risk Scoring Summary

| Component | Risk Level | Evidence |
|-----------|-----------|----------|
| **Authentication** | CRITICAL | None implemented |
| **Encryption** | CRITICAL | HTTP cleartext |
| **Authorization** | CRITICAL | Always returns true |
| **Cryptography** | CRITICAL | Blind signing, no ZK |
| **Network** | HIGH | No TLS, no CORS |
| **Input Validation** | HIGH | Limited sanitization |
| **Error Handling** | MEDIUM | Panics instead of errors |
| **Logging** | MEDIUM | Minimal audit trail |

---

**Conclusion**: System has exploitable attack surface across all layers. All CRITICAL and HIGH findings must be remediated before any production use.

