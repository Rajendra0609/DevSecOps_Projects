# POSTGRESQL ADMINISTRATION & DEVOPS

## Project Overview

This project covers PostgreSQL database administration, deployment, security, and DevOps practices including backup strategies, replication, high availability, and security hardening.

### Objectives
1. Install and configure PostgreSQL
2. Implement security best practices
3. Set up replication and high availability
4. Configure backup and recovery
5. Implement monitoring
6. Deploy PostgreSQL with Docker and Kubernetes

## Prerequisites
- Linux server (Ubuntu/CentOS)
- PostgreSQL knowledge
- Docker and Kubernetes (optional)

## Step 1: Installation

### Install PostgreSQL on Ubuntu

```
bash
# Update package index
sudo apt update

# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Install additional packages
sudo apt install -y postgresql-14 postgresql-14-postgis-3

# Check PostgreSQL status
sudo systemctl status postgresql

# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Access PostgreSQL
sudo -u postgres psql
```

### Install PostgreSQL on CentOS

```
bash
# Install PostgreSQL repository
sudo yum install -y https://download.postgresql.org/pub/repos/yum/recent/pg-redhat-latest.rpm

# Install PostgreSQL
sudo yum install -y postgresql14-server postgresql14-contrib

# Initialize database
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb

# Start PostgreSQL
sudo systemctl start postgresql-14
sudo systemctl enable postgresql-14
```

## Step 2: Basic Configuration

### Configure PostgreSQL

```
bash
# Locate configuration files
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"

# Edit postgresql.conf
sudo vim /etc/postgresql/14/main/postgresql.conf

# Edit pg_hba.conf
sudo vim /etc/postgresql/14/main/pg_hba.conf
```

```
ini
# postgresql.conf - Basic settings
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 4MB
min_wal_size = 1GB
max_wal_size = 4GB
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_timezone = 'UTC'
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

```
ini
# pg_hba.conf - Client Authentication
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             10.0.0.0/8               scram-sha-256
host    all             all             192.168.0.0/16          scram-sha-256

# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256

# Replication connections
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

### Create Users and Databases

```
sql
-- Create user with password
CREATE USER devops_user WITH PASSWORD 'SecurePassword123!';

-- Create database
CREATE DATABASE myapp_db OWNER devops_user;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO devops_user;

-- Connect to database
\c myapp_db

-- Grant schema privileges
GRANT ALL ON SCHEMA public TO devops_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO devops_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO devops_user;

-- Create readonly user
CREATE USER readonly_user WITH PASSWORD 'ReadOnlyPassword123!';
GRANT CONNECT ON DATABASE myapp_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

## Step 3: Security Hardening

### Enable SSL/TLS

```
bash
# Generate SSL certificates
sudo openssl req -new -x509 -days 365 -nodes \
  -text -out /etc/postgresql/14/main/server.crt \
  -keyout /etc/postgresql/14/main/server.key

# Set proper permissions
sudo chown postgres:postgres /etc/postgresql/14/main/server.crt
sudo chown postgres:postgres /etc/postgresql/14/main/server.key
sudo chmod 600 /etc/postgresql/14/main/server.key

# Configure PostgreSQL
sudo vim /etc/postgresql/14/main/postgresql.conf
```

```
ini
# SSL configuration
ssl = on
ssl_cert_file = '/etc/postgresql/14/main/server.crt'
ssl_key_file = '/etc/postgresql/14/main/server.key'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL:!MD5:!SEED:!IDEA'
ssl_prefer_server_ciphers = on
ssl_renegotiation_limit = 0
```

### Enable Authentication

```
sql
-- Create password policy
ALTER SYSTEM SET password_encryption = 'scram-sha-256';

-- Set password
ALTER USER devops_user PASSWORD 'SecurePassword123!';

-- Require password for all connections
SET password_encryption = 'scram-sha-256';

-- Enable row-level security
ALTER TABLE my_table ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY "Users can see own data" ON my_table
    FOR SELECT
    USING (user_id = current_user);
