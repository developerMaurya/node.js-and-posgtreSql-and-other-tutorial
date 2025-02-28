# PostgreSQL Basics

## What is PostgreSQL?
PostgreSQL is an advanced, open-source object-relational database system that supports both SQL (relational) and JSON (non-relational) querying.

## Key Features
- ACID Compliance
- [[Transactions|Transaction Support]]
- [[Data Types|Rich Data Type Support]]
- [[Extensions|Extensibility]]

## Installation
```bash
# Windows (using chocolatey)
choco install postgresql

# Linux
sudo apt-get install postgresql

# macOS (using homebrew)
brew install postgresql
```

## Basic Commands
- `psql` - PostgreSQL interactive terminal
- `createdb` - Create a new database
- `dropdb` - Delete a database
- `pg_dump` - Backup a database

## Common Operations
### Connect to PostgreSQL
```sql
psql -U username -d database_name
```

### Create Database
```sql
CREATE DATABASE database_name;
```

### Create Table
```sql
CREATE TABLE table_name (
    column1 datatype1,
    column2 datatype2,
    ...
);
```

### Basic Queries
```sql
-- Select all records
SELECT * FROM table_name;

-- Insert record
INSERT INTO table_name (column1, column2) VALUES (value1, value2);

-- Update record
UPDATE table_name SET column1 = value1 WHERE condition;

-- Delete record
DELETE FROM table_name WHERE condition;
```

## Related Topics
- [[SQL Fundamentals]]
- [[Database Administration]]
- [[Connection and Authentication]]
