# Practice Exercises

## Basic SQL Exercises
### Exercise 1: Create and Populate Tables
```sql
-- Create tables for a simple blog system
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE
);

CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    author_id INTEGER REFERENCES authors(author_id),
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO authors (name, email) VALUES
    ('John Doe', 'john@example.com'),
    ('Jane Smith', 'jane@example.com');

INSERT INTO posts (author_id, title, content) VALUES
    (1, 'First Post', 'Hello World!'),
    (2, 'PostgreSQL Tips', 'Here are some tips...');
```

### Exercise 2: Basic Queries
```sql
-- Practice these queries
SELECT * FROM posts WHERE author_id = 1;
SELECT authors.name, posts.title 
FROM authors 
JOIN posts ON authors.author_id = posts.author_id;
```

## Intermediate Exercises
### Exercise 3: Advanced Queries
```sql
-- Create an e-commerce schema
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price NUMERIC(10,2),
    category VARCHAR(50)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    price_at_time NUMERIC(10,2)
);

-- Practice these queries:
-- 1. Find total sales by category
SELECT 
    p.category,
    SUM(oi.quantity * oi.price_at_time) as total_sales
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category;

-- 2. Find top 5 products by revenue
SELECT 
    p.name,
    SUM(oi.quantity * oi.price_at_time) as revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.name
ORDER BY revenue DESC
LIMIT 5;
```

### Exercise 4: Window Functions
```sql
-- Practice window functions
SELECT 
    category,
    name,
    price,
    RANK() OVER (PARTITION BY category ORDER BY price DESC) as price_rank
FROM products;
```

## Advanced Exercises
### Exercise 5: Optimization Challenge
```sql
-- Create a large dataset
CREATE TABLE logs (
    log_id SERIAL PRIMARY KEY,
    user_id INTEGER,
    action VARCHAR(50),
    timestamp TIMESTAMP
);

-- Insert million rows
INSERT INTO logs (user_id, action, timestamp)
SELECT 
    floor(random() * 1000)::int,
    (ARRAY['login', 'logout', 'purchase', 'view'])[(random() * 4)::int],
    timestamp '2023-01-01' + random() * interval '365 days'
FROM generate_series(1, 1000000);

-- Optimization tasks:
-- 1. Create appropriate indexes
-- 2. Write efficient queries to find:
--    - Daily active users
--    - Most common actions by hour
--    - User sessions
```

### Exercise 6: Transaction and Concurrency
```sql
-- Practice transaction isolation levels
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Implement a bank transfer ensuring consistency
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

## Real-world Scenarios
### Exercise 7: Data Migration
```sql
-- Create a schema for old and new versions
CREATE SCHEMA old_version;
CREATE SCHEMA new_version;

-- Practice data migration with schema changes
-- 1. Copy data with transformations
-- 2. Verify data integrity
-- 3. Handle edge cases
```

### Exercise 8: Performance Tuning
```sql
-- Create scenarios for:
-- 1. Missing indexes
-- 2. Poor query performance
-- 3. Table bloat
-- 4. Connection overload

-- Task: Identify and fix these issues
```

## Challenge Projects
1. Build a Social Media Database
   - Users, Posts, Comments, Likes
   - Friend relationships
   - Privacy settings

2. Implement an Inventory System
   - Products, Stock levels
   - Orders, Suppliers
   - Automatic reordering

3. Create a Time Series Database
   - Sensor data storage
   - Efficient querying
   - Data retention policies

## Related Topics
- [[SQL Fundamentals]]
- [[Query Optimization]]
- [[Database Design]]
- [[Performance Tuning]]
