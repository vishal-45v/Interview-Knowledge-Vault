# Cloud Fundamentals — Theory Questions

---

## Question 1: What are the 12-Factor App principles most relevant to Java microservices?

| Factor | Principle | Java/Spring Implementation |
|---|---|---|
| I. Codebase | One codebase, tracked in VCS | One Git repo per service, mono-repo per domain |
| II. Dependencies | Explicitly declare, isolate dependencies | Maven/Gradle pom.xml, no system-level jars |
| III. Config | Store config in environment | `${ENV_VAR}`, Spring Cloud Config |
| IV. Backing services | Treat as attached resources | Externalize DB, cache, queue URLs |
| V. Build/release/run | Strictly separate stages | Docker build → Helm release → k8s run |
| VI. Processes | Stateless processes | No sticky sessions, share-nothing |
| VII. Port binding | Export services via port | `server.port=8080`, embedded Tomcat |
| VIII. Concurrency | Scale out via process model | Horizontal scaling in Kubernetes |
| IX. Disposability | Fast startup, graceful shutdown | Spring Boot startup < 5s, `SIGTERM` handler |
| X. Dev/prod parity | Keep dev, staging, prod similar | Docker Compose for local, Helm for prod |
| XI. Logs | Treat logs as event streams | Write to stdout, aggregate with ELK/Fluentd |
| XII. Admin processes | Run admin tasks as one-off processes | `kubectl exec` or k8s Jobs |

**Most critical for interviews:** Factor III (config), VI (stateless), IX (disposability), XI (logs).

---

## Question 2: Explain horizontal vs vertical scaling. When do you choose each?

```
Vertical Scaling:          Horizontal Scaling:
  ┌──────────┐              ┌────┐ ┌────┐ ┌────┐
  │  Server  │              │ S1 │ │ S2 │ │ S3 │
  │ ↑ more   │              └────┘ └────┘ └────┘
  │ CPU/RAM  │                 └─────┬─────┘
  └──────────┘                       LB
```

**Vertical (scale up):**
- Add CPU, RAM, storage to existing server
- Simpler — no application changes needed
- Hard limit: there's a maximum machine size
- Single point of failure
- Expensive at large scale (exponential cost curve)

**Horizontal (scale out):**
- Add more instances of the same server
- Requires stateless application design
- Better for cloud-native (pay per usage, auto-scaling)
- Resilient — one instance can fail without outage
- Load balancer required

**Java applications — horizontal scaling requirements:**
```java
// GOOD — stateless: user data in database, cache, or JWT (not in-process)
@Service
public class SessionService {
    private final RedisTemplate<String, UserSession> redis;  // external state store

    public UserSession getSession(String token) {
        return redis.opsForValue().get("session:" + token);  // any instance can serve
    }
}

// BAD — sticky session: state in JVM heap
private final Map<String, UserSession> sessions = new ConcurrentHashMap<>();
// Instance 2 doesn't have sessions from Instance 1!
```

---

## Question 3: What is containerization? How does Docker work internally?

**Container vs VM:**
```
VM:                             Container:
┌─────────────┐                ┌──────────────────────────┐
│   App A     │                │  App A  │  App B  │  App C│
├─────────────┤                ├─────────┴─────────┴───────┤
│  Guest OS A │                │         Docker            │
├─────────────┤                ├───────────────────────────┤
│  Hypervisor │                │      Host OS Kernel        │
└─────────────┘                └───────────────────────────┘
Boots in minutes               Starts in seconds
GBs of overhead               MBs overhead
Full OS isolation              Process-level isolation
```

**Docker internals:**
- **Namespaces** — isolate PID, network, filesystem, users, IPC per container
- **cgroups** — limit CPU, memory, disk I/O for each container
- **Union filesystem (OverlayFS)** — image layers are read-only; container gets a writable layer on top

**Efficient Java Dockerfile:**
```dockerfile
# Multi-stage build — build artifact in one stage, run in minimal image
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Runtime stage — minimal JRE image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy only the artifact
COPY --from=builder --chown=appuser:appgroup /app/target/app.jar app.jar

# JVM tuning for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport"

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**JVM in containers pitfall:** Java < 10 didn't respect container cgroup limits — used host CPU/memory values. `-XX:+UseContainerSupport` (default since Java 10) fixes this.

---

## Question 4: Explain Kubernetes core concepts: Pod, Deployment, Service, Ingress.

```
Internet → Ingress → Service → Deployment → Pod(s)
                                              ↓
                                         Container(s)
