# POSTGRESQL DEVSECOPS - SECURITY & AUTOMATION

## Project Overview

This project covers PostgreSQL security hardening, DevSecOps practices, automated deployments, vulnerability scanning, and compliance automation.

### Objectives
1. Implement PostgreSQL security hardening
2. Set up automated vulnerability scanning
3. Configure secrets management
4. Implement database auditing
5. Set up infrastructure as code for PostgreSQL
6. Configure compliance automation

## Prerequisites
- PostgreSQL installed
- Vault or AWS Secrets Manager
- Docker and Kubernetes
- CI/CD tools (Jenkins/GitLab)

## Step 1: Security Hardening

### PostgreSQL Security Checklist

```
bash
#!/bin/bash
# postgresql_security_hardening.sh

# 1. Set proper permissions on data directory
chown -R postgres:postgres /var/lib/postgresql
chmod 700 /var/lib/postgresql/*

# 2. Set proper permissions on configuration
chown postgres:postgres /etc/postgresql/14/main/postgresql.conf
chmod 600 /etc/postgresql/14/main/postgresql.conf

# 3. Disable remote root login
# Already configured in pg_hba.conf

# 4. Enable SSL
sudo -u postgres psql -c "ALTER SYSTEM SET ssl = on;"

# 5. Set strong passwords
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'VeryStrongPassword123!';"

# 6. Disable superuser login via network
# Already configured in pg_hba.conf

# 7. Enable logging
sudo -u postgres psql -c "ALTER SYSTEM SET log_destination = 'stderr';"
sudo -u postgres psql -c "ALTER SYSTEM SET logging_collector = on;"
sudo -u postgres psql -c "ALTER SYSTEM SET log_directory = 'log';"

# 8. Restart PostgreSQL
sudo systemctl restart postgresql
```

### Security Configuration

```
ini
# postgresql.conf - Security settings
# Connection Settings
listen_addresses = 'localhost'
port = 5432
max_connections = 100

# SSL
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL:!MD5:!SEED:!IDEA'
ssl_prefer_server_ciphers = on

# Memory
shared_buffers = 128MB
effective_cache_size = 512MB
work_mem = 4MB
maintenance_work_mem = 64MB

# Query Tuning
random_page_cost = 1.1
effective_io_concurrency = 200

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_timezone = 'UTC'

# Log connections and disconnections
log_connections = on
log_disconnections = on

# Log statements (for debugging)
log_statement = 'ddl'
log_min_duration_statement = 1000

# Log locks
log_lock_waits = on
deadlock_timeout = 1s

# Checkpoints
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9

# Archiving
archive_mode = on
archive_command = '/bin/true'

# Resource usage
shared_memory_type = mmap

# Auth
password_encryption = scram-sha-256
authentication_timeout = 1min
```

## Step 2: Database Auditing

### Install pgAudit

```
bash
# Install pgaudit extension
# Ubuntu/Debian
sudo apt install postgresql-14-pgaudit

# Add to shared_preload_libraries
sudo -u postgres psql -c "ALTER SYSTEM SET shared_preload_libraries = 'pgaudit';"

# Create extension
sudo -u postgres psql -c "CREATE EXTENSION pgaudit;"
```

### Configure pgAudit

```
sql
-- Configure audit logging
ALTER SYSTEM SET pgaudit.log = 'DDL, DML, FUNCTION, PROCEDURE, ROLE';
ALTER SYSTEM SET pgaudit.log_level = 'notice';
ALTER SYSTEM SET pgaudit.role = 'audit_user';

-- Create audit user
CREATE ROLE audit_user;
GRANT pgaudit TO audit_user;

-- Audit specific tables
ALTER TABLE myapp.users SET (pgaudit.log = 'ALL');
ALTER TABLE myapp.orders SET (pgaudit.log = 'INSERT, UPDATE, DELETE');

-- Audit specific columns
ALTER TABLE myapp.users SET (pgaudit.log = 'ALL, -passthrough');
ALTER TABLE myapp.users SET (pgaudit.log_column = 'password, ssn');

-- Audit read operations
ALTER TABLE myapp.sensitive_data SET (pgaudit.log = 'READ');
```

### Audit Queries

