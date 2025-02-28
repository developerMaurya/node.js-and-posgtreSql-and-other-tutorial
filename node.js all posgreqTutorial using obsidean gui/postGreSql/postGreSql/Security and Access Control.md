# Security and Access Control

## User Management
### Creating Users
```sql
-- Create user
CREATE USER username WITH PASSWORD 'secure_password';

-- Create superuser
CREATE USER admin_user WITH SUPERUSER PASSWORD 'very_secure_password';

-- Alter user
ALTER USER username WITH PASSWORD 'new_password';

-- Remove user
DROP USER username;
```

## Role Management
### Creating Roles
```sql
-- Create role
CREATE ROLE readonly;

-- Create role with login
CREATE ROLE developer WITH LOGIN PASSWORD 'dev_password';

-- Grant role to user
GRANT readonly TO username;

-- Revoke role
REVOKE readonly FROM username;
```

## Privileges
### Basic Privileges
- SELECT
- INSERT
- UPDATE
- DELETE
- TRUNCATE
- REFERENCES
- TRIGGER
- CREATE
- CONNECT
- TEMPORARY
- EXECUTE

```sql
-- Grant table privileges
GRANT SELECT, INSERT ON table_name TO username;

-- Grant all privileges
GRANT ALL PRIVILEGES ON table_name TO username;

-- Revoke privileges
REVOKE ALL PRIVILEGES ON table_name FROM username;
```

### Schema Level Privileges
```sql
-- Grant schema privileges
GRANT USAGE ON SCHEMA schema_name TO username;

-- Grant all schema privileges
GRANT ALL ON ALL TABLES IN SCHEMA schema_name TO username;
```

## Row Level Security (RLS)
```sql
-- Enable RLS
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY employee_access ON employees
    FOR SELECT
    USING (department = current_user);

-- Policy for specific roles
CREATE POLICY manager_access ON employees
    FOR ALL
    TO managers
    USING (true);
```

## SSL Configuration
```sql
-- Check SSL status
SHOW ssl;

-- Force SSL connections
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = 'server.crt';
ALTER SYSTEM SET ssl_key_file = 'server.key';
```

## Password Policy
```sql
-- Set password encryption
SET password_encryption = 'scram-sha-256';

-- Set password validity
ALTER ROLE username VALID UNTIL '2024-12-31';
```

## Audit Logging
```sql
-- Enable audit logging
ALTER SYSTEM SET log_statement = 'all';
ALTER SYSTEM SET log_min_duration_statement = 0;
```

## Best Practices
1. Principle of Least Privilege
2. Regular Security Audits
3. Strong Password Policies
4. SSL/TLS Encryption
5. Regular Updates and Patches

## Common Security Issues
1. Weak Passwords
2. Excessive Privileges
3. Unencrypted Connections
4. SQL Injection
5. Unpatched Vulnerabilities

## Related Topics
- [[Database Administration]]
- [[Backup and Recovery]]
- [[Configuration Management]]
- [[SSL Setup]]
