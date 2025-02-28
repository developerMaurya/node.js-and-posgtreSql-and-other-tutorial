# PostgreSQL Best Practices

## Database Design
### Naming Conventions
```sql
-- Use snake_case for objects
CREATE TABLE user_profiles (
    user_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

-- Use plural for table names
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY
);
```

### Data Types
1. Use appropriate data types
   - INTEGER vs BIGINT
   - TEXT vs VARCHAR
   - TIMESTAMP vs DATE
   - UUID vs SERIAL

2. Consider space efficiency
```sql
-- Example of efficient data types
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,           -- Instead of BIGINT if < 2 billion users
    status SMALLINT,          -- Instead of INTEGER for small ranges
    description TEXT,         -- Instead of VARCHAR for variable text
    created_at TIMESTAMP     -- With timezone if needed
);
```

## Query Writing
### Efficient Queries
```sql
-- Bad: Using SELECT *
SELECT * FROM users;

-- Good: Specify needed columns
SELECT id, username, email FROM users;

-- Bad: Implicit joins
SELECT * FROM orders, users WHERE orders.user_id = users.id;

-- Good: Explicit joins
SELECT o.*, u.username 
FROM orders o
JOIN users u ON o.user_id = u.id;
```

### Indexing
```sql
-- Create indexes for frequently queried columns
CREATE INDEX idx_users_email ON users(email);

-- Create composite indexes for multiple columns
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Use partial indexes when appropriate
CREATE INDEX idx_active_users ON users(last_login) 
WHERE active = true;
```

## Security
### User Management
```sql
-- Create role with minimal privileges
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_password';
GRANT SELECT, INSERT ON table_name TO app_user;

-- Use connection pooling
max_connections = 100
```

### Data Protection
```sql
-- Enable SSL
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'

-- Use encrypted passwords
password_encryption = scram-sha-256
```

## Performance
### Configuration
```ini
# Memory settings
shared_buffers = 25% of RAM
work_mem = 64MB
maintenance_work_mem = 256MB

# Checkpoints
checkpoint_timeout = 15min
max_wal_size = 1GB
```

### VACUUM
```sql
-- Regular VACUUM
VACUUM ANALYZE;

-- Configure autovacuum
ALTER TABLE large_table SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_scale_factor = 0.05
);
```

## Backup and Recovery
### Backup Strategy
1. Regular full backups
2. Point-in-time recovery capability
3. Test restores regularly

```bash
# Daily backup script
pg_dump dbname > /backups/daily/backup_$(date +%Y%m%d).sql
```

## Monitoring
### Key Metrics
1. Connection count
2. Cache hit ratio
3. Transaction rate
4. Replication lag

```sql
-- Monitor active queries
SELECT pid, query, query_start
FROM pg_stat_activity
WHERE state = 'active';
```

## Development
### Version Control
1. Use migrations for schema changes
2. Document database changes
3. Test migrations before production

### Testing
1. Use test databases
2. Create test data
3. Performance testing

## Maintenance
### Regular Tasks
1. Index maintenance
2. Statistics updates
3. Log rotation
4. Disk space monitoring

## Scalability
### Horizontal Scaling
1. Read replicas
2. Connection pooling
3. Partitioning

### Vertical Scaling
1. Hardware upgrades
2. Configuration optimization
3. Query optimization

## Related Topics
- [[Database Administration]]
- [[Security and Access Control]]
- [[Performance Tuning]]
- [[Query Optimization]]
