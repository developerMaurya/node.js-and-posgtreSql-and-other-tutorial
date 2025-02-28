# Database Administration

## Installation and Setup
### Installation
```bash
# Windows
choco install postgresql

# Linux
sudo apt-get install postgresql

# macOS
brew install postgresql
```

### Initial Configuration
```bash
# Initialize database cluster
initdb -D /path/to/data

# Start PostgreSQL
pg_ctl start -D /path/to/data

# Stop PostgreSQL
pg_ctl stop -D /path/to/data
```

## Configuration Management
### postgresql.conf
```ini
# Memory Configuration
shared_buffers = 2GB
work_mem = 64MB
maintenance_work_mem = 256MB

# Connections
max_connections = 100
superuser_reserved_connections = 3

# Write Ahead Log
wal_level = replica
max_wal_size = 1GB
min_wal_size = 80MB
```

### pg_hba.conf
```ini
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all            all                                     trust
host    all            all             127.0.0.1/32            md5
host    all            all             ::1/128                 md5
```

## Routine Maintenance
### VACUUM
```sql
-- Basic vacuum
VACUUM table_name;

-- Full vacuum
VACUUM FULL table_name;

-- Analyze
VACUUM ANALYZE table_name;

-- Automatic vacuum settings
ALTER TABLE table_name SET (
    autovacuum_vacuum_threshold = 50,
    autovacuum_analyze_threshold = 50,
    autovacuum_vacuum_scale_factor = 0.2
);
```

### Database Statistics
```sql
-- Update statistics
ANALYZE table_name;

-- View table statistics
SELECT * FROM pg_stat_user_tables;

-- View index usage
SELECT * FROM pg_stat_user_indexes;
```

## Monitoring
### Active Queries
```sql
SELECT pid, query, query_start, state
FROM pg_stat_activity
WHERE state != 'idle';
```

### Lock Information
```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype;
```

### Resource Usage
```sql
-- Database size
SELECT pg_size_pretty(pg_database_size('dbname'));

-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables;
```

## Performance Tuning
### Memory Settings
```sql
-- Show current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;

-- Modify settings
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET work_mem = '64MB';
```

### Connection Settings
```sql
-- Check current connections
SELECT count(*) FROM pg_stat_activity;

-- Set connection limits
ALTER SYSTEM SET max_connections = '100';
```

## Troubleshooting
### Common Issues
1. Connection Problems
2. Slow Queries
3. High CPU Usage
4. Disk Space Issues

### Diagnostic Queries
```sql
-- Find long-running queries
SELECT pid, now() - query_start AS duration, query 
FROM pg_stat_activity 
WHERE state = 'active';

-- Find bloated tables
SELECT schemaname, tablename, n_dead_tup 
FROM pg_stat_user_tables 
ORDER BY n_dead_tup DESC;
```

## Best Practices
1. Regular Backups
2. Monitoring and Alerting
3. Performance Optimization
4. Security Updates
5. Documentation

## Related Topics
- [[Backup and Recovery]]
- [[Security and Access Control]]
- [[Performance Monitoring]]
- [[Configuration Management]]
