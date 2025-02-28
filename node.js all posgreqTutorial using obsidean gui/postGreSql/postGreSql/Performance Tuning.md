# Performance Tuning

## Query Analysis
### EXPLAIN ANALYZE
```sql
-- Basic usage
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;

-- With buffers and timing
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT * 
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'completed';
```

### Understanding Query Plans
1. Scan Types
   - Sequential Scan
   - Index Scan
   - Bitmap Scan
   - Index Only Scan

2. Join Types
   - Nested Loop
   - Hash Join
   - Merge Join

## Index Optimization
### Index Types
```sql
-- B-tree (default)
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Hash
CREATE INDEX idx_users_email ON users USING HASH (email);

-- GiST
CREATE INDEX idx_locations ON places USING GIST (location);

-- GIN
CREATE INDEX idx_tags ON posts USING GIN (tags);
```

### Composite Indexes
```sql
-- Multi-column index
CREATE INDEX idx_orders_customer_date 
ON orders(customer_id, order_date);

-- Partial index
CREATE INDEX idx_active_users 
ON users(last_login) 
WHERE status = 'active';

-- Expression index
CREATE INDEX idx_lower_email 
ON users(LOWER(email));
```

## Configuration Tuning
### Memory Settings
```ini
# postgresql.conf

# Buffer settings
shared_buffers = 2GB              # 25% of RAM
work_mem = 64MB                   # For complex sorts
maintenance_work_mem = 256MB      # For maintenance operations
effective_cache_size = 6GB        # 75% of RAM

# Connection settings
max_connections = 100
```

### Write Ahead Log (WAL)
```ini
# WAL settings
wal_level = replica
max_wal_size = 1GB
min_wal_size = 80MB
checkpoint_timeout = 15min
```

## Table Maintenance
### VACUUM
```sql
-- Basic vacuum
VACUUM orders;

-- Full vacuum
VACUUM FULL orders;

-- Analyze
VACUUM ANALYZE orders;

-- Automatic vacuum settings
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_scale_factor = 0.05
);
```

### Table Statistics
```sql
-- Update statistics
ANALYZE orders;

-- View table statistics
SELECT * FROM pg_stat_user_tables
WHERE relname = 'orders';
```

## Query Optimization Techniques
### Rewriting Queries
```sql
-- Bad: Subquery in WHERE
SELECT * FROM orders 
WHERE customer_id IN (
    SELECT id FROM customers WHERE country = 'US'
);

-- Better: JOIN
SELECT o.* FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US';
```

### Using CTEs Effectively
```sql
-- Materialized CTE
WITH MATERIALIZED customer_totals AS (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    GROUP BY customer_id
)
SELECT * FROM customer_totals
WHERE total_spent > 1000;
```

## Monitoring
### Active Queries
```sql
-- Find long-running queries
SELECT 
    pid,
    now() - query_start as duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

### Index Usage
```sql
-- Check index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes;
```

## Common Performance Issues
1. Missing Indexes
```sql
-- Find tables with sequential scans
SELECT 
    relname,
    seq_scan,
    seq_tup_read
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

2. Bloated Tables
```sql
-- Estimate table bloat
SELECT 
    schemaname, tablename, 
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as size,
    n_dead_tup as dead_tuples
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

## Related Topics
- [[Query Optimization]]
- [[Indexing and Performance]]
- [[Database Administration]]
- [[Best Practices]]