```

**Pod:** Smallest deployable unit. One or more containers sharing network namespace and storage. Ephemeral — pods die and are replaced.

**Deployment:** Manages a ReplicaSet. Declares desired state (3 replicas of image:v2). Handles rolling updates and rollbacks.

**Service:** Stable network endpoint for a set of pods (selected by labels). Types:
- `ClusterIP` — internal only
- `NodePort` — accessible on each node's port
- `LoadBalancer` — external cloud load balancer

**Ingress:** HTTP/HTTPS routing rules. Routes external traffic to services based on host/path.

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-password
```

---

## Question 5: What are liveness and readiness probes? What's the difference?

**Liveness probe:** Is the application alive? If it fails, Kubernetes kills and restarts the pod.
- **Use case:** Detect deadlocks, infinite loops, OOM states the app can't recover from
- **Failure action:** Kill and restart the container

**Readiness probe:** Is the application ready to serve traffic? If it fails, Kubernetes stops sending traffic to the pod (removes from Service endpoints).
- **Use case:** App starting up (loading caches, waiting for DB), temporarily overloaded
- **Failure action:** Remove from load balancer, NOT restart

```java
// Spring Boot Actuator — separate liveness and readiness endpoints
management:
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  endpoint:
    health:
      probes:
        enabled: true

// Liveness check — is the process healthy?
// Spring's default: LIVE if ApplicationContext started

// Readiness check — can the app serve requests?
@Component
public class DatabaseReadinessIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            dataSource.getConnection().isValid(2);
            return Health.up().build();
        } catch (Exception e) {
            // During startup, DB connection may not be ready yet
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

**Startup probe (Kubernetes 1.18+):** For slow-starting applications. Replaces liveness check until startup succeeds. Prevents premature killing of slow-starting JVMs.

---

## Question 6: What is Kubernetes rolling deployment? What are blue-green and canary deployments?

**Rolling deployment (default):**
- Gradually replaces old pods with new pods
- `maxUnavailable=1`: at most 1 pod down at a time
- `maxSurge=1`: at most 1 extra pod created
- Zero-downtime if readiness probes are configured correctly

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

**Blue-Green deployment:**
```
Blue (v1):  ●●●●  ← currently live
Green (v2): ●●●●  ← new version deployed but not live
                   Switch: update Service selector from blue to green
                   Instant cutover, zero-downtime
                   Easy rollback: switch back to blue
```
- Double the infrastructure during deployment
- Instant rollback by switching service selector

**Canary deployment:**
```
90% → Service v1 (stable)
10% → Service v2 (canary)
     ↓
Monitor: error rates, latency, business metrics
     ↓
If healthy: gradually shift more traffic to v2
If issues: shift 100% back to v1
```

**Kubernetes canary via Argo Rollouts or Flagger:**
```yaml
# Argo Rollouts canary strategy
strategy:
  canary:
    steps:
    - setWeight: 10
    - pause: {duration: 10m}
    - setWeight: 30
    - pause: {duration: 10m}
    - setWeight: 60
    - pause: {duration: 10m}
    analysis:
      templates:
      - templateName: error-rate-check
```

---

## Question 7: Explain the Circuit Breaker pattern. What are the states?

**Problem:** If Service A calls Service B which is down, A's threads pile up waiting for timeouts, causing A to also go down (cascading failure).

**Circuit Breaker states:**
```
                  failure threshold exceeded
CLOSED ───────────────────────────────────────→ OPEN
(normal: calls pass through)                   (all calls fail immediately)
        ↑                                              │
        │ success threshold met              wait timeout
        │                                              ↓
        └────────────────────────────── HALF-OPEN
                                        (probe: allow one call)
```

```java
// Resilience4j circuit breaker
@Bean
public CircuitBreaker inventoryCircuitBreaker(CircuitBreakerRegistry registry) {
    CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50)          // open if 50% of calls fail
        .slowCallRateThreshold(80)         // treat slow calls as failures
        .slowCallDurationThreshold(Duration.ofSeconds(2))
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .permittedNumberOfCallsInHalfOpenState(3)
        .slidingWindowSize(10)
        .minimumNumberOfCalls(5)
        .build();
    return registry.circuitBreaker("inventory", config);
}

@Service
public class InventoryService {

