# Multi-Stage Docker Builds for Java/Spring Boot

> Reduce image size and improve security by separating build-time and runtime dependencies.

---

## Why Multi-Stage Builds?

A single-stage build includes all build tools (Maven, Gradle, JDK) in the final image:

```
Single-stage image size: ~700MB
  - JDK 21: ~350MB
  - Maven: ~100MB
  - Dependencies: ~200MB
  - Your JAR: ~50MB

Multi-stage image size: ~250MB
  - JRE 21 (runtime only): ~200MB
  - Your JAR: ~50MB
  (Build tools NOT included in final image)
```

---

## Multi-Stage Build for Spring Boot (Maven)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-jammy AS builder

WORKDIR /build

# Copy dependency descriptors first (cache optimization)
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .

# Download dependencies (cached layer — only re-runs if pom.xml changes)
RUN ./mvnw dependency:go-offline -B

# Copy source code (separate layer — rebuilds when source changes)
COPY src/ src/

# Build the application (skip tests for speed — run separately in CI)
RUN ./mvnw package -DskipTests -B

# Stage 2: Extract layers for optimized Docker layer caching
FROM eclipse-temurin:21-jre-jammy AS extractor

WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar

# Spring Boot LayerTools extracts the JAR into layers
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 3: Runtime (final image)
FROM eclipse-temurin:21-jre-jammy AS runtime

# Security: non-root user
RUN addgroup --system spring && adduser --system --ingroup spring spring

WORKDIR /app

# Copy layers in order of how frequently they change (least → most)
# Dependencies (change rarely)
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
# Snapshot dependencies (change occasionally)
COPY --from=extractor /extract/snapshot-dependencies/ ./
# Application code (changes most frequently)
COPY --from=extractor /extract/application/ ./

USER spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "org.springframework.boot.loader.JarLauncher"]
```

---

## Multi-Stage Build for Spring Boot (Gradle)

```dockerfile
FROM eclipse-temurin:21-jdk-jammy AS builder

WORKDIR /build

COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle settings.gradle ./

# Cache Gradle dependencies
RUN ./gradlew dependencies --no-daemon

COPY src/ src/

RUN ./gradlew bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre-jammy AS runtime

RUN addgroup --system spring && adduser --system --ingroup spring spring

WORKDIR /app

COPY --from=builder /build/build/libs/*.jar app.jar

USER spring

EXPOSE 8080

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## Layer Optimization Strategy

Spring Boot JAR structure (with layertools):

```
app.jar
├── BOOT-INF/
│   ├── classes/          ← Your code (changes every build)
│   ├── lib/              ← Dependencies (changes when pom.xml changes)
│   └── classpath.idx
├── META-INF/
└── org/                  ← Spring Boot loader (changes rarely)

Extracted layers:
├── application/          ← Your code + resources
├── snapshot-dependencies/ ← SNAPSHOT dependencies
├── dependencies/         ← Release dependencies (most stable)
└── spring-boot-loader/   ← Loader classes (rarely changes)

Docker cache benefit:
  If only your code changes:
  → Only application/ layer rebuilt
  → All other layers cached ✓
  → Fast push to registry (only small application layer)
```

---

## Build Arguments for Flexible Images

```dockerfile
FROM eclipse-temurin:21-jre-jammy

# Build-time configuration
ARG APP_VERSION=latest
ARG BUILD_DATE
ARG GIT_COMMIT

# Labels for traceability
LABEL org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.vendor="MyCompany"

WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build \
  --build-arg APP_VERSION=1.2.3 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=$(git rev-parse --short HEAD) \
  -t myapp:1.2.3 .
```

---

## GraalVM Native Image (Spring Boot 3+)

```dockerfile
# Stage 1: Build native image (requires GraalVM)
FROM ghcr.io/graalvm/native-image-community:21 AS native-builder

WORKDIR /build

COPY --from=mvn-build /build/target/*.jar app.jar

# Compile to native executable (~1-3 minutes)
RUN native-image \
    -jar app.jar \
    -H:Name=myapp \
    --no-fallback \
    --static

# Stage 2: Minimal runtime image (no JVM needed!)
FROM debian:bookworm-slim AS runtime

WORKDIR /app

# Copy just the native executable
COPY --from=native-builder /build/myapp .

RUN chmod +x myapp

EXPOSE 8080

ENTRYPOINT ["./myapp"]

# Final image size: ~80MB (vs 250MB for JRE image)
# Startup time: ~50ms (vs 2-5 seconds for JVM)
```

---

## Docker Image Security Scanning

```bash
# Scan for vulnerabilities
docker scout cves myapp:latest  # Docker Scout (built-in)
trivy image myapp:latest        # Trivy (popular open-source)

# Common security improvements:
# 1. Use specific image tags (not :latest)
FROM eclipse-temurin:21.0.3_9-jre-jammy  # Specific version

# 2. Non-root user (shown above)

# 3. Read-only filesystem where possible
docker run --read-only -v /tmp:/tmp myapp

# 4. Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# 5. No privileged mode
# Never: docker run --privileged myapp
```
