# Database Security — Follow-Up Traps

Tricky follow-up questions exposing common misconceptions about database security, RLS, encryption, and privilege models.

---

## Trap 1: "Row-Level Security automatically protects the table owner from seeing all rows."

**What most people say:** Once RLS is enabled and policies are created, no one can bypass them.

**Correct answer:** By default, RLS policies do NOT apply to the table owner. The table owner bypasses all RLS policies unless FORCE ROW LEVEL SECURITY is set. This is a critical distinction: if your application role is also the table owner (common in poorly designed setups), RLS provides zero protection. Additionally, SUPERUSER roles always bypass RLS regardless of FORCE ROW LEVEL SECURITY.

```sql
-- Enable RLS on the table:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create a policy for non-owner roles:
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- This does NOT protect the table owner!
-- The table owner (e.g., 'postgres') can still see all rows.

-- Force RLS even for the owner:
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Now even the table owner must pass the policy.
-- Superusers STILL bypass. There is no way to RLS-restrict a superuser.

-- Verify RLS status:
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'orders';
```

---

## Trap 2: "Parameterized queries completely prevent SQL injection."

**What most people say:** If you use prepared statements with $1 parameters, you are immune to SQL injection.

**Correct answer:** Parameterized queries prevent first-order SQL injection (injecting into the parameter value). They do NOT prevent second-order injection or injection in dynamic SQL within stored procedures. Second-order injection occurs when: (1) data is stored via parameterized query (safe), (2) the stored data is later used to construct dynamic SQL inside a stored procedure or application code (unsafe). Also, parameterized queries cannot parameterize: table names, column names, or operators — these must be whitelisted manually.

```sql
-- VULNERABLE stored procedure despite parameterized call from app:
CREATE OR REPLACE FUNCTION get_user_data(tbl_name TEXT, user_id INT)
RETURNS TABLE(data TEXT) AS $$
BEGIN
  -- VULNERABLE: tbl_name is concatenated directly!
  RETURN QUERY EXECUTE 'SELECT data FROM ' || tbl_name || ' WHERE id = $1'
    USING user_id;
END;
$$ LANGUAGE plpgsql;

-- Attacker calls: get_user_data('users; DROP TABLE users; --', 1)

-- SAFE version using format() with %I for identifiers:
CREATE OR REPLACE FUNCTION get_user_data_safe(tbl_name TEXT, user_id INT)
RETURNS TABLE(data TEXT) AS $$
BEGIN
  -- %I quotes the identifier safely (prevents injection):
  RETURN QUERY EXECUTE format('SELECT data FROM %I WHERE id = $1', tbl_name)
    USING user_id;
END;
$$ LANGUAGE plpgsql;

-- Still need to whitelist table names to prevent accessing unauthorized tables:
IF tbl_name NOT IN ('public_data', 'reports') THEN
  RAISE EXCEPTION 'Invalid table name: %', tbl_name;
END IF;
```

---

## Trap 3: "GRANT SELECT ON SCHEMA myschema TO app_role gives SELECT on all tables in the schema."

**What most people say:** Granting on a schema gives access to all objects in it.

**Correct answer:** GRANT privilege ON SCHEMA only grants the ability to LOOK UP objects in the schema (i.e., resolve names) and create new objects (with CREATE). It does NOT grant SELECT, INSERT, or any other privilege on tables within the schema. You must separately GRANT privileges on each table (or use ALTER DEFAULT PRIVILEGES for future tables).

```sql
-- This is what most people think they need:
GRANT SELECT ON SCHEMA myschema TO app_role;
-- Reality: app_role can now SEE that tables exist, but CANNOT read them.

-- This is what actually grants table access:
GRANT USAGE ON SCHEMA myschema TO app_role;          -- allow schema lookup
GRANT SELECT ON ALL TABLES IN SCHEMA myschema TO app_role;  -- current tables

-- This covers FUTURE tables created in the schema:
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
  GRANT SELECT ON TABLES TO app_role;

-- Verify actual effective permissions:
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'myschema'
  AND grantee = 'app_role';
```

---

## Trap 4: "Encrypting the database disk provides comprehensive data security."

**What most people say:** Full-disk encryption (FDE) protects all sensitive data.

**Correct answer:** Full-disk encryption (FDE / TDE) only protects against the specific threat of physical disk theft or improper disk disposal. It does NOT protect against: authenticated database connections (the encryption is transparently decrypted on read), SQL injection attacks, insider threats from authorized database users, compromised application credentials, or in-flight data interception (different threat, handled by TLS/SSL). A common misconception is that FDE protects a database that is running — but while the database process is running, all data is decrypted in memory and accessible to any authorized connection. FDE protects the "laptop stolen from the data center" scenario, not the "hacker got application credentials" scenario.

