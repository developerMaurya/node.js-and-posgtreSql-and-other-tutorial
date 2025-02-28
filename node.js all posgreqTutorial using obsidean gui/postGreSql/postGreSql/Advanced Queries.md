# Advanced Queries

## Window Functions
### Basic Window Functions
```sql
-- ROW_NUMBER
SELECT 
    department,
    employee_name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as salary_rank
FROM employees;

-- RANK and DENSE_RANK
SELECT 
    product_name,
    category,
    price,
    RANK() OVER (PARTITION BY category ORDER BY price DESC) as price_rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY price DESC) as dense_rank
FROM products;
```

### Aggregate Window Functions
```sql
-- Running totals
SELECT 
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) as running_total,
    AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_average
FROM orders;

-- Percentage of total
SELECT 
    category,
    sales,
    sales / SUM(sales) OVER () * 100 as percentage_of_total
FROM category_sales;
```

## Common Table Expressions (CTEs)
### Basic CTEs
```sql
WITH monthly_sales AS (
    SELECT 
        date_trunc('month', order_date) as month,
        SUM(amount) as total_sales
    FROM orders
    GROUP BY 1
)
SELECT 
    month,
    total_sales,
    LAG(total_sales) OVER (ORDER BY month) as prev_month_sales
FROM monthly_sales;
```

### Recursive CTEs
```sql
-- Employee hierarchy
WITH RECURSIVE emp_hierarchy AS (
    -- Base case: top-level employees
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees with managers
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.id
)
SELECT * FROM emp_hierarchy;
```

## JSON Operations
### JSON Creation and Manipulation
```sql
-- Create JSON
SELECT jsonb_build_object(
    'name', first_name || ' ' || last_name,
    'contact', jsonb_build_object(
        'email', email,
        'phone', phone
    )
) as user_info
FROM users;

-- JSON Arrays
SELECT jsonb_agg(
    jsonb_build_object(
        'product', name,
        'price', price
    )
) as product_list
FROM products
GROUP BY category;
```

### JSON Querying
```sql
-- Extract values
SELECT 
    data->>'name' as name,
    (data->'contact'->>'email') as email
FROM user_profiles;

-- Filter JSON arrays
SELECT *
FROM orders
WHERE order_items @> '[{"product_id": 123}]'::jsonb;
```

## Full-Text Search
### Basic Text Search
```sql
-- Create text search vector
ALTER TABLE articles ADD COLUMN ts_vector tsvector;
UPDATE articles SET ts_vector = 
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''));

-- Create index
CREATE INDEX articles_ts_idx ON articles USING GIN (ts_vector);

-- Search
SELECT title, ts_rank(ts_vector, query) as rank
FROM articles, to_tsquery('english', 'postgresql & database') query
WHERE ts_vector @@ query
ORDER BY rank DESC;
```

### Advanced Text Search
```sql
-- Phrase search
SELECT title
FROM articles
WHERE to_tsvector('english', body) @@ phraseto_tsquery('english', 'postgresql database');

-- Weighted search
SELECT title,
    ts_rank_cd(
        setweight(to_tsvector('english', title), 'A') ||
        setweight(to_tsvector('english', body), 'B'),
        to_tsquery('english', 'postgresql & database')
    ) as rank
FROM articles
ORDER BY rank DESC;
```

## Advanced Joins and Set Operations
### Lateral Joins
```sql
SELECT 
    u.username,
    recent_orders.order_date,
    recent_orders.amount
FROM users u
CROSS JOIN LATERAL (
    SELECT order_date, amount
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY order_date DESC
    LIMIT 3
) recent_orders;
```

### Set Operations
```sql
-- UNION, INTERSECT, EXCEPT with CTEs
WITH active_users AS (
    SELECT user_id
    FROM activity_logs
    WHERE action_date >= current_date - interval '30 days'
),
paying_users AS (
    SELECT user_id
    FROM subscriptions
    WHERE status = 'active'
)
SELECT user_id FROM active_users
INTERSECT
SELECT user_id FROM paying_users;
```

## Performance Considerations
1. Use appropriate indexes for JSON columns
2. Consider materialized views for complex queries
3. Monitor query execution plans
4. Use partial indexes when possible

## Related Topics
- [[Query Optimization]]
- [[Indexing and Performance]]
- [[Database Design]]
- [[SQL Fundamentals]]