    public StockLevel getStock(String sku) {
        return circuitBreaker.executeSupplier(
            () -> inventoryClient.getStock(sku),           // supplier
            throwable -> StockLevel.unknown(sku)           // fallback
        );
    }
}
```

**Combined with retry:** Retry retries transient failures. Circuit breaker stops trying when the downstream is consistently down.
- Order: Circuit Breaker → Retry → Timeout → Bulkhead (outermost to innermost)

---

## Question 8: What is a Service Mesh? What problems does it solve?

**Service Mesh:** Infrastructure layer that handles service-to-service communication. Implemented as sidecar proxies (e.g., Envoy in Istio).

```
┌──────────────────────────────────────────────────┐
│  Pod A                          Pod B             │
│  ┌─────────┐   ┌──────────┐   ┌──────┐ ┌───────┐│
│  │ App A   │↔  │ Envoy    │→  │Envoy │↔│ App B ││
│  │         │   │ (sidecar)│   │      │ │       ││
│  └─────────┘   └──────────┘   └──────┘ └───────┘│
└──────────────────────────────────────────────────┘
         ↑                           ↑
    All traffic passes through sidecars
    mTLS, observability, retries handled transparently
```

**Problems solved:**
- **mTLS everywhere** — automatic certificate rotation, no app code changes
- **Observability** — distributed tracing (Jaeger), metrics, traffic visualization
- **Traffic management** — canary deployments, A/B testing, circuit breaking
- **Retries and timeouts** — configured in mesh, not application code

**When to use a service mesh:**
- Large number of microservices (10+)
- Need mTLS without application code changes
- Fine-grained traffic control (canary by header, dark launches)
- Compliance requirements for encrypted internal traffic

**Cost:**
- Each pod gets a sidecar proxy → 25-50MB extra memory per pod
- Latency overhead of ~1ms per hop (proxy-to-proxy)
- Operational complexity of managing Istio/Linkerd

---

## Question 9: Explain graceful shutdown in a Kubernetes Java application.

**Graceful shutdown process:**
```
SIGTERM received → stop accepting new requests
                → complete in-flight requests (30s timeout)
                → close DB connections
                → exit with code 0

If not done in 30s: SIGKILL (forceful termination)
```

```java
// Spring Boot graceful shutdown (enabled by default in 2.3+)
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

// Custom shutdown hooks
@Component
public class GracefulShutdownHook implements DisposableBean {

    private final ConnectionPool connectionPool;
    private final KafkaConsumer consumer;

    @Override
    public void destroy() {
        log.info("Starting graceful shutdown...");

        // 1. Stop accepting new Kafka messages
        consumer.pause(consumer.assignment());

        // 2. Wait for in-flight processing to complete
        awaitCompletion(30, TimeUnit.SECONDS);

        // 3. Commit offsets
        consumer.commitSync();

        // 4. Close connections
        connectionPool.close();

        log.info("Graceful shutdown complete");
    }
}

// Kubernetes preStop hook for readiness probe delay
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # wait for load balancer to stop routing
```

**Why `sleep` in preStop?** There's a race condition: Kubernetes removes the pod from Service endpoints (telling load balancer to stop routing) and sends SIGTERM simultaneously. The load balancer may take a few seconds to stop routing. A short `sleep 5` ensures no requests come in during shutdown.

---

## Question 10: What is the difference between configMap and Secrets in Kubernetes?

| Feature | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config | Sensitive data |
| Storage | Plaintext etcd | base64-encoded etcd (NOT encrypted by default!) |
| Use as | Volume, env var | Volume, env var |
| Max size | 1MB | 1MB |

```yaml
# ConfigMap — application.properties override
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application.yml: |
    server.port: 8080
    logging.level.root: INFO
    spring.datasource.url: jdbc:postgresql://postgres:5432/orders

---
# Secret — credentials (consider Sealed Secrets or External Secrets Operator for GitOps)
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=  # base64 encoded — NOT encrypted!
  jwt-secret: bXlzZWNyZXRrZXk=

---
# Mount both in deployment
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /config
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: order-service-secrets
          key: db-password
  volumes:
  - name: config
    configMap:
      name: order-service-config
```

**Production secret management:** Use External Secrets Operator (syncs from AWS Secrets Manager / HashiCorp Vault) or Sealed Secrets (encrypt secrets before committing to Git).

---

## Question 11: What is observability? Explain the three pillars.

**Observability:** Ability to understand internal system state from external outputs.

**Three pillars:**

```
Logs        → WHAT happened (discrete events)
Metrics     → HOW MUCH / HOW OFTEN (aggregated numbers)
Traces      → HOW IT HAPPENED (request path through services)
```

**Logs:**
```java
// Structured logging with SLF4J + Logback
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

