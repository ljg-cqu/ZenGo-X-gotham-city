# ADR 002: RocksDB for Key Share Storage

## Header

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | ~2018 (project inception) |
| Deciders | ZenGo-X team |
| Impact | Server storage - affects persistence, deployment |

---

## Context

The Gotham Server needs to persist Party1 key shares across protocol rounds and server restarts. Requirements:

1. **Durability**: Key shares must survive server restarts
2. **Performance**: Low-latency reads/writes for MPC protocol
3. **Simplicity**: Minimal operational overhead
4. **Portability**: Easy deployment without external dependencies

---

## Decision

**Use RocksDB as embedded key-value storage** for Party1 key shares.

### Implementation

```rust
pub struct PublicGotham {
    rocksdb_client: rocksdb::DB,
}

impl PublicGotham {
    pub fn new() -> Self {
        let rocksdb_client = rocksdb::DB::open_default(format!("./{}", db_name)).unwrap();
        PublicGotham { rocksdb_client }
    }
}
```

Evidence: [`public_gotham.rs:14-44`](../../gotham-server/src/public_gotham.rs#L14-L44)

---

## Alternatives Considered

### Option 1: PostgreSQL (Rejected)

| Pro | Con |
|-----|-----|
| Rich query capabilities | External dependency |
| Mature ecosystem | Connection pool complexity |
| Built-in encryption | Higher latency |
| | Operational overhead |

### Option 2: Redis (Rejected)

| Pro | Con |
|-----|-----|
| Very fast | Primarily in-memory |
| Simple API | Persistence is secondary |
| | External dependency |
| | Memory-bound |

### Option 3: SQLite (Rejected)

| Pro | Con |
|-----|-----|
| Embedded | Write contention |
| SQL support | Slower for simple KV workloads |
| Zero-config | |

### Option 4: Filesystem (Rejected)

| Pro | Con |
|-----|-----|
| Zero dependencies | No transactions |
| Simple | File handle limits |
| | No atomic operations |
| | Directory structure overhead |

---

## Consequences

### Positive

- **Zero external dependencies**: Embedded in process
- **High performance**: ~1ms read/write latency
- **Simple deployment**: Just a directory of files
- **Proven reliability**: Used by Facebook, LinkedIn, Yahoo
- **Automatic compaction**: Manages storage efficiency

### Negative

- **Single-node**: No built-in replication
- **Horizontal scaling**: Requires application-level sharding
- **Backup complexity**: Need to stop server for consistent backup (or use checkpoints)
- **No encryption at rest**: Must be implemented separately

### Neutral

- **SST file format**: Not human-readable (use `ldb` tool)
- **Memory usage**: Configurable but requires tuning for large datasets
- **AWS option**: Code supports switching to DynamoDB (`db = "aws"`)

---

## Migration Path

The code includes support for alternative backends via the `db` setting:

```toml
# Settings.toml
db = "local"  # or "aws"
```

Evidence: [`Settings.toml:2`](../../gotham-server/Settings.toml#L2)

For production at scale, consider:
1. DynamoDB for managed, replicated storage
2. CockroachDB for distributed SQL
3. TiKV for distributed KV

---

## Evidence

| Claim | Verification |
|-------|--------------|
| RocksDB used | [`public_gotham.rs:15`](../../gotham-server/src/public_gotham.rs#L15) |
| Open default DB | [`public_gotham.rs:41`](../../gotham-server/src/public_gotham.rs#L41) |
| Simple KV API | [`public_gotham.rs:66`](../../gotham-server/src/public_gotham.rs#L66) |
| AWS alternative | [`Settings.toml:2`](../../gotham-server/Settings.toml#L2) |

---

## References

- [RocksDB Documentation](https://rocksdb.org/)
- [RocksDB Rust bindings](https://github.com/rust-rocksdb/rust-rocksdb)
- [RocksDB vs. LevelDB](https://github.com/facebook/rocksdb/wiki/RocksDB-FAQ)
