# Advanced Security Topics

## Row-Level Security (RLS)
### Enabling RLS
```sql
-- Enable RLS on a table
ALTER TABLE sensitive_data ENABLE ROW LEVEL SECURITY;

-- Create policy for read access
CREATE POLICY read_own_data ON sensitive_data
    FOR SELECT
    USING (user_id = current_user_id());

-- Create policy for write access
CREATE POLICY write_own_data ON sensitive_data
    FOR INSERT
    WITH CHECK (user_id = current_user_id());
```

### Complex Policies
```sql
-- Policy based on user role
CREATE POLICY admin_access ON sensitive_data
    TO admin_role
    USING (true);

-- Policy with dynamic conditions
CREATE POLICY department_access ON employee_data
    USING (
        department_id IN (
            SELECT dept_id 
            FROM user_departments 
            WHERE user_id = current_user_id()
        )
    );

-- Time-based policy
CREATE POLICY time_restricted_access ON audit_logs
    USING (
        created_at >= current_date - interval '90 days'
        OR has_archive_access(current_user_id())
    );
```

## Encryption at Rest
### Data Encryption
```sql
-- Create extension for encryption
CREATE EXTENSION pgcrypto;

-- Encrypt specific columns
CREATE TABLE secure_users (
    id SERIAL PRIMARY KEY,
    email TEXT,
    password_hash TEXT,
    sensitive_data BYTEA
);

-- Insert encrypted data
INSERT INTO secure_users (email, password_hash, sensitive_data)
VALUES (
    'user@example.com',
    crypt('userpassword', gen_salt('bf')),
    pgp_sym_encrypt(
        'sensitive information',
        'encryption_key'
    )
);

-- Decrypt data
SELECT 
    email,
    pgp_sym_decrypt(
        sensitive_data::bytea,
        'encryption_key'
    ) as decrypted_data
FROM secure_users;
```

### Tablespace Encryption
```sql
-- Create encrypted tablespace
CREATE TABLESPACE secure_space
    LOCATION '/secure/data/location'
    WITH (encryption_algorithm = 'AES_256');

-- Move table to encrypted tablespace
ALTER TABLE secure_users 
SET TABLESPACE secure_space;
```

## Audit Logging
### Setup Audit Logging
```sql
-- Create audit log table
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT,
    changed_at TIMESTAMP WITH TIME ZONE
);

-- Create audit function
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (
        table_name,
        operation,
        old_data,
        new_data,
        changed_by,
        changed_at
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE 
            WHEN TG_OP = 'DELETE' THEN row_to_json(OLD)::jsonb
            ELSE NULL 
        END,
        CASE 
            WHEN TG_OP IN ('INSERT','UPDATE') THEN row_to_json(NEW)::jsonb
            ELSE NULL 
        END,
        current_user,
        current_timestamp
    );
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply audit trigger
CREATE TRIGGER audit_sensitive_data
    AFTER INSERT OR UPDATE OR DELETE ON sensitive_data
    FOR EACH ROW EXECUTE FUNCTION audit_changes();
```

### Querying Audit Logs
```sql
-- View recent changes
SELECT 
    changed_at,
    changed_by,
    operation,
    table_name,
    old_data,
    new_data
FROM audit_log
WHERE changed_at >= current_date - interval '7 days'
ORDER BY changed_at DESC;

-- View changes by specific user
SELECT 
    changed_at,
    operation,
    table_name,
    new_data - old_data as changes
FROM audit_log
WHERE changed_by = 'specific_user'
AND operation = 'UPDATE';
```

## Security Best Practices
### Authentication
```sql
-- Password policy function
CREATE OR REPLACE FUNCTION check_password_strength(password TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN (
        LENGTH(password) >= 12 AND
        password ~ '[A-Z]' AND    -- Uppercase
        password ~ '[a-z]' AND    -- Lowercase
        password ~ '[0-9]' AND    -- Numbers
        password ~ '[^A-Za-z0-9]' -- Special characters
    );
END;
$$ LANGUAGE plpgsql;

-- Apply password policy
ALTER TABLE users
    ADD CONSTRAINT password_strength
    CHECK (check_password_strength(password));
```

### Connection Security
```sql
-- Configure SSL
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = 'server.crt';
ALTER SYSTEM SET ssl_key_file = 'server.key';

-- Force SSL connections
ALTER USER secure_user WITH SSL REQUIRED;
```

### Role Management
```sql
-- Create role hierarchy
CREATE ROLE app_readonly;
CREATE ROLE app_writer;
CREATE ROLE app_admin;

GRANT app_readonly TO app_writer;
GRANT app_writer TO app_admin;

-- Grant specific permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
GRANT INSERT, UPDATE ON specific_table TO app_writer;
GRANT ALL ON ALL TABLES IN SCHEMA public TO app_admin;
```

## Monitoring and Detection
### Security Monitoring
```sql
-- Monitor failed login attempts
SELECT 
    client_addr,
    COUNT(*) as failed_attempts
FROM pg_stat_activity
WHERE state = 'active'
AND query LIKE '%failed%login%'
GROUP BY client_addr
HAVING COUNT(*) > 5;

-- Track suspicious activities
CREATE VIEW suspicious_activities AS
SELECT 
    session_start_time,
    usename,
    client_addr,
    query
FROM pg_stat_activity
WHERE (
    query ~* 'drop|truncate|delete'
    OR client_addr NOT IN (SELECT trusted_ip FROM allowed_ips)
);
```

### Intrusion Detection
```sql
-- Create function to detect suspicious patterns
CREATE OR REPLACE FUNCTION detect_sql_injection()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.query_text ~ '[;]--' OR
       NEW.query_text ~ 'UNION.*SELECT' OR
       NEW.query_text ~ 'DROP.*TABLE'
    THEN
        INSERT INTO security_alerts (
            alert_type,
            description,
            query_text,
            user_name,
            ip_address
        ) VALUES (
            'POSSIBLE_SQL_INJECTION',
            'Suspicious SQL pattern detected',
            NEW.query_text,
            current_user,
            inet_client_addr()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Related Topics
- [[Security and Access Control]]
- [[Database Administration]]
- [[Error Handling and Logging]]
- [[Best Practices]]
