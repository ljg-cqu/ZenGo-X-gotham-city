# Architecture and Attack Surface (Round 2)

## System Overview (Carried from R1)

Gotham City is a two-party ECDSA signing system consisting of a server (Party 1) and a client (Party 2). It implements the Lindell 2017 threshold ECDSA protocol via the `two-party-ecdsa` crate, with:

- A Rocket-based Rust server exposing HTTP JSON APIs for MPC key generation and signing.
- Rust clients (CLI and library) that participate in MPC rounds and persist wallet state locally.
- Demo wallets for Bitcoin and Ethereum that integrate MPC signing and local key-storage/escrow logic.

### Core Components

1. **Gotham Server (Party 1)**
   - Rust/Rocket web server.
   - Stores Party 1 key shares and MPC state in RocksDB.
   - Exposes REST-style endpoints for keygen/signing (`/ecdsa/keygen/*`, `/ecdsa/sign/*`).
   - Trust boundary: public or semi-public network surface.

2. **Gotham Client (Party 2)**
   - Rust library used by CLI and mobile bindings.
   - Orchestrates MPC rounds (keygen, sign, recover, rotate) with `gotham-server`.
   - Persists Party 2 key shares in local wallet files.
   - Trust boundary: end-user device or application host.

3. **Demo Wallets (Bitcoin and Ethereum)**
   - Bitcoin wallet: UTXO management, address derivation, and MPC-based transaction signing.
   - Ethereum wallet: EIP-155 transaction signing and ERC-20 transfers via a custom `GothamSigner`.
   - Both store MPC key shares and auxiliary data in JSON files.

4. **Escrow and Recovery**
   - Bitcoin escrow module (`demo-wallet/src/bitcoin/escrow.rs`) generates an additional secret and public key pair to support backup and recovery.
   - Recovery logic in `gotham-client/src/ecdsa/recover.rs` reconstructs secrets from encrypted segments and escrow material.

## High-Level Data Flow

```mermaid
graph TD
    User[User] -->|CLI or Mobile| Client[Gotham Client / Demo Wallet]
    Client -->|HTTP JSON (MPC rounds)| Server[Gotham Server]
    Server -->|Read/Write| DB[(RocksDB)]

    subgraph "Client Host"
      Client
      WalletFiles[[Wallet JSON / Escrow JSON]]
    end

    subgraph "Server Host"
      Server
      DB
    end

    style Server fill:#ffcccc,stroke:#333,stroke-width:2px
    style DB fill:#ffcccc,stroke:#333,stroke-width:2px
    style Client fill:#ccffcc,stroke:#333,stroke-width:2px
    style WalletFiles fill:#fff2cc,stroke:#333,stroke-width:1px
```

## R2 Focus: Key Storage and Escrow

Round 2 does not change the architecture but **clarifies where long-lived secrets live** and how they relate to ProblemList 067 (MPC key-shard persistence and encrypted storage design risks).

### Server-Side Storage

- Location: `gotham-server/src/public_gotham.rs`.
- Risk (R1-F-002): RocksDB stores serialized MPC state and key shares as plaintext JSON.
- R2 status: unchanged; still a single point of compromise for Party 1 shares.

### Client Wallet Storage

- Bitcoin wallet storage in `demo-wallet/src/bitcoin/mod.rs` persists `BitcoinWallet` as JSON (including `private_share`).
- Ethereum wallet storage in `demo-wallet/src/ethereum/mod.rs` persists `GothamWallet` via `serde_json::to_writer_pretty`, including `private_share` and HD path.
- Risk (R1-F-003): Party 2 key shares are exposed in plaintext.
- R2 status: clarified to include both Bitcoin and Ethereum wallets.

### Escrow Secret Storage (New R2 Emphasis)

- Location: `demo-wallet/src/bitcoin/escrow.rs`.
- The `Escrow::new` function generates a random secret `FE` and corresponding public key `GE`, then writes `(secret, public)` directly to a JSON file on disk.
- This file is intended to support backup and recovery flows, but it is stored unencrypted alongside application data.
- New risk (R2-F-001): plaintext escrow secret further undermines separation of duties and recovery security.

## Attack Surface Summary (Multi-Round)

1. **Network Surface**
   - Rocket HTTP endpoints (`/ecdsa/keygen/*`, `/ecdsa/sign/*`) over cleartext HTTP.
   - No application-level rate limiting or abuse detection is visible in the code paths reviewed.

2. **Server Host Surface**
   - File-system access to RocksDB database exposes all Party 1 key shares in plaintext.
   - Process crashes from panics (`unwrap`/`expect`) can be triggered by malformed inputs or corrupted state.

3. **Client Host Surface**
   - Wallet JSON files (Bitcoin and Ethereum) expose Party 2 key shares.
   - Escrow JSON file (`escrow-sk.json` or similar) stores additional high-value secrets unencrypted.
   - Malware or local attackers can combine these artifacts to reconstruct keys.

4. **Dependency and Supply-Chain Surface**
   - Outdated `rocket`, `reqwest`, `jsonwebtoken`, and `secp256k1` crates.
   - Git dependencies `gotham-engine` and `two-party-ecdsa` are core to MPC, but their internals are not audited here.

## Key Takeaways for Architects

- The **logical architecture** (two-party ECDSA with server/client split) is sound in principle, but the **operationalization** of storage and authorization is weak.
- Long-lived secrets (Party 1 shares, Party 2 shares, escrow secrets) are all stored in plaintext within their respective trust boundaries, without defense in depth.
- Escrow, which should strengthen recovery posture, currently introduces another critical plaintext secret; it must be re-architected to use hardened storage (HSMs, OS keychain, or encrypted blobs with strong KDFs) and clearer trust-domain separation.
- Any remediation plan should treat server storage, client wallet storage, and escrow storage as a **single, coherent key-management problem**, not as isolated issues.
