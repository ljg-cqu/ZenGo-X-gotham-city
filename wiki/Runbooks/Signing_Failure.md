# Runbook: Signing Failure

## Header

| Field | Value |
|-------|-------|
| Severity | P1 - Critical |
| Owner | On-call engineer |
| Last Updated | 2025-12-06 |
| Review Cycle | Quarterly |

---

## Symptoms

- Client receives `None` response from sign API calls
- HTTP 400 error: "Key not found" or "Invalid session"
- HTTP 500 error during signing
- Transaction fails to broadcast despite successful sign call
- Invalid signature (wrong r, s, or recid values)

---

## Diagnosis

### Step 1: Verify Session ID Exists

```bash
# Check if session exists on server
# (Requires DB access - implementation specific)

# For RocksDB: check if keys with session prefix exist
ls -la /path/to/db/

# Look for files containing session ID patterns
```

### Step 2: Verify Server Health

```bash
# Test sign endpoint with known good session
curl -X POST http://localhost:8000/ecdsa/sign/{session_id}/first \
  -H "Content-Type: application/json" \
  -d '{"d_log_proof": "...", "public_share": "..."}'

# Expected: JSON response
# Error 400: Session not found
# Error 500: Server error
```

### Step 3: Check Authorization

```bash
# Verify auth token (if implemented)
# Check Db::granted() implementation

# Default implementation always returns true
# Custom implementations may reject
```

### Step 4: Validate Input Data

| Field | Validation | Common Issues |
|-------|------------|---------------|
| `message` | Valid hex-encoded BigInt | Wrong encoding, empty |
| `x_pos_child_key` | Valid BigInt | Out of range |
| `y_pos_child_key` | Valid BigInt | Out of range |
| `party_two_sign_message` | Valid JSON structure | Corrupted client state |

### Step 5: Check Signature Output

```bash
# Verify signature format
# r: 32-byte hex
# s: 32-byte hex  
# recid: 0 or 1

# If values look wrong, check:
# - Message hash computation
# - HD derivation indices
# - Client master key integrity
```

---

## Resolution

### Session Not Found

**Cause**: Key share was deleted, server restarted with new DB, or wrong session ID

**Resolution**:
1. Verify session ID is correct (from keygen output)
2. If key share lost, user must re-keygen
3. Restore from backup if available

```bash
# If backup exists
cp -r db.backup/ db/
sudo systemctl restart gotham-server
```

### Authorization Rejected

**Cause**: Custom `granted()` implementation rejected the request

**Resolution**:
1. Check authorization policy
2. Verify customer ID matches
3. Check rate limits
4. Review auth token validity

### Invalid Signature

**Cause**: Wrong message hash, wrong HD indices, or protocol error

**Resolution**:
1. Verify message hash computation:
   - Bitcoin: BIP143 sighash
   - Ethereum: Keccak256 with EIP-155
2. Verify HD derivation indices match address
3. Check master key integrity (re-load from storage)

### Server Error During Signing

**Resolution**:
1. Check server logs for panic/error details
2. Verify DB accessibility
3. Check memory/CPU resources
4. Restart server if needed

---

## Transaction-Specific Issues

### Bitcoin Transaction Fails

| Issue | Cause | Resolution |
|-------|-------|------------|
| Invalid signature | Wrong sighash type | Use SIGHASH_ALL |
| Insufficient funds | UTXO changed | Refresh UTXOs |
| Dust output | Amount too small | Increase output amount |

### Ethereum Transaction Fails

| Issue | Cause | Resolution |
|-------|-------|------------|
| Invalid v value | Wrong chain ID | Verify EIP-155 encoding |
| Nonce too low | Concurrent tx | Get fresh nonce |
| Insufficient gas | Gas limit too low | Increase gas limit |

---

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|---------|
| Key shares missing | Security team | security@company.com |
| Persistent failures | Engineering lead | #eng-leads |
| Funds at risk | Incident commander | #incident-response |

---

## Post-Incident

1. [ ] Document root cause
2. [ ] Verify no funds were lost
3. [ ] Check for duplicate signing attempts
4. [ ] Review affected transactions on blockchain
5. [ ] Update client-side error handling if applicable
6. [ ] Communicate resolution to affected users
