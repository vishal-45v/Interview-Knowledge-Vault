# Database Security — Structured Answers

Complete interview-ready answers with SQL, configuration examples, and security architecture patterns.

---

## Q1: Implement Row-Level Security for a multi-tenant SaaS application.

**Answer:**

RLS enforces tenant isolation at the database layer — even if application code has a bug that omits a WHERE clause, the database ensures no cross-tenant data access.

**Schema setup:**
```sql
-- Multi-tenant orders table
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT NOT NULL,
  customer_id BIGINT,
  total NUMERIC(12,2),
  status TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Force RLS even for table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Create isolation policy using application-set context variable
-- Application MUST SET LOCAL app.tenant_id = '<tenant_id>' at start of each transaction
CREATE POLICY tenant_isolation_select ON orders
  FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_isolation_insert ON orders
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_isolation_update ON orders
  FOR UPDATE
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::bigint);

CREATE POLICY tenant_isolation_delete ON orders
  FOR DELETE
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);
```

**Application layer (Python/SQLAlchemy example):**
```python
# Set tenant context at the start of every transaction
def get_tenant_connection(tenant_id: int):
    with engine.connect() as conn:
        conn.execute(
            text("SET LOCAL app.tenant_id = :tid"),
            {"tid": str(tenant_id)}
        )
        yield conn

# Usage:
with get_tenant_connection(42) as conn:
    # This query automatically filtered to tenant_id = 42
    orders = conn.execute(text("SELECT * FROM orders")).fetchall()
    # Even if the developer forgets WHERE tenant_id = 42 — RLS adds it!
```

**Privileged bypass role for admin operations:**
```sql
-- Create a superuser-exempt admin role that bypasses RLS
CREATE ROLE admin_role WITH LOGIN PASSWORD 'admin_secure';
-- Do NOT FORCE ROW LEVEL SECURITY for admin_role's own operations
-- Admin uses SET ROLE to bypass:
SET ROLE admin_role;
-- Now sees all tenants' data (for support/admin operations)
RESET ROLE;  -- back to restricted role
```

**Verify RLS is working:**
```sql
-- Test as a regular application user:
SET app.tenant_id = '1';
SELECT COUNT(*) FROM orders;  -- Should only return tenant 1's orders

SET app.tenant_id = '2';
SELECT COUNT(*) FROM orders;  -- Should only return tenant 2's orders

-- Attempt cross-tenant access should return 0 rows, not an error:
SET app.tenant_id = '1';
SELECT * FROM orders WHERE tenant_id = 2;  -- Returns 0 rows (RLS filters them out)
```

---

## Q2: Design a complete least-privilege role hierarchy for a production application.

**Answer:**

```sql
-- ============================================================
-- ROLE HIERARCHY FOR PRODUCTION APPLICATION
-- ============================================================

-- ---- Group roles (no LOGIN, used for privilege inheritance) ----

-- Read-only access to application data
CREATE ROLE app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO app_readonly;

-- Read/write access (no DDL, no TRUNCATE)
CREATE ROLE app_readwrite;
GRANT app_readonly TO app_readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE, UPDATE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, UPDATE ON SEQUENCES TO app_readwrite;

-- Migration role: can do DDL
CREATE ROLE app_migrations;
GRANT app_readwrite TO app_migrations;
GRANT CREATE ON SCHEMA public TO app_migrations;
-- Note: does NOT include SUPERUSER or CREATEDB

-- Monitoring role: read pg_stat_* views
CREATE ROLE app_monitoring;
GRANT pg_monitor TO app_monitoring;    -- built-in monitoring role (PG10+)
GRANT USAGE ON SCHEMA public TO app_monitoring;
-- No data table access for monitoring role

-- Backup role: can read all data + replication
CREATE ROLE app_backup WITH REPLICATION;
GRANT app_readonly TO app_backup;
GRANT EXECUTE ON FUNCTION pg_start_backup(text, boolean, boolean) TO app_backup;
GRANT EXECUTE ON FUNCTION pg_stop_backup() TO app_backup;

-- ---- Login roles (actual application users) ----

-- Application service account (most requests)
CREATE ROLE app_service WITH LOGIN PASSWORD 'app_secure_pass';
GRANT app_readwrite TO app_service;
ALTER ROLE app_service
  CONNECTION LIMIT 50
  SET statement_timeout = '30s'
  SET lock_timeout = '5s'
  SET idle_in_transaction_session_timeout = '60s';

-- Migration runner (CI/CD pipeline)
CREATE ROLE app_migrator WITH LOGIN PASSWORD 'migrator_secure_pass';
GRANT app_migrations TO app_migrator;
ALTER ROLE app_migrator
  CONNECTION LIMIT 2
  SET lock_timeout = '30s';

-- Monitoring service
CREATE ROLE prometheus_exporter WITH LOGIN PASSWORD 'monitoring_pass';
GRANT app_monitoring TO prometheus_exporter;
ALTER ROLE prometheus_exporter
  CONNECTION LIMIT 5;

-- Backup agent
CREATE ROLE backup_agent WITH LOGIN REPLICATION PASSWORD 'backup_pass';
GRANT app_backup TO backup_agent;
```