```

### Audit Logging

```
sql
-- Enable audit extension
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure audit
ALTER SYSTEM SET pgaudit.log = 'DDL, WRITE';
ALTER SYSTEM SET pgaudit.role = 'audit_role';

-- Create audit role
CREATE ROLE audit_role;
GRANT audit_role TO postgres;

-- Reload configuration
SELECT pg_reload_conf();
```

### Configure Connection Limits

```
sql
-- Limit connections per user
ALTER USER devops_user CONNECTION LIMIT 50;

-- Create resource queue
CREATE RESOURCE QUEUE devops_queue WITH (
    ACTIVE_STATEMENTS = 10,
    MEMORY_LIMIT = '512MB',
    PRIORITY = 'MEDIUM',
    MAX_RESOURCE_QUOTAS = '1GB'
);

-- Assign user to queue
ALTER USER devops_user RESOURCE QUEUE devops_queue;
```

## Step 4: Replication

### Configure Streaming Replication

```
bash
# On Primary server
sudo -u postgres psql
```

```
sql
-- Create replication user
CREATE USER replicator WITH REPLICATION PASSWORD 'ReplPassword123!';

-- Create replication slot
SELECT * FROM pg_create_physical_replication_slot('replica_slot');

-- Backup primary database
sudo -u postgres pg_basebackup -h localhost -D /var/lib/postgresql/14/replica -U replicator -P -Xs -R
```

```
ini
# On Primary - postgresql.conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
```

```
ini
# On Replica - postgresql.conf
hot_standby = on
primary_conninfo = 'host=primary_server_ip port=5432 user=replicator password=ReplPassword123!'
restore_command = 'cp /path/to/wal/%f %p'
```

### Configure Logical Replication

```
sql
-- On primary
CREATE PUBLICATION myapp_publication FOR ALL TABLES;

-- Add table to publication
ALTER PUBLICATION myapp_publication ADD TABLE users;
ALTER PUBLICATION myapp_publication ADD TABLE orders;

-- Create subscription
CREATE SUBSCRIPTION myapp_subscription
CONNECTION 'host=replica_server_ip port=5432 dbname=myapp_db user=postgres password=password'
PUBLICATION myapp_publication;
```

### Configure Replication Monitoring

```
sql
-- Check replication status
SELECT * FROM pg_stat_replication;

-- Check replication slots
SELECT * FROM pg_replication_slots;

-- Check WAL sender status
SELECT * FROM pg_stat_wal_receiver;

-- Monitor lag
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

## Step 5: Backup and Recovery

### Configure Backups

```
bash
#!/bin/bash
# backup_postgres.sh

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="myapp_db"
DB_USER="postgres"

mkdir -p "$BACKUP_DIR"

# Perform backup
sudo -u $DB_USER pg_dump -Fc $DB_NAME > "$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

# Compress backup
gzip "$BACKUP_DIR/${DB_NAME}_${DATE}.dump"

# Keep only last 7 backups
find "$BACKUP_DIR" -name "*.dump.gz" -mtime +7 -delete

echo "Backup complete: ${DB_NAME}_${DATE}.dump.gz"
```

### Automated Backup with Cron

```
bash
# Add to crontab
crontab -e

# Add lines:
# Daily backup at 2 AM
0 2 * * * /opt/scripts/backup_postgres.sh

# Weekly backup on Sunday at 3 AM
0 3 * * 0 /opt/scripts/backup_postgres.sh
```

### Point-in-Time Recovery

