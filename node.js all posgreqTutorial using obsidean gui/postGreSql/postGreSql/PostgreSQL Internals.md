# PostgreSQL Internals

## Architecture Overview
### Process Structure
1. Postmaster Process
   - Main PostgreSQL server process
   - Listens for connections
   - Spawns backend processes

2. Backend Process
   - Handles client connections
   - Executes queries
   - Manages transactions

3. Background Processes
   - Background Writer
   - WAL Writer
   - Autovacuum Launcher
   - Stats Collector

### Memory Architecture
```text
Shared Memory
├── Shared Buffers
│   ├── Page Headers
│   └── Page Data
├── WAL Buffers
└── Caches
    ├── Catalog Cache
    └── Lock Tables

Local Memory
├── Work Memory
├── Maintenance Work Memory
└── Temp Buffers
```

## Storage Engine
### Table Structure
```sql
-- View table structure
SELECT * FROM pg_class 
WHERE relname = 'your_table';

-- View column information
SELECT * FROM pg_attribute 
WHERE attrelid = 'your_table'::regclass;
```

### Page Layout
```text
Page Structure (8KB default)
├── Page Header (24 bytes)
├── Item Pointers
└── Items (Tuples)
    ├── Item Header
    ├── Null Bitmap
    └── Data
```

### TOAST (The Oversized-Attribute Storage Technique)
```sql
-- Check TOAST tables
SELECT 
    relname as table_name,
    reltoastrelid::regclass as toast_table
FROM pg_class
WHERE reltoastrelid != 0;

-- TOAST storage strategies
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,   -- Will be TOASTed if large
    metadata JSONB  -- Will be TOASTed if large
);
```

## Buffer Management
### Shared Buffers
```sql
-- Check buffer usage
SELECT 
    c.relname,
    count(*) blocks,
    round(count(*) * 8192.0 / 1024/1024, 2) as mb
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY blocks DESC;

-- Buffer configuration
SHOW shared_buffers;
SHOW effective_cache_size;
```

### Buffer Replacement Strategy
1. Clock-sweep Algorithm
2. Background Writer
3. Checkpoint Process

```sql
-- Check checkpoint statistics
SELECT * FROM pg_stat_bgwriter;

-- Configure checkpoints
ALTER SYSTEM SET checkpoint_timeout = '5min';
ALTER SYSTEM SET max_wal_size = '1GB';
```

## Query Processing
### Parser
```sql
-- View parse tree (requires pg_debug)
EXPLAIN (VERBOSE true)
SELECT * FROM users WHERE id = 1;
```

### Planner/Optimizer
```sql
-- View query plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, o.order_date
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed';

-- Statistics used by planner
SELECT 
    schemaname,
    tablename,
    attname,
    inherited,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'your_table';
```

### Executor
```sql
-- Monitor statement execution
SELECT 
    pid,
    query,
    state,
    wait_event,
    wait_event_type
FROM pg_stat_activity
WHERE state != 'idle';
```

## WAL (Write-Ahead Logging)
### WAL Structure
```text
WAL Segment (16MB default)
├── WAL Records
    ├── Record Header
    ├── Record Data
    └── Record CRC
```

### WAL Configuration
```sql
-- WAL settings
SHOW wal_level;
SHOW wal_buffers;
SHOW checkpoint_timeout;

-- Monitor WAL
SELECT 
    pg_current_wal_lsn(),
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        pg_last_wal_replay_lsn()
    ) as replication_lag;
```

## System Catalogs
### Important Catalogs
```sql
-- Table information
SELECT * FROM pg_class;

-- Column information
SELECT * FROM pg_attribute;

-- Index information
SELECT * FROM pg_index;

-- Constraint information
SELECT * FROM pg_constraint;
```

### System Views
```sql
-- Database statistics
SELECT * FROM pg_stat_database;

-- Table statistics
SELECT * FROM pg_stat_user_tables;

-- Index statistics
SELECT * FROM pg_stat_user_indexes;
```

## Vacuum Processing
### Vacuum Operations
```sql
-- Manual vacuum
VACUUM (VERBOSE, ANALYZE) table_name;

-- Monitor vacuum progress
SELECT * FROM pg_stat_progress_vacuum;

-- Check dead tuples
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables;
```

### Autovacuum
```sql
-- Autovacuum settings
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;

-- Configure autovacuum per table
ALTER TABLE table_name SET (
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_scale_factor = 0.2
);
```

## Performance Monitoring
### System Statistics
```sql
-- Buffer statistics
SELECT * FROM pg_stat_bgwriter;

-- Database statistics
SELECT * FROM pg_stat_database;

-- Connection statistics
SELECT * FROM pg_stat_activity;
```

### Query Statistics
```sql
-- Statement statistics
SELECT * FROM pg_stat_statements;

-- Lock information
SELECT * FROM pg_locks;
```

## Best Practices
1. Configuration
   - Proper shared_buffers sizing
   - Effective WAL configuration
   - Appropriate autovacuum settings

2. Monitoring
   - Regular statistics collection
   - Performance metrics tracking
   - Resource usage monitoring

3. Maintenance
   - Regular VACUUM
   - Statistics updates
   - Index maintenance

## Related Topics
- [[Performance Tuning]]
- [[Database Administration]]
- [[Query Optimization]]
- [[Configuration Management]]
