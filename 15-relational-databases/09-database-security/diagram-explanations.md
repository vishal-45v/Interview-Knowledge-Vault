# Database Security — Diagram Explanations

ASCII diagrams illustrating security architectures, authentication flows, and privilege models.

---

## Diagram 1: PostgreSQL Authentication and Authorization Flow

```
  Client Connection Request
          │
          ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 1: pg_hba.conf MATCHING                                    │
  │                                                                  │
  │  Rules checked top-to-bottom, first match wins:                  │
  │  ┌─────────────────────────────────────────────────────────┐     │
  │  │ hostssl  myapp  app_service  10.0.0.0/8   scram-sha-256 │ ✓   │
  │  │ host     all    all          0.0.0.0/0    reject        │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  Connection from 10.0.1.5, SSL=yes, db=myapp, user=app_service  │
  │  → Matches first rule → use scram-sha-256 authentication        │
  └───────────────────────────┬──────────────────────────────────────┘
                              │ authentication method determined
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 2: AUTHENTICATION (SCRAM-SHA-256)                          │
  │                                                                  │
  │  Server: send challenge nonce                                    │
  │  Client: compute HMAC(password + nonce + salt + iterations)     │
  │  Server: verify against stored verifier in pg_shadow            │
  │  → PASS or FAIL                                                  │
  └───────────────────────────┬──────────────────────────────────────┘
                              │ authenticated
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 3: AUTHORIZATION (GRANT/REVOKE privileges)                 │
  │                                                                  │
  │  Query: SELECT * FROM payments WHERE customer_id = $1            │
  │                                                                  │
  │  Check: does app_service have SELECT on public.payments?         │
  │  → GRANT SELECT ON payments TO app_service  → YES               │
  │                                                                  │
  │  Check: does app_service have USAGE on schema public?            │
  │  → GRANT USAGE ON SCHEMA public TO app_service  → YES           │
  └───────────────────────────┬──────────────────────────────────────┘
                              │ privilege granted
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  STEP 4: ROW-LEVEL SECURITY (if enabled on table)                │
  │                                                                  │
  │  Policy: tenant_id = current_setting('app.tenant_id')::bigint   │
  │  app.tenant_id SET to '42' by application                       │
  │  → Filter WHERE tenant_id = 42 added transparently              │
  │  → Only rows matching tenant 42 are returned                    │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Role and Privilege Hierarchy

```
                    SUPERUSER (postgres)
                    └── All privileges, bypasses RLS
                        USE ONLY for admin operations, fully logged

  Group Roles (NOLOGIN)       Login Roles (LOGIN)
  ────────────────────────    ──────────────────────────────────────
  ┌───────────────────┐       ┌──────────────────────────────────┐
  │  app_readonly     │       │  app_service (LOGIN)             │
  │  - SELECT on all  │◄──────│  - INHERITS app_readwrite        │
  │    tables         │       │  - CONNECTION LIMIT 50           │
  └───────────────────┘       │  - statement_timeout = 30s       │
          ▲                   └──────────────────────────────────┘
          │ inherits
  ┌───────────────────┐       ┌──────────────────────────────────┐
  │  app_readwrite    │       │  app_migrator (LOGIN)            │
  │  - INSERT/UPDATE/ │◄──────│  - INHERITS app_migrations       │
  │    DELETE         │       │  - CONNECTION LIMIT 2            │
  └───────────────────┘       └──────────────────────────────────┘
          ▲
          │ inherits
  ┌───────────────────┐       ┌──────────────────────────────────┐
  │  app_migrations   │       │  monitoring (LOGIN)              │
  │  - CREATE on schema│◄─────│  - INHERITS pg_monitor           │
  └───────────────────┘       │  - No table privileges           │
                              └──────────────────────────────────┘

  Permission inheritance flow:
  ─────────────────────────────────────────────────────────────────
  app_service connects
  → has: app_readwrite role
  → which has: app_readonly role
  → therefore has: SELECT, INSERT, UPDATE, DELETE on tables
  → plus sequence USAGE/UPDATE
  → does NOT have: CREATE TABLE, TRUNCATE, DROP TABLE, GRANT

  Verify actual privileges:
  SELECT * FROM information_schema.role_table_grants
  WHERE grantee = 'app_service' OR grantee IN (
    SELECT m.rolname FROM pg_roles r
    JOIN pg_auth_members am ON r.oid = am.member
    JOIN pg_roles m ON am.roleid = m.oid
    WHERE r.rolname = 'app_service'
  );
