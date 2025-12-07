# Audit Findings & Dependency Analysis

## Finding Summary

This partial audit reviewed a narrow slice of gotham-city, focusing on `gotham-server` entrypoints, its RocksDB-backed `PublicGotham` implementation, selected configuration files, and a unit test that drives the HTTP MPC keygen/sign flows. No high- or medium-severity vulnerabilities were confirmed in this slice.

Three LOW/INFO-level issues were identified:

- Configuration hygiene in Rocket debug settings.
- A stubbed transaction-authorization hook in the `Db` implementation.
- Lack of documented at-rest protection for RocksDB-stored MPC state.

Because coverage is incomplete, these findings should be treated as **additive**, not exhaustive.

**Findings by Severity**:
- CRITICAL: —
- HIGH: —
- MEDIUM: —
- LOW: F-001, F-003
- INFO: F-002

---

## Findings (Detailed)

### F-001: Configuration – Rocket Debug Binding on 0.0.0.0

| Attribute | Value |
|-----------|-------|
| **Severity** | LOW |
| **Location** | [`gotham-server/Rocket.toml:1-5`](../../gotham-server/Rocket.toml#L1-L5) |

**Issue**

The debug Rocket configuration binds the server to `0.0.0.0:8000` with a normal logging level and no explicit separation between local/test and production profiles.

**Context**

For development and isolated test environments, binding to `0.0.0.0` is common. However, if the same configuration is reused in less-controlled environments (for example, exposed directly to the internet without a reverse proxy, TLS termination, or network ACLs), it increases the risk of unintended exposure of the MPC signing service.

**Remediation**

- Define and document distinct configuration profiles for local development, staging, and production.
- For production deployments:
  - Bind to an internal interface or rely on a reverse proxy / load balancer as the external entrypoint.
  - Ensure TLS termination, authentication, rate limiting, and monitoring are handled in front of Rocket.
  - Keep a clear, version-controlled `Rocket.toml` or equivalent configuration artifact per environment.

---

### F-002: Business Logic – Transaction Authorization Hook Stubbed Out

| Attribute | Value |
|-----------|-------|
| **Severity** | INFO |
| **Location** | [`gotham-server/src/public_gotham.rs:92-95`](../../gotham-server/src/public_gotham.rs#L92-L95) |

**Issue**

The `Db` implementation used by gotham-server (`PublicGotham`) defines a `granted` method that unconditionally returns `Ok(true)` and ignores its `message` and `customer_id` inputs.

```rust
// gotham-server/src/public_gotham.rs:92-95
/// the granted function implements the logic of tx authorization. If no tx authorization is needed the function returns always true
fn granted(&self, message: &str, customer_id: &str) -> Result<bool, DatabaseError> {
    Ok(true)
}
```

**Context**

- The inline comment indicates that `granted` is meant to encapsulate transaction-authorization logic when such logic is required.
- In this repository, the default implementation acts as a stub: regardless of message content or customer identity, it always signals that the operation is allowed.
- The actual invocation sites of `Db::granted` live in external engine crates (`gotham-engine`), which are not part of this repo and were not audited here. Therefore, this finding does **not** assert that there is an exploitable authorization vulnerability in a deployed environment.

**Remediation**

- Treat this implementation as a sample only; before any production deployment:
  - Replace the stub with concrete transaction-authorization logic aligned with business policy (for example, policy-based approvals, rate limits, risk scoring, and customer-specific controls).
  - Add tests that exercise both allowed and denied paths through whatever code calls `Db::granted`.
- Clearly document that the current implementation is not suitable for production and that deployments must supply their own hardened `Db`/authorization adapter.

---

### F-003: Data Security – RocksDB-Backed MPC State Without At-Rest Hardening

| Attribute | Value |
|-----------|-------|
| **Severity** | LOW |
| **Location** | [`gotham-server/src/public_gotham.rs:14-16`](../../gotham-server/src/public_gotham.rs#L14-L16), [`gotham-server/src/public_gotham.rs:35-43`](../../gotham-server/src/public_gotham.rs#L35-L43), [`gotham-server/src/public_gotham.rs:58-79`](../../gotham-server/src/public_gotham.rs#L58-L79) |

**Issue**

`PublicGotham` persists values implementing `two_party_ecdsa::party_one::Value` directly into a local RocksDB database without any documented encryption-at-rest, access-control layer, or data-retention policy.

```rust
// Struct and DB initialization
pub struct PublicGotham {
    rocksdb_client: rocksdb::DB,
}
...
let rocksdb_client = rocksdb::DB::open_default(format!("./{}", db_name)).unwrap();

// Insert and get methods
async fn insert(&self, key: &DbIndex, table_name: &dyn MPCStruct, value: &dyn Value) -> Result<(), DatabaseError> {
    let identifier = idify(key.clone().customerId, key.clone().id, table_name);
    let v_string = serde_json::to_string(&value).unwrap();
    let _ = self.rocksdb_client.put(identifier, v_string.clone());
    Ok(())
}

async fn get(&self, key: &DbIndex, table_name: &dyn MPCStruct) -> Result<Option<Box<dyn Value>>, DatabaseError> {
    let identifier = idify(key.clone().customerId, key.clone().id, table_name);
    let result = self.rocksdb_client.get(identifier.clone()).unwrap();
    ...
}
```

**Context**

- The `Db` abstraction is used to persist MPC-related state (for example, protocol transcripts and potentially partial secrets or derived key material) via JSON into RocksDB.
- This repository does not implement:
  - Encryption at rest for RocksDB contents.
  - OS-level access control, wiping, or key-rotation logic for the underlying database files.
- The exact sensitivity of the stored values depends on how `gotham-engine` and `two-party-ecdsa` structure and use `Value`. That code is external to this repo and was not audited here, so this finding is framed as a design/security-hardening gap rather than a confirmed exposure of specific secrets.

**Remediation**

- For any environment where RocksDB stores protocol state with security impact:
  - Consider enabling encryption at rest (via an encrypted filesystem, OS-level mechanisms, or an alternative encrypted storage backend).
  - Restrict filesystem permissions and enforce operational controls around backups and snapshots.
  - Define and implement data-retention and secure-deletion policies for MPC state that is no longer required.
- Alternatively, consider abstracting the `Db` interface over a hardened external datastore (for example, a managed database with built-in encryption and access controls) and providing a different `Db` implementation for production.

---

## Dependency Analysis

### Vulnerable Dependencies

This run did **not** perform automated CVE/RustSec scanning or dependency graph analysis. As a result:

- No specific vulnerable dependency instances are reported.
- No CVEs or CVSS scores are associated with packages in this partial audit.

Instead, the following qualitative observations were made:

- The workspace uses a number of third-party crates with potential security impact, including (non-exhaustive):
  - `rocket` (web framework)
  - `rocksdb` (local key-value store)
  - `redis` (clustered Redis client)
  - `serde` / `serde_json` (serialization)
  - `reqwest`, `jsonwebtoken`, `config`, `uuid`, among others
- Two critical components, `two-party-ecdsa` and `gotham-engine`, are pulled in as git dependencies and contain the cryptographic and MPC protocol logic that is central to system security.

A future audit should:

- Run a dependency scanner (for example, `cargo audit` or equivalent RustSec-aware tooling) across all workspace crates.
- For any high/critical issues found, map them to usage sites in code and promote them to F-IDs following the per-finding workflow.

### Unused Dependencies (No Vulnerabilities)

Not assessed in this run. Any future dependency-focused pass should also confirm which crates in `Cargo.toml` are actually used and distinguish real risk from unused, but declared, dependencies.

### Lockfile Integrity

- Lockfiles exist for some crates (for example, `gotham-server/Cargo.lock`, `integration-tests/Cargo.lock`), but their full contents were not reviewed.
- No CISA KEV or other known-exploited vulnerability checks were performed.

## External References & Standards

This partial audit used the following external references conceptually for classification and remediation guidance (no specific issues were mapped yet):

- **OWASP Top 10 (2021)** – General guidance on categories such as Security Misconfiguration and Cryptographic Failures.
- **CWE** – Conceptual backing for categories like Configuration issues and Data Security gaps.
- **RustSec / Cargo Audit ecosystem** – As recommended tooling for future dependency analysis (not yet applied in this run).

A complete audit should:

- Map concrete findings and dependency issues to explicit OWASP and CWE identifiers where appropriate.
- Use RustSec/Cargo audit outputs as input for dependency-related F-IDs.
