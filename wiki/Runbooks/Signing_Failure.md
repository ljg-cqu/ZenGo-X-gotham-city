# Signing Failure

**Severity**: P1  
**Last Tested**: 2025-12-05  
**Owner**: Platform Team

## Symptoms

- Client receives "party1 sign first message request failed" or "party1 sign second message request failed"
- HTTP 400 response on `/ecdsa/sign/{id}/*` endpoints
- Transactions fail to sign, blocking user operations
- Sign latency exceeds 500ms consistently

## Diagnosis

| Step | Command/Action | Expected Output | Evidence |
|------|----------------|-----------------|----------|
| 1 | Verify session ID exists | Key share in DB | Check server logs for "Getting from db" |
| 2 | Verify keygen completed | PrivateShare has valid `id` field | Client wallet file inspection |
| 3 | Check server connectivity | HTTP response | `curl -X POST http://localhost:8000/ecdsa/sign/test/first` |
| 4 | Check client wallet file | Valid JSON, non-empty master_key | `cat wallet.json \| jq .` |
| 5 | Verify HD derivation params | x_pos, y_pos are valid integers | Client logs |

### Error Pattern Analysis

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "party1 sign first message request failed" | Session not found or server error | Re-keygen or check server |
| "party1 sign second message request failed" | Invalid signature computation | Check message hash format |
| HTTP 400 | Malformed request body | Verify JSON serialization |
| HTTP 404 | Invalid session ID | Use ID from keygen |

## Resolution

| Step | Action | Rollback | Evidence |
|------|--------|----------|----------|
| 1 | Verify session ID matches keygen | N/A | Compare `private_share.id` |
| 2 | Retry signing operation | N/A | Client retry |
| 3 | Re-keygen if session lost | Creates new wallet | [`keygen.rs:37`](../../gotham-client/src/ecdsa/keygen.rs#L37) |
| 4 | Check server DB for session | N/A | DB inspection |

### Session Recovery

If server was restarted and DB was lost:

```bash
# Client must re-generate keys
# WARNING: This creates a NEW wallet address

# 1. Backup old wallet
cp wallet.json wallet.json.old

# 2. Re-run keygen
# (Application-specific command)
```

**Important**: Re-keygen creates a new key pair. Funds at old addresses require the old key share.

### Message Hash Verification

For Bitcoin signing:

```rust
// Ensure message is the sighash, not raw transaction
let sig_hash = SigHashCache::new(&transaction).signature_hash(
    idx,
    script_code,
    value,
    SigHashType::All,
);
let message = BigInt::from(&sig_hash[..]);
```

Evidence: [`bitcoin/mod.rs:336-346`](../../demo-wallet/src/bitcoin/mod.rs#L336-L346)

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|----------|
| All signing operations failing | Engineering Lead | #eng-oncall |
| Session data missing from DB | Database Admin | #db-oncall |
| Invalid signatures produced | Security Team | #security |

## Post-Incident

- [ ] Document root cause in incident report
- [ ] Add session existence check before signing
- [ ] Consider session ID validation in client
- [ ] Schedule retrospective if user funds affected
