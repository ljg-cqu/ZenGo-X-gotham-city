# System Architecture & Attack Surface

## Architecture Overview

**Purpose**: Describe gotham-city's design, trust boundaries, and key entry points as context for the audit.

At a high level, gotham-city is a Rust workspace that implements a two-party ECDSA (2P-ECDSA) signing flow:

- **gotham-server**
  - Rocket-based HTTP server that exposes REST-style endpoints for 2P-ECDSA key generation and signing.
  - Mounts handler functions from the external `gotham-engine` crate (via `gotham_engine::routes::wrap_*` entrypoints) and wires them to a `Db` implementation (`PublicGotham`).
  - Persists MPC protocol state via a RocksDB-backed database by default.
- **gotham-client**
  - CLI wallet application (not fully reviewed here) that drives MPC keygen/signing flows against gotham-server, ultimately enabling Bitcoin-like wallet operations.
- **demo-wallet / integration-tests**
  - Additional crates and tests that demonstrate usage, benchmark keygen/sign, and validate end-to-end MPC flows.
- **MPC engine dependencies** (external repos)
  - `two-party-ecdsa`: Rust implementation of Lindell's 2P-ECDSA protocol.
  - `gotham-engine`: Higher-level orchestration for MPC signing, key derivation, and routing. Its code lives outside this repository and was not audited in this run.

### Key Server Components (Reviewed Slice)

- `gotham-server/src/main.rs` – Rocket launch entrypoint using `#[rocket::launch]` to start the server and delegate to `server::get_server`.
- `gotham-server/src/server.rs` – Rocket server configuration and route mounting:
  - Registers error handlers (400/404/500) and mounts `gotham_engine::routes::wrap_*` keygen/sign routes under `/`.
  - Manages a shared, asynchronous `Db` instance (`PublicGotham`) using `tokio::sync::Mutex`.
- `gotham-server/src/public_gotham.rs` – PublicGotham implementation:
  - Loads configuration from `Settings.toml` plus environment variables into a `HashMap<String, String>`.
  - Enforces a simple alphanumeric-only rule for the RocksDB database name before opening `rocksdb::DB` at `./{db_name}`.
  - Implements the `gotham_engine::traits::Db` trait for insert/get and authorization hooks, using JSON serialization into RocksDB.
- `gotham-server/src/tests.rs` – End-to-end tests driving HTTP keygen and signing flows against a tracked Rocket instance.

## Attack Surface Diagram

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#f8f9fa",
    "primaryTextColor": "#1a1a1a",
    "primaryBorderColor": "#7a8591",
    "lineColor": "#8897a8",
    "secondaryColor": "#eff6fb",
    "tertiaryColor": "#f3f5f7",
    "background": "#ffffff",
    "mainBkg": "#f8f9fa",
    "clusterBkg": "#f3f5f7",
    "clusterBorder": "#8897a8",
    "edgeLabelBackground": "#ffffff"
  }
}}%%
flowchart LR
    U[CLI Wallet Client] -->|HTTP JSON (MPC messages)| S[gotham-server (Rocket)]
    S -->|MPC flows| E[MPC Engine (gotham-engine, two-party-ecdsa)]
    S -->|State read/write| D[RocksDB (local disk)]
    S -->|Optional auth integration| I[External IdP or AWS service]

    style U fill:#f8f9fa,stroke:#7a8591,stroke-width:2px,color:#1a1a1a
    style S fill:#faf4f4,stroke:#a87a7a,stroke-width:2px,color:#1a1a1a
    style E fill:#eff6fb,stroke:#7a9fc5,stroke-width:2px,color:#1a1a1a
    style D fill:#faf6f0,stroke:#a89670,stroke-width:2px,color:#1a1a1a
    style I fill:#f3f5f7,stroke:#8897a8,stroke-width:2px,color:#1a1a1a
