# Runbook: Database Recovery

## Header

| Field | Value |
|-------|-------|
| Severity | P1 - Critical |
| Owner | Infrastructure team |
| Last Updated | 2025-12-06 |
| Review Cycle | Quarterly |

---

## Symptoms

- Server fails to start with DB errors
- RocksDB corruption messages in logs
- Missing key shares (sessions exist but data lost)
- "DB open failed" or "SST file corrupted" errors

---

## Diagnosis

### Step 1: Identify Corruption Type

```bash
# Check RocksDB logs
cat /path/to/db/LOG | tail -100

# Look for:
# - "Corruption" messages
# - "SST file" errors
# - "MANIFEST" errors
```

### Step 2: Verify Backup Availability

```bash
# Check backup location
ls -la /path/to/backups/

# Verify backup integrity
ls -la /path/to/backups/db.backup.YYYYMMDD/
```

### Step 3: Assess Data Loss Scope

```bash
# Count keys in corrupted DB (if readable)
# This requires RocksDB tools

# Or check backup age
stat /path/to/backups/db.backup.latest/

# Determine sessions created since last backup
# These will be lost in recovery
```

---

## Resolution

### Option 1: Repair In-Place (Minor Corruption)

```bash
# Stop server
sudo systemctl stop gotham-server

# Backup current state
cp -r db/ db.corrupted.$(date +%Y%m%d)

# Attempt repair with RocksDB tools
ldb repair --db=db/

# Restart server
sudo systemctl start gotham-server

# Verify functionality
curl http://localhost:8000/ecdsa/keygen/first -d '{}'
```

### Option 2: Restore from Backup (Major Corruption)

```bash
# Stop server
sudo systemctl stop gotham-server

# Backup corrupted DB for analysis
mv db/ db.corrupted.$(date +%Y%m%d)

# Restore from backup
cp -r /path/to/backups/db.backup.latest/ db/

# Restart server
sudo systemctl start gotham-server

# Verify functionality
curl http://localhost:8000/ecdsa/keygen/first -d '{}'
```

### Option 3: Fresh Start (Complete Loss)

⚠️ **WARNING**: This destroys all existing key shares. Users will need to re-keygen.

```bash
# Stop server
sudo systemctl stop gotham-server

# Archive corrupted DB
mv db/ db.corrupted.$(date +%Y%m%d)

# Server will create new DB on start
sudo systemctl start gotham-server

# Notify affected users
# They must generate new keys
```

---

## Backup Procedures

### Manual Backup

```bash
# Stop server for consistent backup
sudo systemctl stop gotham-server

# Create backup
cp -r db/ /path/to/backups/db.backup.$(date +%Y%m%d)

# Restart server
sudo systemctl start gotham-server
```

### Automated Backup Script

```bash
#!/bin/bash
# /opt/gotham/backup.sh

BACKUP_DIR="/path/to/backups"
DB_DIR="/path/to/gotham-server/db"
RETENTION_DAYS=30

# Create backup (server running - may have slight inconsistency)
cp -r $DB_DIR $BACKUP_DIR/db.backup.$(date +%Y%m%d_%H%M%S)

# Or use RocksDB checkpoint for online backup
# ldb checkpoint --db=$DB_DIR --checkpoint_dir=$BACKUP_DIR/checkpoint.$(date +%Y%m%d)

# Clean old backups
find $BACKUP_DIR -name "db.backup.*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;
```

### Cron Schedule

```bash
# Daily backup at 2 AM
0 2 * * * /opt/gotham/backup.sh >> /var/log/gotham-backup.log 2>&1
```

---

## Prevention

### Monitoring

- [ ] Monitor disk space usage
- [ ] Alert on DB size growth anomalies
- [ ] Monitor for RocksDB error patterns in logs
- [ ] Verify backup completion daily

### Best Practices

- [ ] Regular backup schedule (daily minimum)
- [ ] Test backup restoration quarterly
- [ ] Use RAID or replicated storage
- [ ] Encrypt backups at rest
- [ ] Store backups in separate location/region

---

## Escalation

| Condition | Escalate To | Contact |
|-----------|-------------|---------|
| No viable backup | Security team + Engineering lead | #incident-response |
| Data loss affects production | Incident commander | #incident-response |
| Corruption cause unclear | Database specialist | #data-team |

---

## Post-Incident

1. [ ] Document root cause (disk failure, software bug, etc.)
2. [ ] Identify affected sessions/users
3. [ ] Communicate data loss to affected users
4. [ ] Review and improve backup procedures
5. [ ] Implement additional monitoring
6. [ ] Schedule post-mortem meeting
7. [ ] Update this runbook with lessons learned
