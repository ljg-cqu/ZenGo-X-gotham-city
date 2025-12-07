# Security Findings

## Table of Contents

- [Critical Severity](#critical-severity)
  - [F-001: No Authentication/Authorization Enforcement](#f-001-authorization---no-authenticationauthorization-enforcement)
  - [F-002: Blind Signing Vulnerability](#f-002-authorization---blind-signing-vulnerability)
  - [F-003: Cleartext Cryptographic Protocol](#f-003-cryptography---cleartext-cryptographic-protocol)
- [High Severity](#high-severity)
  - [F-004: Path Traversal in Database Name](#f-004-injection---path-traversal-in-database-name)
  - [F-005: No Rate Limiting - DoS Vulnerability](#f-005-infrastructure---no-rate-limiting-dos-vulnerability)
  - [F-006: Service Disruption via Panic](#f-006-code-quality---service-disruption-via-panic)
  - [F-007: Database Key Collision Risk](#f-007-business-logic---database-key-collision-risk)
  - [F-008: Public Network Binding Without Authentication](#f-008-infrastructure---public-network-binding-without-authentication)
- [Medium Severity](#medium-severity)
  - [F-009: Missing Security Headers](#f-009-infrastructure---missing-security-headers)
  - [F-010: Error Information Disclosure](#f-010-information-disclosure---error-information-disclosure)
  - [F-011: Unwrap Operations Without Error Handling](#f-011-code-quality---unwrap-operations-without-error-handling)
  - [F-012: Environment Variable Configuration Not Used](#f-012-configuration---environment-variable-configuration-not-used)
- [Low/Info Severity](#lowinfo-severity)
  - [F-013: No Operational Monitoring or Logging](#f-013-infrastructure---no-operational-monitoring-or-logging)

---

## Critical Severity

### F-001: Authorization - No Authentication/Authorization Enforcement

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 9.8 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:H  I:H  A:H |
| **CWE** | [CWE-306: Missing Authentication for Critical Function](https://cwe.mitre.org/data/definitions/306.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:93-95`](../gotham-server/src/public_gotham.rs#L93) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 2-3 days |

**Vulnerability Description**

The Gotham server implements a `granted()` function that is intended to provide transaction authorization logic. However, this function **unconditionally returns `true`**, effectively disabling all authorization checks:

```rust
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)  // Always grants access!
}
```

Additionally, while the client sends a `bearer_auth` token (see `gotham-client/src/lib.rs:90-92`), **the server never validates it**. The `gotham-engine` routes accept requests without any authentication middleware.

**Impact**

* **Anyone on the network** can trigger MPC key generation operations
* **Any attacker** can initiate signing operations for arbitrary messages
* **No audit trail** of who performed which cryptographic operations
* **Complete bypass** of custody controls and multi-party approval workflows
* **Regulatory non-compliance**: Violates KYC/AML requirements for custody providers

**Attack Vector**

```bash
# Attacker anywhere on network:
curl -X POST http://victim-server:8000/ecdsa/keygen/first
# Returns: {"id": "abc123", ...}

curl -X POST http://victim-server:8000/ecdsa/sign/abc123/first \
  -H "Content-Type: application/json" \
  -d '{"d_log_proof": {...}}'
# Server processes request without checking identity
```

**Proof of Concept**

See `integration-tests/tests/ecdsa.rs:116` where `ClientShim` is created with `None` for auth_token, yet the test passes:

```rust
let client_shim = ClientShim::new_with_client(
    "http://localhost:8008".to_string(), 
    None,  // No auth token!
    client
);
let ps: ecdsa::PrivateShare = ecdsa::get_master_key(&client_shim);
// Succeeds without any authentication
```

**Root Cause Analysis**

1. `granted()` function has a placeholder implementation
2. No JWT validation middleware in Rocket server configuration
3. Client bearer token is sent but server-side validation is not implemented
4. Comment in code (line 92) suggests this was a known TODO: "the granted function implements the logic of tx authorization. If no tx authorization is needed the function returns always true"

**Remediation**

**Immediate Fix (Required for Production)**:

1. **Implement JWT Validation**:

```rust
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    exp: usize,
    customer_id: String,
}

impl Db for PublicGotham {
    fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
        // TODO: Extract JWT from request context
        let token = get_token_from_request()?;
        
        // Validate JWT
        let validation = Validation::new(Algorithm::HS256);
        let token_data = decode::<Claims>(
            &token,
            &DecodingKey::from_secret(get_secret_key().as_bytes()),
            &validation,
        ).map_err(|e| DatabaseError::Unauthorized(e.to_string()))?;
        
        // Verify customer_id matches
        if token_data.claims.customer_id != customer_id {
            return Err(DatabaseError::Unauthorized(
                "Customer ID mismatch".to_string()
            ));
        }
        
        // TODO: Add policy-based authorization
        // - Check signing limits
        // - Verify message against whitelist
        // - Enforce multi-party approval workflows
        
        Ok(true)
    }
}
```

2. **Add Rocket Request Guard**:

```rust
use rocket::request::{self, Request, FromRequest};
use rocket::http::Status;

pub struct AuthenticatedUser {
    pub customer_id: String,
    pub jwt_claims: Claims,
}

#[rocket::async_trait]
impl<'r> FromRequest<'r> for AuthenticatedUser {
    type Error = String;

    async fn from_request(req: &'r Request<'_>) -> request::Outcome<Self, Self::Error> {
        let token = req.headers().get_one("Authorization")
            .and_then(|h| h.strip_prefix("Bearer "));
        
        match token {
            Some(t) => {
                // Validate JWT
                match validate_jwt(t) {
                    Ok(claims) => request::Outcome::Success(AuthenticatedUser {
                        customer_id: claims.customer_id.clone(),
                        jwt_claims: claims,
                    }),
                    Err(e) => request::Outcome::Failure((Status::Unauthorized, e)),
                }
            }
            None => request::Outcome::Failure((
                Status::Unauthorized,
                "Missing Authorization header".to_string()
            )),
        }
    }
}
```

3. **Update Route Signatures** in `gotham-engine`:

```rust
#[post("/ecdsa/keygen/first")]
async fn wrap_keygen_first(
    user: AuthenticatedUser,  // Add this guard
    db: &State<Mutex<Box<dyn Db>>>
) -> Result<Json<...>, Status> {
    // Now user.customer_id is validated
    // ...
}
```

**Long-Term Recommendations**:

* Implement policy-based authorization with configurable rules
* Add multi-party approval workflows for high-value operations
* Integrate with identity provider (OAuth2/OIDC)
* Implement rate limiting per authenticated user
* Add comprehensive audit logging of all operations

**References**

* OWASP: [Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
* CWE-306: Missing Authentication for Critical Function
* MPC Problem #128: Backend Authorization Bypass
* MPC Problem #125: Off-Chain Account Compromise

---

### F-002: Authorization - Blind Signing Vulnerability

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 9.1 |
| **CVSS Vector** | AV:N  AC:L  PR:L  UI:N  S:U  C:H  I:H  A:N |
| **CWE** | [CWE-345: Insufficient Verification of Data Authenticity](https://cwe.mitre.org/data/definitions/345.html) |
| **Location** | `gotham-engine` routes (all signing endpoints) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 3-5 days |

**Vulnerability Description**

The Gotham server **signs arbitrary messages without validating their content or purpose**. The signing flow accepts any message hash provided by the client and performs MPC signing without:

1. Validating the message format or structure
2. Checking transaction parameters (recipient, amount, asset type)
3. Enforcing spending limits or velocity controls
4. Requiring human-readable transaction display
5. Implementing multi-party approval for high-value transactions

This is the classic **"blind signing"** vulnerability that has caused major losses in cryptocurrency custody.

**Impact**

* **Asset Theft**: Attacker tricks server into signing malicious transactions
* **Unauthorized Transfers**: No validation of recipient addresses or amounts
* **Policy Bypass**: Spending limits, whitelists, and approval workflows are absent
* **Phishing Attacks**: Users can be tricked into signing malicious transactions
* **Compliance Violations**: No KYC/AML controls on transaction destinations

**Attack Vector**

```rust
// Attacker's client code:
let malicious_tx_hash = BigInt::from_hex("0xMALICIOUS_TRANSACTION_HASH");

let signature = ecdsa::sign(
    &client_shim,
    malicious_tx_hash,  // Server signs without validation!
    &master_key,
    x_pos,
    y_pos,
    &id,
).expect("Server signed malicious transaction");

// Use signature to drain victim's wallet
```

**Code Evidence**

In `gotham-client/src/ecdsa/sign.rs:28-66`, the `sign()` function accepts any `message: BigInt` and sends it to the server:

```rust
pub fn sign<C: Client>(
    client_shim: &ClientShim<C>,
    message: BigInt,  // No validation!
    mk: &MasterKey2,
    x_pos: BigInt,
    y_pos: BigInt,
    id: &str,
) -> Result<party_one::SignatureRecid> {
    // ... directly sends message to server for signing ...
}
```

The server-side `granted()` function (F-001) could enforce transaction policies, but it returns `true` unconditionally.

**Real-World Impact**

Similar vulnerabilities have caused:
* **Ronin Bridge Hack** ($625M, 2022): Blind signing of malicious transactions
* **Nomad Bridge Exploit** ($190M, 2022): Insufficient message validation
* **Wormhole Bridge Hack** ($325M, 2022): Message authentication bypass

**Remediation**

**Immediate Fix**:

1. **Implement Transaction Validation**:

```rust
#[derive(Serialize, Deserialize)]
struct ValidatedTransaction {
    chain_id: u64,
    to: String,
    value: BigInt,
    data: Vec<u8>,
    nonce: u64,
    gas_limit: u64,
    // ... other fields
}

impl PublicGotham {
    fn validate_transaction(
        &self,
        tx: &ValidatedTransaction,
        customer_id: &str,
    ) -> Result<(), DatabaseError> {
        // 1. Verify chain_id is supported
        if !SUPPORTED_CHAINS.contains(&tx.chain_id) {
            return Err(DatabaseError::InvalidTransaction(
                "Unsupported chain".into()
            ));
        }
        
        // 2. Check recipient against whitelist
        if !self.is_address_whitelisted(&tx.to, customer_id)? {
            return Err(DatabaseError::PolicyViolation(
                "Recipient not whitelisted".into()
            ));
        }
        
        // 3. Enforce spending limits
        let daily_spent = self.get_daily_spending(customer_id)?;
        if daily_spent + &tx.value > DAILY_LIMIT {
            return Err(DatabaseError::PolicyViolation(
                "Daily spending limit exceeded".into()
            ));
        }
        
        // 4. Check for high-value transactions requiring approval
        if tx.value > HIGH_VALUE_THRESHOLD {
            if !self.has_approval(customer_id, &tx)? {
                return Err(DatabaseError::ApprovalRequired(
                    "High-value tx requires approval".into()
                ));
            }
        }
        
        // 5. Validate transaction format
        validate_tx_format(&tx)?;
        
        Ok(())
    }
}
```

2. **Update Signing Flow**:

```rust
async fn wrap_sign_second(
    user: AuthenticatedUser,
    tx: Json<ValidatedTransaction>,  // Change from raw message
    db: &State<Mutex<Box<dyn Db>>>
) -> Result<Json<SignatureRecid>, Status> {
    let db = db.lock().await;
    
    // Validate transaction against policies
    db.validate_transaction(&tx, &user.customer_id)
        .map_err(|_| Status::Forbidden)?;
    
    // Compute message hash from validated transaction
    let message_hash = compute_tx_hash(&tx);
    
    // Proceed with signing
    // ...
}
```

3. **Add Human-Readable Transaction Display**:

```rust
struct TransactionSummary {
    from: String,
    to: String,
    amount: String,
    asset: String,
    chain: String,
    estimated_fee: String,
}

fn display_transaction(tx: &ValidatedTransaction) -> TransactionSummary {
    // Convert raw transaction to human-readable format
    // This should be displayed to user before signing
}
```

**Long-Term Recommendations**:

* Implement EIP-712 typed data signing for structured messages
* Add transaction simulation to preview effects before signing
* Integrate with AML/KYC providers for recipient screening
* Implement multi-party approval workflows (2-of-3, 3-of-5, etc.)
* Add anomaly detection for unusual transaction patterns
* Support hardware security module (HSM) integration for policy enforcement

**References**

* MPC Problem #006: Blind Signing Vulnerability
* MPC Problem #032: Transaction Simulation & Human-Readable Signing
* OWASP: [Insufficient Verification of Data Authenticity](https://owasp.org/www-community/vulnerabilities/Insufficient_Verification_of_Data_Authenticity)
* EIP-712: Ethereum typed structured data hashing and signing

---

### F-003: Cryptography - Cleartext Cryptographic Protocol

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 8.2 |
| **CVSS Vector** | AV:A  AC:L  PR:N  UI:N  S:U  C:H  I:H  A:N |
| **CWE** | [CWE-319: Cleartext Transmission of Sensitive Information](https://cwe.mitre.org/data/definitions/319.html) |
| **Location** | [`gotham-server/Rocket.toml:1-6`](../gotham-server/Rocket.toml#L1) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 1-2 days |

**Vulnerability Description**

The Gotham server **binds to HTTP without TLS/HTTPS configuration**, transmitting all MPC protocol messages in cleartext over the network. This includes:

* Key generation protocol messages (EC points, commitments, zero-knowledge proofs)
* Signing protocol ephemeral keys and partial signatures
* Database identifiers and session information
* Authentication tokens (if F-001 is fixed)

**Configuration Evidence**:

```toml
[debug]
address = "0.0.0.0"
port = 8000
keep_alive = 5
log = "normal"
# No TLS/HTTPS configuration!
```

**Impact**

* **Man-in-the-Middle (MITM) Attacks**: Attacker can intercept and modify MPC protocol messages
* **Key Material Leakage**: Partial key shares and ephemeral keys exposed to network sniffing
* **Replay Attacks**: Captured protocol messages can be replayed
* **Session Hijacking**: If auth tokens are added (F-001), they would be transmitted in cleartext
* **Compliance Violations**: Most custody regulations require encryption in transit

**Attack Scenarios**

1. **Passive Eavesdropping**:
```bash
# Attacker on same network segment:
tcpdump -i eth0 -A 'tcp port 8000'
# Captures all MPC protocol messages
```

2. **Active MITM**:
```bash
# Attacker modifies MPC messages in transit
arpspoof -i eth0 -t victim-client -r victim-server
# Then use mitmproxy or Burp Suite to intercept/modify
```

3. **Replay Attack**:
```bash
# Capture keygen session
# Replay messages to force reuse of ephemeral keys
# May enable key extraction in certain MPC protocols
```

**Proof of Concept**

Network capture of `integration_test_ecdsa_key_signing` (if run over real network) would reveal:

```json
POST /ecdsa/keygen/first
{
  "d_log_proof": {
    "pk": "0x...",  // Public key material visible
    "pk_t_rand_commitment": "0x...",
    // ... other protocol messages
  }
}
```

**Remediation**

**Immediate Fix**:

1. **Configure TLS in Rocket**:

```toml
[global]
address = "0.0.0.0"
port = 8000
keep_alive = 5
log = "normal"

# TLS Configuration
[global.tls]
certs = "/path/to/cert-chain.pem"
key = "/path/to/privkey.pem"

# Or for development:
# [debug.tls]
# certs = "/path/to/dev-cert.pem"
# key = "/path/to/dev-key.pem"
```

2. **Force HTTPS Redirect**:

```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Response, Data};

pub struct HttpsRedirect;

#[rocket::async_trait]
impl Fairing for HttpsRedirect {
    fn info(&self) -> Info {
        Info {
            name: "HTTPS Redirect",
            kind: Kind::Request
        }
    }

    async fn on_request(&self, req: &mut Request<'_>, _data: &mut Data<'_>) {
        if req.uri().scheme() != Some("https") {
            // Reject non-HTTPS requests
            req.local_cache(|| false);
        }
    }
}

// In server.rs:
pub fn get_server() -> Rocket<Build> {
    rocket::Rocket::build()
        .attach(HttpsRedirect)
        // ... rest of config
}
```

3. **Obtain Valid Certificates**:

```bash
# Production: Use Let's Encrypt
certbot certonly --standalone -d gotham.example.com

# Development: Generate self-signed cert
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost"
```

**Client-Side Changes**:

```rust
// In ClientShim::new():
pub fn new(endpoint: String, auth_token: Option<String>) -> ClientShim<reqwest::Client> {
    let client = reqwest::Client::builder()
        .https_only(true)  // Force HTTPS
        .min_tls_version(reqwest::tls::Version::TLS_1_2)
        .build()
        .expect("Failed to create HTTPS client");
    
    ClientShim {
        client,
        auth_token,
        endpoint,
    }
}
```

**Long-Term Recommendations**:

* Implement mutual TLS (mTLS) for client authentication
* Use certificate pinning in mobile clients
* Add HSTS (Strict-Transport-Security) header
* Consider additional application-layer encryption for sensitive MPC messages
* Regular certificate rotation and monitoring

**References**

* MPC Problem #012: Mobile Client Security Risks (cleartext transmission)
* MPC Problem #125: Off-Chain Account Compromise
* OWASP: [Insufficient Transport Layer Protection](https://owasp.org/www-project-top-ten/2017/A3_2017-Sensitive_Data_Exposure)
* CWE-319: Cleartext Transmission of Sensitive Information

---

## High Severity

### F-004: Injection - Path Traversal in Database Name

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.5 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:N  I:H  A:N |
| **CWE** | [CWE-22: Improper Limitation of a Pathname to a Restricted Directory](https://cwe.mitre.org/data/definitions/22.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:36-41`](../gotham-server/src/public_gotham.rs#L36) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 4 hours |

**Vulnerability Description**

The `db_name` configuration parameter is validated only for alphanumeric characters, but the validation is **insufficient** and the database path construction is **vulnerable to path traversal**:

```rust
let db_name = settings.get("db_name").unwrap_or(&"db".to_string()).clone();
if !db_name.chars().all(|e| char::is_ascii_alphanumeric(&e)) {
    panic!("DB name is illegal, may only contain alphanumeric characters");
}
let rocksdb_client = rocksdb::DB::open_default(format!("./{}", db_name)).unwrap();
```

**Issues**:

1. **Environment variable override** can bypass validation (line 28: `.merge(config::Environment::new())`)
2. **Panic on invalid input** causes service disruption (should return error)
3. **Relative path construction** (`./{db_name}`) may not be safe in all deployment scenarios

**Attack Vector**

```bash
# Attacker sets environment variable:
export DB_NAME="../../../etc/passwd"

# Or modifies Settings.toml:
db_name = "validname123"  # Passes validation

# But later code or configuration system may allow:
DB_NAME="../../sensitive-data" ./server_exec
```

While the current alphanumeric check prevents direct `../` injection, environment variable merging happens **after** file parsing, potentially bypassing validation.

**Impact**

* Database files created in unexpected locations
* Potential overwrite of system files (if running with elevated privileges)
* Information disclosure if attacker can read database files from arbitrary paths

**Remediation**

```rust
use std::path::{Path, PathBuf};

impl PublicGotham {
    pub fn new() -> Self {
        let settings = get_settings_as_map();
        
        // 1. Get db_name with validation
        let db_name = settings.get("db_name")
            .unwrap_or(&"db".to_string())
            .clone();
        
        // 2. Strict validation
        if !db_name.chars().all(|c| c.is_ascii_alphanumeric() || c == '_' || c == '-') {
            // Return error instead of panic!
            return Err(DatabaseError::InvalidConfig(
                "DB name must be alphanumeric with _ or -".to_string()
            ));
        }
        
        if db_name.len() > 64 {
            return Err(DatabaseError::InvalidConfig(
                "DB name too long".to_string()
            ));
        }
        
        // 3. Construct safe path
        let db_base_dir = PathBuf::from(
            std::env::var("DB_BASE_DIR").unwrap_or_else(|_| "./data".to_string())
        );
        
        // 4. Canonicalize to prevent traversal
        let db_path = db_base_dir.join(&db_name);
        let canonical_path = db_path.canonicalize()
            .map_err(|e| DatabaseError::InvalidConfig(format!("Invalid path: {}", e)))?;
        
        // 5. Verify path is within allowed base directory
        if !canonical_path.starts_with(&db_base_dir.canonicalize()?) {
            return Err(DatabaseError::SecurityViolation(
                "Path traversal attempt detected".to_string()
            ));
        }
        
        // 6. Open database
        let rocksdb_client = rocksdb::DB::open_default(&canonical_path)
            .map_err(|e| DatabaseError::DatabaseFailure(e.to_string()))?;
        
        Ok(PublicGotham { rocksdb_client })
    }
}
```

**References**

* CWE-22: Improper Limitation of a Pathname to a Restricted Directory
* OWASP: [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)

---

### F-005: Infrastructure - No Rate Limiting - DoS Vulnerability

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.5 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:N  I:N  A:H |
| **CWE** | [CWE-770: Allocation of Resources Without Limits](https://cwe.mitre.org/data/definitions/770.html) |
| **Location** | `gotham-server/src/server.rs` (entire server config) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 1 day |

**Vulnerability Description**

The Gotham server has **no rate limiting** on any endpoints. An attacker can:

* Flood keygen endpoints, consuming CPU and memory
* Trigger excessive database writes
* Exhaust server resources with rapid signing requests
* Cause service unavailability for legitimate users

MPC operations are **computationally expensive** (762ms for keygen, 151ms for sign per benchmark), making DoS attacks highly effective.

**Attack Vector**

```bash
# Simple DoS attack:
while true; do
  curl -X POST http://target:8000/ecdsa/keygen/first &
done

# 100 concurrent requests in < 1 minute can overwhelm server
```

**Impact**

* **Service Unavailability**: Legitimate users cannot generate keys or sign transactions
* **Resource Exhaustion**: Server CPU, memory, and disk space consumed
* **Database Corruption**: Excessive concurrent writes may corrupt RocksDB
* **Cascading Failures**: Dependent services affected by timeouts

**Remediation**

**Using Rocket Fairings**:

```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Response, Data};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

pub struct RateLimiter {
    requests: Arc<Mutex<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    pub fn new(max_requests: usize, window: Duration) -> Self {
        Self {
            requests: Arc::new(Mutex::new(HashMap::new())),
            max_requests,
            window,
        }
    }
    
    fn is_allowed(&self, key: &str) -> bool {
        let mut requests = self.requests.lock().unwrap();
        let now = Instant::now();
        
        // Clean old requests
        let cutoff = now - self.window;
        let entry = requests.entry(key.to_string()).or_insert_with(Vec::new);
        entry.retain(|&t| t > cutoff);
        
        // Check limit
        if entry.len() >= self.max_requests {
            return false;
        }
        
        // Record request
        entry.push(now);
        true
    }
}

#[rocket::async_trait]
impl Fairing for RateLimiter {
    fn info(&self) -> Info {
        Info {
            name: "Rate Limiter",
            kind: Kind::Request,
        }
    }
    
    async fn on_request(&self, req: &mut Request<'_>, _: &mut Data<'_>) {
        let key = req.client_ip()
            .map(|ip| ip.to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        if !self.is_allowed(&key) {
            req.local_cache(|| "rate_limited");
        }
    }
    
    async fn on_response<'r>(&self, req: &'r Request<'_>, res: &mut Response<'r>) {
        if req.local_cache(|| "").contains("rate_limited") {
            res.set_status(Status::TooManyRequests);
            res.set_sized_body(None, std::io::Cursor::new(
                "Rate limit exceeded. Please try again later."
            ));
        }
    }
}

// In server.rs:
pub fn get_server() -> Rocket<Build> {
    rocket::Rocket::build()
        .attach(RateLimiter::new(
            10,  // 10 requests
            Duration::from_secs(60),  // per minute
        ))
        // ... rest of config
}
```

**Alternative: Use `rocket_rate_limiter` Crate**:

```toml
[dependencies]
rocket_rate_limiter = "0.1"
```

```rust
use rocket_rate_limiter::{RateLimiter, RateLimiterFairing};

#[launch]
fn rocket() -> _ {
    rocket::build()
        .attach(RateLimiterFairing::default())
        .mount("/", routes![...])
}
```

**References**

* CWE-770: Allocation of Resources Without Limits
* MPC Problem #005: Throughput Limitations
* MPC Problem #086: API Rate Limiting & DDoS Protection

---

### F-006: Code Quality - Service Disruption via Panic

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.1 |
| **CVSS Vector** | AV:N  AC:L  PR:L  UI:N  S:U  C:N  I:L  A:H |
| **CWE** | [CWE-248: Uncaught Exception](https://cwe.mitre.org/data/definitions/248.html) |
| **Location** | Multiple locations (see below) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 2 days |

**Vulnerability Description**

The codebase uses `panic!()` and `.unwrap()` extensively, which **terminates the server process** on error. This causes:

* Complete service outage on malformed input
* No graceful error recovery
* Loss of in-flight MPC sessions
* Difficulty debugging production issues

**Panic Locations**:

1. **`gotham-server/src/public_gotham.rs:39`**: Invalid database name panics instead of returning error
2. **`gotham-server/src/public_gotham.rs:41`**: Database open failure panics
3. **`gotham-server/src/public_gotham.rs:65-66`**: Serialization errors unwrapped
4. **`gotham-server/src/public_gotham.rs:77-86`**: Database operations unwrapped

**Impact**

* Attacker can crash server by sending invalid configuration
* No error recovery or graceful degradation
* Complete loss of service availability

**Attack Vector**

```bash
# Trigger panic via environment variable:
DB_NAME="invalid-name!" ./server_exec
# Server panics and exits: "DB name is illegal..."

# Or trigger via malformed request that causes serialization panic
```

**Remediation**

Replace all `panic!()` and `.unwrap()` with proper error handling:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("Invalid configuration: {0}")]
    InvalidConfig(String),
    
    #[error("Database error: {0}")]
    DatabaseFailure(String),
    
    #[error("Serialization error: {0}")]
    SerializationError(String),
    
    #[error("Unauthorized: {0}")]
    Unauthorized(String),
}

impl PublicGotham {
    pub fn new() -> Result<Self, DatabaseError> {
        let settings = get_settings_as_map();
        let db_name = settings.get("db_name").unwrap_or(&"db".to_string()).clone();
        
        // Validate without panic
        if !db_name.chars().all(|e| char::is_ascii_alphanumeric(&e)) {
            return Err(DatabaseError::InvalidConfig(
                "DB name must be alphanumeric".to_string()
            ));
        }
        
        // Open with error handling
        let rocksdb_client = rocksdb::DB::open_default(format!("./{}", db_name))
            .map_err(|e| DatabaseError::DatabaseFailure(e.to_string()))?;
        
        Ok(PublicGotham { rocksdb_client })
    }
}

#[async_trait]
impl Db for PublicGotham {
    async fn insert(
        &self,
        key: &DbIndex,
        table_name: &dyn MPCStruct,
        value: &dyn Value,
    ) -> Result<(), DatabaseError> {
        let identifier = idify(key.clone().customerId, key.clone().id, table_name);
        
        // Handle serialization errors
        let v_string = serde_json::to_string(&value)
            .map_err(|e| DatabaseError::SerializationError(e.to_string()))?;
        
        // Handle database errors
        self.rocksdb_client.put(identifier, v_string)
            .map_err(|e| DatabaseError::DatabaseFailure(e.to_string()))?;
        
        Ok(())
    }
}
```

**References**

* Rust Error Handling Best Practices
* CWE-248: Uncaught Exception
* MPC Problem #016: Network Partition & Fault Tolerance

---

### F-007: Business Logic - Database Key Collision Risk

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.0 |
| **CVSS Vector** | AV:N  AC:H  PR:L  UI:N  S:U  C:H  I:H  A:N |
| **CWE** | [CWE-341: Predictable from Observable State](https://cwe.mitre.org/data/definitions/341.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:52-54`](../gotham-server/src/public_gotham.rs#L52) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 1 day |

**Vulnerability Description**

The database key construction uses simple string concatenation:

```rust
fn idify(user_id: String, id: String, name: &dyn MPCStruct) -> String {
    format!("{}_{}_{}", user_id, id, name.to_string())
}
```

This creates **collision risks**:

* `user_id="alice_123"` + `id="456"` = `"alice_123_456_keygen"`
* `user_id="alice"` + `id="123_456"` = `"alice_123_456_keygen"`

Same key! This could allow:
* Cross-user data leakage
* MPC session corruption
* Key material mixing between different operations

**Impact**

* User A's key material could be overwritten by User B's data
* MPC protocol state corruption
* Potential key extraction if attacker can trigger collisions

**Remediation**

```rust
use sha2::{Sha256, Digest};

fn idify(user_id: String, id: String, name: &dyn MPCStruct) -> String {
    // Use structured format with delimiters that cannot appear in components
    let structured = format!("user:{};id:{};type:{}", user_id, id, name.to_string());
    
    // Hash to prevent collisions and limit key length
    let mut hasher = Sha256::new();
    hasher.update(structured.as_bytes());
    let hash = hasher.finalize();
    
    format!("{:x}", hash)
}

// Or use a delimiter that's validated to not appear in inputs:
fn idify_v2(user_id: String, id: String, name: &dyn MPCStruct) -> Result<String, DatabaseError> {
    // Validate no forbidden characters
    if user_id.contains('\0') || id.contains('\0') {
        return Err(DatabaseError::InvalidInput("Null bytes not allowed".into()));
    }
    
    // Use null byte as delimiter (cannot appear in validated strings)
    Ok(format!("{}\0{}\0{}", user_id, id, name.to_string()))
}
```

**References**

* CWE-341: Predictable from Observable State
* Database Key Design Best Practices

---

### F-008: Infrastructure - Public Network Binding Without Authentication

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.3 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:L  I:L  A:L |
| **CWE** | [CWE-1188: Initialization of a Resource with an Insecure Default](https://cwe.mitre.org/data/definitions/1188.html) |
| **Location** | [`gotham-server/Rocket.toml:2`](../gotham-server/Rocket.toml#L2) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 2 hours |

**Vulnerability Description**

Server binds to `0.0.0.0` (all interfaces), making it accessible from any network:

```toml
address = "0.0.0.0"  # Accessible from anywhere!
port = 8000
```

Combined with **no authentication (F-001)** and **no TLS (F-003)**, this creates a **completely open service** accessible from the internet.

**Impact**

* Internet-wide exposure of MPC endpoints
* Increased attack surface
* Regulatory non-compliance (custody systems should not be publicly accessible)

**Remediation**

**Production Configuration**:

```toml
[release]
# Bind only to localhost, use reverse proxy for external access
address = "127.0.0.1"
port = 8000
keep_alive = 5
log = "normal"

[release.tls]
certs = "/etc/ssl/certs/gotham-cert.pem"
key = "/etc/ssl/private/gotham-key.pem"
```

**Network Architecture**:

```
Internet → Nginx (with auth, rate limiting, TLS) → Gotham Server (localhost only)
```

**Nginx Configuration**:

```nginx
upstream gotham {
    server 127.0.0.1:8000;
}

server {
    listen 443 ssl http2;
    server_name gotham.example.com;
    
    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=gotham_limit:10m rate=10r/m;
    limit_req zone=gotham_limit burst=5;
    
    # Authentication
    auth_request /auth;
    
    location / {
        proxy_pass http://gotham;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**References**

* CWE-1188: Insecure Default Configuration
* OWASP: Security Misconfiguration
* MPC Problem #044: Backend Server Infrastructure Security

---

## Medium Severity

### F-009: Infrastructure - Missing Security Headers

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 5.3 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:L  I:N  A:N |
| **CWE** | [CWE-693: Protection Mechanism Failure](https://cwe.mitre.org/data/definitions/693.html) |
| **Location** | `gotham-server/src/server.rs` (server configuration) |
| **Status** | Open |
| **Priority** | Medium-term |
| **Effort** | 4 hours |

**Vulnerability Description**

Server does not set security-relevant HTTP headers:

* No `Strict-Transport-Security` (HSTS)
* No `Content-Security-Policy` (CSP)
* No `X-Frame-Options`
* No `X-Content-Type-Options`
* No `Referrer-Policy`

**Impact**

* **Downgrade Attacks**: Without HSTS, attacker can force HTTP
* **ClickjackingIf a web UI is added, it could be framed
* **MIME Sniffing**: Browser may misinterpret responses

**Remediation**

```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Response};

pub struct SecurityHeaders;

#[rocket::async_trait]
impl Fairing for SecurityHeaders {
    fn info(&self) -> Info {
        Info {
            name: "Security Headers",
            kind: Kind::Response,
        }
    }
    
    async fn on_response<'r>(&self, _req: &'r Request<'_>, res: &mut Response<'r>) {
        res.set_raw_header("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        res.set_raw_header("X-Frame-Options", "DENY");
        res.set_raw_header("X-Content-Type-Options", "nosniff");
        res.set_raw_header("Referrer-Policy", "no-referrer");
        res.set_raw_header("Content-Security-Policy", "default-src 'self'");
        res.set_raw_header("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
    }
}

// In server.rs:
pub fn get_server() -> Rocket<Build> {
    rocket::Rocket::build()
        .attach(SecurityHeaders)
        // ... rest
}
```

---

### F-010: Information Disclosure - Error Information Disclosure

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 5.3 |
| **CVSS Vector** | AV:N  AC:L  PR:N  UI:N  S:U  C:L  I:N  A:N |
| **CWE** | [CWE-209: Generation of Error Message Containing Sensitive Information](https://cwe.mitre.org/data/definitions/209.html) |
| **Location** | Multiple error handlers |
| **Status** | Open |
| **Priority** | Medium-term |
| **Effort** | 1 day |

**Vulnerability Description**

Error handlers expose internal information:

```rust
#[catch(500)]
fn internal_error() -> &'static str {
    "Internal server error"  // Generic, but...
}

// Other code panics with detailed messages visible in logs/traces
```

Additionally, if server panics, stack traces may be exposed to client.

**Impact**

* Information leakage about internal implementation
* Easier reconnaissance for attackers
* Potential exposure of database structure or file paths

**Remediation**

```rust
#[catch(500)]
fn internal_error() -> Json<ErrorResponse> {
    // Log detailed error internally
    log::error!("Internal server error occurred");
    
    // Return generic message to client
    Json(ErrorResponse {
        error: "An internal error occurred. Please contact support with request ID: {uuid}".to_string(),
        request_id: generate_request_id(),
    })
}

#[catch(400)]
fn bad_request() -> Json<ErrorResponse> {
    Json(ErrorResponse {
        error: "Invalid request format".to_string(),
        request_id: generate_request_id(),
    })
}
```

---

### F-011: Code Quality - Unwrap Operations Without Error Handling

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 4.9 |
| **CVSS Vector** | AV:N  AC:L  PR:L  UI:N  S:U  C:N  I:N  A:H |
| **CWE** | [CWE-754: Improper Check for Unusual or Exceptional Conditions](https://cwe.mitre.org/data/definitions/754.html) |
| **Location** | Throughout codebase (see below) |
| **Status** | Open |
| **Priority** | Medium-term |
| **Effort** | 3 days |

**Vulnerability Description**

Extensive use of `.unwrap()` throughout:

* `gotham-server/src/public_gotham.rs`: Lines 27, 31, 65, 77, 86
* `gotham-client/src/lib.rs`: Line 94
* `gotham-client/src/ecdsa/keygen.rs`: Lines 41, 48, 71, 82, 91
* `gotham-client/src/ecdsa/sign.rs`: Lines 149, 153

**Impact**

* Service crashes on unexpected input
* Poor user experience
* Difficult debugging

**Remediation**

Perform systematic refactoring:

```rust
// Before:
let value = some_operation().unwrap();

// After:
let value = some_operation()
    .map_err(|e| {
        log::error!("Operation failed: {}", e);
        ErrorType::OperationFailed(e.to_string())
    })?;
```

Use `#![forbid(unwrap_used)]` lint to prevent future unwraps.

---

### F-012: Configuration - Environment Variable Configuration Not Used

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 4.3 |
| **CVSS Vector** | AV:L  AC:L  PR:L  UI:N  S:U  C:L  I:L  A:N |
| **CWE** | [CWE-1188: Initialization of a Resource with an Insecure Default](https://cwe.mitre.org/data/definitions/1188.html) |
| **Location** | [`gotham-server/Settings.toml:6-9`](../gotham-server/Settings.toml#L6) |
| **Status** | Open |
| **Priority** | Medium-term |
| **Effort** | 4 hours |

**Vulnerability Description**

`Settings.toml` defines placeholder fields that are never used:

```toml
region = "" # Override with ENV variable!
pool_id = "" # Override with ENV variable!
issuer = "" # Override with ENV variable!
audience = "" # Override with ENV variable!
```

These appear to be placeholders for AWS Cognito integration that was never implemented. This creates:

* Confusion about what security controls are actually in place
* False sense of security
* Dead code maintenance burden

**Impact**

* Developers may assume JWT validation exists when it doesn't
* Security misconfiguration risk
* Code maintainability issues

**Remediation**

1. **Remove Unused Configuration**:

```toml
# settings.toml
db = "local"
db_name = "gotham_db"

# JWT configuration (if implementing F-001 fix)
jwt_secret = ""  # Set via JWT_SECRET environment variable
jwt_issuer = "https://auth.example.com"
jwt_audience = "gotham-api"
```

2. **Or Implement the JWT Validation** (see F-001 remediation)

---

## Low/Info Severity

### F-013: Infrastructure - No Operational Monitoring or Logging

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **CVSS Score** | N/A |
| **CWE** | [CWE-778: Insufficient Logging](https://cwe.mitre.org/data/definitions/778.html) |
| **Location** | Entire codebase |
| **Status** | Advisory |
| **Priority** | Long-term |
| **Effort** | 2 days |

**Vulnerability Description**

No structured logging, monitoring, or alerting:

* No audit trail of MPC operations
* No metrics collection (request rates, error rates, latency)
* No alerting on anomalous behavior
* Difficult to investigate security incidents

**Impact**

* Cannot detect or respond to active attacks
* No compliance audit trail
* Difficult post-incident forensics

**Remediation**

Implement comprehensive logging:

```rust
use slog::{Logger, Drain, o, info, warn, error};

pub struct AuditLogger {
    logger: Logger,
}

impl AuditLogger {
    pub fn log_keygen(&self, user_id: &str, session_id: &str) {
        info!(self.logger, "MPC keygen initiated";
            "user_id" => user_id,
            "session_id" => session_id,
            "timestamp" => chrono::Utc::now().to_rfc3339(),
        );
    }
    
    pub fn log_sign(&self, user_id: &str, session_id: &str, message_hash: &str) {
        info!(self.logger, "MPC signing initiated";
            "user_id" => user_id,
            "session_id" => session_id,
            "message_hash" => message_hash,
            "timestamp" => chrono::Utc::now().to_rfc3339(),
        );
    }
}
```

Add monitoring with Prometheus:

```rust
use prometheus::{IntCounter, Histogram, Registry};

lazy_static! {
    static ref KEYGEN_TOTAL: IntCounter = 
        IntCounter::new("gotham_keygen_total", "Total keygen operations").unwrap();
    
    static ref SIGN_DURATION: Histogram =
        Histogram::new("gotham_sign_duration_seconds", "Sign operation duration").unwrap();
}
```

**References**

* OWASP: Insufficient Logging & Monitoring
* MPC Problem #113: Forensic Investigation & Incident Response Logging

---

## Summary Statistics

| Severity | Count | Status |
|----------|-------|--------|
| **CRITICAL** | 3 | Open |
| **HIGH** | 5 | Open |
| **MEDIUM** | 4 | Open |
| **LOW/INFO** | 1 | Advisory |
| **TOTAL** | **13** | |

## Glossary & Abbreviations

* **MPC**: Multi-Party Computation
* **ECDSA**: Elliptic Curve Digital Signature Algorithm
* **JWT**: JSON Web Token
* **TLS**: Transport Layer Security
* **MITM**: Man-in-the-Middle attack
* **DoS**: Denial of Service
* **CVSS**: Common Vulnerability Scoring System
* **CWE**: Common Weakness Enumeration
* **HSM**: Hardware Security Module
* **KYC/AML**: Know Your Customer / Anti-Money Laundering

## References & External Standards

* **OWASP Top 10 2021**: https://owasp.org/Top10/
* **CWE Top 25**: https://cwe.mitre.org/top25/
* **NIST SP 800-53**: Security and Privacy Controls
* **Rocket Framework Security**: https://rocket.rs/v0.5/guide/
* **Rust Security Guidelines**: https://anssi-fr.github.io/rust-guide/
* **Lindell'17 Protocol**: Fast Secure Two-Party ECDSA Signing (https://eprint.iacr.org/2017/552)
* **MPC Wallet Problems Library**: 213 scenarios analyzed
