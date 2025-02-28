# Transactions and ACID Properties

## ACID Properties
### Atomicity
- All or nothing
- Transaction either completes entirely or fails entirely

### Consistency
- Database remains in a consistent state
- All constraints are satisfied

### Isolation
- Transactions are isolated from each other
- Concurrent transactions don't interfere

### Durability
- Committed changes are permanent
- Survive system crashes

## Transaction Control
```sql
-- Start transaction
BEGIN;

-- Commit transaction
COMMIT;

-- Rollback transaction
ROLLBACK;

-- Create savepoint
SAVEPOINT my_savepoint;

-- Rollback to savepoint
ROLLBACK TO my_savepoint;
```

## Isolation Levels
### Read Uncommitted
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```
- Lowest isolation level
- Dirty reads possible

### Read Committed
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
- Default in PostgreSQL
- No dirty reads
- Non-repeatable reads possible

### Repeatable Read
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
- No dirty/non-repeatable reads
- Phantom reads possible

### Serializable
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
- Highest isolation level
- Complete isolation
- Performance impact

## Common Patterns
### Basic Transaction
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Transaction with Error Handling
```sql
BEGIN;
    SAVEPOINT my_savepoint;
    
    -- Try operations
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    
    -- If error, rollback to savepoint
    ROLLBACK TO my_savepoint;
    
COMMIT;
```

## Concurrency Issues
1. Dirty Reads
2. Non-repeatable Reads
3. Phantom Reads
4. Lost Updates

## Best Practices
1. Keep transactions short
2. Choose appropriate isolation level
3. Handle deadlocks properly
4. Use proper error handling

## Related Topics
- [[Database Design]]
- [[Performance Tuning]]
- [[Error Handling]]
- [[Concurrency Control]]