```

---

## Diagram 3: SQL Injection Attack and Defense

```
  VULNERABLE: String Concatenation
  ─────────────────────────────────

  User input:  alice@example.com' OR '1'='1

  Application code:
  query = "SELECT * FROM users WHERE email = '" + user_input + "'"

  Resulting SQL:
  SELECT * FROM users WHERE email = 'alice@example.com' OR '1'='1'
  ─────────────────────────────────────────────────────────────────
  This returns ALL rows! The OR '1'='1' is always true.

  Attacker's more dangerous input:
  alice'; DROP TABLE users; --

  Resulting SQL:
  SELECT * FROM users WHERE email = 'alice'; DROP TABLE users; --'
  ─────────────────────────────────────────────────────────────────
  This executes TWO statements: SELECT, then DROP TABLE!

  SECURE: Parameterized Query
  ────────────────────────────

  Query template: SELECT * FROM users WHERE email = $1
  Parameter:      {$1: "alice@example.com' OR '1'='1"}

  What the database sees:
  ┌─────────────────────────────────────────────────────────────────┐
  │ QUERY:  SELECT * FROM users WHERE email = $1                    │
  │ PARAMS: [$1 = "alice@example.com' OR '1'='1"]                  │
  │                                                                 │
  │ The parameter $1 is ALWAYS treated as a string literal.         │
  │ The single quotes, OR, '1'='1' are all part of the string value.│
  │ They are NEVER interpreted as SQL syntax.                       │
  └─────────────────────────────────────────────────────────────────┘

  Result: No rows found (no user with that email address)
  The injection payload is stored as a literal string, not executed.

  Defense in depth layers (multiple failures needed for a breach):
  ┌─────────────────────────────────────────────────────────────────┐
  │ Layer 1: Parameterized queries (prevents injection)             │
  │ Layer 2: App role has only SELECT (no DROP even if injected)    │
  │ Layer 3: Input validation (reject suspicious patterns)          │
  │ Layer 4: WAF / pg_audit (detect and alert on injection attempts)│
  │ Layer 5: Least privilege (app role cannot read sensitive tables)│
  └─────────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: Multi-Tenant Data Isolation Options

```
  OPTION A: Shared Schema + Row-Level Security
  ─────────────────────────────────────────────

  Database: myapp
  Schema: public
  Table: orders
  ┌────────────┬───────────┬──────────────┬─────────┐
  │ id         │ tenant_id │ customer_id  │ total   │
  ├────────────┼───────────┼──────────────┼─────────┤
  │ 1          │ 1 (Acme)  │ 101          │ 500.00  │
  │ 2          │ 2 (Globex)│ 201          │ 750.00  │  ← Acme cannot see this
  │ 3          │ 1 (Acme)  │ 102          │ 300.00  │
  └────────────┴───────────┴──────────────┴─────────┘

  RLS Policy enforces: tenant_id = current_setting('app.tenant_id')
  Acme's session: app.tenant_id = '1' → sees rows 1 and 3 only
  Globex's session: app.tenant_id = '2' → sees row 2 only

  OPTION B: Per-Tenant Schema
  ────────────────────────────

  Database: myapp
  Schema: tenant_acme       | Schema: tenant_globex
  Table: orders             | Table: orders
  ┌────┬────────┬──────┐    | ┌────┬────────┬──────┐
  │ id │ cust   │total │    | │ id │ cust   │total │
  ├────┼────────┼──────┤    | ├────┼────────┼──────┤
  │  1 │   101  │ 500  │    | │  1 │   201  │ 750  │
  │  2 │   102  │ 300  │    | │  2 │   202  │ 900  │
  └────┴────────┴──────┘    | └────┴────────┴──────┘

  acme_app_role: search_path = tenant_acme
  Cannot even address tenant_globex tables (no schema access)

  OPTION C: Per-Tenant Database
  ──────────────────────────────

  Cluster:
  ├── Database: acme_prod
  │   └── Schema: public → tables
  ├── Database: globex_prod
  │   └── Schema: public → tables
  └── Database: startup_prod
      └── Schema: public → tables

  Complete isolation: separate VACUUM, WAL, backups per tenant
  Cross-tenant queries impossible (need FDW)
```

