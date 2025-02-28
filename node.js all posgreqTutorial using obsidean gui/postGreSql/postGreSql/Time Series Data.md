# Time Series Data

## TimescaleDB Integration
### Installation and Setup
```sql
-- Install TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create hypertable
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INTEGER,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION
);

-- Convert to hypertable
SELECT create_hypertable('sensor_data', 'time');
```

### Basic Operations
```sql
-- Insert data
INSERT INTO sensor_data (time, sensor_id, temperature, humidity)
VALUES 
    (NOW(), 1, 25.6, 60),
    (NOW() - interval '1 minute', 1, 25.5, 61);

-- Time-based queries
SELECT time_bucket('15 minutes', time) AS interval,
    avg(temperature) as avg_temp,
    max(temperature) as max_temp
FROM sensor_data
WHERE time > NOW() - interval '1 day'
GROUP BY interval
ORDER BY interval;
```

## Native Time Series Solutions
### Range Partitioning
```sql
-- Create partitioned table
CREATE TABLE metrics (
    timestamp TIMESTAMPTZ NOT NULL,
    metric_name TEXT,
    value DOUBLE PRECISION
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE metrics_2024_01 PARTITION OF metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE metrics_2024_02 PARTITION OF metrics
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

### Continuous Aggregates
```sql
-- Create materialized view for hourly aggregates
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', timestamp) as hour,
    metric_name,
    avg(value) as avg_value,
    max(value) as max_value,
    min(value) as min_value
FROM metrics
GROUP BY hour, metric_name;

-- Refresh policy
SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '1 month',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

## Common Time Series Patterns
### Rolling Windows
```sql
-- Moving average
SELECT 
    time,
    temperature,
    AVG(temperature) OVER (
        ORDER BY time
        ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
    ) as moving_avg
FROM sensor_data
WHERE sensor_id = 1
ORDER BY time DESC;

-- Time-based window
SELECT 
    time,
    temperature,
    AVG(temperature) OVER (
        ORDER BY time
        RANGE BETWEEN INTERVAL '1 hour' PRECEDING AND CURRENT ROW
    ) as hour_avg
FROM sensor_data;
```

### Retention Policies
```sql
-- Create retention policy function
CREATE OR REPLACE FUNCTION delete_old_data()
RETURNS void AS $$
BEGIN
    DELETE FROM sensor_data
    WHERE time < NOW() - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql;

-- Schedule retention policy
SELECT cron.schedule('0 0 * * *', 'SELECT delete_old_data()');
```

## Advanced Time Series Analysis
### Gap Filling
```sql
-- Generate time series with gaps filled
WITH RECURSIVE timepoints AS (
    SELECT 
        generate_series(
            date_trunc('hour', NOW()) - interval '24 hours',
            date_trunc('hour', NOW()),
            interval '1 hour'
        ) as time
)
SELECT 
    t.time,
    COALESCE(s.temperature, 0) as temperature
FROM timepoints t
LEFT JOIN sensor_data s 
    ON date_trunc('hour', s.time) = t.time
ORDER BY t.time;
```

### Anomaly Detection
```sql
-- Detect outliers using z-score
WITH stats AS (
    SELECT 
        avg(temperature) as avg_temp,
        stddev(temperature) as stddev_temp
    FROM sensor_data
    WHERE time > NOW() - interval '24 hours'
)
SELECT 
    time,
    temperature,
    (temperature - avg_temp) / stddev_temp as z_score
FROM sensor_data, stats
WHERE 
    time > NOW() - interval '24 hours'
    AND abs((temperature - avg_temp) / stddev_temp) > 3;
```

## Performance Optimization
### Indexing Strategies
```sql
-- Create time-based index
CREATE INDEX idx_sensor_time 
ON sensor_data (time DESC);

-- Create composite index
CREATE INDEX idx_sensor_metric 
ON sensor_data (sensor_id, time DESC);

-- Partial index for recent data
CREATE INDEX idx_sensor_recent 
ON sensor_data (time DESC)
WHERE time > NOW() - interval '7 days';
```

### Query Optimization
```sql
-- Use time constraints
EXPLAIN ANALYZE
SELECT *
FROM sensor_data
WHERE time >= NOW() - interval '1 hour'
AND time < NOW()
AND sensor_id = 1;

-- Use appropriate time buckets
SELECT 
    time_bucket('5 minutes', time) as interval,
    avg(temperature) as avg_temp
FROM sensor_data
WHERE time >= NOW() - interval '1 day'
GROUP BY interval
ORDER BY interval;
```

## Best Practices
1. Data Management
   - Regular archiving
   - Appropriate partitioning
   - Efficient compression

2. Query Patterns
   - Use time constraints
   - Leverage materialized views
   - Implement efficient aggregations

3. Monitoring
   - Data growth
   - Query performance
   - Resource usage

## Related Topics
- [[Partitioning Strategies]]
- [[Performance Tuning]]
- [[Query Optimization]]
- [[Database Design]]
