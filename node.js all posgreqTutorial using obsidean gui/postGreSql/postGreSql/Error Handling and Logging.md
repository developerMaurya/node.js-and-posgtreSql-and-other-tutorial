# Error Handling and Logging

## Error Handling
### Basic Error Handling
```sql
-- Basic exception handling
DO $$
BEGIN
    -- Some operation that might fail
    INSERT INTO users (email) VALUES ('duplicate@email.com');
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Email already exists';
    WHEN OTHERS THEN
        RAISE NOTICE 'Unknown error: %', SQLERRM;
END $$;
```

### Custom Error Types
```sql
-- Create custom error
CREATE TYPE error_level AS ENUM ('INFO', 'WARNING', 'ERROR');

-- Create error table
CREATE TABLE application_errors (
    error_id SERIAL PRIMARY KEY,
    error_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    error_level error_level,
    error_code TEXT,
    message TEXT,
    details JSONB
);

-- Create error handling function
CREATE OR REPLACE FUNCTION log_error(
    p_level error_level,
    p_code TEXT,
    p_message TEXT,
    p_details JSONB DEFAULT NULL
)
RETURNS void AS $$
BEGIN
    INSERT INTO application_errors 
        (error_level, error_code, message, details)
    VALUES 
        (p_level, p_code, p_message, p_details);
END;
$$ LANGUAGE plpgsql;
```

### Transaction Error Handling
```sql
-- Transaction with savepoints
CREATE OR REPLACE FUNCTION process_order(
    p_order_id INTEGER,
    p_user_id INTEGER
)
RETURNS BOOLEAN AS $$
BEGIN
    -- Start transaction
    BEGIN
        -- Create savepoint
        SAVEPOINT order_start;
        
        -- Update inventory
        UPDATE inventory 
        SET quantity = quantity - (
            SELECT quantity 
            FROM order_items 
            WHERE order_id = p_order_id
        );
        
        -- If successful, process payment
        SAVEPOINT payment_start;
        PERFORM process_payment(p_order_id);
        
        -- Complete order
        UPDATE orders 
        SET status = 'completed' 
        WHERE order_id = p_order_id;
        
        RETURN true;
    EXCEPTION
        WHEN insufficient_stock THEN
            ROLLBACK TO order_start;
            PERFORM log_error('ERROR', 'STOCK_ERR', 'Insufficient stock');
            RETURN false;
        WHEN payment_failed THEN
            ROLLBACK TO payment_start;
            PERFORM log_error('ERROR', 'PAYMENT_ERR', 'Payment failed');
            RETURN false;
        WHEN OTHERS THEN
            ROLLBACK;
            PERFORM log_error('ERROR', 'UNKNOWN', SQLERRM);
            RETURN false;
    END;
END;
$$ LANGUAGE plpgsql;
```

## Logging
### System Logging Configuration
```ini
# postgresql.conf settings

# Log destination
log_destination = 'csvlog'

# Logging collector
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

# What to log
log_min_duration_statement = 1000  # Log queries taking more than 1s
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
```

### Application Level Logging
```sql
-- Create audit log table
CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    log_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_id INTEGER,
    action TEXT,
    table_name TEXT,
    record_id INTEGER,
    old_data JSONB,
    new_data JSONB
);

-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (
            user_id, action, table_name, record_id, old_data
        ) VALUES (
            current_setting('app.current_user_id')::INTEGER,
            TG_OP,
            TG_TABLE_NAME,
            OLD.id,
            row_to_json(OLD)
        );
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (
            user_id, action, table_name, record_id, 
            old_data, new_data
        ) VALUES (
            current_setting('app.current_user_id')::INTEGER,
            TG_OP,
            TG_TABLE_NAME,
            NEW.id,
            row_to_json(OLD),
            row_to_json(NEW)
        );
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (
            user_id, action, table_name, record_id, new_data
        ) VALUES (
            current_setting('app.current_user_id')::INTEGER,
            TG_OP,
            TG_TABLE_NAME,
            NEW.id,
            row_to_json(NEW)
        );
        RETURN NEW;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

## Monitoring
### Query Performance Monitoring
```sql
-- Create monitoring function
CREATE OR REPLACE FUNCTION monitor_slow_queries()
RETURNS TABLE (
    query_id BIGINT,
    database_name TEXT,
    username TEXT,
    query TEXT,
    execution_time INTERVAL,
    calls BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        queryid,
        datname,
        usename,
        query,
        total_time * interval '1 millisecond' as execution_time,
        calls
    FROM pg_stat_statements s
    JOIN pg_database d ON d.oid = s.dbid
    JOIN pg_user u ON u.usesysid = s.userid
    WHERE total_time > 1000  -- queries taking more than 1 second
    ORDER BY total_time DESC
    LIMIT 10;
END;
$$ LANGUAGE plpgsql;
```

### Connection Monitoring
```sql
-- Monitor active connections
CREATE OR REPLACE FUNCTION monitor_connections()
RETURNS TABLE (
    database_name TEXT,
    username TEXT,
    client_addr INET,
    state TEXT,
    query TEXT,
    duration INTERVAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        datname,
        usename,
        client_addr,
        state,
        query,
        now() - query_start as duration
    FROM pg_stat_activity
    WHERE state != 'idle'
    AND pid != pg_backend_pid();
END;
$$ LANGUAGE plpgsql;
```

## Alerting
### Alert Configuration
```sql
-- Create alert table
CREATE TABLE alert_rules (
    rule_id SERIAL PRIMARY KEY,
    rule_name TEXT,
    condition TEXT,
    threshold NUMERIC,
    alert_channel TEXT,
    enabled BOOLEAN DEFAULT true
);

-- Create alert function
CREATE OR REPLACE FUNCTION check_alerts()
RETURNS void AS $$
DECLARE
    rule RECORD;
BEGIN
    FOR rule IN SELECT * FROM alert_rules WHERE enabled = true
    LOOP
        EXECUTE format(
            'INSERT INTO alert_log (rule_id, message)
             SELECT %s, %L
             WHERE EXISTS (
                 SELECT 1
                 FROM (%s) subquery
                 WHERE value > %s
             )',
            rule.rule_id,
            'Alert: ' || rule.rule_name,
            rule.condition,
            rule.threshold
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## Best Practices
1. Error Handling
   - Use specific exception handlers
   - Implement proper transaction management
   - Log errors with context

2. Logging
   - Configure appropriate log levels
   - Implement audit logging
   - Regular log rotation

3. Monitoring
   - Monitor query performance
   - Track connection usage
   - Set up alerting

## Related Topics
- [[Database Administration]]
- [[Performance Tuning]]
- [[Security and Access Control]]
- [[Best Practices]]
