# Audit Findings & Dependency Analysis

## Finding Summary

This audit focused on the Gotham City workspace (`gotham-server`, `gotham-client`, `demo-wallet`, and `integration-tests`) with an emphasis on MPC key management, signing flows, and data-at-rest protection.
The core cryptographic protocol is delegated to `two-party-ecdsa` and `gotham-engine`, which were treated as trusted external components for this review; we examined how they are integrated and configured.

Key issues include a default allow-all authorization hook in the server (`Db::granted()`), plaintext storage of long-lived ECDSA key shares on both server and client, and a lack of encryption-at-rest or structured audit logging.
Additional medium and lower severity findings cover FFI robustness, use of floating-point amounts for Bitcoin/Ethereum transactions, and dependency/deployment hygiene.

**Findings by Severity**:
- CRITICAL: F-001
- HIGH: F-002, F-003
- MEDIUM: F-004, F-005
- LOW: F-006
- INFO: F-007, F-008

---

## Findings (Detailed)

### F-001: Auth/AuthZ – Default Allow-All Authorization for MPC Signing

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 9.1 |
| **CVSS Vector** | AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:L |
| **CWE** | [CWE-285: Improper Authorization](https://cwe.mitre.org/data/definitions/285.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:92-95`](../../gotham-server/src/public_gotham.rs#L92-L95) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 1–2 days |

**Vulnerability Description**

The `Db::granted()` method in the server-side `PublicGotham` implementation always returns `Ok(true)`, regardless of the message content or customer identifier.
This function is the hook that `gotham-engine` uses to decide whether a given operation (for example, signing a transaction) is authorized.
When `gotham-server` is deployed in any environment where untrusted clients can reach the ECDSA endpoints, this default effectively disables authorization and allows any caller with a valid session identifier to request signatures.

**Vulnerable Code**

```rust
// gotham-server/src/public_gotham.rs:92-95
/// the granted function implements the logic of tx authorization. If no tx authorization is needed the function returns always true
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)
}
```

**Attack Vector**

- An attacker obtains or guesses a valid `customerId`/session identifier (for example, via leaked logs, weak session ID handling in surrounding systems, or collusion with a compromised client).
- The attacker sends crafted HTTP requests to the signing endpoints (`/ecdsa/sign/{id}/first` and `/ecdsa/sign/{id}/second`), impersonating a legitimate client.
- Because `Db::granted()` always returns `true`, the server has no effective policy gate to distinguish legitimate from unauthorized signing requests.

Even if the deployment intends to restrict access at the network level (for example, via private networking), this code-level default is unsafe: any misconfiguration or future exposure of the server to additional clients instantly creates a severe risk of unauthorized signatures.

**Proof of Concept**

A simplified proof-of-concept flow (assuming the attacker can reach the server and has a valid session ID `SESSION_ID`):

```bash
# 1. Start a keygen flow as any reachable client to obtain an id
curl -s -X POST http://SERVER:8000/ecdsa/keygen/first \
  -H 'Content-Type: application/json' > /tmp/kg_first.json

# 2. Extract the id from the response (pair of [id, message])
ID=$(jq -r '.[0]' /tmp/kg_first.json)

# 3. Reuse ID to drive signing flows without any authorization checks
# (payload structure derived from integration tests and client library)

# First signing round (attacker-controlled)
curl -s -X POST "http://SERVER:8000/ecdsa/sign/${ID}/first" \
  -H 'Content-Type: application/json' \
  -d '{"dummy":"payload"}'

# Second signing round
curl -s -X POST "http://SERVER:8000/ecdsa/sign/${ID}/second" \
  -H 'Content-Type: application/json' \
  -d '{"dummy":"payload2"}'
```

The exact payload formats are enforced by `gotham-engine` and `two-party-ecdsa`, but the authorization decision is controlled exclusively by `Db::granted()`, which currently cannot reject any request.

**Impact**

- **Confidentiality**: Indirect — if unauthorized signatures are used to move funds, transaction histories and balances may be exposed through public blockchains and external services.
- **Integrity**: Severe — an attacker can potentially trigger MPC signing operations with a compromised or colluding client share, enabling unauthorized transfers from wallets depending on this service.
- **Availability**: Limited — repeated unauthorized signing requests can also increase load but this is secondary to integrity loss.
- **Scope**: System-wide — affects all keys and accounts whose Party1 share is stored in the affected RocksDB instance.

**Remediation**

```rust
// Example: enforce policy in Db::granted()
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    // 1. Extract and validate an authenticated identity (for example, from a JWT
    //    or mTLS client certificate passed via gotham-engine).
    // 2. Look up authorization policy for this customer_id and message type.
    // 3. Enforce limits (amount caps, whitelists, rate limits) and return Ok(true)
    //    only when policy explicitly allows the operation.

    // Placeholder: reject by default until a real policy is implemented.
    Ok(false)
}
```

At minimum, the project should:

- Implement a real `granted()` policy for any deployment that handles real funds.
- Ensure that `gotham-server` is not reachable from untrusted networks unless strong authentication and policy enforcement are in place.
- Add structured audit logging for all `granted()` decisions.

**Additional Notes**

- The `wiki/Security.md` document already calls this out as a critical warning, but the code itself still ships with an allow-all default.
  Code must be the source of truth for security behavior.

---

### F-002: Data Security – Plaintext Server Key Shares in RocksDB

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.5 |
| **CVSS Vector** | AV:L/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:N |
| **CWE** | [CWE-312: Cleartext Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/312.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:14-16`](../../gotham-server/src/public_gotham.rs#L14-L16), [`gotham-server/src/public_gotham.rs:34-44`](../../gotham-server/src/public_gotham.rs#L34-L44), [`gotham-server/src/public_gotham.rs:58-68`](../../gotham-server/src/public_gotham.rs#L58-L68) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 3–5 days |

**Issue**

Party1 MPC key shares and protocol state are persisted in RocksDB without any encryption at rest or application-level access control.
The `PublicGotham` struct embeds a `rocksdb::DB` instance opened via `DB::open_default`, and `Db::insert()` writes serialized values directly under composite keys derived from `customerId`, `id`, and table names.

**Evidence**

```rust
// gotham-server/src/public_gotham.rs:14-16,34-44
pub struct PublicGotham {
    rocksdb_client: rocksdb::DB,
}

impl PublicGotham {
    pub fn new() -> Self {
        let settings = get_settings_as_map();
        let db_name = settings.get("db_name").unwrap_or(&"db".to_string()).clone();
        if !db_name.chars().all(|e| char::is_ascii_alphanumeric(&e)) {
            panic!("DB name is illegal, may only contain alphanumeric characters");
        }
        let rocksdb_client = rocksdb::DB::open_default(format!("./{}", db_name)).unwrap();

        PublicGotham { rocksdb_client }
    }
}

// gotham-server/src/public_gotham.rs:58-67
async fn insert(
    &self,
    key: &DbIndex,
    table_name: &dyn MPCStruct,
    value: &dyn Value,
) -> Result<(), DatabaseError> {
    let identifier = idify(key.clone().customerId, key.clone().id, table_name);
    let v_string = serde_json::to_string(&value).unwrap();
    let _ = self.rocksdb_client.put(identifier, v_string.clone());
    Ok(())
}
```

**Context**

- Server key shares are long-lived secret material that, when combined with a corresponding client share, allow full control over associated wallets.
- The threat model in `wiki/Security.md` explicitly calls out key share theft via server compromise as a primary risk.

**Remediation**

- Introduce encryption-at-rest for RocksDB data. Options include:
  - File-system level encryption (LUKS, volume encryption) with restricted access to the DB directory.
  - Application-level encryption of values before writing, using a key stored in a secure keystore/HSM.
- Tighten filesystem permissions for the DB directory (for example, `chmod 700` owned by the service user).
- Add structured logging and monitoring for access to the DB path.

---

### F-003: Data Security – Plaintext Wallet and Escrow Secrets on Client Disk

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.1 |
| **CVSS Vector** | AV:L/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N |
| **CWE** | [CWE-312: Cleartext Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/312.html) |
| **Location** | [`demo-wallet/src/bitcoin/mod.rs:114-121`](../../demo-wallet/src/bitcoin/mod.rs#L114-L121), [`demo-wallet/src/bitcoin/mod.rs:143-167`](../../demo-wallet/src/bitcoin/mod.rs#L143-L167), [`demo-wallet/src/bitcoin/mod.rs:245-250`](../../demo-wallet/src/bitcoin/mod.rs#L245-L250), [`demo-wallet/src/bitcoin/escrow.rs:31-37`](../../demo-wallet/src/bitcoin/escrow.rs#L31-L37) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 3–5 days |

**Issue**

The demo Bitcoin wallet stores sensitive material (client MPC key shares and escrow private keys) as JSON files on disk without encryption or permission hardening.
A local attacker with filesystem access can recover these secrets and, in combination with server key shares, reconstruct full private keys and sign arbitrary transactions.

**Evidence**

```rust
// demo-wallet/src/bitcoin/mod.rs:114-121
#[derive(Serialize, Deserialize)]
pub struct BitcoinWallet {
    pub id: String,
    pub network: String,
    pub private_share: PrivateShare,
    pub last_derived_pos: u32,
    pub addresses_derivation_map: HashMap<String, AddressDerivation>,
}

// demo-wallet/src/bitcoin/mod.rs:245-250
pub fn save_to(&self, path: &str) {
    let wallet_json = serde_json::to_string_pretty(self).unwrap();
    fs::write(path, wallet_json).expect("Unable to save wallet!");
}
```

```rust
// demo-wallet/src/bitcoin/escrow.rs:31-37
impl Escrow {
    pub fn new(path: &str) -> Escrow {
        let secret: FE = ECScalar::new_random();
        let g: GE = ECPoint::generator();
        let public: GE = g * secret;
        fs::write(path, serde_json::to_string(&(secret, public)).unwrap())
            .expect("Unable to save escrow secret!");

        Escrow { secret, public }
    }
}
```

**Context**

- Wallet files (`wallet.json`) and escrow secrets (`escrow-sk.json` by default) are stored wherever the user runs the CLI, often with default OS file permissions.
- These files contain all material needed for the client side of 2P-ECDSA and, in the escrow case, the private key for decrypting backed-up secrets.

**Remediation**

- Treat the demo wallet as a reference implementation only; for any production-like use:
  - Encrypt wallet and escrow files at rest (for example, using OS keystores, password-based encryption, or platform-specific secure storage).
  - Store secrets in OS-provided secure key stores where available (Keychain, Android Keystore, etc.).
  - Document secure backup procedures and encourage users to store backups in encrypted locations.

---

### F-004: Code Quality – FFI Boundaries Panic on Malformed Inputs

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 5.0 |
| **CWE** | [CWE-248: Uncaught Exception](https://cwe.mitre.org/data/definitions/248.html) |
| **Location** | [`gotham-client/src/ecdsa/keygen.rs:123-155`](../../gotham-client/src/ecdsa/keygen.rs#L123-L155), [`gotham-client/src/ecdsa/recover.rs:135-143`](../../gotham-client/src/ecdsa/recover.rs#L135-L143) |

**Issue**

Several `extern "C"` FFI functions in `gotham-client` (`get_client_master_key`, helpers in `recover.rs`) use `CStr::from_ptr` and `to_str()` and call `panic!` when decoding fails.
A malformed or non-UTF-8 string passed by the embedding application can cause the Rust side to abort the process instead of returning a structured error.
While this is primarily a robustness issue (the host controls the inputs), it makes integrating the library safely more difficult and can turn input validation mistakes into full process crashes.

**Evidence**

```rust
// gotham-client/src/ecdsa/keygen.rs:129-137
#[no_mangle]
pub unsafe extern "C" fn get_client_master_key(
    c_endpoint: *const c_char,
    c_auth_token: *const c_char,
) -> *mut c_char {
    let raw_endpoint = CStr::from_ptr(c_endpoint);
    let endpoint = match raw_endpoint.to_str() {
        Ok(s) => s,
        Err(_) => panic!("Error while decoding raw endpoint"),
    };

    let raw_auth_token = CStr::from_ptr(c_auth_token);
    let auth_token = match raw_auth_token.to_str() {
        Ok(s) => s,
        Err(_) => panic!("Error while decoding auth token"),
    };
    // ...
}
```

```rust
// gotham-client/src/ecdsa/recover.rs:135-143
fn get_str_from_c_char(c: *const c_char) -> String {
    let raw = unsafe { CStr::from_ptr(c) };
    let s = match raw.to_str() {
        Ok(s) => s,
        Err(_) => panic!("Error while decoding c_char to string"),
    };

    s.to_string()
}
```

**Context**

- These FFI functions are intended to be called from iOS/Android or other native code.
- In robust FFI designs, malformed input should result in a well-defined error value or an error string, not a panic that aborts the host process.

**Remediation**

- Replace `panic!` paths with safe error signaling:
  - For C ABI functions, return an error-encoded C string (similar to `error_to_c_string`) instead of panicking.
  - For internal helpers, return `Result<String, Error>` and propagate errors to the FFI boundary.
- Consider providing explicit "free" functions for returned strings to avoid leaks on the host side.

---

### F-005: Code Quality – Floating-Point Amount Handling for Bitcoin and Ethereum Transactions

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **CVSS Score** | 4.0 |
| **CWE** | [CWE-190: Integer Overflow or Wraparound](https://cwe.mitre.org/data/definitions/190.html) (related risk due to float-to-integer conversion) |
| **Location** | [`demo-wallet/src/bitcoin/commands.rs:97-103`](../../demo-wallet/src/bitcoin/commands.rs#L97-L103), [`demo-wallet/src/bitcoin/mod.rs:263-303`](../../demo-wallet/src/bitcoin/mod.rs#L263-L303), [`demo-wallet/src/ethereum/commands.rs:58-63`](../../demo-wallet/src/ethereum/commands.rs#L58-L63), [`demo-wallet/src/ethereum/commands.rs:144-151`](../../demo-wallet/src/ethereum/commands.rs#L144-L151) |

**Issue**

The demo wallet represents Bitcoin and Ethereum transfer amounts as `f32`/`f64` and converts them to integer units (satoshis or wei) using simple multiplication and casting.
This approach is vulnerable to rounding errors and, in edge cases, can produce values that differ from user intent (for example, due to floating-point representation limits), especially for large or precise amounts.

**Evidence**

```rust
// demo-wallet/src/bitcoin/mod.rs:295-303
let amount_satoshi = (amount_btc * 100_000_000 as f32) as u64;
// ...
let total_selected = selected
    .clone()
    .into_iter()
    .fold(0, |sum, val| sum + val.value) as u64;
```

```rust
// demo-wallet/src/ethereum/commands.rs:58-63
#[derive(Args)]
pub struct SendEvmWalletArgs {
    #[arg(long, help = "Amount of ETH to transfer")]
    pub amount: f64,
    // ...
}
```

**Context**

- Production-grade wallets typically use fixed-point or integer representations for currency amounts to avoid rounding surprises.
- While this is demo code, copying this pattern into production code can cause subtle mis-amount transfers and complicate auditing.

**Remediation**

- Replace `f32`/`f64` amount fields with integer or fixed-point types (for example, `u64` satoshis, `U256` wei, or domain-specific decimal types).
- Perform explicit range checks and validation before constructing transactions.

---

### F-006: Configuration – Debug Rocket Profile Listens on All Interfaces Without TLS

| Attribute | Value |
|-----------|-------|
| **Severity** | LOW |
| **Location** | [`gotham-server/Rocket.toml:1-5`](../../gotham-server/Rocket.toml#L1-L5) |

**Issue**

The `Rocket.toml` configuration defines a `[debug]` profile that binds to `0.0.0.0:8000` without TLS.
If this debug configuration is used (or copied) in production or on a shared development host, the signing API can become reachable from unintended networks over plain HTTP.

**Evidence**

```toml
# gotham-server/Rocket.toml:1-5
[debug]
address = "0.0.0.0"
port = 8000
keep_alive = 5
log = "normal"
```

**Remediation**

- Ensure that production deployments use a `[release]` profile or a reverse proxy with TLS termination and network-level access controls.
- Restrict debug profile usage to local development environments.
- Document recommended deployment patterns (for example, binding to `127.0.0.1` behind a reverse proxy).

---

### F-007: Dependencies – Lack of Automated Dependency Risk Management

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **Location** | [`Cargo.toml:9-26`](../../Cargo.toml#L9-L26), [`gotham-server/Cargo.toml:17-38`](../../gotham-server/Cargo.toml#L17-L38), [`demo-wallet/Cargo.toml:7-31`](../../demo-wallet/Cargo.toml#L7-L31), [`integration-tests/Cargo.toml:7-15`](../../integration-tests/Cargo.toml#L7-L15) |

**Issue**

The workspace relies on several security-sensitive dependencies (Rocket, reqwest, RocksDB, cryptographic crates) but does not include any documented or automated dependency risk management (for example, `cargo audit` in CI, explicit policy for updating git-based dependencies like `two-party-ecdsa` and `gotham-engine`).
At the time of this audit, Snyk reports an out-of-bounds read issue for `rocksdb` versions `<0.19.0` (CVE score 5.9, `SNYK-RUST-ROCKSDB-2980271`), which does not affect the current 0.21.0 version, but the project lacks guardrails to catch future advisories.

**Remediation**

- Integrate `cargo audit` (or equivalent) into CI to detect new CVEs.
- Pin git dependencies (`two-party-ecdsa`, `gotham-engine`) to reviewed tags or commit hashes and document upgrade procedures.
- Establish a regular dependency review cadence and document supported versions.

---

### F-008: Logging & Observability – Lack of Structured Audit Logging for Signing Operations

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **Location** | [`gotham-server/src/public_gotham.rs:56-99`](../../gotham-server/src/public_gotham.rs#L56-L99), [`gotham-server/src/server.rs:21-39`](../../gotham-server/src/server.rs#L21-L39) |

**Issue**

The server-side components that persist MPC state and orchestrate keygen/sign operations (`PublicGotham` and Rocket routes via `gotham-engine`) do not emit structured, security-focused audit logs.
Insert and get operations into RocksDB occur without logging of key identifiers, operation types, or authorization decisions, making it difficult to reconstruct events during incident response.

**Remediation**

- Add structured logs (including operation type, customer/session identifiers, and authorization decision outcomes) around keygen and signing flows.
- Ensure that logs avoid leaking sensitive key material while still providing enough detail for forensics.
- Consider integrating with a centralized logging system and setting retention and access policies.

---

## Dependency Analysis

### Vulnerable Dependencies

At the time of this audit:

- `rocksdb` 0.21.0 is **not** affected by Snyk advisory `SNYK-RUST-ROCKSDB-2980271` (which targets versions `<0.19.0`).
- No specific CVEs were confirmed for the exact versions in use via automated tools within this session.

However, the project depends on:

| Package | Type | Current | Notes | Finding(s) |
|---------|------|---------|-------|------------|
| `reqwest` | Direct workspace dependency | 0.9.5 | Older HTTP client version on the critical path for MPC protocol messages | F-007 |
| `rocket` | Direct workspace dependency | 0.5.0-rc.1 | Pre-1.0 web framework; relies on correct TLS and configuration for secure deployment | F-006, F-007 |
| `rocksdb` | Direct dependency | 0.21.0 | Embedded key-value store for key shares; not encrypted at rest in application code | F-002 |
| `two-party-ecdsa` | Git dependency | branch `compatibility_gotham_engine` | Core cryptographic protocol implementation; treated as trusted external component | F-007 |
| `gotham-engine` | Git dependency | GitHub master | Route and protocol glue for the MPC server; treated as trusted external component | F-007 |

### Lockfile Integrity and Next Steps

- Run `cargo audit` against the workspace to obtain a current view of dependency CVEs.
- For any future advisories affecting these crates, ensure that vulnerabilities are:
  - Confirmed as reachable via code paths in this project.
  - Mapped to F-IDs and remediations in future audits, following the Evidence Standards in the main prompt.

### External References & Standards

- Snyk advisory `SNYK-RUST-ROCKSDB-2980271` (Out-of-bounds Read in `rocksdb` <0.19.0) – used to confirm that the current version (0.21.0) is not in the vulnerable range.
- OWASP Top 10 and CWE Top 25 – used for classification and mapping in `Recommendations.md`.
