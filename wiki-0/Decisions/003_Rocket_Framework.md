# 003. Rocket Web Framework

**Date**: 2018 (original), updated 2022 (Rocket 0.5)  
**Status**: Accepted  
**Deciders**: ZenGo-X Team

## Context

The Gotham server exposes HTTP/JSON endpoints for the MPC protocol. Framework requirements:

1. **Rust-native**: Consistent with rest of codebase
2. **JSON support**: Automatic serialization/deserialization
3. **Type safety**: Compile-time route checking
4. **Performance**: Low overhead for crypto-heavy workloads
5. **Async support**: Non-blocking I/O for scalability

Options considered:

| Option | Pros | Cons |
|--------|------|------|
| Rocket | Type-safe routing, excellent DX | Was sync-only (fixed in 0.5) |
| Actix-web | High performance, async | Complex actor model |
| Axum | Modern, tower-based | Newer, less mature (at the time) |
| Warp | Filter-based composition | Steep learning curve |

## Decision

We will use **Rocket** (version 0.5.0-rc.1) as the HTTP framework.

Key characteristics:
- Type-safe request guards
- Automatic JSON serialization via `rocket::serde`
- Async support (Rocket 0.5+)
- Simple route definition with macros

## Consequences

**Positive**:
- **Type safety**: Routes checked at compile time
- **Developer experience**: Clear, readable route definitions
- **JSON handling**: Built-in serde integration
- **Error handling**: Catchers for HTTP errors
- **Testing**: Built-in test client

**Negative**:
- **RC version**: Using release candidate (0.5.0-rc.1)
- **Macro magic**: Some runtime behavior hidden in macros
- **Nightly Rust**: Historically required (now stable with 0.5)
- **Limited middleware**: Less flexible than tower-based frameworks

**Risks**:
- **API stability**: RC version may have breaking changes
  - Mitigation: Pin version, test thoroughly on upgrade
- **Async complexity**: Mixing async with blocking crypto
  - Mitigation: Spawn blocking tasks for heavy computation

## Evidence

- Server setup: [`server.rs:21-39`](../../gotham-server/src/server.rs#L21-L39)
- Route mounting: [`server.rs:27-36`](../../gotham-server/src/server.rs#L27-L36)
- Error catchers: [`server.rs:6-19`](../../gotham-server/src/server.rs#L6-L19)
- Dependency: [`Cargo.toml:18`](../../Cargo.toml#L18)

## Route Definition Example

```rust
pub fn get_server() -> Rocket<Build> {
    rocket::Rocket::build()
        .register("/", catchers![internal_error, not_found, bad_request])
        .mount(
            "/",
            routes![
                gotham_engine::routes::wrap_keygen_first,
                gotham_engine::routes::wrap_keygen_second,
                // ... more routes
            ],
        )
        .manage(Mutex::new(Box::new(x) as Box<dyn gotham_engine::traits::Db>))
}
```

Evidence: [`server.rs:21-39`](../../gotham-server/src/server.rs#L21-L39)
