# Docker Production Mistakes — What NOT to Do

---

## Mistake 1: Running as Root

```dockerfile
# BAD: Runs as root — security risk
FROM eclipse-temurin:21-jre-jammy
COPY app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]

# GOOD: Non-root user
FROM eclipse-temurin:21-jre-jammy
RUN addgroup --system spring && adduser --system --ingroup spring spring
COPY --chown=spring:spring app.jar app.jar
USER spring
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Mistake 2: Using :latest Tag

```bash
# BAD: Non-deterministic builds
FROM eclipse-temurin:latest      # Which version? Could change tomorrow!
FROM eclipse-temurin:21-jre      # Still could change on patch updates

# GOOD: Pin to specific version
FROM eclipse-temurin:21.0.3_9-jre-jammy  # Exact version
```

---

## Mistake 3: No Health Check

```bash
# BAD: No health check — container shows as running but app may be broken
docker run myapp  # Container starts, Kubernetes routes traffic, but app is stuck

# GOOD: Health check so orchestrators know the true app state
HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1
```

---

## Mistake 4: Hard-Coded Secrets in Dockerfile

```dockerfile
# BAD: Secret in image
ENV DB_PASSWORD=supersecret123  # Anyone with the image can see this!

# GOOD: Pass at runtime
docker run -e DB_PASSWORD=$DB_PASSWORD myapp
# Or use Docker secrets / Kubernetes secrets
```

---

## Mistake 5: Not Handling SIGTERM

```dockerfile
# BAD: PID 1 is shell — doesn't forward signals
ENTRYPOINT ["sh", "-c", "java -jar app.jar"]
# docker stop → SIGTERM to shell → shell killed → app hard-killed

# GOOD: Java is PID 1 — receives SIGTERM directly
ENTRYPOINT ["java", "-jar", "app.jar"]
# docker stop → SIGTERM to java → Spring's graceful shutdown → clean exit
```

---

## Mistake 6: Large Docker Build Context

```bash
# Add .dockerignore to exclude unnecessary files
# .dockerignore:
.git
target/
*.log
*.md
.idea/
node_modules/
```

---

## Mistake 7: Not Using Multi-Stage Builds

Single-stage Spring Boot image: ~700MB
Multi-stage Spring Boot image: ~250MB
Smaller image = faster pull = less attack surface