```
bash
#!/bin/bash
# restore_postgres.sh

BACKUP_FILE=$1
TARGET_TIME=$2

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_TIME" ]; then
    echo "Usage: $0 <backup_file> <target_time>"
    echo "Example: $0 backup.dump '2024-01-15 10:30:00'"
    exit 1
fi

# Stop PostgreSQL
sudo systemctl stop postgresql

# Backup current data
sudo cp -r /var/lib/postgresql/14/main /var/lib/postgresql/14/main.old

# Clean data directory
sudo rm -rf /var/lib/postgresql/14/main/*
sudo mkdir /var/lib/postgresql/14/main
sudo chown -R postgres:postgres /var/lib/postgresql/14/main
sudo chmod 700 /var/lib/postgresql/14/main

# Restore base backup
sudo -u postgres pg_restore -C -d postgres "$BACKUP_FILE"

# Configure PITR
sudo -u postgres psql -c "ALTER SYSTEM SET restore_command = 'cp /var/wal/%f %p';"
sudo -u postgres psql -c "SELECT pg_wal_restore();" || true

# Restart PostgreSQL
sudo systemctl start postgresql

echo "Restore complete!"
```

### Continuous Archiving

```
ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/wal/%f && cp %p /var/wal/%f'
archive_timeout = 300
```

```
bash
# Create WAL archive directory
sudo mkdir -p /var/wal
sudo chown postgres:postgres /var/wal
```

## Step 6: High Availability

### Configure Patroni

```
yaml
# patroni.yml
scope: myapp-postgres
namespace: /service
name: postgres-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: postgres-1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    postgresql:
      parameters:
        wal_level: replica
        max_connections: 100
        max_wal_senders: 10
        wal_keep_size: 1GB
      recovery_conf:
        restore_command: cp /wal/%f %p
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host all all 0.0.0.0/0 scram-sha-256
    - host all all ::0/0 scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: postgres-1:5432
  data_dir: /data/postgresql
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: ReplPassword123!
    superuser:
      username: postgres
      password: PostgresPassword123!

watchdog:
  enabled: true
  mode: required
  device: /dev/watchdog
  safety_margin: 5
```

### Configure HAProxy

```
ini
# haproxy.cfg
listen postgres
    bind *:5432
    mode tcp
    option tcplog
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres-1 10.0.1.10:5432 check port 8008
    server postgres-2 10.0.1.11:5432 check port 8008
    server postgres-3 10.0.1.12:5432 check port 8008
```

## Step 7: Monitoring

### Install pgMonitor

```
bash
# Install Prometheus and Grafana
wget https://packages.grafana.com/gpg.key -O- | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana prometheus

# Configure Prometheus
sudo vim /etc/prometheus/prometheus.yml
```

```
yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres_exporter'
    static_configs:
      - targets: ['localhost:9187']
```

### Create Monitoring Queries

```
sql
-- Database size
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;

-- Index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Slow queries
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Connection status
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Lock information
SELECT relname, mode, granted, pid, usename
FROM pg_locks l
JOIN pg_database d ON l.database = d.oid
WHERE d.datname = 'myapp_db';

-- Replication lag
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

### Prometheus Exporter

```
bash
# Install PostgreSQL exporter
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.11.0/postgres_exporter-0.11.0.linux-amd64.tar.gz
tar -xzf postgres_exporter-*.linux-amd64.tar.gz
sudo cp postgres_exporter-*/postgres_exporter /usr/local/bin/

# Create systemd service
sudo vim /etc/systemd/system/postgres_exporter.service
```

```
ini
[Unit]
Description=PostgreSQL Exporter
After=network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/postgres_exporter --config.postgresql-alerts="queries.yaml"
Environment="DATA_SOURCE_NAME=postgresql://monitoring:monpass@localhost:5432/postgres?sslmode=disable"
Restart=always

[Install]
WantedBy=multi-user.target
```

## Step 8: PostgreSQL with Docker

### Docker Compose Setup

```
yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14
    container_name: myapp-postgres
    environment:
      POSTGRES_DB: myapp_db
      POSTGRES_USER: dbuser
      POSTGRES_PASSWORD: SecurePassword123!
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backup:/backup
    networks:
      - postgres-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dbuser -d myapp_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=256MB"
      - "-c"
      - "work_mem=4MB"
      - "-c"
      - "maintenance_work_mem=128MB"
      - "-c"
      - "wal_buffers=16MB"
      - "-c"
      - "checkpoint_completion_target=0.9"
      - "-c"
      - "random_page_cost=1.1"
      - "-c"
      - "effective_io_concurrency=200"

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://dbuser:SecurePassword123!@postgres:5432/myapp_db?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - postgres-network
    restart: unless-stopped
    depends_on:
      - postgres

