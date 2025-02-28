# High Availability

## Replication
### Streaming Replication
#### Primary Server Configuration
```ini
# postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_segments = 32

# pg_hba.conf
host replication replicator 192.168.1.0/24 md5
```

#### Standby Server Setup
```ini
# recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=192.168.1.100 port=5432 user=replicator password=secret'
trigger_file = '/tmp/postgresql.trigger'
```

### Logical Replication
```sql
-- Publisher (Primary)
CREATE PUBLICATION my_publication FOR TABLE users, orders;

-- Subscriber (Replica)
CREATE SUBSCRIPTION my_subscription 
CONNECTION 'host=192.168.1.100 dbname=mydb' 
PUBLICATION my_publication;
```

## Failover Strategies
### Manual Failover
```bash
# On standby server
pg_ctl promote

# Or create trigger file
touch /tmp/postgresql.trigger
```

### Automatic Failover
#### Using Patroni
```yaml
# patroni.yml
scope: postgres-cluster
namespace: /db/
name: postgresql0

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.100:8008

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.100:5432
```

## Load Balancing
### PgPool-II Configuration
```ini
# pgpool.conf
listen_addresses = '*'
port = 9999
backend_hostname0 = 'primary.example.com'
backend_port0 = 5432
backend_weight0 = 1
backend_hostname1 = 'replica.example.com'
backend_port1 = 5432
backend_weight1 = 1
```

### Connection Pooling
```ini
# pgbouncer.ini
[databases]
* = host=localhost port=5432

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
pool_mode = session
max_client_conn = 100
```

## Monitoring and Health Checks
### Basic Health Check
```sql
-- Check replication lag
SELECT client_addr, 
       replay_lag, 
       sync_state
FROM pg_stat_replication;

-- Check WAL receiver status
SELECT status, 
       received_lsn, 
       latest_end_lsn
FROM pg_stat_wal_receiver;
```

### Advanced Monitoring
```sql
-- Replication slots
SELECT slot_name,
       plugin,
       slot_type,
       active
FROM pg_replication_slots;

-- WAL sender status
SELECT pid,
       state,
       sent_lsn,
       write_lsn
FROM pg_stat_replication;
```

## Backup Strategies
### Continuous Archiving
```bash
# Enable WAL archiving
archive_mode = on
archive_command = 'cp %p /archive/%f'

# Take base backup
pg_basebackup -D backup -Ft -z
```

## Disaster Recovery
### Recovery Time Objective (RTO)
1. Failover time
2. Application recovery
3. Data verification

### Recovery Point Objective (RPO)
1. Synchronous replication
2. Asynchronous replication
3. Backup recovery

## Best Practices
1. Regular Testing
2. Documentation
3. Monitoring
4. Automation

## Common Issues
1. Replication Lag
2. Split Brain
3. Network Issues
4. Data Corruption

## Related Topics
- [[Backup and Recovery]]
- [[Database Administration]]
- [[Performance Monitoring]]
- [[Security and Access Control]]
