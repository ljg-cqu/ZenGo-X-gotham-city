# Recommendations

## Prioritized Roadmap

| Priority | Finding ID | Title | Effort |
|----------|------------|-------|--------|
| **Immediate** | R1-F-001 | Fix Authorization Bypass in `granted` | Low |
| **Immediate** | R1-F-002 | Encrypt Server Key Storage | Medium |
| **Immediate** | R1-F-003 | Encrypt Client Wallet Storage | Medium |
| **Short-term** | R1-F-004 | Enable TLS (HTTPS) | Low |
| **Short-term** | R1-F-005 | Implement Proper Error Handling | Medium |
| **Long-term** | R1-F-006 | Update Dependencies | High |

## Detailed Recommendations

### Immediate Actions

1.  **Fix Authorization Bypass (R1-F-001)**
    -   **Action**: Modify `gotham-server/src/public_gotham.rs` to implement real authorization checks in `granted`.
    -   **Details**: Verify that the `customer_id` owns the key being used and that the transaction details match the user's intent/policy.

2.  **Encrypt Key Storage (R1-F-002, R1-F-003)**
    -   **Action**: Implement encryption at rest for RocksDB (server) and wallet files (client).
    -   **Details**: Use a standard encryption library (e.g., `ring` or `sodiumoxide`) to encrypt sensitive data before writing to disk. Manage encryption keys securely (e.g., via environment variables or KMS for server, password derivation for client).

### Short-term Actions

3.  **Enable TLS (R1-F-004)**
    -   **Action**: Configure Rocket to use TLS.
    -   **Details**: Update `Rocket.toml` to include `[global.tls]` settings pointing to valid certificates.

4.  **Improve Error Handling (R1-F-005)**
    -   **Action**: Audit and replace `unwrap()`/`expect()` calls.
    -   **Details**: Use `Result` types and proper error propagation to ensure the server returns HTTP 500/400 responses instead of crashing.

### Long-term Actions

5.  **Update Dependencies (R1-F-006)**
    -   **Action**: Upgrade `rocket`, `reqwest`, `secp256k1`, and other dependencies.
    -   **Details**: This may require significant refactoring due to breaking changes in major versions (e.g., Rocket 0.4 -> 0.5, async changes).
