# Dockerized Deployments — Patterns and Anti-Patterns

---

## Twelve-Factor App Principles for Docker

1. **Codebase**: One repo, many deployments
2. **Dependencies**: Declare in pom.xml / build.gradle
3. **Config**: Environment variables (never in code)
4. **Backing services**: Treat as attached resources (DB URL in env var)
5. **Build/Release/Run**: Separate stages (Docker build → image → run)
6. **Processes**: Stateless processes (store state in DB/Redis)
7. **Port binding**: Export via port (EXPOSE in Dockerfile)
8. **Concurrency**: Scale via process model (replica count)
9. **Disposability**: Fast startup, graceful shutdown
10. **Dev/Prod parity**: Same Docker image everywhere
11. **Logs**: Write to stdout (Docker captures to log driver)
12. **Admin processes**: Run as one-off tasks (migrate DB before main app)

---

## Database Migration Pattern

```yaml
# Separate init container for migrations before app starts
initContainers:
- name: db-migrate
  image: flyway/flyway:9
  command: ["flyway", "migrate"]
  env:
  - name: FLYWAY_URL
    value: jdbc:postgresql://db:5432/myapp
  - name: FLYWAY_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

This pattern ensures:
1. Migrations run before the app starts
2. App doesn't start if migration fails
3. Only one migration job runs (Kubernetes init containers)