```
sql
-- Create audit log table
CREATE TABLE audit_log (
    audit_id SERIAL PRIMARY KEY,
    session_id BIGINT NOT NULL,
    session_seq BIGINT NOT NULL,
    statement_id BIGINT NOT NULL,
    substatement_id BIGINT,
    classid OID NOT NULL,
    objid OID NOT NULL,
    objsubid INTEGER NOT NULL,
    command_tag TEXT,
    object_type TEXT,
    object_identity TEXT,
    statement TEXT,
    parameter TEXT,
    audit_timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create trigger for audit
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (session_id, session_seq, statement_id, classid, objid, objsubid, command_tag, object_type, object_identity, statement, parameter)
        VALUES (pg_backend_pid(),
                txid_current(),
                EXTRACT(EPOCH FROM clock_timestamp())::BIGINT,
                TG_relid, TG_relid, TG_nargs,
                TG_OP, TG_TABLE_SCHEMA, TG_TABLE_NAME,
                current_query(), row_to_json(NEW)::TEXT);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (session_id, session_seq, statement_id, classid, objid, objsubid, command_tag, object_type, object_identity, statement, parameter)
        VALUES (pg_backend_pid(),
                txid_current(),
                EXTRACT(EPOCH FROM clock_timestamp())::BIGINT,
                TG_relid, TG_relid, TG_nargs,
                TG_OP, TG_TABLE_SCHEMA, TG_TABLE_NAME,
                current_query(), row_to_json(NEW)::TEXT);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (session_id, session_seq, statement_id, classid, objid, objsubid, command_tag, object_type, object_identity, statement, parameter)
        VALUES (pg_backend_pid(),
                txid_current(),
                EXTRACT(EPOCH FROM clock_timestamp())::BIGINT,
                TG_relid, TG_relid, TG_nargs,
                TG_OP, TG_TABLE_SCHEMA, TG_TABLE_NAME,
                current_query(), row_to_json(OLD)::TEXT);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create audit trigger
CREATE TRIGGER audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON myapp.users
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

## Step 3: Secrets Management

### Integrate with HashiCorp Vault

```
bash
# Install Vault
wget https://releases.hashicorp.com/vault/1.12.0/vault_1.12.0_linux_amd64.zip
unzip vault_1.12.0_linux_amd64.zip
sudo mv vault /usr/local/bin/

# Start Vault in dev mode
vault server -dev &
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root-token'

# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/myapp-postgres \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/myapp_db" \
    allowed_roles="app-role" \
    username="postgres" \
    password="VaultPassword123!"

# Create role
vault write database/roles/app-role \
    db_name=myapp-postgres \
    creation_statements="CREATE ROLE {{name}} WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO {{name}};" \
    default_ttl=1h \
    max_ttl=24h
```

### Application Integration

```
python
# app.py - Using Vault for database credentials
import os
import hvac
import psycopg2
from contextlib import contextmanager

class VaultDB:
    def __init__(self):
        self.vault_client = hvac.Client(
            url=os.environ.get('VAULT_ADDR', 'http://localhost:8200'),
            token=os.environ.get('VAULT_TOKEN')
        )
        self.db_role = 'app-role'
        self.lease_id = None
    
    def get_credentials(self):
        """Get database credentials from Vault"""
        response = self.vault_client.secrets.database.generate_credentials(
            name=self.db_role
        )
        self.lease_id = response['lease_id']
        return {
            'username': response['data']['username'],
            'password': response['data']['password']
        }
    
    def renew_credentials(self):
        """Renew credentials before expiry"""
        if self.lease_id:
            self.vault_client.sys.renew_lease(lease_id=self.lease_id)
    
    def revoke_credentials(self):
        """Revoke credentials"""
        if self.lease_id:
            self.vault_client.sys.revoke_lease(lease_id=self.lease_id)
    
    @contextmanager
    def get_connection(self):
        """Get database connection using Vault credentials"""
        creds = self.get_credentials()
        conn = psycopg2.connect(
            host=os.environ.get('DB_HOST', 'localhost'),
            port=os.environ.get('DB_PORT', '5432'),
            database=os.environ.get('DB_NAME', 'myapp_db'),
            user=creds['username'],
            password=creds['password']
        )
        try:
            yield conn
        finally:
            conn.close()
            self.revoke_credentials()

# Usage
vault_db = VaultDB()
with vault_db.get_connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        print(cur.fetchall())
```

### Integrate with AWS Secrets Manager

```
bash
# Store PostgreSQL credentials in AWS Secrets Manager
aws secretsmanager create-secret \
    --name "myapp/postgres" \
    --description "PostgreSQL credentials" \
    --secret-string '{"username":"dbuser","password":"SecurePassword123!","engine":"postgres","host":"myapp.db.example.com","port":5432,"dbname":"myapp_db"}'

