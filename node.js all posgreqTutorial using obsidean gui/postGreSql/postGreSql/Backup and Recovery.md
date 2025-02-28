# Backup and Recovery

## Backup Types
### Logical Backups (pg_dump)
```bash
# Backup single database
pg_dump dbname > backup.sql

# Backup specific tables
pg_dump -t table1 -t table2 dbname > tables_backup.sql

# Custom format backup (compressed)
pg_dump -Fc dbname > backup.dump

# Directory format backup
pg_dump -Fd dbname -f backup_dir/
```

### Physical Backups
```bash
# Stop PostgreSQL
pg_ctl stop

# Copy data directory
cp -R $PGDATA backup_dir/

# Start PostgreSQL
pg_ctl start
```

## Backup Strategies
### Full Database Backup
```bash
# Backup all databases
pg_dumpall > full_backup.sql

# With roles and tablespaces
pg_dumpall --roles-only > roles.sql
pg_dumpall --tablespaces-only > tablespaces.sql
```

### Continuous Archiving
```bash
# In postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /archive/%f'

# Take base backup
pg_basebackup -D backup_dir -Ft -z
```

## Recovery Methods
### Simple Recovery
```bash
# Restore from SQL dump
psql dbname < backup.sql

# Restore from custom format
pg_restore -d dbname backup.dump

# Restore specific tables
pg_restore -t table1 -d dbname backup.dump
```

### Point-in-Time Recovery (PITR)
```bash
# In recovery.conf
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2023-12-31 23:59:59'
```

## Backup Best Practices
1. Regular Scheduling
```bash
# Cron job for daily backup
0 0 * * * pg_dump dbname > /backups/daily/backup_$(date +%Y%m%d).sql
```

2. Compression
```bash
# Compress backup
pg_dump dbname | gzip > backup.sql.gz

# Decompress and restore
gunzip -c backup.sql.gz | psql dbname
```

3. Validation
```bash
# Test restore
psql -f backup.sql test_restore_db

# Verify backup
pg_restore --list backup.dump
```

## Monitoring and Maintenance
### Backup Size
```sql
SELECT pg_size_pretty(pg_database_size('dbname'));
```

### Backup History
```sql
SELECT * FROM pg_stat_archiver;
```

## Disaster Recovery
1. Recovery Time Objective (RTO)
2. Recovery Point Objective (RPO)
3. Disaster Recovery Plan
4. Testing Recovery Procedures

## Common Issues
1. Insufficient Disk Space
2. Backup Corruption
3. Long Backup Times
4. Recovery Failures

## Related Topics
- [[Database Administration]]
- [[Security and Access Control]]
- [[Performance Monitoring]]
- [[High Availability]]
