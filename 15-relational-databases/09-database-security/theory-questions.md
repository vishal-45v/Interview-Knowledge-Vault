# Database Security — Theory Questions

Senior engineer / database architect level. No answers provided — use for self-assessment.

---

1. Explain the PostgreSQL role and privilege system. What is the difference between a role with LOGIN privilege and a group role? How does role inheritance work, and what does NOINHERIT change? How do you use SET ROLE versus SET SESSION AUTHORIZATION, and when would you choose each?

2. What is the difference between object-level privileges in PostgreSQL (GRANT SELECT ON TABLE to role) and schema-level default privileges (ALTER DEFAULT PRIVILEGES)? If a new table is created in a schema, does an existing GRANT on the schema automatically cover the new table?

3. Explain Row-Level Security (RLS) in PostgreSQL. What is the difference between a USING policy (for SELECT/DELETE/UPDATE) and a WITH CHECK policy (for INSERT/UPDATE)? What does FORCE ROW LEVEL SECURITY do, and why is it needed even for table owners?

4. What is column-level security in PostgreSQL? How do you grant SELECT on specific columns but not others? What happens when a user tries to SELECT * from a table where they only have access to certain columns? How does this interact with RLS?

5. Describe the SQL injection attack pattern. What makes parameterized queries safe from SQL injection while string concatenation is not? Give an example of a second-order SQL injection attack and explain why parameterized queries alone may not prevent it in all cases.

6. What does the pg_audit extension provide beyond standard PostgreSQL logging? What are the two audit log methods (session and object) and how do they differ? What is the pgaudit.log parameter and what classes can be logged?

7. What are the authentication methods in pg_hba.conf? Explain md5, scram-sha-256, cert, ldap, and peer authentication. Why is scram-sha-256 preferred over md5, and what version of PostgreSQL introduced it? What is the trust method and why is it dangerous?

8. Explain SSL/TLS configuration for PostgreSQL. What are the server-side parameters (ssl, ssl_cert_file, ssl_key_file, ssl_ca_file)? What does ssl_min_protocol_version do? How do you require (not just allow) SSL connections from clients, and how do you verify this is enforced?

9. What is the principle of least privilege in the context of database access? Describe a complete privilege model for a production application with separate roles for: application reads, application writes, migration execution, monitoring, and backup operations.

10. What is pg_hba.conf and how does PostgreSQL use it? What is the processing order of entries in pg_hba.conf? Can you have multiple lines matching the same connection — which one wins? What is the consequence of a misconfigured pg_hba.conf on startup?

11. Explain transparent data encryption (TDE) in the context of PostgreSQL. Does PostgreSQL have built-in TDE? What alternatives exist (OS-level encryption, pgcrypto, file system encryption)? What threat model does TDE address and what does it NOT protect against?

12. What is the pgcrypto extension? What functions does it provide for symmetric encryption, asymmetric encryption, and hashing? How would you store encrypted credit card numbers in PostgreSQL using pgcrypto, and what are the key management challenges?

13. Describe schema isolation patterns for multi-tenant applications. Compare: shared schema (RLS), per-tenant schema (separate schema per tenant), and per-tenant database (separate database per tenant). What are the security, scalability, and maintenance trade-offs of each?

14. What is database auditing and why is it required for compliance frameworks (SOC2, HIPAA, PCI DSS)? What specifically must be logged to satisfy PCI DSS requirements for a database storing cardholder data? What PostgreSQL configuration achieves this?

15. Explain how HashiCorp Vault integrates with PostgreSQL for database credential management. What is dynamic secrets generation in Vault, and how does it eliminate the "static credential" problem? What happens when a Vault lease expires?

16. What is a database firewall? How does pgBadger or a network-level WAF differ from pg_hba.conf in terms of what they protect against? Describe a defense-in-depth security architecture for a PostgreSQL instance accessible from the internet.

17. What does GRANT EXECUTE ON FUNCTION mean in PostgreSQL? How do SECURITY DEFINER functions differ from SECURITY INVOKER functions? What security vulnerability can a SECURITY DEFINER function introduce if not written carefully?

18. How do you implement data masking in PostgreSQL? What is the difference between static data masking (for non-production copies) and dynamic data masking (masking at query time)? How can views and RLS work together to implement dynamic masking?

19. What is the REVOKE ALL PRIVILEGES statement, and what does it NOT revoke? Explain the difference between REVOKE on a TABLE versus REVOKE on a SCHEMA. What does REVOKE privilege CASCADE do compared to REVOKE privilege RESTRICT?

20. Describe the security implications of the search_path configuration parameter in PostgreSQL. What is a "search_path injection" attack? How can a malicious schema in a user's search_path lead to privilege escalation, and how do you prevent it in security-sensitive applications?
