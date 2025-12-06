# ADR 003: Rocket Web Framework

## Header

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | ~2018 (project inception), updated for Rocket 0.5 |
| Deciders | ZenGo-X team |
| Impact | Server framework - affects API design, async model |

---

## Context

The Gotham Server needs an HTTP framework to expose MPC protocol endpoints. Requirements:

1. **Rust-native**: Type safety, memory safety, performance
2. **JSON support**: MPC protocol uses JSON serialization
3. **Async support**: Non-blocking I/O for concurrent requests
4. **Simple routing**: RESTful API patterns
5. **State management**: Shared database connection

---

## Decision

**Use Rocket 0.5.0-rc.1** as the HTTP server framework.

### Implementation

```rust
pub fn get_server() -> Rocket<Build> {
    let x = PublicGotham::new();
    rocket::Rocket::build()
        .register("/", catchers![internal_error, not_found, bad_request])
        .mount("/", routes![
            gotham_engine::routes::wrap_keygen_first,
            // ... other routes
        ])
        .manage(Mutex::new(Box::new(x) as Box<dyn Db>))
}
```

Evidence: [`server.rs:21-39`](../../gotham-server/src/server.rs#L21-L39)

---

## Alternatives Considered

### Option 1: Actix-web (Rejected)

| Pro | Con |
|-----|-----|
| Highest performance | Actor model complexity |
| Mature ecosystem | Less ergonomic API |
| | More verbose code |

### Option 2: warp (Rejected)

| Pro | Con |
|-----|-----|
| Composable filters | Steeper learning curve |
| Type-safe routing | Filter composition can be complex |
| | Less documentation |

### Option 3: axum (Not available at decision time)

| Pro | Con |
|-----|-----|
| Tower ecosystem | Released after project started |
| Type-safe extractors | Would require rewrite |
| Modern design | |

### Option 4: hyper (Rejected)

| Pro | Con |
|-----|-----|
| Low-level control | Too low-level |
| Maximum flexibility | Requires more boilerplate |
| | No built-in routing |

---

## Consequences

### Positive

- **Ergonomic API**: Declarative routing with macros
- **Type safety**: Request guards, responders
- **Built-in JSON**: `rocket::serde::json` support
- **Async ready**: Rocket 0.5 is fully async
- **Error handling**: Catchers for HTTP errors
- **State management**: `.manage()` for shared state

### Negative

- **Release candidate**: 0.5.0-rc.1 is not stable release
- **Breaking changes**: May need updates when 0.5 stable releases
- **Macro magic**: Some compile errors are hard to debug
- **Nightly Rust**: Previously required nightly (now stable in 0.5)

### Neutral

- **Tokio runtime**: Uses tokio for async (same as most Rust async)
- **Performance**: Adequate for MPC workloads (not a bottleneck)
- **Community**: Active but smaller than actix-web

---

## Configuration

### Rocket.toml

```toml
[default]
address = "127.0.0.1"
port = 8000

[release]
address = "0.0.0.0"
port = 8000
```

### Features Used

```toml
# Cargo.toml
rocket = { version = "0.5.0-rc.1", default-features = false, features=["json"]}
```

Evidence: [`Cargo.toml:18`](../../Cargo.toml#L18)

---

## Migration Considerations

When Rocket 0.5 stable releases:

1. Update version in `Cargo.toml`
2. Review release notes for breaking changes
3. Update deprecated APIs if any
4. Test all endpoints

Alternative migration paths:
- axum: Most similar modern framework
- actix-web: Highest performance if needed

---

## Evidence

| Claim | Verification |
|-------|--------------|
| Rocket 0.5.0-rc.1 | [`Cargo.toml:18`](../../Cargo.toml#L18) |
| Route registration | [`server.rs:27-36`](../../gotham-server/src/server.rs#L27-L36) |
| Error catchers | [`server.rs:6-19`](../../gotham-server/src/server.rs#L6-L19) |
| State management | [`server.rs:38`](../../gotham-server/src/server.rs#L38) |
| JSON feature | [`Cargo.toml:18`](../../Cargo.toml#L18) |

---

## References

- [Rocket Documentation](https://rocket.rs/)
- [Rocket 0.5 Migration Guide](https://rocket.rs/v0.5-rc/guide/upgrading-from-0.4/)
- [Rocket vs. Actix vs. warp comparison](https://www.arewewebyet.org/)
