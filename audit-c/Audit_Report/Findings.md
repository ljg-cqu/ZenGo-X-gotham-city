# Audit Findings & Dependency Analysis

## Finding Summary

This FULL-tier audit of the `ZenGo-X/gotham-city` repository focused on the core 2P-ECDSA signing path and key management surfaces:

- Gotham server HTTP API and RocksDB-backed key-share storage.
- Gotham client library (Rust + FFI bindings).
- Demo wallet (Bitcoin and EVM flows), including escrow backup.
- Integration tests validating end-to-end 2P-ECDSA signing.

Within that scope we identified:

- 1 CRITICAL authorization issue on the server (`Db::granted` always returns `true`).
- 2 HIGH data-security issues (plaintext storage of server and client key shares).
- 2 INFO-level hygiene/operational gaps (lack of automated dependency auditing and rate limiting / DoS protection).

Because the codebase is larger than the slices reviewed and `CoverageMode = COMPREHENSIVE`, **this audit is labeled _"Comprehensive coverage incomplete"_**. Many non-essential files (for example, additional ECDSA modules and utilities) were not fully reviewed.

**Findings by Severity**:
- CRITICAL: F-001
- HIGH: F-002, F-003
- MEDIUM: (none)
- LOW: (none)
- INFO: F-004, F-005

---

## Findings (Detailed)

