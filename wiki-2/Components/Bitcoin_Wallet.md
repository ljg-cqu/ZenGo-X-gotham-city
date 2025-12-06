# Bitcoin Wallet Component [P2]

**Location**: [`demo-wallet/src/bitcoin/`](../../demo-wallet/src/bitcoin/)  
**Priority**: P2 — Demonstration of 2P-ECDSA integration with Bitcoin network  
**Purpose**: CLI wallet implementation for Bitcoin using gotham-client and Electrum API  
**Design**: SegWit P2WPKH addresses with Electrum backend — chosen for modern Bitcoin compatibility and lightweight indexing  
**Limitations**: Demo-quality code; Electrum dependency for UTXO queries; single-address wallet (non-HD address rotation)

## Overview

The Bitcoin wallet demonstrates practical integration of 2P-ECDSA with the Bitcoin network. It supports wallet creation, balance queries, and transaction signing/broadcasting via Electrum servers.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#f8f9fa", "primaryTextColor": "#1a1a1a", "primaryBorderColor": "#7a8591", "lineColor": "#8897a8", "secondaryColor": "#eff6fb", "tertiaryColor": "#f3f5f7"}}}%%
flowchart LR
    subgraph CLI["CLI Commands"]
        CREATE[create]
        BALANCE[balance]
        SEND[send]
        BACKUP[backup]
    end
    subgraph Wallet["Wallet Logic"]
        KEYGEN[2P Keygen]
        SIGN[2P Sign]
        TX[Transaction Builder]
    end
    subgraph External["External Services"]
        GOTHAM[Gotham Server]
        ELECTRUM[Electrum Server]
        BTC[Bitcoin Network]
    end
    
    CREATE --> KEYGEN --> GOTHAM
    BALANCE --> ELECTRUM
    SEND --> TX --> SIGN --> GOTHAM
    SEND --> ELECTRUM
    ELECTRUM --> BTC
    
    classDef default fill:#f8f9fa,stroke:#7a8591,stroke-width:2px,color:#1a1a1a
```

## CLI Commands

| Command | Purpose | Evidence |
|---------|---------|----------|
| `bitcoin create` | Generate new 2P-ECDSA wallet | [`main.rs:76`](../../demo-wallet/src/main.rs#L76) |
| `bitcoin balance` | Query wallet balance via Electrum | CLI implementation |
| `bitcoin send` | Sign and broadcast Bitcoin transaction | CLI implementation |
| `bitcoin backup` | Create encrypted key backup | CLI implementation |

## Configuration

Settings from `settings.toml` or environment:

| Setting | Purpose | Default | Evidence |
|---------|---------|---------|----------|
| `electrum_server_url` | Electrum server endpoint | Required | [`main.rs:51`](../../demo-wallet/src/main.rs#L51) |
| `wallet_file` | Path to wallet JSON | `wallet.json` | [`main.rs:70-72`](../../demo-wallet/src/main.rs#L70-L72) |
| `gotham_server_url` | Gotham server endpoint | `http://127.0.0.1:8000` | [`main.rs:66-68`](../../demo-wallet/src/main.rs#L66-L68) |

## Transaction Flow

### Signing Process

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#f8f9fa", "primaryTextColor": "#1a1a1a", "primaryBorderColor": "#7a8591", "lineColor": "#8897a8", "secondaryColor": "#eff6fb", "tertiaryColor": "#f3f5f7"}}}%%
flowchart TD
    A[Query UTXOs] --> B[Select Inputs]
    B --> C[Build Transaction]
    C --> D{For Each Input}
    D --> E[Compute BIP143 SigHash]
    E --> F[2P-ECDSA Sign]
    F --> G[Build Witness]
    G --> D
    D -->|Done| H[Serialize Transaction]
    H --> I[Broadcast via Electrum]
    
    classDef default fill:#f8f9fa,stroke:#7a8591,stroke-width:2px,color:#1a1a1a
```

### Transaction Building

| Stage | Input | Transform | Output |
|-------|-------|-----------|--------|
| UTXO Query | Wallet addresses | Electrum `listunspent` | UTXO list with values |
| Selection | Amount, UTXOs | Greedy selection | Selected inputs |
| Build | UTXOs, recipient, change | BIP143 structure | Unsigned transaction |
| Sign | SigHash, master_key | 2P-ECDSA protocol | Witness data |
| Broadcast | Signed transaction | Electrum `broadcast` | TXID |

## Address Format

Uses **P2WPKH** (Pay-to-Witness-Public-Key-Hash) SegWit addresses:

- Prefix: `bc1` (mainnet) or `tb1` (testnet)
- Lower fees than legacy addresses
- BIP143 signature hash algorithm

## Key Derivation

BIP32-style hierarchical derivation:

```rust
// Derive child key for address index
let mk_child = master_key.get_child(vec![x_pos, y_pos]);
```

Where:
- `x_pos`: Account index (typically 0)
- `y_pos`: Address index

## Wallet Storage

Wallet data is stored as JSON:

```rust
struct Wallet {
    id: String,              // Session ID from keygen
    master_key: MasterKey2,  // Client's key share
    addresses: Vec<Address>, // Derived addresses
}
```

Storage location: `wallet_file` setting (default: `wallet.json`)

## Electrum Integration

The wallet uses Electrum protocol for:

| Operation | Electrum Method | Purpose |
|-----------|-----------------|---------|
| Balance query | `blockchain.scripthash.get_balance` | Get confirmed/unconfirmed balance |
| UTXO listing | `blockchain.scripthash.listunspent` | Get spendable outputs |
| Transaction broadcast | `blockchain.transaction.broadcast` | Submit signed transaction |
| Fee estimation | `blockchain.estimatefee` | Get fee rate |

## Key Backup (Centipede)

Encrypted backup using verifiable encryption:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Segment Size | 8 | Encryption granularity |
| Num Segments | 32 | Backup redundancy |

Evidence: Referenced in wiki-0 as `bitcoin/escrow.rs`

## Error Handling

| Error Type | Handling | Recovery |
|------------|----------|----------|
| Electrum connection | Return error | Retry or use different server |
| Insufficient funds | Return error | Add more UTXOs |
| Signing failure | Return error | Check server availability |
| Broadcast failure | Return error | Check transaction validity |

## Dependencies

| Crate | Purpose |
|-------|---------|
| `gotham-client` | 2P-ECDSA key/sign operations |
| `bitcoin` | Transaction structures, hashing |
| `electrum-client` | Electrum protocol client |
| `clap` | CLI argument parsing |
| `serde_json` | Wallet file serialization |
