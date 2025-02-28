# Database Design

## Database Design Principles
### Normalization
1. First Normal Form (1NF)
   - Atomic values
   - No repeating groups

2. Second Normal Form (2NF)
   - 1NF requirements
   - No partial dependencies

3. Third Normal Form (3NF)
   - 2NF requirements
   - No transitive dependencies

### Entity Relationship Model
- Entities
- Relationships
- Attributes
- Cardinality

## Best Practices
### Naming Conventions
```sql
-- Use snake_case for objects
CREATE TABLE user_profiles (
    user_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

-- Use plural for table names
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY
);
```

### Data Integrity
```sql
-- Primary Keys
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Foreign Keys
CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER CHECK (quantity > 0)
);

-- Unique Constraints
CREATE TABLE users (
    email VARCHAR(100) UNIQUE,
    username VARCHAR(50) UNIQUE
);
```

## Common Patterns
### One-to-Many Relationship
```sql
CREATE TABLE departments (
    dept_id SERIAL PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    dept_id INTEGER REFERENCES departments(dept_id),
    name VARCHAR(100)
);
```

### Many-to-Many Relationship
```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    course_id SERIAL PRIMARY KEY,
    title VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(student_id),
    course_id INTEGER REFERENCES courses(course_id),
    PRIMARY KEY (student_id, course_id)
);
```

### Inheritance
```sql
-- Parent table
CREATE TABLE vehicles (
    vehicle_id SERIAL PRIMARY KEY,
    brand VARCHAR(50),
    model VARCHAR(50)
);

-- Child table
CREATE TABLE cars (
    car_id INTEGER PRIMARY KEY REFERENCES vehicles(vehicle_id),
    num_doors INTEGER
) INHERITS (vehicles);
```

## Performance Considerations
1. Proper Indexing
2. Appropriate Data Types
3. Partitioning Strategy
4. Denormalization when needed

## Common Anti-patterns
1. Over-normalization
2. Inappropriate use of NULLS
3. Poor primary key choices
4. Lack of constraints

## Related Topics
- [[Data Types]]
- [[Tables and Schema]]
- [[Indexing and Performance]]
- [[Query Optimization]]
