# Extensions and Plugins

## Popular Extensions
### PostGIS (Spatial Database)
```sql
-- Install PostGIS
CREATE EXTENSION postgis;

-- Create spatial table
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOMETRY(Point, 4326)
);

-- Insert point
INSERT INTO locations (name, location)
VALUES ('Central Park', 
    ST_SetSRID(ST_MakePoint(-73.965355, 40.782865), 4326));

-- Spatial queries
SELECT name, 
    ST_Distance(
        location, 
        ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326)
    ) as distance
FROM locations
ORDER BY distance;
```

### pgcrypto (Encryption)
```sql
-- Install pgcrypto
CREATE EXTENSION pgcrypto;

-- Hash passwords
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    password_hash TEXT
);

-- Insert with hashed password
INSERT INTO users (email, password_hash)
VALUES (
    'user@example.com',
    crypt('mypassword', gen_salt('bf'))
);

-- Verify password
SELECT id 
FROM users 
WHERE email = 'user@example.com' 
AND password_hash = crypt('mypassword', password_hash);
```

### uuid-ossp (UUID Generation)
```sql
-- Install uuid-ossp
CREATE EXTENSION "uuid-ossp";

-- Create table with UUID
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title VARCHAR(255),
    content TEXT
);

-- Insert with auto-generated UUID
INSERT INTO documents (title, content)
VALUES ('My Document', 'Content here');
```

### pg_stat_statements (Query Analysis)
```sql
-- Install pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- View query statistics
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### hstore (Key-Value Store)
```sql
-- Install hstore
CREATE EXTENSION hstore;

-- Create table with hstore
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes hstore
);

-- Insert with hstore data
INSERT INTO products (name, attributes)
VALUES (
    'Laptop',
    'color => "black", ram => "16GB", storage => "512GB"'
);

-- Query hstore data
SELECT name, attributes -> 'color' as color
FROM products
WHERE attributes ? 'ram'
AND attributes -> 'ram' = '16GB';
```

### pg_trgm (Fuzzy String Matching)
```sql
-- Install pg_trgm
CREATE EXTENSION pg_trgm;

-- Create index for similarity search
CREATE INDEX idx_products_name_trgm 
ON products USING gin (name gin_trgm_ops);

-- Find similar names
SELECT name, similarity(name, 'labtop')
FROM products
WHERE name % 'labtop'
ORDER BY similarity(name, 'labtop') DESC;
```

## Full-Text Search Extensions
### unaccent
```sql
-- Install unaccent
CREATE EXTENSION unaccent;

-- Use in queries
SELECT title
FROM articles
WHERE unaccent(title) ILIKE unaccent('%café%');
```

### fuzzystrmatch
```sql
-- Install fuzzystrmatch
CREATE EXTENSION fuzzystrmatch;

-- Find similar sounding names
SELECT name, levenshtein(name, 'John') as distance
FROM users
WHERE levenshtein(name, 'John') <= 2;
```

## Performance Extensions
### pg_prewarm (Buffer Cache Management)
```sql
-- Install pg_prewarm
CREATE EXTENSION pg_prewarm;

-- Load table into buffer cache
SELECT pg_prewarm('mytable');
```

### pg_buffercache (Buffer Analysis)
```sql
-- Install pg_buffercache
CREATE EXTENSION pg_buffercache;

-- Analyze buffer usage
SELECT c.relname, count(*) blocks
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY 2 DESC;
```

## Development Extensions
### plv8 (JavaScript Procedures)
```sql
-- Install plv8
CREATE EXTENSION plv8;

-- Create JavaScript function
CREATE FUNCTION hello_js(name text)
RETURNS text AS $$
    return "Hello, " + name + "!";
$$ LANGUAGE plv8;

-- Use function
SELECT hello_js('World');
```

### pgtap (Unit Testing)
```sql
-- Install pgtap
CREATE EXTENSION pgtap;

-- Create test
CREATE OR REPLACE FUNCTION test_users()
RETURNS SETOF TEXT AS $$
BEGIN
    RETURN NEXT has_table('users');
    RETURN NEXT col_is_pk('users', 'id');
    RETURN NEXT col_not_null('users', 'email');
END;
$$ LANGUAGE plpgsql;

-- Run tests
SELECT * FROM runtests();
```

## Best Practices
1. Security Considerations
   - Review extension source
   - Limit extension access
   - Regular updates

2. Performance Impact
   - Monitor resource usage
   - Test before production
   - Regular maintenance

3. Compatibility
   - Version matching
   - Dependency management
   - Upgrade planning

## Related Topics
- [[Database Administration]]
- [[Security and Access Control]]
- [[Performance Tuning]]
- [[Development Tools]]