# Get credentials
aws secretsmanager get-secret-value \
    --secret-id myapp/postgres
```

## Step 4: Infrastructure as Code

### Terraform for PostgreSQL

```
hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

# RDS PostgreSQL Instance
resource "aws_db_instance" "postgres" {
  identifier           = "myapp-postgres"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  max_allocated_storage = 100
  storage_encrypted    = true
  storage_type         = "gp3"
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.postgres.id]
  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "mon:04:00-mon:05:00"
  
  skip_final_snapshot       = false
  final_snapshot_identifier = "myapp-postgres-final-snapshot"
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  tags = {
    Environment = var.environment
    Project     = "myapp"
  }
}

# Security Group
resource "aws_security_group" "postgres" {
  name        = "postgres-sg"
  description = "Security group for PostgreSQL"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
    description = "PostgreSQL"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "postgres-sg"
  }
}

# Subnet Group
resource "aws_db_subnet_group" "postgres" {
  name       = "postgres-subnet-group"
  subnet_ids = var.subnet_ids
  
  tags = {
    Name = "postgres-subnet-group"
  }
}

# Outputs
output "postgres_endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "postgres_port" {
  value = aws_db_instance.postgres.port
}
```

### Ansible for PostgreSQL

```
yaml
# postgresql.yml
---
- name: Install and configure PostgreSQL
  hosts: postgres_servers
  become: yes
  vars:
    postgresql_version: 14
    postgresql_databases:
      - name: myapp_db
        encoding: UTF8
        lc_collate: en_US.UTF-8
        lc_ctype: en_US.UTF-8
    postgresql_users:
      - name: app_user
        password: "{{ vault_postgres_password }}"
        db: myapp_db
        priv: "CONNECT,SELECT,INSERT,UPDATE,DELETE"
    postgresql_pg_hba_custom:
      - host all all 10.0.0.0/16 scram-sha-256
      - local all all peer

  tasks:
    - name: Install PostgreSQL
      apt:
        name:
          - postgresql-{{ postgresql_version }}
          - postgresql-contrib-{{ postgresql_version }}
          - python3-psycopg2
        state: present
        update_cache: yes

    - name: Start PostgreSQL service
      service:
        name: postgresql
        state: started
        enabled: yes

    - name: Configure PostgreSQL
      template:
        src: postgresql.conf.j2
        dest: /etc/postgresql/{{ postgresql_version }}/main/postgresql.conf
        mode: '0600'
      notify: restart postgresql

    - name: Configure pg_hba.conf
      template:
        src: pg_hba.conf.j2
        dest: /etc/postgresql/{{ postgresql_version }}/main/pg_hba.conf
        mode: '0600'
      notify: restart postgresql

    - name: Create databases
      postgresql_db:
        name: "{{ item.name }}"
        encoding: "{{ item.encoding | default('UTF8') }}"
        lc_collate: "{{ item.lc_collate | default('en_US.UTF-8') }}"
        lc_ctype: "{{ item.lc_ctype | default('en_US.UTF-8') }}"
      loop: "{{ postgresql_databases }}"

    - name: Create users
      postgresql_user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        db: "{{ item.db }}"
        priv: "{{ item.priv }}"
      loop: "{{ postgresql_users }}"

  handlers:
    - name: restart postgresql
      service:
        name: postgresql
        state: restarted
```

## Step 5: Vulnerability Scanning

### Install and Configure pgNag

```
bash
# Install pgNag
pip install pgnag

# Create configuration
mkdir -p ~/.pgnag
cat > ~/.pgnag/config.yaml <<EOF
database:
  host: localhost
  port: 5432
  user: postgres
  password: password
  name: postgres

checks:
  - name: weak_passwords
    enabled: true
  - name: ssl_disabled
    enabled: true
  - name: public_objects
    enabled: true
  - name: unwanted_extensions
    enabled: true
  - name: excessive_grants
    enabled: true

output:
  format: json
  file: /var/log/pgnag/results.json
EOF

# Run scan
pgnag scan
```

### Custom Security Checks

```
python
# security_scanner.py
import psycopg2
from datetime import datetime