### F-001: Auth/AuthZ – Default allow-all authorization on signing and keygen endpoints

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 9.1 |
| **CVSS Vector** | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:L |
| **CWE** | [CWE-306: Missing Authentication for Critical Function](https://cwe.mitre.org/data/definitions/306.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:92-95`](../../gotham-server/src/public_gotham.rs#L92-L95) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 1–2 days (design + implementation + tests) |

**Vulnerability Description**

The `Db::granted` authorization hook in the Gotham server always returns `Ok(true)` and is used by the `gotham-engine` routes to decide whether a given customer is allowed to perform key generation and signing. In its default configuration, **all requests are authorized**, regardless of caller identity, transaction parameters, or rate.

**Vulnerable Code**

```rust
// gotham-server/src/public_gotham.rs:92-95
/// the granted function implements the logic of tx authorization. If no tx authorization is needed the function returns always true
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)
}
```

**Attack Vector**

- An attacker with network access to the Gotham server (for example, `http://host:8000`) can:
  - Initiate 2P-ECDSA key generations via `/ecdsa/keygen/*` routes.
  - Request 2P-ECDSA signatures via `/ecdsa/sign/*` routes.
- Because `granted()` never denies a request, **any caller** can:
  - Obtain key material indirectly by driving protocol state evolution.
  - Request signatures over arbitrary messages (for example, transaction digests), so long as the higher-level application exposes those flows.
- If the server is exposed beyond a tightly controlled internal network, this becomes a remotely exploitable issue with no authentication barrier.

**Proof of Concept**

Conceptually (using the integration-test flow as a model):

```text
1. POST /ecdsa/keygen/first
2. POST /ecdsa/keygen/{id}/second
3. POST /ecdsa/keygen/{id}/third
4. POST /ecdsa/keygen/{id}/fourth
5. POST /ecdsa/keygen/{id}/chaincode/first
6. POST /ecdsa/keygen/{id}/chaincode/second
7. POST /ecdsa/sign/{id}/first
8. POST /ecdsa/sign/{id}/second
```

At no point is the caller authenticated or authorized. Any script that can reach the HTTP port can drive this sequence and obtain valid ECDSA signatures, subject only to protocol correctness.

**Impact**

- **Confidentiality**: If the server is wired into higher-level wallet or custody flows, an attacker can cause signatures over attacker-chosen messages, potentially exfiltrating or moving funds.
- **Integrity**: Unauthorized signing breaks transaction-approval policies and business rules (for example, amount limits, whitelists).
- **Availability**: Attackers can also abuse signing as a CPU-intensive operation, contributing to DoS.
- **Scope**: All tenants / customers sharing the same Gotham server instance are affected when the instance is deployed without a custom `granted()` implementation and network isolation.

**Remediation**

```rust
// Example sketch – final design should follow project auth architecture
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    // 1. Extract and validate bearer token (or mTLS identity) from request context.
    // 2. Map token → customer identity and authorization policy (limits, whitelists).
    // 3. Enforce per-customer rules for this message type and amount.
    // 4. Return Ok(true) only when the request is explicitly authorized.

    // Placeholder: production deployments must NOT ship with unconditional Ok(true).
    Err(DatabaseError::AuthorizationRequired)
}
```

- Implement `granted()` as a **mandatory policy enforcement point** for:
  - Key-generation flows (`/ecdsa/keygen/*`).
  - Signing flows (`/ecdsa/sign/*`).
- Require callers to present a verifiable identity (JWT, mTLS, API key) and enforce per-tenant authorization and rate limiting.
- Ensure the default build for any production deployment fails closed (deny by default) rather than `Ok(true)`.

**Additional Notes**

- This behavior is documented as a risk in [`wiki/Security.md`](../../wiki/Security.md#L96-L123), but the code-level default remains insecure.
- Any production deployment must either:
  - Replace `PublicGotham` with a hardened implementation, or
  - Override `Db::granted` to implement real authorization and remove the unconditional `Ok(true)` default.

---

### F-002: Data Security – Server key shares stored unencrypted in RocksDB

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.5 |
| **CVSS Vector** | AV:L/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:N |
| **CWE** | [CWE-311: Missing Encryption of Sensitive Data](https://cwe.mitre.org/data/definitions/311.html) |
| **Location** | [`gotham-server/src/public_gotham.rs:14-16`](../../gotham-server/src/public_gotham.rs#L14-L16), [`gotham-server/src/public_gotham.rs:58-87`](../../gotham-server/src/public_gotham.rs#L58-L87) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 3–5 days (design + implementation + migration) |

**Issue**

`PublicGotham` persists protocol state and key-share material to a local RocksDB instance on the filesystem without any encryption at rest. The data is written as JSON-serialized `Value` objects representing MPC protocol state, which includes sensitive key-share information.

**Relevant Code**

```rust
// gotham-server/src/public_gotham.rs:14-16,35-43
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
async fn insert(&self, key: &DbIndex, table_name: &dyn MPCStruct, value: &dyn Value)
    -> Result<(), DatabaseError> {
    let identifier = idify(key.clone().customerId, key.clone().id, table_name);
    let v_string = serde_json::to_string(&value).unwrap();
    let _ = self.rocksdb_client.put(identifier, v_string.clone());
    Ok(())
}
```

There is no encryption layer, key wrapping, or separation of duties between application code and storage of long-lived key shares.

**Context**

- [`wiki/Security.md`](../../wiki/Security.md#L85-L90) explicitly notes that server key shares are stored in RocksDB in plaintext and flags this as a high-risk area.
- `Settings.toml` defaults to a local DB type (`db = "local"`) and does not introduce encryption at rest or HSM integration.

**Impact**

- An attacker who obtains filesystem or volume access (for example, via OS compromise, container breakout, or backup leak) can:
  - Read serialized protocol state and key shares from RocksDB.
  - Combine these with a compromised client share to reconstruct full private keys.
- This undermines the core security goal of 2P-ECDSA: that no single compromise (client or server) suffices to obtain the effective private key.

**Remediation**

- Introduce a **storage abstraction** for key-share material that supports:
  - Encryption at rest (for example, with a KMS-managed master key).
  - Pluggable backends (RocksDB, HSM-backed stores, cloud KMS, etc.).
- Encrypt all values before writing to RocksDB, and decrypt only in-memory just before use.
- Ensure encryption keys are:
  - Not stored on the same filesystem as the encrypted RocksDB data.
  - Rotated according to a defined key-management policy.
- For high-assurance deployments, prefer hardware-backed key storage (HSM or TEE) over filesystem-based RocksDB.

**Additional Notes**

- The alphanumeric-only `db_name` validation prevents trivial path traversal (for example, `../secrets`), which is positive, but does not mitigate plaintext-at-rest risks.
- Migration of existing deployments requires a one-time re-encryption or re-keying process and must be planned carefully to avoid key loss.

---

### F-003: Data Security – Client wallet and escrow secrets stored unencrypted on disk

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **CVSS Score** | 7.1 |
| **CVSS Vector** | AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N |
| **CWE** | [CWE-922: Insecure Storage of Sensitive Information in a File or Directory](https://cwe.mitre.org/data/definitions/922.html) |
| **Location** | [`demo-wallet/src/bitcoin/escrow.rs:32-37`](../../demo-wallet/src/bitcoin/escrow.rs#L32-L37), [`demo-wallet/src/bitcoin/escrow.rs:42-45`](../../demo-wallet/src/bitcoin/escrow.rs#L42-L45), [`demo-wallet/src/bitcoin/mod.rs:143-167`](../../demo-wallet/src/bitcoin/mod.rs#L143-L167), [`demo-wallet/src/bitcoin/mod.rs:245-260`](../../demo-wallet/src/bitcoin/mod.rs#L245-L260) |
| **Status** | Open |
| **Priority** | Short-term |
| **Effort** | 3–5 days (design + implementation + testing across platforms) |

**Issue**

The demo Bitcoin wallet and its escrow component persist sensitive key material (escrow secret keys and MPC master keys / private shares) as **plaintext JSON files** on the local filesystem. There is no encryption, OS-provided secure storage, or file-permission hardening.

**Relevant Code**

```rust
// demo-wallet/src/bitcoin/escrow.rs:32-37
pub fn new(path: &str) -> Escrow {
    let secret: FE = ECScalar::new_random();
    let g: GE = ECPoint::generator();
    let public: GE = g * secret;
    fs::write(path, serde_json::to_string(&(secret, public)).unwrap())
        .expect("Unable to save escrow secret!");

    Escrow { secret, public }
}

// demo-wallet/src/bitcoin/mod.rs:143-167 (backup)
pub fn backup(&self, escrow_service: escrow::Escrow, path: &str) {
    let g: GE = ECPoint::generator();
    let y = escrow_service.get_public_key();
    let (segments, encryptions) = self.private_share.master_key.private.to_encrypted_segment(
        &escrow::SEGMENT_SIZE,
        escrow::NUM_SEGMENTS,
        &y,
        &g,
    );

    let proof = Proof::prove(&segments, &encryptions, &g, &y, &escrow::SEGMENT_SIZE);

    let client_backup_json = serde_json::to_string(&(
        encryptions,
        proof,
        self.private_share.master_key.public.clone(),
        self.private_share.master_key.chain_code.clone(),
        self.private_share.id.clone(),
    ))
    .unwrap();

    fs::write(path, client_backup_json).expect("Unable to save client backup!");
}

// demo-wallet/src/bitcoin/mod.rs:245-260 (wallet save/load)
pub fn save_to(&self, path: &str) {
    let wallet_json = serde_json::to_string_pretty(self).unwrap();
    fs::write(path, wallet_json).expect("Unable to save wallet!");
}

pub fn load_from(path: &str) -> BitcoinWallet {
    let data = fs::read_to_string(path).expect("Unable to load wallet!");
    let wallet: BitcoinWallet = serde_json::from_str(&data).unwrap();
    wallet
}
```

**Impact**

- Theft or leakage of `wallet.json`, `escrow-bitcoin.json`, or backup files allows an attacker with filesystem access to:
  - Reconstruct key shares, escrow secrets, and derived private keys.
  - Combine compromised client files with server-side compromise (or vice versa) to obtain the full effective private key.
- On shared or poorly protected hosts, malware or other users can trivially copy these JSON files.

**Remediation**

- Use platform-appropriate secure storage for long-lived secrets:
  - Desktop/server: OS keyring, encrypted filesystem, or dedicated key vault.
  - Mobile: Secure Enclave / Keychain (iOS), Keystore (Android).
- For file-based backups:
  - Encrypt backup files using a strong symmetric cipher (for example, AES-256-GCM) with a user-supplied passphrase or device-bound key.
  - Store only ciphertext and necessary metadata on disk, not raw MPC state.
- Harden file permissions and locations:
  - Restrict access to the wallet and escrow files to the owning user (for example, `chmod 600`).
  - Store backups in locations intended for sensitive data (not arbitrary working directories).

**Additional Notes**

- This is consistent with the wiki’s characterization that client key-share protection is an application responsibility, but in practice many users will adopt the demo wallet as-is. The code should either:
  - Ship with secure defaults, or
  - Make the risk explicit in the CLI help and documentation and gate unsafe modes behind flags.

---

### F-004: Security Hygiene – No automated dependency vulnerability scanning

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **Location** | [`Cargo.toml`](../../Cargo.toml), [`.github/`](../../.github/) |

The workspace uses multiple security-sensitive dependencies (for example, `rocket`, `reqwest`, `secp256k1`, `two-party-ecdsa`, `gotham-engine`), but the repository does not include any automated dependency scanning or `cargo audit` invocation in CI (the `.github/` directory is currently empty).

This is not a direct vulnerability on its own, but it increases the risk that known-critical CVEs in transitive crates remain unnoticed.

**Recommended Action**

- Add a CI job that runs `cargo audit` (or an equivalent Rust-focused dependency scanner) on all workspace members.
- Gate releases on a clean vulnerability report or explicit, documented risk acceptance for any remaining findings.

---

### F-005: Availability – No rate limiting or DoS controls on expensive ECDSA endpoints

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **Location** | [`gotham-server/src/server.rs:21-38`](../../gotham-server/src/server.rs#L21-L38), [`gotham-server/src/tests.rs:17-158`](../../gotham-server/src/tests.rs#L17-L158) |

The Gotham server exposes CPU-intensive multi-round 2P-ECDSA keygen and signing endpoints (`/ecdsa/keygen/*`, `/ecdsa/sign/*`) without any built-in rate limiting or request throttling. The integration tests demonstrate that these endpoints can be called in tight loops.

While this is acceptable for local testing and research use, any production deployment that exposes the server to untrusted networks should implement rate limiting, per-customer quotas, and monitoring to prevent intentional or accidental denial-of-service.

**Recommended Action**

- Introduce rate limiting (per IP and per authenticated principal) at the ingress layer (reverse proxy, API gateway) or within Rocket via request guards.
- Instrument and monitor request rates and latencies for keygen/signing routes.

---

## Dependency Analysis

This audit did **not** perform a full CVE-level dependency analysis (for example, via `cargo audit`). However, we note:

- The workspace depends on several security-critical crates:
  - `rocket` 0.5.0-rc.1 (pre-release web framework).
  - `secp256k1` 0.21.0 and `two-party-ecdsa` (cryptographic primitives and MPC protocol implementation).
  - `reqwest` 0.9.5 (HTTP client, relatively old version).
  - `gotham-engine` (external MPC engine implementation).
- The repository contains no automated dependency scanning or CI configuration.

**Policy for this audit**:

- We do **not** report individual CVEs as findings without code-verified exploitability and version confirmation.
- Instead, we surface the lack of automated scanning as **F-004 (INFO)** and recommend:
  - Running `cargo audit` regularly.
  - Reviewing any HIGH/CRITICAL issues against actual usage sites in the code.

## External References & Standards

- [OWASP Top 10](https://owasp.org/Top10/) – used for high-level classification of auth, data security, and configuration issues.
- [CWE-306](https://cwe.mitre.org/data/definitions/306.html) – missing authentication for critical functions.
- [CWE-311](https://cwe.mitre.org/data/definitions/311.html) – missing encryption of sensitive data.
- [CWE-922](https://cwe.mitre.org/data/definitions/922.html) – insecure storage of sensitive information in files.
- [Lindell 2017 – Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552) – background for protocol security claims (referenced via wiki, not audited here).
