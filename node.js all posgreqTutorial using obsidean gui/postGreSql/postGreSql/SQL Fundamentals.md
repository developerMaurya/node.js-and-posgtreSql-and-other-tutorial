# SQL Fundamentals

## Basic Query Structure
```sql
SELECT column1, column2
FROM table_name
WHERE condition
GROUP BY column
HAVING group_condition
ORDER BY column DESC/ASC;
```

## SELECT Queries
### Basic Selection
```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;

-- Select with alias
SELECT first_name AS name, salary AS annual_pay FROM employees;
```

### WHERE Clause
```sql
-- Basic conditions
SELECT * FROM employees WHERE salary > 50000;

-- Multiple conditions
SELECT * FROM employees 
WHERE salary > 50000 
AND department = 'IT';

-- IN operator
SELECT * FROM employees 
WHERE department IN ('IT', 'HR', 'Finance');

-- LIKE operator
SELECT * FROM employees 
WHERE last_name LIKE 'Sm%';
```

## JOIN Operations
### Types of JOINs
- INNER JOIN
- LEFT JOIN
- RIGHT JOIN
- FULL OUTER JOIN

```sql
-- Inner Join Example
SELECT employees.name, departments.dept_name
FROM employees
INNER JOIN departments 
ON employees.dept_id = departments.id;

-- Left Join Example
SELECT customers.name, orders.order_date
FROM customers
LEFT JOIN orders 
ON customers.id = orders.customer_id;
```

## Aggregation Functions
```sql
-- COUNT
SELECT COUNT(*) FROM employees;

-- SUM
SELECT SUM(salary) FROM employees;

-- AVG
SELECT AVG(salary) FROM employees;

-- GROUP BY
SELECT department, AVG(salary) 
FROM employees 
GROUP BY department;

-- HAVING
SELECT department, AVG(salary) 
FROM employees 
GROUP BY department
HAVING AVG(salary) > 60000;
```

## Subqueries
```sql
-- Subquery in WHERE
SELECT name 
FROM employees 
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Subquery in FROM
SELECT dept_name, avg_salary
FROM (
    SELECT department as dept_name, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
) AS dept_stats;
```

## Related Topics
- [[Advanced Queries]]
- [[Query Optimization]]
- [[Database Design]]
- [[PostgreSQL Basics]]