class SecurityScanner:
    def __init__(self, conn_params):
        self.conn = psycopg2.connect(**conn_params)
        self.results = []
    
    def check_weak_passwords(self):
        """Check for weak or default passwords"""
        query = """
            SELECT usename, valuntil 
            FROM pg_shadow 
            WHERE passwd IS NOT NULL 
            AND (valuntil IS NULL OR valuntil < NOW())
        """
        with self.conn.cursor() as cur:
            cur.execute(query)
            for row in cur.fetchall():
                self.results.append({
                    'check': 'weak_passwords',
                    'severity': 'HIGH',
                    'message': f"User {row[0]} has weak or expired password",
                    'details': {'username': row[0], 'expiry': row[1]}
                })
    
    def check_ssl_status(self):
        """Check if SSL is enforced"""
        query = "SHOW ssl"
        with self.conn.cursor() as cur:
            cur.execute(query)
            result = cur.fetchone()
            if result[0] != 'on':
                self.results.append({
                    'check': 'ssl_disabled',
                    'severity': 'CRITICAL',
                    'message': 'SSL is not enabled on PostgreSQL server',
                    'details': {'ssl_status': result[0]}
                })
    
    def check_public_access(self):
        """Check for public access to sensitive tables"""
        query = """
            SELECT schemaname, tablename, privilege_type
            FROM information_schema.table_privileges
            WHERE grantee = 'PUBLIC'
            AND privilege_type IN ('SELECT', 'INSERT', 'UPDATE', 'DELETE')
        """
        with self.conn.cursor() as cur:
            cur.execute(query)
            for row in cur.fetchall():
                self.results.append({
                    'check': 'public_access',
                    'severity': 'MEDIUM',
                    'message': f"PUBLIC has {row[2]} on {row[0]}.{row[1]}",
                    'details': {'schema': row[0], 'table': row[1], 'privilege': row[2]}
                })
    
    def check_unwanted_extensions(self):
        """Check for potentially dangerous extensions"""
        unwanted = ['dblink', 'postgres_fdw', 'unaccent']
        query = "SELECT extname FROM pg_extension"
        with self.conn.cursor() as cur:
            cur.execute(query)
            for row in cur.fetchall():
                if row[0] in unwanted:
                    self.results.append({
                        'check': 'unwanted_extensions',
                        'severity': 'MEDIUM',
                        'message': f"Potentially dangerous extension {row[0]} is enabled",
                        'details': {'extension': row[0]}
                    })
    
    def check_excessive_grants(self):
        """Check for excessive privileges"""
        query = """
            SELECT grantor, grantee, table_schema, table_name, privilege_type
            FROM information_schema.table_privileges
            WHERE privilege_type = 'DELETE'
            AND grantee NOT IN ('postgres', 'owner')
        """
        with self.conn.cursor() as cur:
            cur.execute(query)
            for row in cur.fetchall():
                self.results.append({
                    'check': 'excessive_grants',
                    'severity': 'LOW',
                    'message': f"User {row[1]} has DELETE on {row[2]}.{row[3]}",
                    'details': {'grantor': row[0], 'grantee': row[1], 'schema': row[2], 'table': row[3]}
                })
    
    def run_all_checks(self):
        """Run all security checks"""
        self.check_weak_passwords()
        self.check_ssl_status()
        self.check_public_access()
        self.check_unwanted_extensions()
        self.check_excessive_grants()
        return self.results

