# Indexing and Performance

## Types of Indexes
### B-tree Index (Default)
- Best for equality and range queries
- Default index type in PostgreSQL
```sql
CREATE INDEX idx_name ON table_name (column_name);
```

### Hash Index
- Only good for equality comparisons
```sql
CREATE INDEX idx_name ON table_name USING HASH (column_name);
```

### GiST Index
- Geometry, full-text search
```sql
CREATE INDEX idx_name ON table_name USING GIST (column_name);
```

### GIN Index
- Full-text search, arrays
```sql
CREATE INDEX idx_name ON table_name USING GIN (column_name);
```

## Index Operations
### Creating Indexes
```sql
-- Simple index
CREATE INDEX idx_employee_name ON employees(last_name);

-- Multi-column index
CREATE INDEX idx_employee_name_dept 
ON employees(last_name, department);

-- Unique index
CREATE UNIQUE INDEX idx_employee_email 
ON employees(email);

-- Partial index
CREATE INDEX idx_high_salary 
ON employees(salary) 
WHERE salary > 50000;
```

### Maintaining Indexes
```sql
-- Rebuild index
REINDEX INDEX idx_name;

-- Rebuild all indexes on table
REINDEX TABLE table_name;

-- Remove unused indexes
DROP INDEX idx_name;
```

## Query Performance

### EXPLAIN ANALYZE
```sql
EXPLAIN ANALYZE
SELECT * FROM employees 
WHERE salary > 50000;
```

### Common Performance Issues
1. Missing Indexes
2. Poor Query Structure
3. Table Statistics
4. Inefficient Joins

### Performance Tips
1. Use appropriate indexes
2. Regular VACUUM and ANALYZE
3. Proper table partitioning
4. Efficient query writing

## Monitoring Tools
- pg_stat_statements
- pg_stat_activity
- pg_stat_user_tables
- pg_stat_user_indexes

## Query Optimization
```sql
-- Update table statistics
ANALYZE table_name;

-- Clean up dead tuples
VACUUM table_name;

-- Both operations
VACUUM ANALYZE table_name;
```

## Related Topics
- [[Query Optimization]]
- [[Database Administration]]
- [[Tables and Schema]]
- [[Performance Tuning]]