```

### Entry Points

- **HTTP API (Rocket)**
  - Mounted under `/` by `gotham-server/src/server.rs` via `routes![gotham_engine::routes::wrap_*]`.
  - Key paths (from tests):
    - `/ecdsa/keygen/first`, `/ecdsa/keygen/{id}/second`, `/ecdsa/keygen/{id}/third`, `/ecdsa/keygen/{id}/fourth`
    - `/ecdsa/keygen/{id}/chaincode/first`, `/ecdsa/keygen/{id}/chaincode/second`
    - `/ecdsa/sign/{id}/first`, `/ecdsa/sign/{id}/second`
- **CLI Wallet**
  - Invokes server endpoints according to MPC protocol steps (not fully reviewed in this run).

### Trust Boundaries

1. **Client ↔ Server (HTTP)**
   - Boundary between local CLI wallet and Rocket-based server.
   - Transport security, authentication, and request validation must be provided by the deployment environment and any additional middleware; the reviewed server slice does not itself implement explicit auth.
2. **Server ↔ MPC Engine**
   - Logical boundary between HTTP layer (`gotham-server`) and the MPC orchestration / crypto implementations (`gotham-engine`, `two-party-ecdsa`).
   - Those engine crates are external to this repo and were not audited in this run.
3. **Server ↔ Data Store (RocksDB)**
   - Persistence of MPC protocol state and related values via a `Db` abstraction, stored in a local RocksDB database under `./{db_name}`.
   - No encryption or access-control configuration is defined in this repo.
4. **Server ↔ External Identity / Cloud Services**
   - `Settings.toml` defines `region`, `pool_id`, `issuer`, and `audience` keys, and tests set corresponding environment variables.
   - Actual integration logic with any IdP or cloud service was not observed in the reviewed slice.

## Technology Stack

| Component | Technology | Version / Source | Security Notes (this run) |
|-----------|-----------|------------------|----------------------------|
| **Language / Runtime** | Rust | Edition 2021 | Memory-safe by design; unsafe usage and engine code not reviewed here. |
| **Web Framework** | Rocket | 0.5.0-rc.1 (workspace) | Debug config binds to `0.0.0.0` in `Rocket.toml`; production profile and TLS/offload are deployment concerns. |
| **MPC Engine** | gotham-engine, two-party-ecdsa | Git dependencies | External cryptographic and protocol logic; treated as black boxes in this run and require dedicated audits. |
| **Database (default)** | RocksDB | `rocksdb = "0.21.0"` (local feature) | Used as a local on-disk KV store without encryption or retention policies in this repo. |
| **Config Loader** | config crate | `config = "0.9.2"` | Used with `include_str!("../Settings.toml")` and environment overrides. |
| **Serialization** | serde / serde_json | Workspace deps | Used for MPC state serialization into RocksDB and HTTP JSON payloads. |

## Data Flow Analysis

### Sensitive Data Handling (Reviewed Slice)

- **MPC Protocol Messages & State**
  - HTTP tests in `gotham-server/src/tests.rs` send and receive MPC keygen and signing messages as JSON.
  - `PublicGotham` implements `Db::insert` and `Db::get` using JSON serialization via `serde_json`, storing values in RocksDB with keys derived from user identifiers and MPC table names.
  - The exact nature of the stored values (for example, whether they include partial keys or only protocol transcripts) is defined in external engine crates and was not inspected here.

- **Configuration**
  - `Settings.toml` plus environment variables are merged into a `HashMap<String, String>` used to derive database name and other settings.
  - A simple character-level validation enforces alphanumeric-only database names before creating/opening RocksDB.

### Trust Boundaries (Detail)

1. **User ↔ CLI ↔ Server**
   - CLI wallet issues HTTP requests with JSON payloads to the Rocket server. Validation, rate limiting, authentication, and authorization policies for these requests are not implemented in the reviewed slice and are expected to be added by deployments or other components.
2. **Server ↔ Data Store**
   - `PublicGotham` constructs RocksDB keys by concatenating user ID, an ID field, and a logical table name. There is no encryption at rest and no explicit key-rotation or deletion logic in this implementation.
3. **Server ↔ Engine**
   - All MPC-specific operations are delegated to external crates; this repo acts primarily as a transport and state-persistence adapter around them.

## Deployment Model (Observed / Inferred from Repo)

- **Infrastructure**
  - Rocket server configured for debug mode in `gotham-server/Rocket.toml` with:
    - `address = "0.0.0.0"`
    - `port = 8000`
    - `log = "normal"`
  - No container, orchestration, or infrastructure-as-code definitions are present in this repo; deployment topology (TLS, reverse proxies, network ACLs) is environment-specific.

- **Configuration Management**
  - `Settings.toml` is compiled into the binary via `include_str!` and combined with environment variables (for example, `region`, `pool_id`, `issuer`, `audience`).
  - Database name is controlled via configuration and validated for alphanumeric-only characters before use.
  - No explicit key-management or secret-rotation logic appears in the reviewed slice; such responsibilities likely fall to external systems or non-reviewed parts of the engine.

> This architecture description is intentionally conservative and only covers the code that was actually reviewed in this run. A follow-up audit should expand this section after reviewing gotham-client, demo-wallet, integration-tests, and the external MPC engine crates.