# Usage
scanner = SecurityScanner({
    'host': 'localhost',
    'port': 5432,
    'user': 'postgres',
    'password': 'password',
    'dbname': 'postgres'
})
results = scanner.run_all_checks()
print(results)
```

## Step 6: CI/CD Integration

### Jenkins Pipeline for PostgreSQL

```
groovy
// Jenkinsfile for PostgreSQL deployment
pipeline {
    agent any
    
    environment {
        DB_HOST = credentials('postgres-host')
        DB_USER = credentials('postgres-user')
        DB_PASSWORD = credentials('postgres-password')
        VAULT_ADDR = 'http://vault:8200'
    }
    
    stages {
        stage('Security Scan') {
            steps {
                sh '''
                    pip install pgnag
                    pgnag scan --config /etc/pgnag/config.yaml
                '''
            }
            post {
                failure {
                    emailext body: 'PostgreSQL security scan failed!', 
                               subject: 'Security Alert', 
                               to: 'security@example.com'
                }
            }
        }
        
        stage('Database Migration') {
            steps {
                sh '''
                    pip install alembic
                    alembic upgrade head
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                sh '''
                    pytest tests/ --db-host=$DB_HOST --db-user=$DB_USER --db-password=$DB_PASSWORD
                '''
            }
        }
        
        stage('Backup Before Deploy') {
            steps {
                sh '''
                    pg_dump -h $DB_HOST -U $DB_USER -Fc myapp_db > backup_$(date +%Y%m%d_%H%M%S).dump
                '''
            }
        }
        
        stage('Deploy Schema') {
            steps {
                sh '''
                    psql -h $DB_HOST -U $DB_USER -d myapp_db -f deploy/schema.sql
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    psql -h $DB_HOST -U $DB_USER -d myapp_db -c "SELECT version();"
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Database deployment successful!'
        }
        failure {
            sh '''
                # Rollback if needed
                psql -h $DB_HOST -U $DB_USER -d myapp_db -f rollback/rollback.sql
            '''
        }
    }
}
```

### GitLab CI for PostgreSQL

```
yaml
# .gitlab-ci.yml
stages:
  - security
  - test
  - deploy
  - backup

variables:
  POSTGRES_DB: myapp_db
  POSTGRES_USER: dbuser

security_scan:
  stage: security
  image: python:3.11
  before_script:
    - pip install pgnag
  script:
    - pgnag scan
  only:
    - merge_requests
    - main

test:
  stage: test
  image: postgres:14
  services:
    - postgres:14
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_pass
  script:
    - pip install pytest pytest-postgresql
    - pytest tests/

deploy:
  stage: deploy
  script:
    - pip install psycopg2-binary
    - python deploy.py
  only:
    - main
  when: manual

backup:
  stage: backup
  image: postgres:14
  variables:
    POSTGRES_DB: myapp_db
  script:
    - pg_dump -h $POSTGRES_HOST -U $POSTGRES_USER -Fc > backup_$(date +%Y%m%d_%H%M%S).dump
  artifacts:
    paths:
      - *.dump
    expire_in: 7 days
  only:
    - main
```

## Step 7: Compliance Automation

### CIS Compliance Script

```
bash
#!/bin/bash
# cis_compliance_check.sh

PASS=0
FAIL=0

echo "=== CIS PostgreSQL Compliance Check ==="

# 1. Check PostgreSQL version
VERSION=$(psql --version | awk '{print $3}')
echo "PostgreSQL Version: $VERSION"

# 2. Check SSL is enabled
SSL_STATUS=$(sudo -u postgres psql -t -c "SHOW ssl;" | xargs)
if [ "$SSL_STATUS" = "on" ]; then
    echo "[PASS] SSL is enabled"
    ((PASS++))
else
    echo "[FAIL] SSL is not enabled"
    ((FAIL++))
fi

# 3. Check password encryption
ENCRYPTION=$(sudo -u postgres psql -t -c "SHOW password_encryption;" | xargs)
if [ "$ENCRYPTION" = "scram-sha-256" ]; then
    echo "[PASS] Password encryption is set to scram-sha-256"
    ((PASS++))
else
    echo "[FAIL] Password encryption is not set to scram-sha-256"
    ((FAIL++))
fi

# 4. Check logging connections
LOG_CONN=$(sudo -u postgres psql -t -c "SHOW log_connections;" | xargs)
if [ "$LOG_CONN" = "on" ]; then
    echo "[PASS] Connection logging is enabled"
    ((PASS++))
else
    echo "[FAIL] Connection logging is not enabled"
    ((FAIL++))
fi

# 5. Check logging disconnections
LOG_DISC=$(sudo -u postgres psql -t -c "SHOW log_disconnections;" | xargs)
if [ "$LOG_DISC" = "on" ]; then
    echo "[PASS] Disconnection logging is enabled"
    ((PASS++))
else
    echo "[FAIL] Disconnection logging is not enabled"
    ((FAIL++))
fi

# 6. Check pgaudit extension
if sudo -u postgres psql -t -c "SELECT extname FROM pg_extension WHERE extname = 'pgaudit';" | grep -q pgaudit; then
    echo "[PASS] pgAudit extension is installed"
    ((PASS++))
else
    echo "[FAIL] pgAudit extension is not installed"
    ((FAIL++))
fi

# 7. Check shared_buffers setting
SHARED_BUFFERS=$(sudo -u postgres psql -t -c "SHOW shared_buffers;" | xargs)
if [ "$SHARED_BUFFERS" != "128MB" ]; then
    echo "[PASS] shared_buffers is not default"
    ((PASS++))
else
    echo "[FAIL] shared_buffers is at default value"
    ((FAIL++))
fi

# 8. Check superuser roles
SUPERUSERS=$(sudo -u postgres psql -t -c "SELECT rolname FROM pg_roles WHERE rolsuper = true AND rolname != 'postgres';" | wc -l)
if [ "$SUPERUSERS" = "0" ]; then
    echo "[PASS] No extra superuser roles found"
    ((PASS++))
else
    echo "[FAIL] Extra superuser roles found"
    ((FAIL++))
fi

echo ""
echo "=== Summary ==="
echo "Passed: $PASS"
echo "Failed: $FAIL"
```

### GDPR Compliance

```
sql
-- GDPR: Data access logging
CREATE TABLE data_access_log (
    id SERIAL PRIMARY KEY,
    user_id BIGINT,
    table_name TEXT,
    operation TEXT,
    access_timestamp TIMESTAMP DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT
);

-- Function to log access
CREATE OR REPLACE FUNCTION log_data_access()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO data_access_log (user_id, table_name, operation)
    VALUES (
        current_setting('app.user_id', true)::BIGINT,
        TG_TABLE_NAME,
        TG_OP
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to PII tables
CREATE TRIGGER log_users_access
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION log_data_access();

-- Create data retention policy
CREATE OR REPLACE FUNCTION enforce_retention()
RETURNS void AS $$
BEGIN
    -- Delete data older than 90 days
    DELETE FROM users WHERE created_at < NOW() - INTERVAL '90 days';
    DELETE FROM orders WHERE created_at < NOW() - INTERVAL '90 days';
    DELETE FROM audit_log WHERE audit_timestamp < NOW() - INTERVAL '365 days';
END;
$$ LANGUAGE plpgsql;

-- Schedule retention policy
-- Add to cron: 0 2 * * * psql -c "SELECT enforce_retention();"
```

## Step 8: Backup Automation

### Automated Backup with Retention

```
bash
#!/bin/bash
# postgresql_backup.sh

set -e

# Configuration
BACKUP_DIR="/var/backups/postgresql"
S3_BUCKET="s3://my-backup-bucket/postgres"
DB_NAME="myapp_db"
DB_USER="postgres"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Generate backup filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

echo "Starting backup for $DB_NAME..."

# Perform backup
pg_dump -Fc -h localhost -U "$DB_USER" -f "$BACKUP_FILE" "$DB_NAME"

# Compress backup
gzip "$BACKUP_FILE"

# Upload to S3
aws s3 cp "${BACKUP_FILE}.gz" "$S3_BUCKET/${DB_NAME}/"

# Cleanup old local backups
find "$BACKUP_DIR" -name "*.dump.gz" -mtime +$RETENTION_DAYS -delete

# Cleanup old S3 backups
aws s3 ls "$S3_BUCKET/${DB_NAME}/" | while read -r line; do
    file_date=$(echo "$line" | awk '{print $1}')
    file_name=$(echo "$line" | awk '{print $4}')
    if [[ $(date -d "$file_date" +%s) -lt $(date -d "-$RETENTION_DAYS days" +%s) ]]; then
        aws s3 rm "$S3_BUCKET/${DB_NAME}/$file_name"
    fi
done

# Verify backup
if [ -f "${BACKUP_FILE}.gz" ]; then
    echo "Backup completed successfully!"
    echo "Backup file: ${BACKUP_FILE}.gz"
else
    echo "Backup failed!"
    exit 1
fi
```

## Conclusion

This project covers:
- PostgreSQL security hardening
- Database auditing with pgAudit
- Secrets management with Vault and AWS
- Infrastructure as Code with Terraform and Ansible
- Vulnerability scanning
- CI/CD integration with Jenkins and GitLab
- Compliance automation (CIS, GDPR)
- Automated backup with retention

## Additional Resources
- [PostgreSQL Security](https://www.postgresql.org/docs/14/security.html)
- [pgAudit Documentation](https://github.com/pgaudit/pgaudit)
- [HashiCorp Vault Database Secrets](https://www.vaultproject.io/docs/secrets/databases)
- [PostgreSQL CIS Benchmark](https://www.cisecurity.org/)