networks:
  postgres-network:
    driver: bridge

volumes:
  postgres_data:
```

### Docker Compose with Replication

```
yaml
# docker-compose-replica.yml
version: '3.8'

services:
  postgres-primary:
    image: postgres:14
    container_name: postgres-primary
    environment:
      POSTGRES_DB: myapp_db
      POSTGRES_USER: replicator
      POSTGRES_PASSWORD: ReplPassword123!
      POSTGRES_INITDB_ARGS: "--encoding=UTF8"
    ports:
      - "5432:5432"
    volumes:
      - primary_data:/var/lib/postgresql/data
    networks:
      - postgres-cluster
    command:
      - "postgres"
      - "-c"
      - "wal_level=replica"
      - "-c"
      - "max_wal_senders=10"
      - "-c"
      - "max_replication_slots=10"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U replicator"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-replica:
    image: postgres:14
    container_name: postgres-replica
    environment:
      POSTGRES_DB: myapp_db
      POSTGRES_USER: replicator
      POSTGRES_PASSWORD: ReplPassword123!
    volumes:
      - replica_data:/var/lib/postgresql/data
    networks:
      - postgres-cluster
    depends_on:
      - postgres-primary
    entrypoint: ["/bin/bash", "-c"]
    command:
      - |
        chown -R postgres:postgres /var/lib/postgresql/data
        exec docker-entrypoint.sh postgres -c hot_standby=on

networks:
  postgres-cluster:
    driver: bridge

volumes:
  primary_data:
  replica_data:
```

## Step 9: PostgreSQL with Kubernetes

### Deploy PostgreSQL on Kubernetes

```
yaml
# postgres-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: myapp_db
  POSTGRES_USER: dbuser
  POSTGRES_MAX_CONNECTIONS: "200"
  POSTGRES_SHARED_BUFFERS: 256MB
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: SecurePassword123!
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - dbuser
            - -d
            - myapp_db
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - dbuser
            - -d
            - myapp_db
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
```

### PostgreSQL StatefulSet

```
yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: myapp_db
        - name: POSTGRES_USER
          value: dbuser
        - name: POSTGRES_PASSWORD
          value: SecurePassword123!
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Step 10: Performance Tuning

### Query Optimization

```
sql
-- Analyze query performance
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Create indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id) WHERE status = 'active';
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Partial index
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';

-- Composite index
CREATE INDEX idx_users_name_email ON users(name, email);

-- Check index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Remove unused indexes
DROP INDEX IF EXISTS idx_unused_index;
```

### Configuration Tuning

```
sql
-- Memory settings
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET work_mem = '4MB';
ALTER SYSTEM SET maintenance_work_mem = '128MB';
ALTER SYSTEM SET temp_buffers = '16MB';

-- WAL settings
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;

-- Query tuning
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET default_statistics_target = 100;

-- Connections
ALTER SYSTEM SET max_connections = 200;

-- Apply changes
SELECT pg_reload_conf();
```

### Connection Pooling with PgBouncer

```
ini
# pgbouncer.ini
[databases]
myapp_db = host=localhost port=5432 dbname=myapp_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5
max_db_connections = 100
max_user_connections = 50
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

[admin_users]
admin
```

## Conclusion

This project covers:
- PostgreSQL installation and configuration
- Security hardening (SSL, authentication, audit)
- Streaming and logical replication
- Backup and recovery strategies
- High availability with Patroni
- Monitoring with Prometheus and Grafana
- PostgreSQL with Docker and Docker Compose
- PostgreSQL on Kubernetes
- Performance tuning and optimization

## Additional Resources
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [PgBouncer Documentation](https://www.pgbouncer.org/)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [PostGIS Documentation](https://postgis.net/documentation/)
