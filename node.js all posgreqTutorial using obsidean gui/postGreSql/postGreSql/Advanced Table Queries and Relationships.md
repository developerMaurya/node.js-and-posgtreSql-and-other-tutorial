# Advanced Table Queries and Relationships

## Sample Database Schema
```sql
-- Create sample e-commerce database schema
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    category_id INTEGER REFERENCES categories(category_id),
    price DECIMAL(10,2),
    stock_quantity INTEGER
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50)
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE product_reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    customer_id INTEGER REFERENCES customers(customer_id),
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    review_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Basic to Advanced JOIN Operations

### Inner Joins
```sql
-- Get all orders with customer information
SELECT 
    o.order_id,
    c.name as customer_name,
    o.order_date,
    o.status
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;

-- Get order details with product information
SELECT 
    o.order_id,
    p.name as product_name,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) as total_price
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id;
```

### Left Joins
```sql
-- Find customers who haven't placed any orders
SELECT 
    c.customer_id,
    c.name,
    c.email,
    COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.email
HAVING COUNT(o.order_id) = 0;

-- Get all products and their reviews (including products with no reviews)
SELECT 
    p.product_id,
    p.name,
    COUNT(r.review_id) as review_count,
    COALESCE(AVG(r.rating), 0) as avg_rating
FROM products p
LEFT JOIN product_reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name;
```

### Multiple Joins with Conditions
```sql
-- Get comprehensive order details
SELECT 
    o.order_id,
    c.name as customer_name,
    p.name as product_name,
    cat.name as category_name,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) as total_price,
    o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY o.order_date DESC;
```

## Subqueries and Derived Tables

### Subqueries in SELECT
```sql
-- Get products with their sales rank
SELECT 
    p.product_id,
    p.name,
    p.price,
    (
        SELECT COUNT(*) 
        FROM order_items oi 
        WHERE oi.product_id = p.product_id
    ) as times_ordered,
    (
        SELECT COALESCE(AVG(rating), 0)
        FROM product_reviews pr
        WHERE pr.product_id = p.product_id
    ) as avg_rating
FROM products p;
```

### Subqueries in WHERE
```sql
-- Find customers who have spent more than average
SELECT 
    c.customer_id,
    c.name,
    SUM(oi.quantity * oi.unit_price) as total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name
HAVING SUM(oi.quantity * oi.unit_price) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT 
            customer_id,
            SUM(oi2.quantity * oi2.unit_price) as customer_total
        FROM orders o2
        JOIN order_items oi2 ON o2.order_id = oi2.order_id
        GROUP BY customer_id
    ) as customer_totals
);
```

### Correlated Subqueries
```sql
-- Find products that have higher than average rating in their category
SELECT 
    p.product_id,
    p.name,
    cat.name as category_name,
    avg_rating.rating as product_avg_rating,
    cat_avg.rating as category_avg_rating
FROM products p
JOIN categories cat ON p.category_id = cat.category_id
JOIN (
    SELECT 
        product_id,
        AVG(rating) as rating
    FROM product_reviews
    GROUP BY product_id
) avg_rating ON p.product_id = avg_rating.product_id
JOIN (
    SELECT 
        p2.category_id,
        AVG(pr.rating) as rating
    FROM products p2
    JOIN product_reviews pr ON p2.product_id = pr.product_id
    GROUP BY p2.category_id
) cat_avg ON cat.category_id = cat_avg.category_id
WHERE avg_rating.rating > cat_avg.rating;
```

## Common Table Expressions (CTEs)

### Basic CTE
```sql
-- Calculate customer lifetime value
WITH customer_purchases AS (
    SELECT 
        c.customer_id,
        c.name,
        COUNT(DISTINCT o.order_id) as order_count,
        SUM(oi.quantity * oi.unit_price) as total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id, c.name
)
SELECT 
    customer_id,
    name,
    order_count,
    total_spent,
    total_spent / order_count as avg_order_value
FROM customer_purchases
ORDER BY total_spent DESC;
```

### Recursive CTE
```sql
-- Get category hierarchy
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT 
        category_id,
        name,
        parent_id,
        1 as level,
        name as path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: child categories
    SELECT 
        c.category_id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT 
    category_id,
    name,
    level,
    path
FROM category_tree
ORDER BY path;
```

## Advanced Aggregations

### Window Functions
```sql
-- Calculate running total of orders by date
SELECT 
    order_date::date,
    COUNT(*) as daily_orders,
    SUM(COUNT(*)) OVER (ORDER BY order_date::date) as running_total,
    LAG(COUNT(*)) OVER (ORDER BY order_date::date) as previous_day_orders,
    ROUND(
        (COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY order_date::date))::numeric /
        LAG(COUNT(*)) OVER (ORDER BY order_date::date) * 100,
        2
    ) as daily_growth_percent
FROM orders
GROUP BY order_date::date
ORDER BY order_date::date;

-- Customer purchase patterns
SELECT 
    c.customer_id,
    c.name,
    o.order_date,
    SUM(oi.quantity * oi.unit_price) as order_total,
    AVG(SUM(oi.quantity * oi.unit_price)) OVER (
        PARTITION BY c.customer_id
        ORDER BY o.order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name, o.order_date
ORDER BY c.customer_id, o.order_date;
```

### Complex Aggregations
```sql
-- Product performance matrix
SELECT 
    p.category_id,
    cat.name as category_name,
    COUNT(DISTINCT p.product_id) as product_count,
    COUNT(DISTINCT oi.order_id) as total_orders,
    SUM(oi.quantity) as units_sold,
    SUM(oi.quantity * oi.unit_price) as total_revenue,
    ROUND(AVG(pr.rating), 2) as avg_rating,
    ROUND(
        SUM(oi.quantity * oi.unit_price) / 
        COUNT(DISTINCT p.product_id)::numeric,
        2
    ) as revenue_per_product
FROM products p
LEFT JOIN categories cat ON p.category_id = cat.category_id
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id
GROUP BY p.category_id, cat.name
HAVING COUNT(DISTINCT p.product_id) > 0
ORDER BY total_revenue DESC NULLS LAST;
```

## Best Practices
1. Always use appropriate indexes for JOIN columns
2. Use EXPLAIN ANALYZE to understand query performance
3. Consider using materialized views for complex, frequently-used queries
4. Use CTEs for better code organization and readability
5. Be careful with recursive queries on large datasets
6. Use appropriate JOIN types (INNER, LEFT, RIGHT) based on requirements
7. Consider partitioning large tables for better performance

## Related Topics
- [[Query Optimization]]
- [[Indexing and Performance]]
- [[Database Design]]
- [[Performance Tuning]]
