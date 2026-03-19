# Database Security — Analogy Explanations

Conceptual analogies for security concepts that are often misunderstood or superficially understood.

---

## Analogy 1: Row-Level Security — The Hospital Floor with Patient Privacy Curtains

**Story:**
A hospital floor has dozens of patient rooms. Nurses have general access to the floor — they can walk anywhere. But each nurse can only open the medical chart for patients assigned to them. Even if a nurse physically enters a room to help with an emergency, their login badge won't unlock the chart terminal for a different nurse's patient. The medical records system enforces this: it checks the badge ID against the patient assignment list before revealing ANY information. A doctor (superuser) can access any chart — but must still log in with their badge, creating an audit trail. The hospital administrator (table owner) can override any restriction — so the administrator badge should be locked in the safe except during legitimate administrative operations.

**Database connection:**
```sql
-- The badge check in code (RLS policy):
CREATE POLICY patient_access ON medical_records
  FOR SELECT
  USING (
    assigned_nurse_id = (
      SELECT nurse_id FROM current_nurses
      WHERE session_token = current_setting('app.session_token')
    )
  );

-- The administrator override:
ALTER TABLE medical_records FORCE ROW LEVEL SECURITY;
-- Even the table owner must pass the policy now

-- The doctor (superuser) always bypasses — which is why:
-- 1. Don't use superuser as the application role
-- 2. Audit all superuser connections
SET pgaudit.role = 'superuser_audit';  -- log all superuser operations
```

**Why it matters:** Without FORCE ROW LEVEL SECURITY, the table owner role bypasses all policies. If your application role IS the table owner (a common mistake), RLS is completely ineffective.

---

## Analogy 2: SQL Injection — The Forged Check

**Story:**
A bank receives a check. The check form has pre-printed fields: "Pay to the order of: [NAME]" and "Amount: [AMOUNT]". Normally a customer fills in "Alice Jones" and "$500." A fraudster, however, writes: "Alice Jones; and also pay $50,000 to Fraudster Corp" in the NAME field, exploiting the fact that the bank's automated system reads the form literally without understanding where the form field ends and the instruction begins.

A parameterized query is like a check with tamper-proof ink that turns red if anyone writes outside the designated fields — the fraudster's extra text is rejected because it extends beyond the field boundary.

**Database connection:**
```sql
-- VULNERABLE (forged check): user input becomes part of the SQL instruction
query = "SELECT * FROM users WHERE name = '" + user_input + "'"
-- user_input = "'; DROP TABLE users; --"
-- Actual SQL: SELECT * FROM users WHERE name = ''; DROP TABLE users; --'

-- SECURE (parameterized): the field boundary is enforced
query = "SELECT * FROM users WHERE name = $1"
-- $1 is always treated as data, never as SQL instruction
-- user_input = "'; DROP TABLE users; --" is stored as literal string

-- Python example:
import psycopg2
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
# The psycopg2 driver sends the query and parameter separately
# The database server NEVER interprets the parameter as SQL
```

**Second-order injection — the delayed forged check:**
The fraudster deposits a check that says "see back for memo" and on the back writes SQL. The bank files it. Six months later, a new automated system reads the stored memo and executes it as a new check instruction. The fraud occurs not at deposit but at retrieval time — when the stored value is used to construct a new SQL statement without re-sanitizing.

---

## Analogy 3: Principle of Least Privilege — The Hotel Key Card System

**Story:**
A hotel issues key cards. Guests get a card that opens only their room (437). Housekeeping gets a card for their assigned floor's rooms but not the restaurant storage. The restaurant manager's card opens the restaurant storage and kitchen but not guest rooms. The general manager has a master card — but it is locked in the safe and only retrieved with authorization and a logged procedure. An important design principle: the hotel staff badge you use to check guests in should NEVER also be a master key. If a check-in clerk is fooled into giving their badge to a scammer, only the check-in desk is compromised, not the entire hotel.

**Database connection:**
```sql
-- Guest key: application role (just the room it needs)
CREATE ROLE app_service;
GRANT SELECT, INSERT, UPDATE ON orders, customers TO app_service;
-- NOT: GRANT ALL ON ALL TABLES TO app_service

-- Housekeeping: read-only analytics role
CREATE ROLE analytics_reader;
GRANT SELECT ON reports, summaries TO analytics_reader;

-- Restaurant manager: migration role (just schema changes)
CREATE ROLE migrator;
GRANT CREATE ON SCHEMA public TO migrator;
GRANT app_service TO migrator;

-- General manager / master key: superuser
-- Locked in the safe — never used as application connection
-- Retrieved only for explicit administrative operations, fully logged
```

**Why it matters:** The application role (used by thousands of requests per minute) should have the minimum needed. When it is compromised (it will be, eventually), the blast radius is limited to only what that role can access.

---

## Analogy 4: Encryption at Rest vs Encryption in Transit — The Sealed Envelope vs The Secure Tunnel

**Story:**
Imagine your bank statement. Encryption at rest is like sealing the statement in a tamper-evident envelope before filing it in a cabinet — if someone breaks into the filing room and steals the cabinet, they cannot read the statements without breaking the seal (decryption key). Encryption in transit is like using a pneumatic tube sealed with a combination lock to send the statement to you — it cannot be intercepted on the way. They solve different problems: the sealed envelope protects filed data; the pneumatic tube protects moving data. A stolen cabinet full of sealed envelopes is still safe from physical theft, but someone with your combination can open your envelope when it arrives.