**Verify privilege structure:**
```sql
-- View role membership
SELECT r.rolname, m.rolname AS member_of
FROM pg_roles r
JOIN pg_auth_members am ON r.oid = am.member
JOIN pg_roles m ON am.roleid = m.oid
WHERE r.rolname LIKE 'app_%'
ORDER BY r.rolname;

-- View effective table privileges per role
SELECT grantee, table_name, string_agg(privilege_type, ', ') AS privileges
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
  AND grantee IN ('app_readonly', 'app_readwrite', 'app_service')
GROUP BY grantee, table_name
ORDER BY grantee, table_name;
```

---

## Q3: Implement pgcrypto for column-level encryption of sensitive data.

**Answer:**

Column-level encryption protects specific sensitive fields even if an attacker gets full database access.

```sql
-- Install pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Table with encrypted PII columns
CREATE TABLE patient_records (
  id BIGSERIAL PRIMARY KEY,
  patient_name TEXT NOT NULL,
  -- Encrypted columns stored as BYTEA
  ssn_encrypted BYTEA,       -- Social Security Number
  dob_encrypted BYTEA,       -- Date of Birth
  created_at TIMESTAMPTZ DEFAULT now()
);

-- SYMMETRIC ENCRYPTION using pgp_sym_encrypt
-- Key should come from environment/Vault, NOT hardcoded
-- Using AES-256 via pgcrypto's OpenPGP symmetric encryption

-- INSERT with encryption (application provides key from Vault):
INSERT INTO patient_records (patient_name, ssn_encrypted, dob_encrypted)
VALUES (
  'John Smith',
  pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key')),
  pgp_sym_encrypt('1985-03-22', current_setting('app.encryption_key'))
);

-- SELECT with decryption (requires key):
SELECT
  patient_name,
  pgp_sym_decrypt(ssn_encrypted, current_setting('app.encryption_key')) AS ssn,
  pgp_sym_decrypt(dob_encrypted, current_setting('app.encryption_key')) AS dob
FROM patient_records
WHERE id = $1;

-- Application sets key at connection time (from Vault dynamic secret):
SET app.encryption_key = 'vault-retrieved-key-abc123';
```

**Key management with AWS KMS (recommended approach):**
```python
# Application layer: decrypt with AWS KMS
import boto3
import base64

def get_encryption_key():
    kms = boto3.client('kms')
    # KMS Data Key — envelope encryption pattern
    response = kms.generate_data_key(
        KeyId='arn:aws:kms:us-east-1:123456789:key/abc-def',
        KeySpec='AES_256'
    )
    return response['Plaintext']  # Use in memory, never store

# Set key in session, use for queries, key never stored in DB
with conn:
    key = get_encryption_key()
    conn.execute("SET LOCAL app.encryption_key = %s", [key.hex()])
    result = conn.execute("SELECT pgp_sym_decrypt(ssn_encrypted, ...) ...")
```

**Hashing for lookup (passwords, tokens):**
```sql
-- Store hashed values for searchable fields (bcrypt via pgcrypto):
INSERT INTO users (email, password_hash)
VALUES (
  'alice@example.com',
  crypt('user_password', gen_salt('bf', 12))  -- bcrypt with cost=12
);

-- Verify password:
SELECT id FROM users
WHERE email = $1
  AND password_hash = crypt($2, password_hash);
-- $2 = plain text password provided by user
```

---

## Q4: Configure pg_audit for comprehensive audit logging.

**Answer:**