```
Threat model vs protection:
┌────────────────────────────────────┬────────────────────────────────┐
│ Threat                             │ Protection                     │
├────────────────────────────────────┼────────────────────────────────┤
│ Physical disk theft                │ FDE (full-disk encryption)     │
│ Data in transit interception       │ SSL/TLS (sslmode=verify-full)  │
│ SQL injection                      │ Parameterized queries          │
│ Unauthorized DB user               │ Least privilege + RLS          │
│ Compromised app credentials        │ Column-level encryption        │
│ Insider threat (DBA)               │ Column encryption + key mgmt  │
│ Backup theft                       │ Encrypted backups (pgBackRest) │
└────────────────────────────────────┴────────────────────────────────┘
```

---

## Trap 5: "The 'trust' authentication method is fine for local connections."

**What most people say:** Localhost connections are from internal processes only, so trust is safe.

**Correct answer:** The trust method is almost never appropriate in production. trust means ANY connection from the matching host/user combination is accepted with NO password verification. On localhost, this means any process running on the machine (including malware, compromised application processes, or container escape scenarios) can connect to PostgreSQL as any database user including superuser — with zero authentication. The correct method for local connections is peer (verifies the OS user matches the database role name, appropriate for maintenance) or scram-sha-256 (password auth, appropriate for application connections).

```
# BAD pg_hba.conf entries:
local   all   all        trust       # Any local process = superuser access
host    all   all   127.0.0.1/32  trust  # Same problem over TCP

# GOOD pg_hba.conf entries:
local   all   postgres   peer        # OS user 'postgres' can connect as DB role 'postgres'
local   all   all        scram-sha-256  # All other local connections need password
host    all   all   10.0.0.0/8   scram-sha-256  # Network connections: password required
hostssl all   all   0.0.0.0/0   scram-sha-256   # Only SSL connections accepted
```

---

## Trap 6: "A read-only role cannot cause data loss or security issues."

**What most people say:** SELECT-only access is harmless.

**Correct answer:** A read-only database role can still cause serious security issues: (1) Mass data exfiltration — SELECT on all tables means downloading the entire database. (2) Reconnaissance — can read schema definitions (pg_catalog access) to understand table structures for other attacks. (3) Resource exhaustion — expensive queries can slow down the entire database affecting all users. (4) Timing attacks — can infer information about other users' activities via timing differences. Least privilege for read-only roles should still include: specific table/column restrictions, RLS policies, connection throttling, and query timeout limits.

```sql
-- Truly least-privilege read-only role:
CREATE ROLE readonly_limited WITH LOGIN PASSWORD 'secure';

-- Only specific tables, not all:
GRANT USAGE ON SCHEMA public TO readonly_limited;
GRANT SELECT ON products, categories TO readonly_limited;
-- NOT: GRANT SELECT ON ALL TABLES

-- Limit connection count and timeout:
ALTER ROLE readonly_limited
  CONNECTION LIMIT 5                  -- max 5 simultaneous connections
  SET statement_timeout = '30s'       -- kill runaway queries
  SET lock_timeout = '5s'
  SET idle_in_transaction_session_timeout = '60s';

-- Column-level restriction (exclude sensitive columns):
-- Cannot grant column-level SELECT and table-level SELECT together
-- Use a VIEW instead:
CREATE VIEW products_public AS
  SELECT id, name, category, price  -- exclude cost_price, supplier_id
  FROM products;
GRANT SELECT ON products_public TO readonly_limited;
```

---

## Trap 7: "Disabling remote superuser login (pg_hba.conf) is sufficient to prevent superuser access."

**What most people say:** If superuser cannot log in remotely, the database is protected.

**Correct answer:** Restricting remote superuser login in pg_hba.conf is one layer of defense, but several other paths to superuser access remain: (1) Any role with CREATEUSER or CREATEROLE can create new superuser roles. (2) Any SUPERUSER role that CAN log in locally (peer auth) provides a path. (3) A SECURITY DEFINER function owned by a superuser, callable by non-privileged roles, can execute arbitrary SQL as superuser. (4) pg_ctl on the OS provides direct PostgreSQL control without any authentication. True superuser protection requires defense in depth: OS-level access controls, no SUPERUSER for application roles, no SECURITY DEFINER functions with dangerous privileges, and audit logging on superuser-capable role creation.

---

## Trap 8: "MD5 password hashing in pg_shadow is secure for authentication."

**What most people say:** MD5 is a hash function, so passwords are protected.

**Correct answer:** PostgreSQL's md5 auth stores passwords as md5(password + username) — an unsalted MD5 hash. MD5 is cryptographically broken for password hashing: it is extremely fast (billions of hashes per second on GPUs), rainbow tables exist for common passwords, and there is no salt per credential (same password + different username = different hash, but same password + same username = same hash always). scram-sha-256 (default since PostgreSQL 14) uses PBKDF2 with random salt and high iteration count — dramatically harder to brute-force.

