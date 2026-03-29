---
name: backup-and-restore
description: >
  Backup strategies and disaster recovery procedures for databases.
  Use when: database backup, backup strategy, disaster recovery, restore procedure, backup validation,
  point in time recovery, PITR, pg_dump, pg_basebackup, mysqldump, xtrabackup, WAL archiving,
  incremental backup, full backup, backup retention, backup testing, backup encryption,
  RTO RPO, recovery time objective, recovery point objective, offsite backup, backup monitoring,
  restore drill, backup automation, S3 backup, backup compression, backup integrity, DR plan.
argument-hint: >
  Describe the database and recovery requirements (e.g., "PostgreSQL 16, 500GB, RPO < 1 hour,
  RTO < 30 minutes, multi-region requirements"), and specify the current backup tooling or gap being addressed.
---

# Backup and Restore Specialist

## When to Use

Invoke this skill when you need to:
- Design or review a database backup strategy against RPO/RTO requirements
- Select and configure backup tools (pg_basebackup, WAL archiving, xtrabackup)
- Implement backup automation, monitoring, and alerting
- Design and execute a restore drill to validate recoverability
- Define backup retention policies and offsite storage requirements
- Build a disaster recovery runbook

---

## Step 1 — Define Recovery Objectives

Before choosing any backup technology, define objectives with stakeholders:

| Objective | Definition | Example Target |
|---|---|---|
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss window | 1 hour — lose at most 1h of data |
| **RTO** (Recovery Time Objective) | Maximum acceptable recovery time | 30 minutes — system back online within 30m |

**RPO → Backup frequency mapping:**
| RPO | Required Backup Strategy |
|---|---|
| < 5 minutes | Continuous WAL archiving / binlog streaming |
| < 1 hour | WAL archiving with hourly base backup rotation |
| < 24 hours | Daily full backup |
| < 7 days | Weekly full + daily incremental |

Checklist:
- [ ] RPO agreed with product/business stakeholders
- [ ] RTO agreed with engineering and operations
- [ ] Objectives documented in the DR plan and reviewed annually
- [ ] Recovery objectives tested empirically — not assumed from theory

---

## Step 2 — Choose Backup Types

| Type | Description | Tool Examples |
|---|---|---|
| **Logical backup** | SQL dump of objects and data | `pg_dump`, `mysqldump`, `mongodump` |
| **Physical / file-level backup** | Binary copy of data directory | `pg_basebackup`, `xtrabackup`, file snapshot |
| **Snapshot backup** | Cloud volume snapshot | AWS EBS Snapshot, Azure Managed Disk Snapshot |
| **WAL archiving / binlog streaming** | Continuous log shipping for PITR | `archive_command`, `pgBackRest`, Barman |
| **Managed backup** | Cloud-native automated backup | AWS RDS automated backup, GCP Cloud SQL |

**Decision guide:**

| Use Case | Recommended |
|---|---|
| Small DB (< 10GB), simple restore, portability | Logical (`pg_dump`) |
| Large DB, fast restore, PITR required | Physical + WAL archiving |
| Cloud-hosted DB with managed service | Cloud snapshot + WAL archiving |
| Continuous RPO < 15 min | WAL archiving (streaming) |

Checklist:
- [ ] Backup type matches the RPO and RTO requirements
- [ ] Logical + physical backups used together for flexibility (logical for selective restore, physical for speed)
- [ ] Managed backups not the only backup — test that they can be restored independently

---

## Step 3 — Configure PostgreSQL Backup

**Logical backup (pg_dump):**
```bash
pg_dump \
  --host=localhost \
  --username=postgres \
  --format=custom \          # compressed, parallel-restore capable
  --file=/backups/mydb_$(date +%Y%m%d_%H%M%S).dump \
  mydb
```

**Physical backup (pg_basebackup):**
```bash
pg_basebackup \
  --host=localhost \
  --username=replication_user \
  --pgdata=/backups/basebackup_$(date +%Y%m%d) \
  --format=tar \
  --gzip \
  --checkpoint=fast \
  --wal-method=stream      # include WAL needed to make backup consistent
```

**WAL archiving for PITR (postgresql.conf):**
```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=mydb archive-push %p'
# or: aws s3 cp %p s3://my-wal-archive/%f
wal_level = replica
```

**pgBackRest (recommended for production):**
```bash
pgbackrest --stanza=mydb --type=full backup    # Full backup
pgbackrest --stanza=mydb --type=incr backup    # Incremental
pgbackrest --stanza=mydb --type=diff backup    # Differential
```

Checklist:
- [ ] `--format=custom` used for `pg_dump` — enables parallel restore with `pg_restore -j`
- [ ] WAL archiving configured and archive destination verified as accessible
- [ ] `archive_command` failure tested — PostgreSQL stops if archiving fails repeatedly (protects WAL)
- [ ] Base backup taken at minimum daily; WAL archived continuously

---

## Step 4 — Configure MySQL Backup

**Logical backup:**
```bash
mysqldump \
  --single-transaction \    # consistent snapshot without locking (InnoDB)
  --master-data=2 \         # record binlog position in dump header
  --databases mydb \
  | gzip > /backups/mydb_$(date +%Y%m%d).sql.gz
```

**Physical backup (xtrabackup — preferred for large DBs):**
```bash
xtrabackup --backup --target-dir=/backups/full_$(date +%Y%m%d) --user=root --password=...
xtrabackup --prepare --target-dir=/backups/full_$(date +%Y%m%d)
```

**Binlog archiving:**
```ini
# my.cnf
log_bin = /var/lib/mysql/binlog
expire_logs_days = 7          # or: binlog_expire_logs_seconds = 604800
binlog_format = ROW           # required for PITR
```

Checklist:
- [ ] `--single-transaction` used for `mysqldump` — avoids table locks on InnoDB
- [ ] `xtrabackup` used for databases > 10GB — faster and non-blocking
- [ ] Binlog archiving enabled and binlogs stored offsite for PITR
- [ ] `binlog_format = ROW` confirmed — required for reliable PITR

---

## Step 5 — Backup Storage, Encryption, and Retention

**Storage:**
- Primary: local or NAS (fastest restore)
- Offsite: S3, GCS, Azure Blob (disaster recovery)
- Multi-region: replicate backup to a second region for regional failure coverage

**Encryption:**
```bash
# Encrypt before upload:
openssl enc -aes-256-cbc -pbkdf2 -in dump.sql -out dump.sql.enc -k "$BACKUP_PASSPHRASE"
# Or: pgBackRest built-in encryption with cipher-type=aes-256-cbc
```

**Retention schedule:**
| Period | Frequency | Retention |
|---|---|---|
| Daily | Every 24h | 7 days |
| Weekly | Every 7 days | 4 weeks |
| Monthly | Every 30 days | 12 months |
| Annual | January 1 | 7 years (compliance) |

Checklist:
- [ ] Backups encrypted at rest; encryption key stored separately from the backup
- [ ] Backups stored in at least two geographic locations
- [ ] Retention policy enforced automatically (S3 lifecycle policy, pgBackRest retention config)
- [ ] Backup storage access restricted — backup files not readable by application service accounts
- [ ] Backup storage alerts configured if backup size drops unexpectedly (may indicate failed backup)

---

## Step 6 — Restore Procedures

**PostgreSQL PITR restore:**
```bash
# 1. Stop the database
pg_ctl stop -D /var/lib/postgresql/data

# 2. Restore base backup
pgbackrest --stanza=mydb --delta restore

# 3. Configure recovery target (postgresql.conf / recovery.conf)
echo "restore_command = 'pgbackrest --stanza=mydb archive-get %f %p'" >> recovery.conf
echo "recovery_target_time = '2026-03-29 14:30:00 UTC'" >> recovery.conf

# 4. Start the database — recovery applies WAL until target time
pg_ctl start -D /var/lib/postgresql/data
```

**MySQL PITR restore:**
```bash
# 1. Restore full backup
xtrabackup --copy-back --target-dir=/backups/full_20260329

# 2. Replay binlogs from the point recorded in the backup header
mysqlbinlog --start-position=<pos> /var/log/mysql/binlog.000042 ... | mysql
```

Checklist:
- [ ] Restore procedure documented step-by-step in a runbook (not from memory during incident)
- [ ] Runbook includes: which backup to use, how to decrypt, how to apply WAL/binlogs, how to verify
- [ ] Restore always targets a **non-production** environment during drills
- [ ] Post-restore data validation queries defined to verify integrity

---

## Step 7 — Backup Monitoring and Restore Drills

**Monitoring:**
- Alert if backup has not completed within the expected window
- Alert if backup size is < 80% of previous backup (possible data loss or failed dump)
- Alert if WAL archiving is lagging (affects PITR coverage)

**Restore drill:**
- Schedule restore drills at minimum quarterly
- Restore to an isolated environment
- Measure actual RTO during the drill
- Verify data integrity post-restore (row counts, checksums, application smoke test)
- Document any deviations from the expected runbook

Checklist:
- [ ] Backup success/failure alert configured and tested
- [ ] Restore drill scheduled quarterly — not just annually
- [ ] RTO measured during drill and compared to the SLO
- [ ] Drill results stored and reviewed — trend of recovery time tracked
- [ ] Backup encryption key rotation procedure tested during a drill

---

## Output Report

### Critical
- No backups exist or backups have never been verified as restorable
- Backup stored only in the same region/AZ as the primary database — regional failure loses both
- Backup unencrypted and stored in a publicly accessible location
- WAL archiving not configured — PITR is impossible; minimum RPO is the last full backup

### High
- Restore drills never performed — RTO is unknown and runbook may be wrong
- `mysqldump` used without `--single-transaction` — table locks during backup window
- Backup retention not enforced — storage grows indefinitely or old backups overwritten too soon

### Medium
- Backup monitoring not configured — silent failures possible for days
- No offsite copy — local disaster destroys both primary and backup
- `pg_dump` used for large databases — restore time exceeds RTO

### Low
- Backup files not compressed — unnecessary storage cost
- Restore runbook not documented — recovery improvised during incident
- Backup encryption key stored adjacent to backup files — encryption provides no protection

### Passed
- RPO and RTO defined, documented, and tested through restore drills
- Physical + WAL archiving configured for PITR with RPO < target
- Backups encrypted, stored offsite in at least two regions, and retention enforced
- Backup success/failure monitored with alerts
- Restore drills conducted quarterly with measured and improving RTO
