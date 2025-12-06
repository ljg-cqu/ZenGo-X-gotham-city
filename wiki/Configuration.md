# Configuration

## Server Configuration

### Settings.toml

**Location**: [`gotham-server/Settings.toml`](../gotham-server/Settings.toml)

| Variable | Type | Default | Required | Description | Evidence |
|----------|------|---------|----------|-------------|----------|
| `db` | String | `"local"` | No | Storage backend: `"local"` (RocksDB) or `"aws"` | [`Settings.toml:2`](../gotham-server/Settings.toml#L2) |
| `db_name` | String | `"db"` | No | RocksDB directory name (alphanumeric only) | [`public_gotham.rs:37`](../gotham-server/src/public_gotham.rs#L37) |
| `region` | String | `""` | If db=aws | AWS region for DynamoDB | [`Settings.toml:6`](../gotham-server/Settings.toml#L6) |
| `pool_id` | String | `""` | If auth | AWS Cognito User Pool ID | [`Settings.toml:7`](../gotham-server/Settings.toml#L7) |
| `issuer` | String | `""` | If auth | JWT token issuer URL | [`Settings.toml:8`](../gotham-server/Settings.toml#L8) |
| `audience` | String | `""` | If auth | JWT token audience | [`Settings.toml:9`](../gotham-server/Settings.toml#L9) |

### Environment Variables (Server)

All settings can be overridden via environment variables:

| Variable | Overrides | Example |
|----------|-----------|---------|
| `DB` | `db` | `DB=aws` |
| `DB_NAME` | `db_name` | `DB_NAME=production_db` |
| `REGION` | `region` | `REGION=us-east-1` |
| `POOL_ID` | `pool_id` | `POOL_ID=us-east-1_xxxxx` |
| `ISSUER` | `issuer` | `ISSUER=https://cognito-idp.us-east-1.amazonaws.com/...` |
| `AUDIENCE` | `audience` | `AUDIENCE=client-app-id` |

Evidence: [`public_gotham.rs:28`](../gotham-server/src/public_gotham.rs#L28)

### Rocket.toml

**Location**: [`gotham-server/Rocket.toml`](../gotham-server/Rocket.toml)

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `address` | String | `"127.0.0.1"` | Bind address |
| `port` | Integer | `8000` | Listen port |
| `workers` | Integer | CPU cores | Async worker threads |
| `log_level` | String | `"normal"` | Logging verbosity |

---

## Client Configuration

### Demo Wallet Settings

**Location**: `settings.toml` (working directory) or CLI `--settings` flag

| Variable | Type | Default | Required | Description | Evidence |
|----------|------|---------|----------|-------------|----------|
| `gotham_server_url` | String | `"http://127.0.0.1:8000"` | No | Gotham server endpoint | [`main.rs:66-68`](../demo-wallet/src/main.rs#L66-L68) |
| `wallet_file` | String | `"wallet.json"` | No | Path to wallet JSON file | [`main.rs:70-72`](../demo-wallet/src/main.rs#L70-L72) |
| `electrum_server_url` | String | — | For Bitcoin | Electrum server endpoint | [`main.rs:51`](../demo-wallet/src/main.rs#L51) |
| `rpc_url` | String | — | For Ethereum | Ethereum JSON-RPC endpoint | [`main.rs:45`](../demo-wallet/src/main.rs#L45) |
| `chain_id` | Integer | — | For Ethereum | EIP-155 chain ID | [`main.rs:49`](../demo-wallet/src/main.rs#L49) |
| `private_key` | String | — | Optional | Private key (non-MPC mode) | [`main.rs:48`](../demo-wallet/src/main.rs#L48) |

### Environment Variables (Client)

Environment variables are prefixed with `GOTHAM_`:

| Variable | Overrides | Example |
|----------|-----------|---------|
| `GOTHAM_SERVER_URL` | `gotham_server_url` | `GOTHAM_SERVER_URL=https://mpc.example.com` |
| `GOTHAM_WALLET_FILE` | `wallet_file` | `GOTHAM_WALLET_FILE=/secure/wallet.json` |
| `GOTHAM_ELECTRUM_SERVER_URL` | `electrum_server_url` | `GOTHAM_ELECTRUM_SERVER_URL=ssl://electrum.example.com:50002` |
| `GOTHAM_RPC_URL` | `rpc_url` | `GOTHAM_RPC_URL=https://eth.llamarpc.com` |
| `GOTHAM_CHAIN_ID` | `chain_id` | `GOTHAM_CHAIN_ID=1` |

Evidence: [`main.rs:60`](../demo-wallet/src/main.rs#L60)

---

## Configuration by Environment

### Development

```toml
# gotham-server/Settings.toml
db = "local"
db_name = "dev_db"

# demo-wallet/settings.toml
gotham_server_url = "http://127.0.0.1:8000"
wallet_file = "dev_wallet.json"
electrum_server_url = "ssl://testnet.aranguren.org:51002"  # Bitcoin testnet
rpc_url = "https://rpc.sepolia.org"  # Ethereum Sepolia testnet
chain_id = 11155111
```

### Production

```toml
# gotham-server/Settings.toml
db = "local"  # or "aws" for DynamoDB
db_name = "prod_db"
region = "us-east-1"
pool_id = "us-east-1_XXXXXXXX"
issuer = "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXX"
audience = "your-app-client-id"

# demo-wallet/settings.toml
gotham_server_url = "https://mpc.yourcompany.com"
wallet_file = "/secure/storage/wallet.json"
electrum_server_url = "ssl://electrum.yourcompany.com:50002"
rpc_url = "https://mainnet.infura.io/v3/YOUR_KEY"
chain_id = 1
```

---

## Configuration Loading Order

Configuration is loaded in the following order (later sources override earlier):

### Server

1. Default values in code
2. `Settings.toml` file
3. Environment variables

Evidence: [`public_gotham.rs:21-28`](../gotham-server/src/public_gotham.rs#L21-L28)

### Client

1. Default values in code
2. Settings file (path from `--settings` CLI flag or `settings.toml`)
3. Environment variables (prefixed with `GOTHAM_`)

Evidence: [`main.rs:58-64`](../demo-wallet/src/main.rs#L58-L64)

---

## Secrets Management

| Secret Type | Storage Method | Evidence |
|-------------|----------------|----------|
| **Server key shares** | RocksDB (plaintext) | [`public_gotham.rs:41`](../gotham-server/src/public_gotham.rs#L41) |
| **Client key shares** | JSON file (plaintext) | demo-wallet |
| **AWS credentials** | Environment variables | AWS SDK default |
| **Auth tokens** | Config/environment | [`lib.rs:22`](../gotham-client/src/lib.rs#L22) |

⚠️ **Security Note**: Key shares are stored in plaintext. Production deployments should:
1. Encrypt RocksDB at rest
2. Encrypt wallet JSON files
3. Use hardware security modules (HSM) for key storage
4. Implement proper access controls

→ See [Security.md](./Security.md) for secrets management recommendations.

---

## Validation

### DB Name Validation

```rust
if !db_name.chars().all(|e| char::is_ascii_alphanumeric(&e)) {
    panic!("DB name is illegal, may only contain alphanumeric characters");
}
```

Evidence: [`public_gotham.rs:38-40`](../gotham-server/src/public_gotham.rs#L38-L40)

### Required Configuration

| Scenario | Required Settings |
|----------|-------------------|
| Local development | None (all defaults work) |
| Bitcoin wallet | `electrum_server_url` |
| Ethereum wallet | `rpc_url`, `chain_id` |
| AWS storage | `db=aws`, `region`, AWS env vars |
| Cognito auth | `pool_id`, `issuer`, `audience` |

---

## CLI Arguments

### Demo Wallet

```bash
demo-wallet [OPTIONS] <COMMAND>

Options:
  -s, --settings <FILE>  Settings file [default: settings.toml]
  -h, --help             Print help
  -V, --version          Print version

Commands:
  bitcoin  Bitcoin blockchain operations
  evm      EVM-compatible blockchain operations
```

Evidence: [`main.rs:11-19`](../demo-wallet/src/main.rs#L11-L19)