---

## Diagram 5: pg_hba.conf Processing and Authentication Chain

```
  Connection attempt:
  FROM: 10.0.1.5 (application server)
  TO:   database=myapp, user=app_service, SSL=yes

  pg_hba.conf (rules checked in ORDER, first match wins):
  ┌────┬────────┬────────────┬────────────┬─────────────────────┐
  │ #  │ Type   │ Database   │ Address     │ Method             │
  ├────┼────────┼────────────┼────────────┼────────────────────┤
  │ 1  │ local  │ all        │ -           │ peer              │
  │ 2  │ hostssl│ all        │ 127.0.0.1  │ scram-sha-256     │
  │ 3  │ hostssl│ myapp      │ 10.0.0.0/8 │ scram-sha-256 ✓  │ ← MATCH
  │ 4  │ host   │ all        │ 10.0.0.0/8 │ reject            │ not checked
  │ 5  │ hostssl│ all        │ 0.0.0.0/0  │ scram-sha-256     │ not checked
  │ 6  │ hostnossl│ all      │ 0.0.0.0/0  │ reject            │ not checked
  └────┴────────┴────────────┴────────────┴────────────────────┘

  Rule 3 matches: myapp database, 10.x.x.x network, SSL=yes
  Authentication: scram-sha-256 challenge/response begins

  ┌─────────────────────────────────────────────────────────────────┐
  │ SCRAM-SHA-256 Authentication Flow:                              │
  │                                                                 │
  │ Client → Server: ClientFirst (username, random nonce)          │
  │ Server → Client: ServerFirst (combined nonce, salt, iterations)│
  │ Client → Server: ClientFinal (proof = HMAC of client key)      │
  │ Server → Client: ServerFinal (server proof) → connection OK    │
  │                                                                 │
  │ Properties:                                                     │
  │ - Password never sent in plaintext                              │
  │ - Mutual authentication (server proves it knows verifier too)  │
  │ - Salt + iterations = expensive brute force                     │
  │ - Stored verifier ≠ password (pg_shadow holds verifier only)   │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Diagram 6: Defense in Depth — Database Security Layers

```
  INTERNET / UNTRUSTED NETWORK
          │
          ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 1: NETWORK FIREWALL                                      │
  │  - Deny all inbound to port 5432 except from application VPC   │
  │  - No direct public internet access to PostgreSQL               │
  │  - VPC security groups / firewall rules                         │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │ (only app VPC traffic)
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 2: TLS/SSL                                               │
  │  - sslmode=verify-full on all clients                           │
  │  - ssl_min_protocol_version = TLSv1.2                           │
  │  - hostnossl → reject in pg_hba.conf                            │
  │  - Prevents: network sniffing, MITM attacks                     │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │ (encrypted connection)
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 3: AUTHENTICATION (pg_hba.conf + scram-sha-256)          │
  │  - Dynamic credentials from Vault (expire every 8 hours)        │
  │  - No static credentials in config files                        │
  │  - Prevents: unauthorized database connections                  │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │ (authenticated)
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 4: AUTHORIZATION (GRANT/REVOKE + Least Privilege)        │
  │  - Application role: only SELECT/INSERT/UPDATE on needed tables  │
  │  - No SUPERUSER, no CREATEDB, no DDL privileges                 │
  │  - Prevents: unauthorized data access, privilege escalation      │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │ (authorized)
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 5: ROW-LEVEL SECURITY                                    │
  │  - Tenant isolation enforced at database layer                  │
  │  - Even bugs in application code cannot cross tenant boundary   │
  │  - Prevents: accidental/deliberate cross-tenant data access     │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 6: ENCRYPTION                                            │
  │  - Column-level encryption (pgcrypto) for PII/sensitive data    │
  │  - Full-disk encryption for disks-at-rest threat                │
  │  - Encrypted backups                                            │
  │  - Prevents: backup theft, disk theft, DBA snooping             │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  LAYER 7: AUDIT LOGGING (pg_audit + pg_stat_activity)           │
  │  - All data access logged with user, timestamp, query           │
  │  - Logs shipped to immutable external store                     │
  │  - Provides: compliance evidence, incident forensics            │
  └─────────────────────────────────────────────────────────────────┘
```