```sql
-- Check current authentication method:
SELECT usename, passwd FROM pg_shadow;
-- md5 hash starts with 'md5' followed by 32 hex characters
-- scram hash starts with 'SCRAM-SHA-256$'

-- Migrate users from MD5 to SCRAM-SHA-256:
-- 1. Set in postgresql.conf:
--    password_encryption = scram-sha-256

-- 2. Force password reset (hashing algorithm applied on next SET PASSWORD):
ALTER USER app_user PASSWORD 'new_secure_password';
-- Now stored as SCRAM-SHA-256 hash

-- 3. Update pg_hba.conf to require scram-sha-256:
--    host  all  all  0.0.0.0/0  scram-sha-256
```

---

## Trap 9: "RLS policies apply to the user running the query, so they're always evaluated correctly."

**What most people say:** The current_user function in a policy always reflects who is making the query.

**Correct answer:** When a SECURITY DEFINER function is called, it executes with the privileges of the function OWNER, not the caller. If an RLS policy uses current_user to filter rows, it will filter based on the function owner's username, not the calling user's username — potentially bypassing the policy or applying the wrong filter. The safe approach is to use session_user (the original login role) or a custom setting (current_setting('app.current_user_id')) set by the application:

```sql
-- VULNERABLE: current_user in RLS when called from SECURITY DEFINER function
CREATE POLICY user_isolation ON private_data
  USING (owner_user = current_user);
-- If called from a SECURITY DEFINER function owned by 'admin',
-- current_user = 'admin' and the policy sees ALL rows owned by 'admin'!

-- SAFE: use session_user or application-level context:
CREATE POLICY user_isolation ON private_data
  USING (owner_user = session_user);  -- original login user, not definer

-- Or use application-set context (must be set by app before each query):
CREATE POLICY tenant_isolation ON private_data
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- Application sets: SET LOCAL app.tenant_id = '42';
```

---

## Trap 10: "SSL connections to PostgreSQL prevent man-in-the-middle attacks by default."

**What most people say:** If the application uses SSL, communication is secure.

**Correct answer:** SSL in PostgreSQL has multiple modes, and the default client mode does NOT verify the server certificate, leaving connections vulnerable to MITM attacks:

- `sslmode=disable`: No SSL at all.
- `sslmode=allow`: Use SSL if server offers it, no verification.
- `sslmode=prefer` (default): Use SSL if available, no certificate verification.
- `sslmode=require`: Require SSL but do NOT verify server certificate (VULNERABLE TO MITM).
- `sslmode=verify-ca`: Verify server cert is signed by a trusted CA.
- `sslmode=verify-full`: Verify cert AND that hostname matches CN/SAN (only truly safe for MITM prevention).

Most applications use sslmode=require and believe they are protected, but without certificate verification, an attacker can intercept with their own SSL certificate. Always use sslmode=verify-full with the CA certificate in production:

```ini
# In application connection string or psql:
sslmode=verify-full
sslrootcert=/path/to/ca-certificate.crt

# In postgresql.conf on server:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
ssl_ca_file   = 'root.crt'   # for client certificate authentication

# Force SSL for all connections (pg_hba.conf):
# hostssl  all  all  0.0.0.0/0  scram-sha-256
# (host lines without ssl prefix also allow non-SSL — remove them)
```

---

## Trap 11: "GRANT privilege TO role WITH GRANT OPTION creates a manageable privilege chain."

**What most people say:** WITH GRANT OPTION is a useful delegation feature.

**Correct answer:** WITH GRANT OPTION creates a potentially uncontrollable privilege chain. Once a role has GRANT OPTION, it can grant that privilege to any other role, including roles you did not intend to have it. When you REVOKE the privilege from the original grantee, you must use CASCADE to also revoke it from everyone they granted it to — and if they granted to others who granted further, the cascade can unexpectedly revoke privileges from roles across the system. In production, GRANT OPTION should almost never be used. Centralize all privilege grants in a controlled DBA role or migration script that you explicitly control.

```sql
-- DANGEROUS: creates uncontrolled delegation chain
GRANT SELECT ON orders TO developer_role WITH GRANT OPTION;
-- developer_role can now grant to anyone without DBA knowledge

-- When revoking, cascade is required:
REVOKE SELECT ON orders FROM developer_role CASCADE;
-- This may revoke from other roles that developer_role granted to!

-- Check who has grant option and what they've re-granted:
SELECT grantor, grantee, table_name, privilege_type, is_grantable
FROM information_schema.role_table_grants
WHERE table_name = 'orders'
ORDER BY grantor, grantee;
```

---

## Trap 12: "The application should connect as a different user for reads vs writes to improve security."

**What most people say:** Separate read and write users is a best practice that improves security.

**Correct answer:** Separate read and write roles is a worthwhile isolation — but it is only security theater if the application code has access to BOTH credentials simultaneously (which it usually does). True security benefit requires: (1) the read credential is used in application code paths that genuinely should never write, (2) the write credential is used only when writes are needed, (3) the credentials are stored separately in a secret manager (not both in the same config file), (4) the read role has NO ability to write even if credential is leaked. The real security benefit is limiting blast radius: if a read-code-path is exploited via SQL injection, the attacker cannot INSERT/DELETE even with the connection they have. This is valuable, but only if the read connection truly cannot write.
