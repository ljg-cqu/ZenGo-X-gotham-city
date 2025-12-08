# Findings

**Repository**: `ZenGo-X-gotham-city`  
**Date**: 2025-12-08  

---

## F-001: Blind Signing in CLI Wallet

**Severity**: CRITICAL (CVSS: 9.0)  
**Problem Map**: `006_Blind_Signing_Vulnerability.md`  
**Location**: `demo-wallet/src/ethereum/commands.rs`, `demo-wallet/src/ethereum/mod.rs`

### Description
The CLI wallet (`demo-wallet`) constructs and signs transactions without displaying the critical transaction details to the user for verification. Specifically, in `transfer_erc20` and `send_transaction`, the transaction hash is signed immediately after construction. The user only sees the transaction hash *after* it has been sent to the network.

For ERC-20 transfers, the `data` field (which encodes the function call and arguments) is constructed programmatically but never shown to the user. A compromised or malicious client could swap the `to` address or `amount` (or `approve` unlimited allowance) without the user knowing.

### Evidence
-   `demo-wallet/src/ethereum/mod.rs:266`: `let receipt = contract_call.send().await?.await?.unwrap();` is called directly.
-   `demo-wallet/src/ethereum/mod.rs:267`: `println!("Transaction Hash: {:?}", receipt.transaction_hash);` prints hash after sending.
-   No `println!` or user confirmation prompt exists before `contract_call.send()`.

### Recommendation
1.  Decode the transaction payload (RLP decoding for ETH, ABI decoding for ERC-20).
2.  Display `To`, `Amount`, `Fee`, `Contract Address`, and `Function Call` (e.g., `transfer(to, amount)`) to the user.
3.  Require an explicit confirmation (e.g., "Type 'yes' to sign") before proceeding with the signing operation.

---

## F-002: Unencrypted Key Share Storage

**Severity**: CRITICAL (CVSS: 9.5)  
**Problem Map**: `067_MPC_Key_Shard_Persistence_and_Encrypted_Storage_Design_Risks.md`  
**Location**: `gotham-server/src/public_gotham.rs`

### Description
The `gotham-server` stores MPC key shares and state directly into a RocksDB database without any application-level encryption. The `PublicGotham` struct implements the `Db` trait, where the `insert` method serializes the value to JSON and writes it to RocksDB.

If the server's filesystem is compromised (e.g., via backup leakage, physical access, or OS vulnerability), the attacker can read all key shares in plaintext (or JSON format), leading to a complete compromise of the MPC wallet system.

### Evidence
-   `gotham-server/src/public_gotham.rs:65`: `let v_string = serde_json::to_string(&value).unwrap();`
-   `gotham-server/src/public_gotham.rs:66`: `let _ = self.rocksdb_client.put(identifier, v_string.clone());`
-   No encryption step is performed before `put`.

### Recommendation
1.  Implement a "Key Wrapping" strategy.
2.  Encrypt the `v_string` using a symmetric key (e.g., AES-GCM) before storing it in RocksDB.
3.  The encryption key should be managed securely, ideally derived from a Hardware Security Module (HSM) or a cloud KMS (e.g., AWS KMS) and never stored on the disk alongside the database.

---

## F-003: Missing API Rate Limiting

**Severity**: HIGH (CVSS: 7.5)  
**Problem Map**: `086_API_Rate_Limiting_DDoS_Protection_MPC_Signing_Services.md`  
**Location**: `gotham-server/src/server.rs`

### Description
The `gotham-server` exposes public API endpoints for MPC key generation and signing (e.g., `/keygen/first`, `/sign/first`) without any visible rate limiting middleware. MPC operations are computationally expensive (involving heavy cryptographic math).

An attacker can flood the server with requests, exhausting CPU and memory resources, causing a Denial of Service (DoS) for legitimate users. This is a known asymmetry in MPC systems where verification/signing is much more expensive than request generation.

### Evidence
-   `gotham-server/src/server.rs:23`: `rocket::Rocket::build()` mounts routes directly.
-   No rate limiting fairing or middleware (e.g., `rocket_governor`) is registered.

### Recommendation
1.  Integrate a rate limiting library (e.g., `rocket_governor` or a reverse proxy like Nginx/Cloudflare).
2.  Configure rate limits based on IP address or API token.
3.  Set stricter limits for computationally intensive endpoints (KeyGen, Sign) compared to lightweight endpoints.

---

## F-004: Implicit/Opaque Randomness Usage

**Severity**: HIGH (CVSS: 7.0)  
**Problem Map**: `053_Weak_Randomness_and_Entropy_in_MPC_Key_Generation_and_Signing.md`  
**Location**: `gotham-client/src/ecdsa/keygen.rs`, `gotham-client/src/ecdsa/sign.rs`

### Description
The client and server code rely on implicit randomness sources within the `two-party-ecdsa` dependency. Functions like `MasterKey2::key_gen_first_message()` and `MasterKey2::sign_first_message()` are called without passing an explicit Random Number Generator (RNG).

While the underlying library might use a secure default (like `OsRng`), the lack of explicit RNG management makes it difficult to audit and verify the quality of entropy. In high-security MPC contexts, it is best practice to explicitly pass a CSPRNG (Cryptographically Secure PRNG) to ensure that weak entropy sources (like user-space PRNGs or unseeded RNGs) are not used.

### Evidence
-   `gotham-client/src/ecdsa/keygen.rs:43`: `MasterKey2::key_gen_first_message()` called without RNG argument.
-   `gotham-client/src/ecdsa/sign.rs:37`: `MasterKey2::sign_first_message()` called without RNG argument.

### Recommendation
1.  Refactor the `two-party-ecdsa` library (if possible) or the wrapper code to accept an explicit RNG trait object (e.g., `rand::RngCore + rand::CryptoRng`).
2.  In the application layer, instantiate a `OsRng` or `ThreadRng` and pass it explicitly.
3.  Add tests to verify that the RNG is working as expected (e.g., statistical tests on outputs, though difficult for MPC messages).
