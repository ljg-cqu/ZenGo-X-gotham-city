# Database Recovery

**Severity**: P1  
**Last Tested**: 2025-12-05  
**Owner**: Platform Team

## Symptoms

- Server fails to start with RocksDB errors
- "Unable to open database" or corruption errors in logs
- Key generation succeeds but signing fails (session not found)
- Disk full alerts on server

## Diagnosis

| Step | Command/Action | Expected Output | Evidence |
|------|----------------|-----------------|----------|
| 1 | Check DB directory exists | Directory present | `ls -la ./db/` |
| 2 | Check disk space | >10% free | `df -h .` |
| 3 | Check file permissions | Read/write for server user | `ls -la ./db/` |
| 4 | Check RocksDB integrity | No errors | `rocksdb_ldb --db=./db scan` |
| 5 | Check recent backups | Backup files exist | `ls -la ./backups/` |

### RocksDB Error Patterns

| Error | Cause | Solution |
|-------|-------|----------|
| "Corruption: block checksum mismatch" | Data corruption | Restore from backup |
| "IO error: No space left on device" | Disk full | Free space, expand disk |
| "IO error: Permission denied" | File permissions | Fix permissions |
| "lock hold by current process" | Stale lock file | Remove LOCK file |

## Resolution

### Option 1: Clear Stale Lock

If server crashed and left a lock file:

| Step | Action | Rollback | Evidence |
|------|--------|----------|----------|
| 1 | Stop any running server | N/A | `pkill -f gotham-server` |
| 2 | Remove lock file | N/A | `rm ./db/LOCK` |
| 3 | Restart server | N/A | `cargo run` |

### Option 2: Restore from Backup

| Step | Action | Rollback | Evidence |
|------|--------|----------|----------|
| 1 | Stop server | N/A | `pkill -f gotham-server` |
| 2 | Move corrupted DB | Keep for analysis | `mv ./db ./db.corrupted.$(date +%Y%m%d)` |
| 3 | Restore from backup | N/A | `cp -r ./backups/latest ./db` |
| 4 | Start server | N/A | `cargo run` |
| 5 | Verify operation | Test keygen/sign | Integration test |

### Option 3: Full Reset (Data Loss)

**⚠️ WARNING**: This destroys all key shares. Users will lose access to funds.

| Step | Action | Rollback | Evidence |
|------|--------|----------|----------|
| 1 | Stop server | N/A | `pkill -f gotham-server` |
| 2 | Archive corrupted DB | N/A | `mv ./db ./db.archived.$(date +%Y%m%d)` |
| 3 | Start fresh | N/A | `cargo run` (creates new DB) |
| 4 | Notify affected users | N/A | Communication plan |

### Disk Space Recovery

```bash
# 1. Check disk usage
du -sh ./db/

# 2. If DB is large, consider compaction
# (Requires RocksDB tools)
rocksdb_ldb --db=./db compact

# 3. Archive old backups
find ./backups -mtime +30 -delete
```

## Backup Procedure

Implement regular backups:

```bash
#!/bin/bash
# backup_db.sh - Run daily via cron

BACKUP_DIR="/backups/gotham"
DB_DIR="./db"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup
cp -r "$DB_DIR" "$BACKUP_DIR/db_$DATE"

# Keep only last 7 days
find "$BACKUP_DIR" -type d -mtime +7 -exec rm -rf {} +

# Create symlink to latest
ln -sfn "$BACKUP_DIR/db_$DATE" "$BACKUP_DIR/latest"
```

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|----------|
| Backup restoration fails | Database Admin | #db-oncall |
| Data corruption cause unknown | Engineering Lead | #eng-oncall |
| User funds affected | Executive + Legal | #incident-response |

## Post-Incident

- [ ] Document root cause (disk failure, corruption, etc.)
- [ ] Verify backup system is functioning
- [ ] Add disk space monitoring alert
- [ ] Consider redundant storage (RAID, cloud backup)
- [ ] Review RocksDB configuration for durability
- [ ] Schedule retrospective if data loss occurred
