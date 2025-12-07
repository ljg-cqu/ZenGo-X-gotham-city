# Remediation Recommendations & Roadmap

## Table of Contents

- [Remediation Philosophy](#remediation-philosophy)
- [Phase 1: Immediate (Blocking for Production)](#phase-1-immediate-blocking-for-production)
- [Phase 2: Short-Term (1 Week)](#phase-2-short-term-1-week)
- [Phase 3: Medium-Term (1 Month)](#phase-3-medium-term-1-month)
- [Compliance Mapping](#compliance-mapping)
- [Implementation Guidance](#implementation-guidance)
- [Verification & Testing](#verification--testing)
- [Measurement & Follow-up](#measurement--follow-up)

---

## Remediation Philosophy

**Core Principle**: Security must be **layered** (defense in depth), **verifiable** (testable), and **maintainable** (sustainable).

### Remediation Priorities

1. **Immediate (Phase 1)**: Fix CRITICAL findings that enable trivial asset theft
2. **Short-Term (Phase 2)**: Fix HIGH findings and implement monitoring
3. **Medium-Term (Phase 3)**: Fix MEDIUM findings and establish security processes

### Cost-Benefit Analysis

| Phase | Effort (hours) | Impact | Risk Reduction |
|-------|----------------|--------|----------------|
| Phase 1 | 40-60 | **CRITICAL** | 70-80% of critical risk |
| Phase 2 | 30-40 | **HIGH** | 15-20% remaining risk |
| Phase 3 | 60-80 | **MEDIUM** | Final 5-10% + long-term sustainability |

**Total Estimated Effort**: 130-180 hours (16-23 developer-days)

---

## Phase 1: Immediate (Blocking for Production)

**Timeline**: 3-5 days  
**Status**: **MANDATORY** before ANY production deployment  
**Risk**: Without Phase 1, system is **UNSUITABLE FOR PRODUCTION**

### P1.1: Implement Authentication & Authorization (F-001)

**Priority**: #1 - CRITICAL  
**Effort**: 16-24 hours  
**Assigned To**: Backend Security Team

**Tasks**:

1. **JWT Validation Infrastructure**:

```rust
// src/auth/jwt.rs
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use rocket::http::Status;
use rocket::request::{self, Request, FromRequest};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,           // User ID
    pub customer_id: String,   // Customer identifier
    pub exp: usize,            // Expiration time
    pub iat: usize,            // Issued at
    pub roles: Vec<String>,    // User roles (admin, user, readonly)
}

pub struct AuthenticatedUser {
    pub claims: Claims,
}

#[rocket::async_trait]
impl<'r> FromRequest<'r> for AuthenticatedUser {
    type Error = String;

    async fn from_request(req: &'r Request<'_>) -> request::Outcome<Self, Self::Error> {
        // Extract JWT from Authorization header
        let token = req.headers().get_one("Authorization")
            .and_then(|h| h.strip_prefix("Bearer "));
        
        match token {
            Some(jwt) => {
                let secret = std::env::var("JWT_SECRET")
                    .expect("JWT_SECRET must be set");
                
                let validation = Validation::default();
                match decode::<Claims>(
                    jwt,
                    &DecodingKey::from_secret(secret.as_bytes()),
                    &validation,
                ) {
                    Ok(token_data) => {
                        // Log successful auth
                        log::info!("Authenticated user: {}", token_data.claims.sub);
                        request::Outcome::Success(AuthenticatedUser {
                            claims: token_data.claims,
                        })
                    }
                    Err(e) => {
                        log::warn!("JWT validation failed: {}", e);
                        request::Outcome::Failure((
                            Status::Unauthorized,
                            format!("Invalid token: {}", e),
                        ))
                    }
                }
            }
            None => request::Outcome::Failure((
                Status::Unauthorized,
                "Missing Authorization header".to_string(),
            )),
        }
    }
}
```

2. **Update gotham-engine Routes**:

```rust
// Modify all route signatures to require authentication
#[post("/ecdsa/keygen/first")]
async fn wrap_keygen_first(
    user: AuthenticatedUser,  // Add this!
    db: &State<Mutex<Box<dyn Db>>>,
) -> Result<Json<party_one::KeyGenFirstMsg>, Status> {
    // Now user.claims.customer_id is validated
    // ...
}
```

3. **Authorization Policy Engine**:

```rust
// src/auth/policy.rs
pub struct PolicyEngine {
    // Policy rules stored here
}

impl PolicyEngine {
    pub fn can_generate_key(&self, user: &Claims) -> Result<(), PolicyError> {
        // Check user permissions
        if !user.roles.contains(&"keygen".to_string()) {
            return Err(PolicyError::InsufficientPermissions);
        }
        
        // Check rate limits (daily keygen quota)
        let daily_count = self.get_daily_keygen_count(&user.customer_id)?;
        if daily_count >= MAX_DAILY_KEYGEN {
            return Err(PolicyError::QuotaExceeded);
        }
        
        Ok(())
    }
    
    pub fn can_sign_transaction(
        &self,
        user: &Claims,
        tx: &ValidatedTransaction,
    ) -> Result<(), PolicyError> {
        // Check signing permissions
        if !user.roles.contains(&"sign".to_string()) {
            return Err(PolicyError::InsufficientPermissions);
        }
        
        // Check recipient whitelist
        if !self.is_whitelisted(&tx.to, &user.customer_id)? {
            return Err(PolicyError::RecipientNotWhitelisted);
        }
        
        // Check spending limits
        let daily_spent = self.get_daily_spending(&user.customer_id)?;
        if daily_spent + &tx.value > user.daily_limit {
            return Err(PolicyError::DailyLimitExceeded);
        }
        
        // Check high-value approval requirement
        if tx.value > HIGH_VALUE_THRESHOLD {
            if !self.has_multi_party_approval(&user.customer_id, &tx)? {
                return Err(PolicyError::MultiPartyApprovalRequired);
            }
        }
        
        Ok(())
    }
}
```

**Testing**:
```bash
# Test without auth
curl -X POST http://localhost:8000/ecdsa/keygen/first
# Should return 401 Unauthorized

# Test with valid JWT
curl -X POST http://localhost:8000/ecdsa/keygen/first \
  -H "Authorization: Bearer eyJ..."
# Should succeed
```

**Verification Criteria**:
- [ ] All endpoints require valid JWT
- [ ] Invalid/expired tokens are rejected (401)
- [ ] JWT validation logged
- [ ] Test suite updated with auth scenarios

---

### P1.2: Add Transaction Validation (F-002)

**Priority**: #2 - CRITICAL  
**Effort**: 12-16 hours  
**Assigned To**: Backend + Smart Contract Team

**Implementation**:

```rust
// src/validation/transaction.rs
use serde::{Deserialize, Serialize};
use bigint::BigInt;

#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct ValidatedTransaction {
    pub chain_id: u64,
    pub nonce: u64,
    pub to: String,             // Recipient address
    pub value: BigInt,          // Amount in wei
    pub gas_limit: u64,
    pub gas_price: BigInt,
    pub data: Vec<u8>,          // Contract call data (if any)
}

pub struct TransactionValidator {
    policy: PolicyEngine,
}

impl TransactionValidator {
    pub fn validate_and_authorize(
        &self,
        tx: &ValidatedTransaction,
        user: &Claims,
    ) -> Result<(), ValidationError> {
        // 1. Basic format validation
        self.validate_format(tx)?;
        
        // 2. Chain ID validation
        if !SUPPORTED_CHAINS.contains(&tx.chain_id) {
            return Err(ValidationError::UnsupportedChain(tx.chain_id));
        }
        
        // 3. Address validation
        self.validate_address(&tx.to)?;
        
        // 4. Value validation
        if tx.value < BigInt::zero() {
            return Err(ValidationError::NegativeValue);
        }
        
        // 5. Policy checks
        self.policy.can_sign_transaction(user, tx)?;
        
        // 6. AML/KYC screening (if required)
        if let Some(aml_provider) = &self.aml_provider {
            aml_provider.screen_transaction(tx).await?;
        }
        
        // 7. Transaction simulation (optional but recommended)
        if self.simulate_enabled {
            let simulation = self.simulate_transaction(tx).await?;
            if simulation.reverts {
                return Err(ValidationError::TransactionWouldRevert(
                    simulation.revert_reason
                ));
            }
        }
        
        Ok(())
    }
    
    fn validate_address(&self, address: &str) -> Result<(), ValidationError> {
        // EVM address validation
        if !address.starts_with("0x") || address.len() != 42 {
            return Err(ValidationError::InvalidAddress(address.to_string()));
        }
        
        // Checksum validation (EIP-55)
        if !self.verify_checksum(address) {
            return Err(ValidationError::InvalidChecksum(address.to_string()));
        }
        
        Ok(())
    }
}
```

**Update Signing Endpoint**:

```rust
#[derive(Serialize, Deserialize)]
pub struct SignRequest {
    pub transaction: ValidatedTransaction,  // Changed from raw message!
    pub party_two_sign_message: party2::SignMessage,
    pub x_pos: BigInt,
    pub y_pos: BigInt,
}

#[post("/ecdsa/sign/<id>/second", data = "<request>")]
async fn wrap_sign_second(
    user: AuthenticatedUser,
    id: String,
    request: Json<SignRequest>,
    db: &State<Mutex<Box<dyn Db>>>,
    validator: &State<TransactionValidator>,
) -> Result<Json<party_one::SignatureRecid>, Status> {
    // Validate transaction
    validator.validate_and_authorize(&request.transaction, &user.claims)
        .map_err(|e| {
            log::warn!("Transaction validation failed: {}", e);
            Status::Forbidden
        })?;
    
    // Compute message hash from validated transaction
    let message_hash = compute_tx_hash(&request.transaction);
    
    // Log signing attempt
    log::info!(
        "Signing transaction for user {}, to: {}, value: {}",
        user.claims.customer_id,
        request.transaction.to,
        request.transaction.value
    );
    
    // Proceed with MPC signing
    // ...
}
```

**Verification Criteria**:
- [ ] Raw message signing rejected
- [ ] Only ValidatedTransaction accepted
- [ ] Policy violations return 403 Forbidden
- [ ] High-value transactions require approval
- [ ] Transaction details logged

---

### P1.3: Deploy TLS/HTTPS (F-003)

**Priority**: #3 - CRITICAL  
**Effort**: 8-12 hours (including certificate procurement)  
**Assigned To**: DevOps

**Implementation Steps**:

1. **Obtain TLS Certificates**:

```bash
# Production: Use Let's Encrypt
certbot certonly --standalone \
  --preferred-challenges http \
  -d gotham.example.com

# Certificates will be in:
# /etc/letsencrypt/live/gotham.example.com/fullchain.pem
# /etc/letsencrypt/live/gotham.example.com/privkey.pem
```

2. **Update Rocket Configuration**:

```toml
# Rocket.toml
[release]
address = "127.0.0.1"  # Bind to localhost only!
port = 8000
keep_alive = 5
log = "critical"

[release.tls]
certs = "/etc/letsencrypt/live/gotham.example.com/fullchain.pem"
key = "/etc/letsencrypt/live/gotham.example.com/privkey.pem"
```

3. **Deploy Nginx Reverse Proxy** (Recommended):

```nginx
# /etc/nginx/sites-available/gotham
upstream gotham_backend {
    server 127.0.0.1:8000;
}

server {
    listen 443 ssl http2;
    server_name gotham.example.com;
    
    # TLS Configuration
    ssl_certificate /etc/letsencrypt/live/gotham.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/gotham.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=gotham_limit:10m rate=10r/m;
    limit_req zone=gotham_limit burst=5 nodelay;
    
    # Proxy to backend
    location / {
        proxy_pass http://gotham_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name gotham.example.com;
    return 301 https://$server_name$request_uri;
}
```

4. **Update Client Configuration**:

```rust
// gotham-client/src/lib.rs
impl ClientShim<reqwest::Client> {
    pub fn new(endpoint: String, auth_token: Option<String>) -> Self {
        let client = reqwest::Client::builder()
            .https_only(true)  // Enforce HTTPS
            .min_tls_version(reqwest::tls::Version::TLS_1_2)
            .timeout(Duration::from_secs(60))
            .build()
            .expect("Failed to build HTTPS client");
        
        Self {
            client,
            auth_token,
            endpoint: endpoint.replace("http://", "https://"),  // Force HTTPS
        }
    }
}
```

**Verification Criteria**:
- [ ] HTTP requests redirected to HTTPS
- [ ] TLS 1.2+ enforced
- [ ] Certificate valid and trusted
- [ ] HSTS header present
- [ ] Client enforces HTTPS-only

---

### P1.4: Implement Rate Limiting (F-005)

**Priority**: #4 - HIGH  
**Effort**: 6-8 hours  
**Assigned To**: Backend

**Implementation** (Nginx-based):

Already included in P1.3 Nginx config. For application-level:

```rust
// src/middleware/rate_limit.rs
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Data, Response};
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::Mutex;
use std::time::{Duration, Instant};

pub struct RateLimiter {
    requests: Arc<Mutex<HashMap<String, Vec<Instant>>>>,
    config: RateLimitConfig,
}

pub struct RateLimitConfig {
    pub max_requests: usize,
    pub window: Duration,
    pub ban_duration: Duration,
}

impl Default for RateLimitConfig {
    fn default() -> Self {
        Self {
            max_requests: 10,
            window: Duration::from_secs(60),
            ban_duration: Duration::from_secs(300),  // 5 min ban
        }
    }
}

impl RateLimiter {
    pub fn new(config: RateLimitConfig) -> Self {
        Self {
            requests: Arc::new(Mutex::new(HashMap::new())),
            config,
        }
    }
    
    async fn is_allowed(&self, key: &str) -> bool {
        let mut requests = self.requests.lock().await;
        let now = Instant::now();
        
        // Clean old requests
        let cutoff = now - self.config.window;
        let entry = requests.entry(key.to_string()).or_insert_with(Vec::new);
        entry.retain(|&t| t > cutoff);
        
        // Check limit
        if entry.len() >= self.config.max_requests {
            log::warn!("Rate limit exceeded for {}", key);
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
            kind: Kind::Request | Kind::Response,
        }
    }
    
    async fn on_request(&self, req: &mut Request<'_>, _: &mut Data<'_>) {
        let key = if let Some(auth_user) = req.local_cache(|| None::<String>) {
            auth_user.clone()  // Rate limit by user ID
        } else {
            req.client_ip()
                .map(|ip| ip.to_string())
                .unwrap_or_else(|| "unknown".to_string())
        };
        
        if !self.is_allowed(&key).await {
            req.local_cache(|| "rate_limited");
        }
    }
    
    async fn on_response<'r>(&self, req: &'r Request<'_>, res: &mut Response<'r>) {
        if let Some(limited) = req.local_cache(|| None::<String>) {
            if limited == "rate_limited" {
                res.set_status(Status::TooManyRequests);
                res.set_sized_body(None, std::io::Cursor::new(
                    json!({
                        "error": "Rate limit exceeded",
                        "retry_after": 60
                    }).to_string()
                ));
            }
        }
    }
}

// In server.rs:
pub fn get_server() -> Rocket<Build> {
    rocket::Rocket::build()
        .attach(RateLimiter::new(RateLimitConfig::default()))
        // ...
}
```

**Verification Criteria**:
- [ ] Legitimate requests succeed
- [ ] Excessive requests return 429 Too Many Requests
- [ ] Rate limit persists across requests
- [ ] Different users have independent limits

---

### P1 Verification & Sign-Off

Before proceeding to Phase 2, verify:

- [ ] **Authentication**: All endpoints require valid JWT
- [ ] **Authorization**: Transaction policies enforced
- [ ] **TLS**: All traffic encrypted (verify with Wireshark)
- [ ] **Rate Limiting**: DoS attacks mitigated
- [ ] **Testing**: All Phase 1 tests pass
- [ ] **Documentation**: Updated deployment guide
- [ ] **Security Review**: Independent verification by security team

**Sign-Off Required**: CTO + CISO

---

## Phase 2: Short-Term (1 Week)

**Timeline**: 5-7 days after Phase 1  
**Risk Reduction**: Additional 15-20%

### P2.1: Fix All HIGH Severity Findings

**Tasks**:
- F-004: Path traversal sanitization
- F-006: Replace panics with error handling
- F-007: Fix database key collision
- F-008: Restrict network binding to localhost

**Effort**: 16-20 hours

### P2.2: Add Security Headers (F-009)

See remediation in F-009 in Findings.md.  
**Effort**: 4 hours

### P2.3: Implement Comprehensive Logging (F-013)

```rust
use slog::{Logger, Drain, o, info, warn};
use slog_json::Json;

pub struct AuditLogger {
    logger: Logger,
}

impl AuditLogger {
    pub fn new() -> Self {
        let drain = slog_json::Json::new(std::io::stdout())
            .add_default_keys()
            .build()
            .fuse();
        let drain = slog_async::Async::new(drain).build().fuse();
        
        Self {
            logger: Logger::root(drain, o!("service" => "gotham")),
        }
    }
    
    pub fn log_keygen(&self, user_id: &str, session_id: &str, success: bool) {
        info!(self.logger, "MPC keygen";
            "event" => "keygen",
            "user_id" => user_id,
            "session_id" => session_id,
            "success" => success,
            "timestamp" => chrono::Utc::now().to_rfc3339(),
        );
    }
    
    pub fn log_sign(&self, user_id: &str, tx: &ValidatedTransaction, success: bool) {
        info!(self.logger, "MPC signing";
            "event" => "sign",
            "user_id" => user_id,
            "chain_id" => tx.chain_id,
            "to" => &tx.to,
            "value" => tx.value.to_string(),
            "success" => success,
            "timestamp" => chrono::Utc::now().to_rfc3339(),
        );
    }
}
```

**Effort**: 12 hours

### P2.4: Error Handling Refactor (F-006, F-011)

Systematic refactoring of all `.unwrap()` and `panic!()` calls.  
**Effort**: 16-20 hours

---

## Phase 3: Medium-Term (1 Month)

**Timeline**: 2-4 weeks  
**Risk Reduction**: Final 5-10% + sustainability

### P3.1: Dependency Updates & CVE Remediation

- Update Rocket to stable release
- Update all dependencies
- Regular `cargo audit` integration

**Effort**: 8 hours

### P3.2: Enhanced Testing

- Integration tests with authentication
- Fuzzing for input validation
- Property-based testing for cryptographic operations
- Penetration testing

**Effort**: 40 hours

### P3.3: Formal Security Processes

- Security code review checklist
- Secure SDLC implementation
- Incident response playbook
- Regular security training

**Effort**: 24 hours

---

## Compliance Mapping

### OWASP Top 10 2021 Remediation

| Category | Current | After P1 | After P2 | After P3 |
|----------|---------|----------|----------|----------|
| A01: Broken Access Control | ❌ | ✅ | ✅ | ✅ |
| A02: Cryptographic Failures | ❌ | ✅ | ✅ | ✅ |
| A03: Injection | ⚠️ | ⚠️ | ✅ | ✅ |
| A05: Security Misconfiguration | ❌ | ⚠️ | ✅ | ✅ |
| A07: Authentication Failures | ❌ | ✅ | ✅ | ✅ |
| A09: Logging & Monitoring | ❌ | ❌ | ✅ | ✅ |

### CWE Top 25 Coverage

After full remediation, system will address:
- CWE-306: Missing Authentication ✅
- CWE-319: Cleartext Transmission ✅
- CWE-345: Insufficient Verification ✅
- CWE-22: Path Traversal ✅
- CWE-770: Resource Allocation Without Limits ✅

---

## Measurement & Follow-up

* **Phase 1 Completion Target**: 2025-12-17
* **Phase 1 Re-Audit**: 2025-12-20
* **Phase 2 Completion Target**: 2025-12-27
* **Phase 3 Completion Target**: 2026-01-17
* **Quarterly Security Review**: Every 3 months post-deployment
* **Responsible Party**: ZenGo Engineering + Security Teams
* **Escalation**: CTO (engineering) / CISO (security)

## Success Metrics

| Metric | Current | After P1 | After P3 | Target |
|--------|---------|----------|----------|--------|
| Critical Findings | 3 | 0 | 0 | 0 |
| High Findings | 5 | 0 | 0 | 0 |
| Auth Coverage | 0% | 100% | 100% | 100% |
| TLS Coverage | 0% | 100% | 100% | 100% |
| Code Coverage | ~40% | ~60% | ~80% | 80%+ |
| Incident Response Time | N/A | <1hr | <15min | <15min |
