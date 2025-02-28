# Query Optimization

## EXPLAIN ANALYZE
### Basic Usage
```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 50000;
```

### Reading Query Plans
1. Scan Types
   - Sequential Scan
   - Index Scan
   - Bitmap Scan
   - Index Only Scan

2. Join Types
   - Nested Loop
   - Hash Join
   - Merge Join

## Common Performance Issues
### Missing Indexes
```sql
-- Create appropriate indexes
CREATE INDEX idx_employee_salary ON employees(salary);
CREATE INDEX idx_employee_dept_salary ON employees(department_id, salary);
```

### Poor JOIN Performance
```sql
-- Bad (Cartesian Join)
SELECT * FROM employees, departments;

-- Good (Explicit Join)
SELECT * FROM employees e
JOIN departments d ON e.department_id = d.id;
```

### Subquery vs JOIN
```sql
-- Subquery (can be slower)
SELECT * FROM employees 
WHERE department_id IN (SELECT id FROM departments WHERE location = 'NY');

-- JOIN (often faster)
SELECT e.* FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.location = 'NY';
```

## Query Writing Best Practices
### Use Specific Columns
```sql
-- Bad
SELECT * FROM employees;

-- Good
SELECT id, first_name, last_name FROM employees;
```

### Avoid DISTINCT When Possible
```sql
-- Bad
SELECT DISTINCT department_id FROM employees;

-- Good
SELECT department_id FROM employees GROUP BY department_id;
```

### Efficient WHERE Clauses
```sql
-- Bad (non-sargable)
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';

-- Good
SELECT * FROM employees WHERE last_name ILIKE 'smith';
```

## Advanced Optimization Techniques
### Partitioning
```sql
-- Range partitioning
CREATE TABLE orders (
    order_date DATE,
    amount NUMERIC
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

### Materialized Views
```sql
-- Create materialized view
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT 
    date_trunc('month', order_date) as month,
    SUM(amount) as total_sales
FROM orders
GROUP BY 1;

-- Refresh view
REFRESH MATERIALIZED VIEW monthly_sales;
```

### Using CTEs
```sql
WITH ranked_employees AS (
    SELECT 
        *,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as rank
    FROM employees
)
SELECT * FROM ranked_employees WHERE rank = 1;
```

## Monitoring Tools
### pg_stat_statements
```sql
-- Enable extension
CREATE EXTENSION pg_stat_statements;

-- Query statistics
SELECT 
    query,
    calls,
    total_time,
    rows,
    100.0 * shared_blks_hit /
        nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_time DESC;
```

## Related Topics
- [[Indexing and Performance]]
- [[Database Design]]
- [[Configuration Management]]
- [[Performance Monitoring]]
