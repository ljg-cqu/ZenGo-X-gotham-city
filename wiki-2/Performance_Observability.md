# Performance & Observability

## Performance Baselines

### Benchmark Results

| Priority | Operation | p50 | p95 | p99 | Throughput | Conditions | Evidence |
|----------|-----------|-----|-----|-----|------------|------------|----------|
| P1 | **Key Generation** | ~700ms | 762ms | ~800ms | ~1.3 ops/s | M2 MacBook, localhost | [`README.md:90`](../README.md#L90) |
| P1 | **Signing** | ~140ms | 151ms | ~160ms | ~6.6 ops/s | M2 MacBook, localhost | [`README.md:91`](../README.md#L91) |

### Performance Budget

| Priority | Endpoint | Budget (p95) | Current (p95) | Status | Headroom | Optimization Opportunity | Evidence |
|----------|----------|--------------|---------------|--------|----------|-------------------------|----------|
| P1 | **Keygen (6 rounds)** | 1000ms | 762ms | ✅ OK | 238ms | Paillier acceleration possible | [`README.md:90`](../README.md#L90) |
| P1 | **Sign (2 rounds)** | 200ms | 151ms | ✅ OK | 49ms | Connection pooling | [`README.md:91`](../README.md#L91) |
| P2 | **DB Read** | 10ms | ~1ms | ✅ OK | 9ms | N/A - already fast | RocksDB |
| P2 | **DB Write** | 20ms | ~2ms | ✅ OK | 18ms | N/A - already fast | RocksDB |

### Latency Breakdown

#### Key Generation (~762ms)

| Phase | Estimated Time | % of Total | Bottleneck |
|-------|----------------|------------|------------|
| Round 1 (commitment) | ~50ms | 7% | Network |
| Round 2 (ECDH + Paillier) | ~400ms | 52% | **Paillier keygen** |
| Round 3 (PDL challenge) | ~50ms | 7% | Network |
| Round 4 (PDL verify) | ~100ms | 13% | Crypto verification |
| Chain code (2 rounds) | ~100ms | 13% | Network |
| Client assembly | ~60ms | 8% | Local crypto |

#### Signing (~151ms)

| Phase | Estimated Time | % of Total | Bottleneck |
|-------|----------------|------------|------------|
| Round 1 (ephemeral) | ~50ms | 33% | Network |
| Round 2 (signature) | ~100ms | 67% | Crypto + Network |

---

## Service Level Indicators (SLIs)

| SLI | Definition | Measurement | Target (SLO) | Evidence |
|-----|------------|-------------|--------------|----------|
| **Availability** | Successful requests / Total requests | HTTP 2xx count | 99.9% | Proxy metrics |
| **Keygen Latency** | p95 key generation time | Request timing | <1000ms | Benchmarks |
| **Sign Latency** | p95 signing time | Request timing | <200ms | Benchmarks |
| **Error Rate** | 4xx + 5xx / Total requests | HTTP status codes | <0.1% | Proxy metrics |

---

## Observability Stack

### Current State

| Layer | Tool | Status | Evidence |
|-------|------|--------|----------|
| Metrics | None built-in | ❌ Not implemented | — |
| Logging | `log` crate (info level) | ✅ Basic | [`lib.rs:53`](../gotham-client/src/lib.rs#L53) |
| Tracing | None | ❌ Not implemented | — |
| Alerting | None | ❌ Not implemented | — |

### Recommended Stack

| Layer | Tool | Purpose | Integration |
|-------|------|---------|-------------|
| Metrics | Prometheus | Time-series metrics | Rocket fairings |
| Logging | ELK / Loki | Log aggregation | `tracing` crate |
| Tracing | Jaeger | Distributed tracing | `tracing` crate |
| Alerting | Prometheus Alertmanager | Incident notification | Alert rules |

### Logging Patterns

Current logging in the codebase:

```rust
// Request timing (client-side)
info!("(req {}, took: {:?})", path, TimeFormat(start.elapsed()));

// Keygen completion
println!("(id: {}) Took: {:?}", id, TimeFormat(start.elapsed()));
```

Evidence: [`lib.rs:53`](../gotham-client/src/lib.rs#L53), [`keygen.rs:118`](../gotham-client/src/ecdsa/keygen.rs#L118)

---

## Performance Optimization Opportunities

### P1 Critical Optimizations

| Opportunity | Current | Target | Effort | Impact | Evidence |
|-------------|---------|--------|--------|--------|----------|
| **HTTP Connection Pooling** | New connection per request | Connection reuse | Medium | -20ms sign latency | [`lib.rs:28`](../gotham-client/src/lib.rs#L28) |
| **Upgrade reqwest** | 0.9.5 (sync) | 0.11+ (async) | High | Better throughput | [`Cargo.toml:15`](../Cargo.toml#L15) |

### P2 Important Optimizations

| Opportunity | Current | Target | Effort | Impact | Evidence |
|-------------|---------|--------|--------|--------|----------|
| **Paillier Acceleration** | Software | Hardware/optimized lib | High | -200ms keygen | two-party-ecdsa |
| **RocksDB Tuning** | Defaults | Tuned for workload | Medium | -10% DB latency | RocksDB docs |
| **Async Client** | Blocking | Full async | High | Better concurrency | Future work |

### P3 Nice-to-Have

| Opportunity | Current | Target | Effort | Impact |
|-------------|---------|--------|--------|--------|
| Protocol batching | Single-request | Batch multiple ops | High | Reduced round trips |
| Response compression | None | gzip | Low | Reduced bandwidth |

---

## Capacity Planning

### Current Capacity (Single Instance)

| Resource | Current Usage | Estimated Capacity | Utilization | Headroom |
|----------|---------------|-------------------|-------------|----------|
| **CPU** | Variable | ~6.6 signs/s per core | Low | High |
| **Memory** | ~100MB | ~1GB available | 10% | 90% |
| **Storage** | ~1KB/session | ~50GB available | <1% | >99% |
| **Network** | ~10KB/sign | ~100Mbps available | <1% | >99% |

### Growth Projections

| Scenario | Signs/Day | Required Capacity | Current Sufficiency |
|----------|-----------|-------------------|---------------------|
| Small wallet | 100 | 1 instance | ✅ Yes |
| Medium service | 10,000 | 1 instance | ✅ Yes |
| Large service | 100,000 | 2+ instances + shared DB | ⚠️ Needs architecture change |

### Scaling Triggers

| Trigger | Threshold | Action | Lead Time |
|---------|-----------|--------|-----------|
| CPU >80% sustained | 1 hour | Scale up instance | 30 min |
| Memory >80% | 1 hour | Scale up instance | 30 min |
| p95 latency >2x baseline | 15 min | Investigate + scale | 1 hour |
| Error rate >1% | 5 min | Incident response | Immediate |

---

## Benchmarking

### Running Benchmarks

```bash
# Key generation benchmark
cd gotham-server
cargo bench --bench keygen_bench

# Signing benchmark
cd gotham-server
cargo bench --bench sign_bench

# Full benchmark suite
cargo bench
```

### Custom Benchmarking

```rust
// Example using criterion
use criterion::{criterion_group, criterion_main, Criterion};

fn keygen_benchmark(c: &mut Criterion) {
    let client = ClientShim::new("http://127.0.0.1:8000".to_string(), None);
    c.bench_function("keygen", |b| {
        b.iter(|| keygen::get_master_key(&client))
    });
}

criterion_group!(benches, keygen_benchmark);
criterion_main!(benches);
```

---

## Performance Monitoring Checklist

### Production Readiness

- [ ] Add Prometheus metrics endpoint
- [ ] Configure request latency histograms
- [ ] Set up alerting for SLO breaches
- [ ] Implement health check endpoint
- [ ] Add distributed tracing headers
- [ ] Configure log aggregation
- [ ] Create performance dashboard

### Ongoing Operations

- [ ] Weekly p95 latency review
- [ ] Monthly capacity planning review
- [ ] Quarterly benchmark regression tests
- [ ] Annual performance audit
