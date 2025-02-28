# PostgreSQL Data Types

## Numeric Types
- `INTEGER` - whole numbers
- `BIGINT` - large whole numbers
- `NUMERIC/DECIMAL` - exact decimal numbers
- `REAL` - 6 decimal digits precision
- `DOUBLE PRECISION` - 15 decimal digits precision

## Character Types
- `CHAR(n)` - fixed-length string
- `VARCHAR(n)` - variable-length string
- `TEXT` - unlimited length string

## Date/Time Types
- `DATE` - date only
- `TIME` - time only
- `TIMESTAMP` - date and time
- `INTERVAL` - time periods

## Boolean Type
- `BOOLEAN` - true/false

## Special Types
- `UUID` - universally unique identifiers
- `JSON/JSONB` - JSON data
- `ARRAY` - array of other types
- `BYTEA` - binary data

## Examples
```sql
-- Numeric
CREATE TABLE products (
    id INTEGER,
    price NUMERIC(10,2),
    weight REAL
);

-- Character
CREATE TABLE users (
    username VARCHAR(50),
    description TEXT
);

-- Date/Time
CREATE TABLE events (
    event_date DATE,
    event_time TIME,
    created_at TIMESTAMP
);
```

## Type Conversion
```sql
-- Cast syntax
SELECT CAST(string_field AS INTEGER);
-- or
SELECT string_field::INTEGER;
```

## Related Topics
- [[SQL Fundamentals]]
- [[Tables and Schema]]
- [[Data Validation]]
