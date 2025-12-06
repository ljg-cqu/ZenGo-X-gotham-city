# Key Generation Failure

**Severity**: P2  
**Last Tested**: 2025-12-05  
**Owner**: Platform Team

## Symptoms

- Client receives HTTP 500 or connection timeout during keygen
- Keygen latency exceeds 2000ms consistently
- Client logs show "party1 keygen message request failed"
- Incomplete wallet creation (missing `PrivateShare`)

## Diagnosis

| Step | Command/Action | Expected Output | Evidence |
|------|----------------|-----------------|----------|
| 1 | Check server is running | Process active, port 8000 listening | `ps aux \| grep gotham` |
| 2 | Check server logs | No panic/crash messages | `tail -f gotham-server.log` |
| 3 | Test basic connectivity | HTTP 404 (no /health endpoint) | `curl http://localhost:8000/` |
| 4 | Check DB access | RocksDB directory writable | `ls -la ./db/` |
| 5 | Check disk space | >10% free | `df -h` |
| 6 | Check memory | <90% used | `free -m` |

### Common Error Patterns

| Error | Cause | Solution |
|-------|-------|----------|
| "DB name is illegal" | Invalid `db_name` in Settings.toml | Use alphanumeric characters only |
| Connection refused | Server not running | Start server |
| 500 Internal Error | Crypto operation failed | Check logs for stack trace |
| Timeout | Network or server overload | Check latency, scale if needed |

## Resolution

| Step | Action | Rollback | Evidence |
|------|--------|----------|----------|
| 1 | Restart server | N/A | `cd gotham-server && cargo run` |
| 2 | Clear DB if corrupted | Restore from backup | `rm -rf ./db && restore_backup.sh` |
| 3 | Check Settings.toml | Revert to known good | [`Settings.toml`](../../gotham-server/Settings.toml) |
| 4 | Retry keygen from client | N/A | Client restart |

### Server Restart Procedure

```bash
# 1. Stop existing server (if running)
pkill -f gotham-server

# 2. Backup current DB (precaution)
cp -r ./db ./db.backup.$(date +%Y%m%d)

# 3. Start server
cd gotham-server && cargo run --release
```

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|----------|
| Server won't start after restart | Engineering Lead | #eng-oncall |
| Data corruption suspected | Database Admin | #db-oncall |
| Crypto panic in logs | Security Team | #security |

## Post-Incident

- [ ] Document root cause in incident report
- [ ] Add monitoring if gap identified (e.g., keygen latency alert)
- [ ] Update runbook if new failure mode discovered
- [ ] Schedule retrospective if P1 impact
