# Docker Backend Architecture Patterns

---

## Local Development Architecture

```
docker-compose.yml:

┌─────────────────────────────────────────────────────────┐
│  Docker Network: backend                                 │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Spring Boot  │  │ PostgreSQL   │  │   Redis      │  │
│  │ :8080        │  │ :5432        │  │ :6379        │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                │                  │            │
│         └────────────────┴──────────────────┘            │
│                    Internal DNS: service names            │
└─────────────────────────────────────────────────────────┘
         │
         │ Port mapping to host
         ▼
  http://localhost:8080  (developer accesses app)
  localhost:5432         (IDE connects to DB for inspection)
```

---

## Container Startup Order with Health Checks

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy  # Wait until DB passes health check
      redis:
        condition: service_healthy

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s  # Grace period before health checks start
```

---

## Resource Limits in Production

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

Set JVM flags to match container limits:
```
Container: 1GB
MaxRAMPercentage=75 → ~768MB heap
```
