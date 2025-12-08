# Audit Findings & Dependency Analysis (Round R2)

## Finding Summary (Current Round R2 Only)

This file lists **only new findings from Round 2 (R2)**. Baseline findings from Round 1 (R1) are summarized separately below.

**Findings by Severity (R2):**
- CRITICAL: R2-F-001
- HIGH: *(none)*
- MEDIUM: *(none)*
- LOW: *(none)*
- INFO: *(none)*

Total new findings in R2: **1**.

---

## Findings (Detailed, Round R2)

### R2-F-001: Data Security – Insecure Escrow Secret Storage in Plaintext JSON

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **CVSS Score** | 8.8 |
| **CVSS Vector** | AV:L/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:N |
| **CWE** | [CWE-312: Cleartext Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/312.html) |
| **Location** | [`demo-wallet/src/bitcoin/escrow.rs:31-47`](../demo-wallet/src/bitcoin/escrow.rs#L31-L47) |
| **Status** | Open |
| **Priority** | Immediate |
| **Effort** | 1–2 days (design + implementation + testing) |
| **ProblemList** | [067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md](../../../../knowledge/Blockchain/Wallets/MPC/Problems/067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md) |

**Vulnerability Description**

The Bitcoin demo wallet’s escrow component generates a long-lived *escrow secret* and writes it **directly to disk in plaintext JSON**, together with its associated public key:

```rust
// demo-wallet/src/bitcoin/escrow.rs
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

The default CLI settings store this file under a user-supplied or default path such as `escrow-bitcoin.json` on the local filesystem. There is no encryption, key derivation, passphrase, or OS keychain integration protecting this material.

**Attack Vector**

- An attacker who gains local or malware-level access to the client machine can read the escrow secret file directly.
- The escrow secret participates in the backup and recovery workflow for the MPC key share. Combined with:
  - Existing plaintext wallet key-share files (R1-F-003), and/or
  - Access to server-side state (R1-F-002),

  the attacker can accelerate or fully enable reconstruction of the underlying ECDSA private key.
- Because the escrow file is stored as JSON, it is trivial to parse and reuse across tools.

In typical institutional MPC custody deployments, escrow or recovery secrets are expected to be held in **separate, hardened trust domains** (for example, HSMs, separate key-management systems, or physical custodians). Storing them unencrypted alongside the wallet application undermines that separation.

**Impact**

- **Confidentiality**: Disclosure of escrow secrets materially lowers the bar for full key reconstruction when combined with other compromised components.
- **Integrity**: Attackers with escrow + wallet + server access can sign arbitrary transactions that appear fully authorized.
- **Scope**: Affects any Bitcoin MPC wallet that uses this demo escrow implementation and stores escrow JSON on disk without additional controls.
- **Domain Impact**: Directly maps to ProblemList 067 (MPC key-shard persistence and encrypted storage design risks) by extending the set of unencrypted long-lived secrets.

**Remediation**

At minimum:

```rust
// High-level remediation sketch (not a drop-in patch):
// 1. Do NOT write raw (secret, public) to disk.
// 2. Encrypt escrow material with a strong key that is not stored alongside the wallet.
// 3. Prefer hardware-backed or OS keychain storage when available.

pub fn new(path: &str, encrypted_store: &mut dyn EscrowKeyStore) -> Escrow {
    let secret: FE = ECScalar::new_random();
    let g: GE = ECPoint::generator();
    let public: GE = g * secret;

    // Serialize in-memory only
    let escrow_blob = serde_json::to_vec(&(secret, public))
        .expect("Unable to serialize escrow secret");

    // Delegate encryption + persistence to a hardened keystore interface
    encrypted_store.store(path, &escrow_blob)
        .expect("Unable to save encrypted escrow secret");

    Escrow { secret, public }
}
```

Recommended design changes:

- Introduce an abstraction for **escrow secret storage** that can be backed by:
  - OS keychain / secure enclave on mobile.
  - Hardware tokens or HSMs for institutional deployments.
  - At minimum, AES-GCM (or similar) using a key derived from a user passphrase via a modern KDF (Argon2, scrypt) with per-device salt.
- Avoid sharing storage locations between wallet state and escrow secrets; treat them as **separate trust domains**.
- Document the threat model for escrow usage clearly so integrators understand that plaintext local storage is unsafe outside of controlled demos.

**Additional Notes**

- **Related Problems**: Extends R1’s coverage of ProblemList 067 beyond RocksDB and wallet JSON files to escrow material.
- **Related Findings**:
  - R1-F-002 (server RocksDB plaintext key shares).
  - R1-F-003 (wallet JSON plaintext key shares).
- **Verification**: File-system inspection on a test run confirms human-readable JSON escrow files are created without any encryption or OS-backed protections.

---

## Previously Reported Findings (Baseline from R1)

These findings were reported in **Round 1 (R1)** and remain **Open** in Round 2. They are not re-numbered here; instead, R2 references them as baseline.

| Previous F-ID | Location | Severity | Status in R2 | Notes |
|---------------|----------|----------|--------------|-------|
| R1-F-001 | `gotham-server/src/public_gotham.rs:93` | CRITICAL | Verified still present | `granted` continues to return `Ok(true)` unconditionally; no authorization policy added. |
| R1-F-002 | `gotham-server/src/public_gotham.rs:62-67` | CRITICAL | Verified still present | Server still writes MPC key-share state to RocksDB in plaintext JSON. |
| R1-F-003 | `demo-wallet/src/bitcoin/mod.rs:246-249` | CRITICAL | Verified still present (impact clarified) | Wallet JSON files (including EVM `GothamWallet` JSON) continue to store client key shares in plaintext; R2 confirms same pattern in `demo-wallet/src/ethereum/mod.rs:69-76`. |
| R1-F-004 | `gotham-server/Rocket.toml:1-5` | HIGH | Verified still present | Rocket still configured for HTTP on port 8000 with no TLS; demo wallet defaults to `http://127.0.0.1:8000`. |
| R1-F-005 | Multiple (`public_gotham.rs`, client ECDSA flows) | MEDIUM | Verified still present | `unwrap`/`expect` calls and lack of rate limiting remain; R2 reconfirms patterns but does not add new locations. |
| R1-F-006 | `Cargo.toml` workspace dependencies | MEDIUM | Verified still present | Dependency versions unchanged; no evidence of systematic upgrade or `cargo audit` integration.

R2 focuses on expanding the coverage of **ProblemList 067** (key-share and secret persistence risks) by identifying insecure escrow secret storage. All R1 findings continue to represent blocking issues for any production deployment.
