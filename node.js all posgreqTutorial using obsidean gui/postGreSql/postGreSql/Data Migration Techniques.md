# Data Migration Techniques

## Migration Planning
### Pre-Migration Checklist
1. Source Analysis
   - Schema structure
   - Data volume
   - Dependencies
   - Constraints

2. Target Requirements
   - Storage capacity
   - Performance needs
   - Compatibility issues
   - Downtime window

3. Migration Strategy
   - Big bang vs. incremental
   - Downtime requirements
   - Rollback plan
   - Validation approach

## Schema Migration
### Basic Schema Migration
```sql
-- Create new schema version
CREATE SCHEMA version_2;

-- Create new table structure
CREATE TABLE version_2.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    full_name VARCHAR(100),  -- Combined from first_name, last_name
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Migrate data with transformation
INSERT INTO version_2.users (id, email, full_name, status)
SELECT 
    id,
    email,
    first_name || ' ' || last_name,
    CASE 
        WHEN active = true THEN 'active'
        ELSE 'inactive'
    END
FROM public.users;
```

### Zero-Downtime Schema Changes
```sql
-- Add new column without blocking
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100);

-- Update in batches
DO $$
DECLARE
    batch_size INTEGER := 1000;
    max_id INTEGER;
    current_id INTEGER := 0;
BEGIN
    SELECT MAX(id) INTO max_id FROM users;
    
    WHILE current_id < max_id LOOP
        -- Update batch
        UPDATE users
        SET full_name = first_name || ' ' || last_name
        WHERE id > current_id 
        AND id <= current_id + batch_size;
        
        current_id := current_id + batch_size;
        COMMIT;
        
        -- Pause to reduce load
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;

-- Drop old columns when ready
ALTER TABLE users 
DROP COLUMN first_name,
DROP COLUMN last_name;
```

## Data Migration
### Bulk Data Migration
```sql
-- Using COPY command
COPY (
    SELECT id, email, full_name, status 
    FROM source_table
) TO '/tmp/data.csv' WITH CSV HEADER;

COPY target_table (id, email, full_name, status)
FROM '/tmp/data.csv' WITH CSV HEADER;

-- Using pg_dump/pg_restore
-- Export
pg_dump -t source_table -F c -f data.dump dbname

-- Import
pg_restore -t target_table -d dbname data.dump
```

### Incremental Migration
```sql
-- Create migration tracking table
CREATE TABLE migration_progress (
    table_name VARCHAR(100),
    last_migrated_id INTEGER,
    last_run TIMESTAMP,
    status VARCHAR(20)
);

-- Incremental migration function
CREATE OR REPLACE FUNCTION migrate_data_increment(
    p_batch_size INTEGER
)
RETURNS INTEGER AS $$
DECLARE
    v_last_id INTEGER;
    v_count INTEGER := 0;
BEGIN
    -- Get last migrated ID
    SELECT COALESCE(last_migrated_id, 0)
    INTO v_last_id
    FROM migration_progress
    WHERE table_name = 'users';
    
    -- Migrate next batch
    INSERT INTO new_users (id, email, data)
    SELECT id, email, data
    FROM users
    WHERE id > v_last_id
    ORDER BY id
    LIMIT p_batch_size;
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    
    -- Update progress
    IF v_count > 0 THEN
        UPDATE migration_progress
        SET last_migrated_id = v_last_id + v_count,
            last_run = CURRENT_TIMESTAMP
        WHERE table_name = 'users';
    END IF;
    
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;
```

## Validation
### Data Validation
```sql
-- Create validation function
CREATE OR REPLACE FUNCTION validate_migration(
    p_source_schema TEXT,
    p_target_schema TEXT,
    p_table_name TEXT
)
RETURNS TABLE (
    validation_type TEXT,
    source_count BIGINT,
    target_count BIGINT,
    mismatch_count BIGINT
) AS $$
BEGIN
    RETURN QUERY
    
    -- Count comparison
    SELECT 
        'row_count' as validation_type,
        (SELECT COUNT(*) FROM source_schema.table_name) as source_count,
        (SELECT COUNT(*) FROM target_schema.table_name) as target_count,
        ABS((SELECT COUNT(*) FROM source_schema.table_name) - 
            (SELECT COUNT(*) FROM target_schema.table_name)) as mismatch_count
    
    UNION ALL
    
    -- Data checksum comparison
    SELECT 
        'checksum',
        COUNT(*),
        COUNT(*),
        COUNT(CASE WHEN s.checksum != t.checksum THEN 1 END)
    FROM (
        SELECT id, MD5(ROW(s.*)::text) as checksum
        FROM source_schema.table_name s
    ) s
    FULL OUTER JOIN (
        SELECT id, MD5(ROW(t.*)::text) as checksum
        FROM target_schema.table_name t
    ) t USING (id);
END;
$$ LANGUAGE plpgsql;
```

## Rollback Procedures
### Rollback Planning
```sql
-- Create rollback table
CREATE TABLE rollback_points (
    id SERIAL PRIMARY KEY,
    migration_id TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    schema_backup TEXT,
    data_backup TEXT
);

-- Create rollback function
CREATE OR REPLACE FUNCTION create_rollback_point(
    p_migration_id TEXT,
    p_schema_name TEXT
)
RETURNS void AS $$
DECLARE
    v_backup_schema TEXT;
BEGIN
    -- Create backup schema
    v_backup_schema := p_schema_name || '_backup_' || 
                      to_char(CURRENT_TIMESTAMP, 'YYYYMMDD_HH24MISS');
    
    EXECUTE 'CREATE SCHEMA ' || v_backup_schema;
    
    -- Copy schema structure and data
    EXECUTE 'CREATE TABLE ' || v_backup_schema || '.schema_backup AS 
            SELECT * FROM pg_catalog.pg_namespace 
            WHERE nspname = ' || quote_literal(p_schema_name);
            
    -- Store rollback point
    INSERT INTO rollback_points (
        migration_id, schema_backup
    ) VALUES (
        p_migration_id, v_backup_schema
    );
END;
$$ LANGUAGE plpgsql;
```

## Best Practices
1. Planning
   - Thorough testing
   - Performance impact assessment
   - Rollback strategy
   - Communication plan

2. Execution
   - Batch processing
   - Progress monitoring
   - Error handling
   - Data validation

3. Post-Migration
   - Performance verification
   - Data integrity checks
   - Cleanup procedures
   - Documentation

## Related Topics
- [[Database Administration]]
- [[Performance Tuning]]
- [[Error Handling and Logging]]
- [[Best Practices]]