public Order placeOrder(CreateOrderRequest request) {
    log.info("Placing order customerId={} items={} totalAmount={}",
        request.customerId(), request.items().size(), request.totalAmount());
    // Structured fields for easy filtering in Kibana/Splunk
}
```

**Metrics with Micrometer:**
```java
// Micrometer + Prometheus
@Service
public class OrderService {
    private final Counter ordersPlaced;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry registry) {
        ordersPlaced = registry.counter("orders.placed",
            "service", "order-service");
        orderProcessingTime = registry.timer("orders.processing.time",
            "service", "order-service");
    }

    public Order placeOrder(CreateOrderRequest request) {
        return orderProcessingTime.recordCallable(() -> {
            Order order = doPlaceOrder(request);
            ordersPlaced.increment();
            return order;
        });
    }
}
```

**Distributed Tracing with OpenTelemetry:**
```java
// Auto-instrumented with Spring Boot + OpenTelemetry agent
// No code changes needed for basic tracing
// Manual spans for business events
@Autowired private Tracer tracer;

public Order placeOrder(CreateOrderRequest request) {
    Span span = tracer.spanBuilder("placeOrder")
        .setAttribute("customer.id", request.customerId())
        .startSpan();
    try (Scope scope = span.makeCurrent()) {
        return doPlaceOrder(request);
    } finally {
        span.end();
    }
}
```

---

## Question 12: What is eventual consistency and CAP theorem?

**CAP Theorem:** In a distributed system, you can have at most two of:
- **C**onsistency — every read returns the most recent write
- **A**vailability — every request receives a response (not error)
- **P**artition tolerance — system works despite network partitions

**In practice:** Network partitions happen → you must choose CP or AP.

```
CP (Consistency + Partition):
  - If partition: return error rather than stale data
  - Example: HBase, ZooKeeper, etcd
  - Use when: financial transactions, leader election

AP (Availability + Partition):
  - If partition: return potentially stale data
  - Example: Cassandra, DynamoDB, CouchDB
  - Use when: shopping carts, social feeds, DNS
```

**Eventual Consistency:** All replicas converge to the same value given enough time and no new updates. Used in AP systems.

```java
// Eventual consistency example: inventory update
// Order service writes to its DB immediately (strong consistency)
// Inventory service updates asynchronously via event (eventual consistency)

@Service
public class OrderService {
    @Transactional
    public Order placeOrder(CreateOrderRequest req) {
        Order order = createOrder(req);
        // Immediate, consistent — order is placed
        orderRepository.save(order);

        // Eventual — inventory will be updated eventually
        eventPublisher.publish(new OrderPlacedEvent(order));
        return order;
    }
}

// Inventory service processes asynchronously
@KafkaListener(topics = "order-placed")
public void updateInventory(OrderPlacedEvent event) {
    // Might process this seconds later — eventually consistent
    inventoryService.decrementStock(event.getItems());
}
```

**PACELC theorem:** Extension of CAP — even without partitions, there's a Latency vs Consistency trade-off.

---

## Question 13: What is Infrastructure as Code (IaC)? How do Helm charts work?

**IaC:** Manage infrastructure using code (version-controlled, repeatable, auditable).

**Tools:**
- **Terraform** — cloud resources (VMs, databases, networking)
- **Helm** — Kubernetes application deployments
- **Ansible** — configuration management

**Helm Chart structure:**
```
order-service/
├── Chart.yaml          # chart metadata
├── values.yaml         # default configuration values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # shared template functions
└── charts/             # dependency charts
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myregistry/order-service
  tag: "1.2.3"
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi
ingress:
  host: api.example.com

# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        resources: {{ .Values.resources | toYaml | nindent 12 }}
```

```bash
# Deploy to environment with override values
helm upgrade --install order-service ./order-service \
  --namespace production \
  --set image.tag=1.3.0 \
  --values ./envs/production-values.yaml

# Rollback
helm rollback order-service 2
```

---

## Question 14: Explain the Sidecar and Ambassador patterns in Kubernetes.

**Sidecar Pattern:** A helper container in the same pod as the main application. Shares network and storage.

```yaml
spec:
  containers:
  - name: app
    image: order-service:latest
    # App writes logs to /var/log/app.log
  - name: log-forwarder  # Sidecar
    image: fluent-bit:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
    # Reads logs and ships to Elasticsearch
```

**Use cases for sidecars:**
- Log shipping (Fluent Bit, Filebeat)
- Metrics collection (Prometheus exporters)
- Service mesh proxy (Envoy in Istio)
- Config reload watchers

**Ambassador Pattern:** Proxy sidecar that handles external communication on behalf of the main container.

```yaml
spec:
  containers:
  - name: app
    image: my-service:latest
    env:
    - name: DB_URL
      value: "localhost:5432"  # connects to ambassador sidecar
  - name: db-ambassador  # Ambassador
    image: pgbouncer:latest
    # Pools connections, handles retries, does connection management
    # App doesn't need to know real DB location or handle pooling
