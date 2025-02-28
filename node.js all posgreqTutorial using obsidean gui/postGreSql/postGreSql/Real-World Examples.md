# Real-World Examples

## E-Commerce System
### Schema Design
```sql
-- Core tables
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE,
    name VARCHAR(100),
    description TEXT,
    price NUMERIC(10,2),
    stock_quantity INTEGER,
    category_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    status VARCHAR(20),
    total_amount NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER,
    unit_price NUMERIC(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

### Common Queries
```sql
-- Popular products
SELECT 
    p.name,
    SUM(oi.quantity) as total_sold,
    SUM(oi.quantity * oi.unit_price) as revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.created_at >= current_date - interval '30 days'
GROUP BY p.product_id, p.name
ORDER BY total_sold DESC;

-- Customer lifetime value
WITH customer_orders AS (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    GROUP BY customer_id
)
SELECT 
    c.email,
    co.order_count,
    co.total_spent,
    co.total_spent / co.order_count as avg_order_value
FROM customers c
JOIN customer_orders co ON c.customer_id = co.customer_id
ORDER BY total_spent DESC;
```

## Social Media Platform
### Schema Design
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100) UNIQUE,
    password_hash TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE follows (
    follower_id INTEGER REFERENCES users(user_id),
    following_id INTEGER REFERENCES users(user_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id)
);

CREATE TABLE likes (
    user_id INTEGER REFERENCES users(user_id),
    post_id INTEGER REFERENCES posts(post_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
);
```

### Common Queries
```sql
-- User feed
WITH followed_users AS (
    SELECT following_id 
    FROM follows 
    WHERE follower_id = :current_user_id
)
SELECT 
    p.*,
    u.username,
    COUNT(l.user_id) as like_count
FROM posts p
JOIN users u ON p.user_id = u.user_id
LEFT JOIN likes l ON p.post_id = l.post_id
WHERE p.user_id IN (SELECT following_id FROM followed_users)
GROUP BY p.post_id, u.username
ORDER BY p.created_at DESC;

-- Trending posts
SELECT 
    p.*,
    u.username,
    COUNT(l.user_id) as like_count
FROM posts p
JOIN users u ON p.user_id = u.user_id
LEFT JOIN likes l ON p.post_id = l.post_id
WHERE p.created_at >= current_timestamp - interval '24 hours'
GROUP BY p.post_id, u.username
HAVING COUNT(l.user_id) > 10
ORDER BY like_count DESC;
```

## Content Management System
### Schema Design
```sql
CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    slug VARCHAR(200) UNIQUE,
    content TEXT,
    author_id INTEGER REFERENCES users(user_id),
    status VARCHAR(20),
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    slug VARCHAR(50) UNIQUE,
    parent_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE article_categories (
    article_id INTEGER REFERENCES articles(article_id),
    category_id INTEGER REFERENCES categories(category_id),
    PRIMARY KEY (article_id, category_id)
);

CREATE TABLE tags (
    tag_id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE
);

CREATE TABLE article_tags (
    article_id INTEGER REFERENCES articles(article_id),
    tag_id INTEGER REFERENCES tags(tag_id),
    PRIMARY KEY (article_id, tag_id)
);
```

### Common Queries
```sql
-- Article search with categories and tags
SELECT 
    a.title,
    a.slug,
    u.username as author,
    string_agg(DISTINCT c.name, ', ') as categories,
    string_agg(DISTINCT t.name, ', ') as tags
FROM articles a
JOIN users u ON a.author_id = u.user_id
LEFT JOIN article_categories ac ON a.article_id = ac.article_id
LEFT JOIN categories c ON ac.category_id = c.category_id
LEFT JOIN article_tags at ON a.article_id = at.article_id
LEFT JOIN tags t ON at.tag_id = t.tag_id
WHERE 
    a.status = 'published' 
    AND a.published_at <= current_timestamp
GROUP BY a.article_id, u.username
ORDER BY a.published_at DESC;

-- Category tree
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT 
        category_id,
        name,
        parent_id,
        1 as level,
        ARRAY[name] as path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: child categories
    SELECT 
        c.category_id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT 
    level,
    repeat('  ', level - 1) || name as category_name,
    array_to_string(path, ' > ') as full_path
FROM category_tree
ORDER BY path;
```

## Related Topics
- [[Database Design]]
- [[Query Optimization]]
- [[Indexing and Performance]]
- [[Advanced Queries]]