**Database connection:**
```
Threat model:
┌─────────────────────────┬────────────────────────────────────────────┐
│ Full-disk encryption    │ Protects against: stolen disk, data center  │
│ (encryption at rest)    │ access, improper disk disposal              │
│                         │ Does NOT protect: running database,         │
│                         │ authorized connections, SQL injection        │
├─────────────────────────┼────────────────────────────────────────────┤
│ SSL/TLS                 │ Protects against: network sniffing,         │
│ (encryption in transit) │ man-in-the-middle attacks                   │
│                         │ Does NOT protect: stored data, server-side  │
│                         │ compromise, authorized sessions             │
├─────────────────────────┼────────────────────────────────────────────┤
│ Column-level encryption │ Protects against: DBA access, SQL injection,│
│ (pgcrypto)              │ database backup theft                       │
│                         │ Does NOT protect: key compromise,           │
│                         │ application memory dumps                    │
└─────────────────────────┴────────────────────────────────────────────┘
```

---

## Analogy 5: pg_hba.conf Authentication — The Building Security Checkpoint

**Story:**
A corporate headquarters has multiple security checkpoints: (1) The building perimeter — only employees from the company IP addresses can enter (firewall). (2) The lobby — a security guard checks your badge for your name and the department you claim to work in (pg_hba.conf matching rules). (3) The badge reader at each department door — verifies the badge is authentic and not forged (authentication method: scram-sha-256). (4) The department directory — confirms you have access to that specific room (GRANT/REVOKE privileges). Bypassing any one checkpoint does not give full access — you need all four.

**Database connection:**
```
# pg_hba.conf — The lobby security checkpoint

# TYPE  DATABASE  USER      ADDRESS         METHOD
# ─────────────────────────────────────────────────────────────────
local   all       postgres  -               peer         # OS user check
local   all       all       -               reject       # No other local
host    myapp     app_svc   10.0.0.0/8      scram-sha-256  # Network with password
hostssl all       all       0.0.0.0/0       scram-sha-256  # SSL required
hostnossl all     all       0.0.0.0/0       reject       # Reject non-SSL

# Processing order: first matching line wins
# If no line matches: connection refused
```

**The nuance:** pg_hba.conf controls WHO can attempt to connect and HOW they authenticate. It does not control WHAT they can do once connected — that is the privilege system (GRANT/REVOKE).

---

## Analogy 6: Dynamic Secrets with Vault — The Vending Machine Access System

**Story:**
A traditional office issues master keys to employees. If an employee leaves or is compromised, you must physically collect all key copies — a slow, unreliable process. A Vault-based system is like a smart vending machine for door codes: when you need access, you request a code from the machine. The machine generates a unique code just for you, valid only for the next 8 hours, and automatically expires it. No one holds a permanent key — codes are issued fresh for each work session and expire automatically. If your code is stolen, it expires in hours. The code was yours alone, so the audit log shows exactly who used it and when.

**Database connection:**
```
Vault Dynamic Secrets for PostgreSQL:
──────────────────────────────────────

1. Application starts → requests credential from Vault
2. Vault generates: app_service_2024_06_15_14:30_xyz (unique username)
              with: random 32-char password
              valid for: 8 hours (lease_duration)
3. Vault creates this role in PostgreSQL:
   CREATE ROLE app_service_2024_06_15_1430_xyz
     LOGIN PASSWORD '<random>' VALID UNTIL 'now + 8 hours';
   GRANT app_readwrite TO app_service_2024_06_15_1430_xyz;
4. Application connects with these dynamic credentials
5. After 8 hours: Vault drops the role automatically
   No credential to rotate, no key to steal
6. Audit trail: every DB access linked to this unique credential

Benefits:
- Leaked credential auto-expires
- Per-service unique credentials → blast radius limited
- Audit trail per credential (not shared 'app_user' for everything)
- No secrets in config files or environment variables
```

---

## Analogy 7: Schema Isolation (Multi-Tenancy) — The Office Building Floors

**Story:**
A shared office building has three tenant isolation options:
1. **Shared floor with name tags** (RLS): All companies share one big floor. Each desk has a company name tag. Employees can only look at desks with their company's tag — enforced by a floor manager who checks before anyone approaches a desk. Efficient use of space, but relies entirely on the floor manager never making a mistake.
2. **Separate floors per company** (per-tenant schemas): Each company has its own floor. The elevator only goes to your floor with your keycard. No accidental mingling possible. More expensive (separate infrastructure per floor) but clearer isolation.
3. **Separate buildings** (per-tenant database): Each company has their own building. Complete isolation — a fire in one building doesn't affect others. Most expensive, hardest to manage, but greatest security guarantee.

**Database connection:**
```sql
-- Option 1: Shared schema + RLS
-- All tenants in public.orders, RLS enforces isolation
-- Pros: easy to manage, efficient; Cons: single policy bug leaks data

-- Option 2: Per-tenant schemas
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;
-- Separate orders table per tenant:
CREATE TABLE tenant_acme.orders (...);
CREATE TABLE tenant_globex.orders (...);
-- Separate DB roles with search_path set to their schema:
ALTER ROLE acme_app_user SET search_path = tenant_acme, public;
ALTER ROLE globex_app_user SET search_path = tenant_globex, public;
-- Pros: schema-level isolation, no RLS required
-- Cons: DDL changes require N schema updates, harder to query across tenants

-- Option 3: Per-tenant database
-- Pros: complete isolation, separate vacuum/backup schedules
-- Cons: connection overhead, cannot JOIN across tenants, PgBouncer complexity
```