```

---

## Question 15: What is the difference between Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA)?

**HPA (Horizontal Pod Autoscaler):**
- Scales the *number* of pod replicas based on metrics
- Metrics: CPU/memory (built-in), custom metrics via Prometheus Adapter

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 100
```

**VPA (Vertical Pod Autoscaler):**
- Adjusts CPU/memory *requests and limits* for pods
- Useful for right-sizing — finds optimal resource allocation
- Cannot be used with HPA on the same metric

**KEDA (Kubernetes Event Driven Autoscaling):**
- Scales based on external events (Kafka lag, SQS queue depth, Redis list length)
- Scale to zero when no events

```yaml
# KEDA ScaledObject — scale order-processors based on Kafka consumer lag
kind: ScaledObject
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0   # scale to zero when queue is empty
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      topic: orders
      bootstrapServers: kafka:9092
      consumerGroup: order-processors
      lagThreshold: "100"  # 1 replica per 100 messages of lag
```

---

## Question 16: What is a ConfigMap vs environment variable vs command-line arg in Spring Boot?

**Priority order (highest to lowest) in Spring Boot:**

```
1. Command-line arguments (--server.port=8081)
2. OS environment variables (SERVER_PORT=8081)
3. application-{profile}.yml
4. application.yml
5. @PropertySource
6. Default values
```

```java
// Kubernetes injects config as env vars or mounted files
@Value("${spring.datasource.url}")  // reads from env DB_URL (kebab-case auto-converted)
private String dbUrl;

// Prefer @ConfigurationProperties for grouped config
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentConfig {
    @NotBlank private String apiKey;
    @Positive private int connectionTimeout;
    @NotNull private RetryConfig retry;
}

// Spring Cloud Config — centralized config server
spring:
  config:
    import: configserver:http://config-server:8888
  cloud:
    config:
      profile: production
      label: main  # git branch
```

---

## Question 17: Explain the Strangler Fig pattern for microservices migration.

**Problem:** Migrating a monolith to microservices. You can't rewrite everything at once — too risky.

**Strangler Fig:** Incrementally replace monolith functionality behind a facade/proxy.

```
Phase 1:                      Phase 2:                      Phase 3:
                               ┌──────────────────────┐
  Client → Monolith            Client → Facade         Client → Microservices
                               ├──────────────────────┤
                               │ /orders → Monolith   │    all routes → μservices
                               │ /users  → μservice   │    Monolith decommissioned
                               └──────────────────────┘
```

```java
// Strangler facade — Spring Cloud Gateway routing
spring:
  cloud:
    gateway:
      routes:
      # New microservice for users
      - id: user-service
        uri: http://user-service:8080
        predicates:
        - Path=/api/users/**
        filters:
        - RewritePath=/api/users/(?<segment>.*), /users/${segment}

      # Everything else still goes to the monolith
      - id: monolith
        uri: http://legacy-monolith:8080
        predicates:
        - Path=/**
```

**Migration order:** Start with bounded contexts that have clear domain boundaries and few dependencies (e.g., user management, notifications). Leave complex, coupled modules for last.

---

## Question 18: What is a health check endpoint and what should it verify?

```java
@Component
public class ApplicationHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;
    private final RedisTemplate<String, String> redis;
    private final KafkaTemplate<String, Object> kafka;

    @Override
    public Health health() {
        Map<String, Object> details = new LinkedHashMap<>();
        boolean healthy = true;

        // Check database
        try {
            try (Connection conn = dataSource.getConnection()) {
                conn.isValid(2);
                details.put("database", "UP");
            }
        } catch (Exception e) {
            details.put("database", "DOWN: " + e.getMessage());
            healthy = false;
        }

        // Check Redis
        try {
            redis.opsForValue().get("healthcheck");
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DEGRADED");
            // Redis down might not be critical — depends on usage
        }

        return healthy
            ? Health.up().withDetails(details).build()
            : Health.down().withDetails(details).build();
    }
}
```

**What liveness check should NOT verify:** External dependencies (DB, cache). If the DB is down, killing and restarting the pod won't fix it — it'll just cause a restart loop. Liveness = "is this JVM process in a healthy state."

**What readiness check should verify:** External dependencies. If DB is down, the pod can't serve requests → remove from load balancer rotation.

