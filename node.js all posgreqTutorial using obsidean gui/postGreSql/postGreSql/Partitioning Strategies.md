# Partitioning Strategies

## Types of Partitioning
### Range Partitioning
```sql
-- Create partitioned table
CREATE TABLE measurements (
    id SERIAL,
    timestamp TIMESTAMP NOT NULL,
    device_id INTEGER,
    temperature NUMERIC,
    humidity NUMERIC
) PARTITION BY RANGE (timestamp);

-- Create partitions
CREATE TABLE measurements_2023 
    PARTITION OF measurements
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE measurements_2024 
    PARTITION OF measurements
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Automatically create partitions
CREATE OR REPLACE FUNCTION create_partition_and_insert()
RETURNS trigger AS $$
BEGIN
    -- Create partition if it doesn't exist
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS measurements_%s 
         PARTITION OF measurements
         FOR VALUES FROM (%L) TO (%L)',
        to_char(NEW.timestamp, 'YYYY'),
        date_trunc('year', NEW.timestamp),
        date_trunc('year', NEW.timestamp) + interval '1 year'
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER measurements_insert_trigger
    BEFORE INSERT ON measurements
    FOR EACH ROW
    EXECUTE FUNCTION create_partition_and_insert();
```

### List Partitioning
```sql
-- Create partitioned table by region
CREATE TABLE sales (
    id SERIAL,
    order_date DATE,
    region TEXT,
    amount NUMERIC
) PARTITION BY LIST (region);

-- Create partitions for different regions
CREATE TABLE sales_north 
    PARTITION OF sales
    FOR VALUES IN ('NORTH', 'NORTHEAST', 'NORTHWEST');

CREATE TABLE sales_south 
    PARTITION OF sales
    FOR VALUES IN ('SOUTH', 'SOUTHEAST', 'SOUTHWEST');
```

### Hash Partitioning
```sql
-- Create hash-partitioned table
CREATE TABLE users (
    id SERIAL,
    username TEXT,
    email TEXT,
    created_at TIMESTAMP
) PARTITION BY HASH (id);

-- Create 4 hash partitions
CREATE TABLE users_0 
    PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE users_1 
    PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE users_2 
    PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE users_3 
    PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Partition Management
### Adding New Partitions
```sql
-- Add new partition for future data
CREATE TABLE measurements_2025 
    PARTITION OF measurements
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

### Detaching Partitions
```sql
-- Detach old partition
ALTER TABLE measurements 
    DETACH PARTITION measurements_2023;

-- Convert to regular table
ALTER TABLE measurements_2023 
    ADD PRIMARY KEY (id);
```

### Dropping Partitions
```sql
-- Drop old partition
DROP TABLE measurements_2023;
```

## Partition Pruning
### Query Examples
```sql
-- Efficient query (uses partition pruning)
EXPLAIN ANALYZE
SELECT * FROM measurements 
WHERE timestamp >= '2024-01-01' 
AND timestamp < '2024-02-01';

-- Create indexes on partitions
CREATE INDEX ON measurements_2024 (device_id, timestamp);
```

## Sub-partitioning
```sql
-- Create table with multiple partition levels
CREATE TABLE sales_data (
    id SERIAL,
    sale_date DATE,
    region TEXT,
    product_id INTEGER,
    amount NUMERIC
) PARTITION BY RANGE (sale_date);

-- Create yearly partition
CREATE TABLE sales_2024 
    PARTITION OF sales_data
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    PARTITION BY LIST (region);

-- Create sub-partitions for regions
CREATE TABLE sales_2024_north 
    PARTITION OF sales_2024
    FOR VALUES IN ('NORTH', 'NORTHEAST');

CREATE TABLE sales_2024_south 
    PARTITION OF sales_2024
    FOR VALUES IN ('SOUTH', 'SOUTHEAST');
```

## Best Practices
1. Choose Partition Key
   - High cardinality
   - Even distribution
   - Query patterns

2. Partition Size
   - Not too many partitions
   - Not too few partitions
   - Consider maintenance

3. Maintenance
```sql
-- Analyze partitions
ANALYZE measurements_2024;

-- Vacuum partitions
VACUUM measurements_2024;

-- Check partition sizes
SELECT 
    partition_name,
    pg_size_pretty(pg_relation_size(partition_name::regclass)) as size
FROM (
    SELECT tablename as partition_name
    FROM pg_tables
    WHERE tablename LIKE 'measurements_%'
) t;
```

## Common Use Cases
1. Time-Series Data
   - Log data
   - Sensor readings
   - Financial transactions

2. Geographic Data
   - Regional sales
   - Location-based services

3. Multi-tenant Applications
   - Customer data isolation
   - Performance optimization

## Related Topics
- [[Performance Tuning]]
- [[Database Design]]
- [[Query Optimization]]
- [[Time Series Data]]
