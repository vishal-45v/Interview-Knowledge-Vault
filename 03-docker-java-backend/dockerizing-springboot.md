# Dockerizing Spring Boot Applications

---

## Production-Ready Spring Boot Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-jammy

# Non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

WORKDIR /app

# Copy JAR
COPY target/app.jar app.jar

# JVM flags for container environment
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=3 \
    CMD curl -sf http://localhost:8080/actuator/health | grep -q '"status":"UP"' || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## Spring Boot application.yml for Docker

```yaml
# application-docker.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:myapp}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

All configuration via environment variables — 12-factor app principle.

---

## Common Issues When Dockerizing Spring Boot

1. **Application won't start:** DB not ready when app starts
   Fix: Use depends_on with healthcheck, add retry logic, use spring-retry

2. **Out of memory:** JVM uses host memory (old JVM versions)
   Fix: Java 11+ is container-aware, use -XX:MaxRAMPercentage

3. **Slow startup:** Classpath scanning takes too long
   Fix: spring.main.lazy-initialization=true or native image

4. **Large image size:** JDK in image instead of JRE
   Fix: Use eclipse-temurin:21-jre instead of eclipse-temurin:21-jdk

5. **Security:** Running as root user
   Fix: Add non-root user in Dockerfile
