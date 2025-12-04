# Implementation

## Key Generation Process [CRITICAL]

**Entry Point**: [`gotham-client/src/ecdsa/keygen.rs:37`](../gotham-client/src/ecdsa/keygen.rs#L37)  
**Trigger**: Client application initiates wallet creation  
**Performance**: p95 = 762ms (localhost, M2 MacBook)  
**Traffic Share**: Primary operation for new wallet creation

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant DB as RocksDB
    
    Note over C,S: Round 1 - Commitment
    C->>S: POST /ecdsa/keygen/first
    S->>S: Generate Party1 commitment
    S->>DB: Store kg_party_one_first_message
    S-->>C: (session_id, KeyGenFirstMsg)
    
    Note over C,S: Round 2 - ECDH + Paillier
    C->>C: Generate Party2 EC keypair
    C->>S: POST /keygen/{id}/second (DLogProof)
    S->>S: Verify proof, generate Paillier keys
    S->>DB: Store kg_party_one_second_message
    S-->>C: KeyGenParty1Message2
    
    Note over C,S: Rounds 3-4 - PDL Verification
    C->>S: POST /keygen/{id}/third (PDLFirstMsg)
    S-->>C: PDLFirstMessage
    C->>S: POST /keygen/{id}/fourth (PDLSecondMsg)
    S->>DB: Store final key share
    S-->>C: PDLSecondMessage
    
    Note over C,S: Chain Code (2 rounds)
    C->>S: POST /keygen/{id}/chaincode/first
    S-->>C: Party1FirstMessage
    C->>S: POST /keygen/{id}/chaincode/second
    S-->>C: Party1SecondMessage
    C->>C: Compute final MasterKey2
```

| Step | Action | Evidence |
|------|--------|----------|
| 1 | Client requests session, server generates commitment | [`keygen.rs:40-41`](../gotham-client/src/ecdsa/keygen.rs#L40-L41) |
| 2 | Client generates EC keypair with DLog proof | [`keygen.rs:43-44`](../gotham-client/src/ecdsa/keygen.rs#L43-L44) |
| 3 | Server verifies proof, returns ECDH + Paillier keys | [`keygen.rs:47-57`](../gotham-client/src/ecdsa/keygen.rs#L47-L57) |
| 4 | PDL protocol verifies Paillier correctness | [`keygen.rs:59-80`](../gotham-client/src/ecdsa/keygen.rs#L59-L80) |
| 5 | Chain code derivation for HD support | [`keygen.rs:82-106`](../gotham-client/src/ecdsa/keygen.rs#L82-L106) |
| 6 | Client assembles MasterKey2 | [`keygen.rs:108-120`](../gotham-client/src/ecdsa/keygen.rs#L108-L120) |

**Edge Cases**:
- Network timeout: Client must retry full protocol (no partial resumption)
- Server restart: Session lost, requires new keygen
- Invalid proof: Server returns 400, client must regenerate

---

## Signing Process [CRITICAL]

**Entry Point**: [`gotham-client/src/ecdsa/sign.rs:28`](../gotham-client/src/ecdsa/sign.rs#L28)  
**Trigger**: Transaction signing request  
**Performance**: p95 = 151ms (localhost, M2 MacBook)  
**Traffic Share**: Most frequent operation (multiple signs per wallet)

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant DB as RocksDB
    
    Note over C,S: Round 1 - Ephemeral Keys
    C->>C: Generate ephemeral EC keypair
    C->>S: POST /ecdsa/sign/{id}/first
    S->>DB: Retrieve stored key share
    S->>S: Generate Party1 ephemeral key
    S-->>C: EphKeyGenFirstMsg
    
    Note over C,S: Round 2 - Signature
    C->>C: Compute partial signature
    C->>S: POST /ecdsa/sign/{id}/second
    Note right of C: SignSecondMsgRequest
    S->>S: Derive child key (x_pos, y_pos)
    S->>S: Combine partial signatures
    S-->>C: SignatureRecid (r, s, recid)
```

| Step | Action | Evidence |
|------|--------|----------|
| 1 | Client generates ephemeral commitment | [`sign.rs:36-37`](../gotham-client/src/ecdsa/sign.rs#L36-L37) |
| 2 | Server returns its ephemeral public key | [`sign.rs:40-44`](../gotham-client/src/ecdsa/sign.rs#L40-L44) |
| 3 | Client computes partial signature | [`sign.rs:46-51`](../gotham-client/src/ecdsa/sign.rs#L46-L51) |
| 4 | Server combines, returns final (r, s, recid) | [`sign.rs:53-65`](../gotham-client/src/ecdsa/sign.rs#L53-L65) |

**Edge Cases**:
- Invalid session ID: Server returns 400 (key not found)
- Concurrent signing: Safe - each sign uses fresh ephemeral keys
- Message replay: Not prevented at protocol level (application responsibility)

---

## Bitcoin Transaction Process [IMPORTANT]

**Entry Point**: [`demo-wallet/src/bitcoin/mod.rs:263`](../demo-wallet/src/bitcoin/mod.rs#L263)  
**Trigger**: `bitcoin send` CLI command  
**Performance**: Depends on input count (151ms × inputs + network latency)

```mermaid
flowchart LR
    A[Query UTXOs] --> B[Select Inputs]
    B --> C[Build Transaction]
    C --> D{For Each Input}
    D --> E[Compute SigHash]
    E --> F[2P-ECDSA Sign]
    F --> G[Attach Witness]
    G --> D
    D --> H[Broadcast]
```

| Stage | Input | Transform | Output | Evidence |
|-------|-------|-----------|--------|----------|
| UTXO Query | Wallet addresses | Electrum API call | List<GetListUnspentResponse> | [`mod.rs:449-458`](../demo-wallet/src/bitcoin/mod.rs#L449-L458) |
| Selection | Amount, UTXOs | Greedy algorithm | Selected UTXOs | [`mod.rs:418-447`](../demo-wallet/src/bitcoin/mod.rs#L418-L447) |
| Build | UTXOs, recipient | Transaction construction | Unsigned Transaction | [`mod.rs:278-320`](../demo-wallet/src/bitcoin/mod.rs#L278-L320) |
| Sign | Transaction, keys | BIP143 + 2P-ECDSA | Witness data | [`mod.rs:324-368`](../demo-wallet/src/bitcoin/mod.rs#L324-L368) |
| Broadcast | Signed tx | Electrum broadcast | TXID | [`mod.rs:370-373`](../demo-wallet/src/bitcoin/mod.rs#L370-L373) |

---

## Data Flow

```mermaid
flowchart TD
    subgraph Client["Client Side"]
        UI[User Input]
        CL[Client Library]
        WL[Wallet Logic]
    end
    
    subgraph Server["Server Side"]
        RK[Rocket Handler]
        GE[Gotham Engine]
        DB[(RocksDB)]
    end
    
    subgraph External["External Services"]
        EL[Electrum]
        RPC[Ethereum RPC]
        BC[Blockchain]
    end
    
    UI --> WL
    WL --> CL
    CL -->|HTTP/JSON| RK
    RK --> GE
    GE --> DB
    WL --> EL
    WL --> RPC
    EL --> BC
    RPC --> BC
```

---

## Cross-Cutting Concerns

| Concern | Implementation | Evidence |
|---------|----------------|----------|
| Error: Network | `failure::Error` with retryable classification | [`gotham-client/src/lib.rs:17`](../gotham-client/src/lib.rs#L17) |
| Error: Protocol | Explicit `Result` returns, panic on crypto failure | [`sign.rs:43-44`](../gotham-client/src/ecdsa/sign.rs#L43-L44) |
| Error: DB | `DatabaseError` type in gotham-engine | [`public_gotham.rs:63`](../gotham-server/src/public_gotham.rs#L63) |
| Serialization | serde JSON for all wire formats | [`Cargo.toml:12-13`](../Cargo.toml#L12-L13) |
| Logging | `log` crate with `info!` for timing | [`lib.rs:53`](../gotham-client/src/lib.rs#L53) |
| Timing | `floating_duration::TimeFormat` for benchmarks | [`keygen.rs:118`](../gotham-client/src/ecdsa/keygen.rs#L118) |

### Error Handling Strategy

| Error Type | Handling | Recovery |
|------------|----------|----------|
| HTTP 400 | Return `None` from client | Application retry with corrected input |
| HTTP 500 | Return `None` from client | Application retry |
| Network timeout | reqwest default timeout | Application retry |
| Crypto proof failure | `expect()` panic | Bug - should not occur with valid keys |
| DB read failure | Return error to handler | 500 response |

### Logging Patterns

```rust
// Request timing (client)
info!("(req {}, took: {:?})", path, TimeFormat(start.elapsed()));

// Keygen completion
println!("(id: {}) Took: {:?}", id, TimeFormat(start.elapsed()));

// Wallet operations (debug level)
debug!("(wallet id: {}) Saved wallet to disk", self.id);
```

Evidence: 
- [`lib.rs:53`](../gotham-client/src/lib.rs#L53)
- [`keygen.rs:118`](../gotham-client/src/ecdsa/keygen.rs#L118)
- [`bitcoin/mod.rs:250`](../demo-wallet/src/bitcoin/mod.rs#L250)

### State Management

| Component | State Location | Persistence | Evidence |
|-----------|----------------|-------------|----------|
| Server key shares | RocksDB | Durable | [`public_gotham.rs:41`](../gotham-server/src/public_gotham.rs#L41) |
| Client key shares | In-memory / JSON file | Application-managed | [`bitcoin/mod.rs:245-261`](../demo-wallet/src/bitcoin/mod.rs#L245-L261) |
| Session state | Server DB (keyed by session ID) | Durable | [`public_gotham.rs:52-54`](../gotham-server/src/public_gotham.rs#L52-L54) |
| Derived addresses | Client wallet JSON | Application-managed | [`bitcoin/mod.rs:119`](../demo-wallet/src/bitcoin/mod.rs#L119) |
