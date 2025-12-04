# 002. RocksDB for Key Share Storage

**Date**: 2018 (original implementation)  
**Status**: Accepted  
**Deciders**: ZenGo-X Team

## Context

The Gotham server must persist Party1's key shares across restarts. Storage requirements:

1. **Durability**: Key shares must survive server restarts
2. **Performance**: Low-latency reads during signing protocol
3. **Simplicity**: Easy deployment without external dependencies
4. **Scalability**: Support growing number of key shares

Options considered:

| Option | Pros | Cons |
|--------|------|------|
| PostgreSQL | ACID, familiar, scalable | External dependency, network latency |
| Redis | Fast, simple | Persistence complexity, memory-bound |
| RocksDB | Embedded, fast, durable | No network access, single-node |
| SQLite | Embedded, SQL interface | Write contention, less optimized for KV |

## Decision

We will use **RocksDB** as an embedded key-value store for server-side key share persistence.

Key characteristics:
- Embedded (no external process)
- LSM-tree based (write-optimized)
- Configurable durability (fsync)
- Used by major projects (Facebook, CockroachDB)

## Consequences

**Positive**:
- **Zero external dependencies**: Single binary deployment
- **High performance**: Sub-millisecond reads, optimized writes
- **Proven reliability**: Battle-tested at scale by Facebook
- **Simple operations**: No database administration required
- **Crash recovery**: Built-in WAL for durability

**Negative**:
- **Single-node limitation**: No built-in replication
- **No SQL**: Key-value only, no complex queries
- **Memory usage**: LSM-tree requires tuning for large datasets
- **Backup complexity**: Requires filesystem-level backup

**Risks**:
- **Data loss on disk failure**: Single disk, no replication
  - Mitigation: Regular backups, RAID storage
- **Scaling limitation**: Single-node capacity bound
  - Mitigation: DB sharding if needed (not implemented)

## Evidence

- RocksDB initialization: [`public_gotham.rs:34-44`](../../gotham-server/src/public_gotham.rs#L34-L44)
- DB operations: [`public_gotham.rs:58-91`](../../gotham-server/src/public_gotham.rs#L58-L91)
- Key format: [`public_gotham.rs:52-54`](../../gotham-server/src/public_gotham.rs#L52-L54)
