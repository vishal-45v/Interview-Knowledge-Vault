# Docker Fundamentals for Java Engineers

> Core Docker concepts with Java/Spring Boot focused examples.

---

## What Is Docker?

Docker is a containerization platform that packages an application and its dependencies into a lightweight, portable container. Containers run the same regardless of the host environment.

```
Without Docker:
Developer's Mac        Staging Server         Production
Java 17, Spring 3.x    Java 11, Spring 2.7    Java 21, Spring 3.x
Works on my machine!   Different behavior!    Different again!

With Docker:
Developer's Mac        Staging Server         Production
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Container     │    │  Container     │    │  Container     │
│  Same image    │    │  Same image    │    │  Same image    │
└────────────────┘    └────────────────┘    └────────────────┘
Identical behavior everywhere!
```

---

## Core Concepts

### Image vs Container

```
Image: Read-only template (like a class)
  - Built from Dockerfile
  - Stored in registry (Docker Hub, ECR, GCR)
  - Layers stacked on base image

Container: Running instance of an image (like an object)
  - Writable layer on top of image
  - Has its own network, filesystem, process space
  - Ephemeral by default (destroyed = data lost unless volume)
```

### Dockerfile for Spring Boot Application

```dockerfile
# Base image with JRE (not JDK — smaller for runtime)
FROM eclipse-temurin:21-jre-jammy

# Create non-root user for security
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# Set working directory
WORKDIR /app

# Copy JAR (built separately with Maven/Gradle)
COPY target/myapp-*.jar app.jar

# Switch to non-root user
USER appuser

# Expose port (documentation only — doesn't actually publish)
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Entry point
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml for Local Development

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/myapp
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=postgres
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  postgres_data:

networks:
  backend:
    driver: bridge
```

---

## Docker Networking

```
Bridge Network (default):
  Containers on same bridge network communicate by service name
  app → db:5432 (service name "db" resolves to container IP)

Host Network:
  Container shares host's network stack
  No isolation — ports directly on host
  Better performance, worse isolation

None:
  No networking — isolated container

Port Mapping:
  -p 8080:8080   (host_port:container_port)
  App inside container listens on 8080
  Accessible from host on port 8080
```

---

## Docker Volumes for Persistence

```bash
# Named volume (managed by Docker)
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres

# Bind mount (host directory)
docker run -v /home/user/data:/app/data myapp
# Changes in host visible in container and vice versa

# tmpfs (in-memory, ephemeral)
docker run --tmpfs /tmp myapp
```

---

## Essential Docker Commands for Java Development

```bash
# Build image
docker build -t myapp:1.0.0 .
docker build -t myapp:latest --build-arg JAR_FILE=target/myapp.jar .

# Run container
docker run -d -p 8080:8080 --name myapp myapp:latest
docker run -d -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod myapp:latest

# Logs
docker logs myapp --follow          # Follow live logs
docker logs myapp --tail 100        # Last 100 lines

# Execute inside container
docker exec -it myapp bash          # Interactive shell
docker exec myapp java -version     # Run single command

# Container management
docker ps -a                        # All containers
docker stop myapp && docker rm myapp
docker inspect myapp                # Detailed container info

# Image management
docker images
docker rmi myapp:1.0.0
docker image prune -f               # Remove unused images

# docker-compose commands
docker-compose up -d                # Start all services
docker-compose logs -f app          # Follow app logs
docker-compose down -v              # Stop and remove volumes
docker-compose ps                   # Status of services
```

---

## JVM Container Awareness

**Problem:** Old JVMs (pre-Java 8u191) read CPU and memory from the HOST, not the container. A container limited to 512MB might still configure a 4GB JVM heap if the host has 32GB RAM.

**Modern JVM (Java 8u191+, Java 11+):** Container-aware by default.

```dockerfile
# Good: Let JVM detect container limits automatically
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \          # Enabled by default in Java 11+
    "-XX:MaxRAMPercentage=75.0", \          # Use 75% of container RAM for heap
    "-XX:InitialRAMPercentage=50.0", \
    "-jar", "app.jar"]

# Example: Container with 1GB limit
# MaxRAMPercentage=75 → 768MB heap
# Remaining 25% for off-heap (metaspace, threads, native memory)

# Avoid: Explicit -Xmx without knowing container size
ENTRYPOINT ["java", "-Xmx512m", "-jar", "app.jar"]  # OK if you know the container size
```

---

## Common Docker Interview Questions

**Q: What is the difference between CMD and ENTRYPOINT?**

ENTRYPOINT: The command that always runs. Cannot be overridden with docker run arguments (unless --entrypoint flag).

CMD: Default arguments to ENTRYPOINT, OR the default command if no ENTRYPOINT. Can be overridden with docker run arguments.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=default"]
# docker run myapp → java -jar app.jar --spring.profiles.active=default
# docker run myapp --spring.profiles.active=prod → java -jar app.jar --spring.profiles.active=prod
```

**Q: What is the difference between COPY and ADD?**

COPY: Copies local files into the image. Recommended for most cases.

ADD: Can also extract archives (.tar.gz) and download from URLs. Use COPY unless you specifically need these features.

**Q: What is a Docker layer?**

Each instruction in a Dockerfile creates a layer. Layers are cached — if an instruction's context hasn't changed, Docker reuses the cached layer. This makes rebuilds fast.

Put frequently-changing instructions (COPY app.jar) at the bottom, rarely-changing ones (apt-get install) at the top.
