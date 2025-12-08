# Findings

## Summary

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| R1-F-001 | CRITICAL | Authorization Bypass in `granted` Function | Open |
| R1-F-002 | CRITICAL | Insecure Key Storage in RocksDB (Plaintext) | Open |
| R1-F-003 | CRITICAL | Insecure Key Storage in Client Wallet File (Plaintext) | Open |
| R1-F-004 | HIGH | Missing Encryption in Transit (HTTP) | Open |
| R1-F-005 | MEDIUM | Denial of Service via Panic on Invalid Input | Open |
| R1-F-006 | MEDIUM | Outdated and Vulnerable Dependencies | Open |

---

## R1-F-001: Authorization Bypass in `granted` Function

**Severity**: CRITICAL (CVSS: 9.8)  
**Status**: Open  
**CWE**: CWE-285: Improper Authorization  
**ProblemList**: [044_Backend_Server_Infrastructure_Operational_Security.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/044_Backend_Server_Infrastructure_Operational_Security.md)

**Description**  
The `granted` function in `gotham-server/src/public_gotham.rs` is intended to implement transaction authorization logic. However, it is hardcoded to always return `true`, effectively bypassing any authorization checks. This allows any user (even if authenticated via other means, though auth seems weak) to authorize any transaction or operation that relies on this check.

**Impact**  
An attacker who can communicate with the server can bypass authorization controls, potentially allowing them to sign unauthorized transactions or access sensitive operations. In an MPC context, this could lead to unauthorized usage of key shares.

**Recommendation**  
Implement proper authorization logic in `granted`. Verify that the `customer_id` matches the owner of the resource and that the `message` (transaction) is authorized by policy.

**Location**  
`gotham-server/src/public_gotham.rs:93`
```rust
    /// the granted function implements the logic of tx authorization. If no tx authorization is needed the function returns always true
    fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
        Ok(true)
    }
```

---

## R1-F-002: Insecure Key Storage in RocksDB (Plaintext)

**Severity**: CRITICAL (CVSS: 9.3)  
**Status**: Open  
**CWE**: CWE-312: Cleartext Storage of Sensitive Information  
**ProblemList**: [067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md)

**Description**  
The server stores key shares and other sensitive values in RocksDB using `serde_json::to_string`. There is no encryption applied to the data before storage. If the server's file system is compromised, all key shares are exposed in plaintext.

**Impact**  
Compromise of the server's storage leads to immediate theft of all Party1 key shares. Combined with a client compromise or collusion, this allows full private key reconstruction and theft of assets.

**Recommendation**  
Implement encryption at rest for all sensitive data stored in RocksDB. Use a strong encryption scheme (e.g., AES-GCM) with keys managed by a separate KMS or HSM.

**Location**  
`gotham-server/src/public_gotham.rs:65`
```rust
        let v_string = serde_json::to_string(&value).unwrap();
        let _ = self.rocksdb_client.put(identifier, v_string.clone());
```

---

## R1-F-003: Insecure Key Storage in Client Wallet File (Plaintext)

**Severity**: CRITICAL (CVSS: 9.3)  
**Status**: Open  
**CWE**: CWE-312: Cleartext Storage of Sensitive Information  
**ProblemList**: [067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md)

**Description**  
The `demo-wallet` saves the wallet state, including the client's private share, to a JSON file (`wallet.json`) in plaintext. `serde_json::to_string_pretty` is used without any encryption.

**Impact**  
Malware or an attacker with local access to the client machine can easily steal the Party2 key share.

**Recommendation**  
Encrypt the wallet file using a user-provided password or system keychain. Do not store private shares in plaintext JSON.

**Location**  
`demo-wallet/src/bitcoin/mod.rs:246`
```rust
    pub fn save_to(&self, path: &str) {
        let wallet_json = serde_json::to_string_pretty(self).unwrap();
        fs::write(path, wallet_json).expect("Unable to save wallet!");
```

---

## R1-F-004: Missing Encryption in Transit (HTTP)

**Severity**: HIGH (CVSS: 7.5)  
**Status**: Open  
**CWE**: CWE-319: Cleartext Transmission of Sensitive Information  
**ProblemList**: [044_Backend_Server_Infrastructure_Operational_Security.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/044_Backend_Server_Infrastructure_Operational_Security.md)

**Description**  
The server is configured to listen on HTTP (port 8000) without TLS. The `Rocket.toml` configuration does not specify any TLS settings. While the documentation mentions HTTPS is an "Application responsibility", the default configuration is insecure.

**Impact**  
Man-in-the-Middle (MitM) attackers can intercept MPC protocol messages. While the MPC protocol might have some internal security, metadata and potentially sensitive values (like auth tokens) are exposed.

**Recommendation**  
Configure Rocket to use TLS (HTTPS) in production. Provide a `Rocket.toml` with TLS settings or use a reverse proxy (Nginx) with SSL termination.

**Location**  
`gotham-server/Rocket.toml:1`
```toml
[debug]
address = "0.0.0.0"
port = 8000
```

---

## R1-F-005: Denial of Service via Panic on Invalid Input

**Severity**: MEDIUM (CVSS: 5.3)  
**Status**: Open  
**CWE**: CWE-248: Uncaught Exception  
**ProblemList**: [086_API_Rate_Limiting_DDoS_Protection_MPC_Signing_Services.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/086_API_Rate_Limiting_DDoS_Protection_MPC_Signing_Services.md)

**Description**  
The server uses `unwrap()` and `expect()` on results from database operations and JSON deserialization. If the database contains invalid data (corruption) or if a request triggers a path that fails an unwrap, the server thread or process may panic, causing a Denial of Service.

**Impact**  
An attacker might be able to trigger a panic by sending malformed data or exploiting race conditions, crashing the server and making the signing service unavailable.

**Recommendation**  
Replace `unwrap()` and `expect()` with proper error handling (e.g., `match` or `?` operator) and return appropriate HTTP error codes (500 or 400) instead of crashing.

**Location**  
`gotham-server/src/public_gotham.rs:86`
```rust
                .unwrap();
```

---

## R1-F-006: Outdated and Vulnerable Dependencies

**Severity**: MEDIUM (CVSS: 5.0)  
**Status**: Open  
**CWE**: CWE-1104: Use of Unmaintained Third Party Components  
**ProblemList**: [039_Cryptographic_Library_Supply_Chain_Attacks.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/039_Cryptographic_Library_Supply_Chain_Attacks.md)

**Description**  
The project uses significantly outdated dependencies:
- `reqwest` 0.9.5 (Current: 0.11+)
- `rocket` 0.5.0-rc.1 (Current: 0.5.0 stable or newer)
- `jsonwebtoken` 8 (Current: 9+)
- `secp256k1` 0.21.0 (Current: 0.27+)

These versions may contain known security vulnerabilities and lack modern security features.

**Impact**  
Exposure to known vulnerabilities in dependencies.

**Recommendation**  
Update all dependencies to their latest stable versions. Run `cargo audit` to identify specific vulnerabilities.

**Location**  
`Cargo.toml:15`
```toml
reqwest = "0.9.5"
```