```sql
-- Install pg_audit (requires shared_preload_libraries)
-- postgresql.conf:
-- shared_preload_libraries = 'pgaudit'
-- pgaudit.log = 'write, ddl'     -- log all writes and DDL
-- pgaudit.log_catalog = off      -- don't log system catalog queries
-- pgaudit.log_parameter = on     -- log bind parameters
-- pgaudit.log_relation = on      -- log each relation accessed

-- Verify pgaudit is loaded:
SHOW pgaudit.log;

-- Configure at session/role level for fine-grained control:
-- Log all access to the payments table for compliance role:
ALTER ROLE compliance_auditor SET pgaudit.log = 'read, write';

-- Per-object audit (object auditing mode):
-- Set pgaudit.log = 'none' globally, then use object-level:
SET pgaudit.log = 'none';
-- Then create a role specifically for audited objects:
CREATE ROLE audit_payments;
GRANT SELECT, INSERT, UPDATE, DELETE ON payments TO audit_payments;
-- Grant audit role to application role:
GRANT audit_payments TO app_service;
-- pgaudit object auditing logs all operations by roles that have the audit role
```

**postgresql.conf for compliance logging:**
```ini
# pgaudit configuration
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl, role'  # Log writes, DDL changes, role changes
pgaudit.log_catalog = off          # Reduce noise from catalog queries
pgaudit.log_parameter = on         # Include bind parameter values
pgaudit.log_relation = on          # Separate log entry per relation accessed

# Standard PostgreSQL logging to supplement pgaudit
log_connections = on               # Log all new connections
log_disconnections = on            # Log all disconnections
log_duration = on                  # Log duration of all statements
log_min_duration_statement = 0     # Log ALL statements with duration
log_line_prefix = '%m [%p] %q%u@%d '  # timestamp, pid, user, database
log_destination = 'csvlog'         # Structured CSV for log parsing
logging_collector = on
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 1GB
```

**Sample pgaudit log output:**
```
2024-06-15 14:23:45 UTC [1234] app_service@myapp LOG:
  AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.payments,
  "INSERT INTO payments (amount, customer_id) VALUES ($1, $2)",
  <not logged>
```

**Tamper-proof log shipping:**
```bash
# Ship logs to immutable S3 with object lock (WORM storage):
# Use AWS CloudWatch Logs with log retention and WORM protection
# Or: send to a separate syslog server the database cannot write to

# filebeat configuration for log shipping:
filebeat.inputs:
  - type: log
    paths:
      - /var/log/postgresql/*.csv
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  # Or output to immutable S3 via Logstash
```

---

## Q5: Implement and harden SSL/TLS for PostgreSQL.

**Answer:**

```ini
# postgresql.conf — Server SSL configuration
ssl = on
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file  = '/etc/postgresql/ssl/server.key'
ssl_ca_file   = '/etc/postgresql/ssl/root.crt'     # For client cert auth
ssl_crl_file  = '/etc/postgresql/ssl/root.crl'     # Certificate revocation

# Enforce modern TLS only:
ssl_min_protocol_version = 'TLSv1.2'               # Reject TLS 1.0 and 1.1
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'           # Strong ciphers only

# pg_hba.conf — Require SSL for all network connections:
# Use 'hostssl' prefix to only match SSL connections
# Use 'hostnossl' to explicitly reject non-SSL

# Require SSL for all connections:
hostssl   all   all   0.0.0.0/0   scram-sha-256

# Certificate-based authentication (mutual TLS) for sensitive roles:
hostssl   monitoring_role   monitoring_user   10.0.0.0/8   cert

# Reject non-SSL connections:
hostnossl all   all   0.0.0.0/0   reject
```

**Verify SSL enforcement:**
```sql
-- Check SSL status of current connection:
SELECT ssl, version, cipher, bits, compression
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();

-- Check all current connections' SSL status:
SELECT
  a.pid,
  a.usename,
  a.client_addr,
  s.ssl,
  s.version,
  s.cipher
FROM pg_stat_activity a
JOIN pg_stat_ssl s ON a.pid = s.pid
WHERE a.backend_type = 'client backend';
```

**Client connection string (application):**
```python
# Python psycopg2 — verify-full for production
import psycopg2

conn = psycopg2.connect(
    host="db.example.com",
    port=5432,
    database="myapp",
    user="app_service",
    password="secure_password",
    sslmode="verify-full",                    # MITM protection
    sslrootcert="/etc/ssl/certs/ca-bundle.crt"  # CA certificate
)
```
