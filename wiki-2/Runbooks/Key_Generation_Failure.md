# Runbook: Key Generation Failure

## Header

| Field | Value |
|-------|-------|
| Severity | P1 - Critical |
| Owner | On-call engineer |
| Last Updated | 2025-12-06 |
| Review Cycle | Quarterly |

---

## Symptoms

- Client receives `None` response from keygen API calls
- HTTP 400/500 errors during key generation
- Protocol timeout (request hangs)
- Panic in client library (`unwrap()` failure)
- "Protocol error" or "pdl error" messages

---

## Diagnosis

### Step 1: Verify Server Health

```bash
# Check if server is running
curl -f http://localhost:8000/ecdsa/keygen/first -d '{}' -H "Content-Type: application/json"

# Expected: JSON response with session ID
# Error: Connection refused = server down
# Error: 500 = server error
```

### Step 2: Check Server Logs

```bash
# If running directly
journalctl -u gotham-server -n 100

# If running in Docker
docker logs gotham-server --tail 100

# Look for:
# - Panic messages
# - DB errors
# - Crypto failures
```

### Step 3: Verify Database Access

```bash
# Check RocksDB directory exists and is writable
ls -la /path/to/gotham-server/db/

# Check disk space
df -h /path/to/gotham-server/
```

### Step 4: Check Network Connectivity

```bash
# From client machine
nc -zv server-host 8000

# Check for firewall issues
iptables -L -n | grep 8000
```

### Step 5: Identify Specific Round Failure

| Round | Endpoint | Common Failures |
|-------|----------|-----------------|
| 1 | `/ecdsa/keygen/first` | Server startup, DB init |
| 2 | `/ecdsa/keygen/{id}/second` | Proof verification, Paillier |
| 3 | `/ecdsa/keygen/{id}/third` | Session not found |
| 4 | `/ecdsa/keygen/{id}/fourth` | PDL verification |
| CC1 | `/ecdsa/keygen/{id}/chaincode/first` | Session state |
| CC2 | `/ecdsa/keygen/{id}/chaincode/second` | Chain code proof |

---

## Resolution

### Server Not Running

```bash
# Restart server
cd /path/to/gotham-server
cargo run --release

# Or via systemd
sudo systemctl restart gotham-server
```

### Database Corruption

```bash
# Backup corrupted DB
mv db/ db.backup.$(date +%Y%m%d)

# Server will create new DB on restart
# WARNING: Existing key shares will be lost
```

### Out of Disk Space

```bash
# Free space
sudo apt-get clean
docker system prune -f

# Move DB to larger volume
mv db/ /mnt/larger-volume/db
ln -s /mnt/larger-volume/db db
```

### Session Not Found (Round 2+)

- Cause: Server restarted between rounds, or wrong session ID
- Resolution: Client must restart keygen from round 1

### Proof Verification Failed

- Cause: Client bug, network corruption, or attack
- Resolution: 
  1. Verify client library version matches server
  2. Check for network issues (proxy, firewall)
  3. Retry from round 1

---

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|---------|
| Server won't start | Infrastructure team | #infra-oncall |
| Persistent crypto failures | Security team | security@company.com |
| Multiple users affected | Engineering lead | #eng-leads |

---

## Post-Incident

1. [ ] Document root cause
2. [ ] Update monitoring if detection was delayed
3. [ ] Consider adding retry logic if applicable
4. [ ] Review affected sessions for data integrity
5. [ ] Communicate resolution to affected users
