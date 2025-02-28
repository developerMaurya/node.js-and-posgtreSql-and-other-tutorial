# Tables and Schema

## Creating Tables
### Basic Table Creation
```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    hire_date DATE DEFAULT CURRENT_DATE,
    salary NUMERIC(10,2)
);
```

## Constraints
- PRIMARY KEY
- FOREIGN KEY
- UNIQUE
- NOT NULL
- CHECK
- DEFAULT

```sql
-- Example with multiple constraints
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount NUMERIC(10,2) CHECK (total_amount > 0),
    status VARCHAR(20) DEFAULT 'pending'
);
```

## Altering Tables
```sql
-- Add column
ALTER TABLE employees 
ADD COLUMN department VARCHAR(50);

-- Remove column
ALTER TABLE employees 
DROP COLUMN department;

-- Modify column
ALTER TABLE employees 
ALTER COLUMN salary TYPE NUMERIC(12,2);

-- Add constraint
ALTER TABLE employees 
ADD CONSTRAINT salary_check 
CHECK (salary > 0);
```

## Schemas
### Creating Schemas
```sql
-- Create new schema
CREATE SCHEMA hr;

-- Create table in schema
CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);
```

### Schema Management
```sql
-- List all schemas
SELECT * FROM information_schema.schemata;

-- Set search path
SET search_path TO hr, public;

-- Drop schema
DROP SCHEMA hr CASCADE;
```

## Indexes
```sql
-- Create index
CREATE INDEX idx_employee_email 
ON employees(email);

-- Create unique index
CREATE UNIQUE INDEX idx_unique_email 
ON employees(email);

-- Drop index
DROP INDEX idx_employee_email;
```

## Views
```sql
-- Create view
CREATE VIEW employee_details AS
SELECT 
    e.id,
    e.first_name,
    e.last_name,
    d.department_name
FROM 
    employees e
JOIN 
    departments d ON e.dept_id = d.id;

-- Materialized view
CREATE MATERIALIZED VIEW dept_salary_stats AS
SELECT 
    department_name,
    AVG(salary) as avg_salary,
    COUNT(*) as employee_count
FROM 
    employee_details
GROUP BY 
    department_name;
```

## Related Topics
- [[Database Design]]
- [[Indexing and Performance]]
- [[Data Types]]
- [[SQL Fundamentals]]
