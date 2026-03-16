# Cloud Fundamentals — Scenario Questions

---

## Scenario 1: Kubernetes Deployment with Zero-Downtime Rolling Update

**Setup:** Deploy a new version of your Java service with zero downtime. The service connects to a PostgreSQL database.

```yaml
# deployment.yaml with production-grade configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
    version: "1.3.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # never reduce below 3 pods during update
      maxSurge: 1        # allow 1 extra pod (so max 4 during rollout)
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 60  # matches graceful shutdown timeout
      containers:
      - name: order-service
        image: myregistry/order-service:1.3.0
        ports:
        - containerPort: 8080
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]  # drain in-flight requests
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 30  # 30 * 10s = 5 minutes for startup
          periodSeconds: 10
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1024Mi"
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-password
```

**Application.yml for graceful shutdown:**
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 50s  # less than terminationGracePeriodSeconds
```

---

## Scenario 2: Implementing Health Checks for a Downstream-Dependent Service

**Setup:** Your order service depends on a payment provider API. If the payment API is down, order placement is impossible. Implement health checks that reflect this correctly.

```java
@Component
public class PaymentProviderHealthIndicator implements HealthIndicator {

    private final PaymentClient paymentClient;
    private final CircuitBreaker circuitBreaker;

    @Override
    public Health health() {
        // Check circuit breaker state first (cheap)
        CircuitBreaker.State state = circuitBreaker.getState();

        if (state == CircuitBreaker.State.OPEN) {
            return Health.down()
                .withDetail("reason", "Circuit breaker is OPEN")
                .withDetail("circuitBreakerState", state.name())
                .build();
        }

        // Do a lightweight ping to payment provider
        try {
            boolean available = paymentClient.ping(); // Timeout: 2 seconds
            return available
                ? Health.up()
                    .withDetail("circuitBreakerState", state.name())
                    .build()
                : Health.down()
                    .withDetail("reason", "Payment provider ping failed")
                    .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("reason", e.getMessage())
                .withDetail("circuitBreakerState", state.name())
                .build();
        }
    }
}

// Separate liveness and readiness
@Component
public class OrderServiceReadinessIndicator implements HealthIndicator {
    private final DataSource dataSource;
    private final PaymentProviderHealthIndicator paymentHealth;

    @Override
    public Health health() {
        // DB must be up for readiness
        if (!isDatabaseHealthy()) {
            return Health.down().withDetail("reason", "Database unavailable").build();
        }

        // Payment provider down = degraded but still accepting orders
        // (orders might queue for later processing)
        Health paymentStatus = paymentHealth.health();
        if (paymentStatus.getStatus().equals(Status.DOWN)) {
            return Health.status("DEGRADED")
                .withDetail("payment", "unavailable — orders will queue")
                .build();
        }

        return Health.up().build();
    }
}
```

---

## Scenario 3: Circuit Breaker with Fallback for Product Catalog

**Setup:** The product catalog service is called by the order service. Implement a circuit breaker with a fallback to a local cache.

```java
@Service
public class ProductCatalogClient {

    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;
    private final Bulkhead bulkhead;
    private final Cache<String, ProductDetails> localCache;

    public ProductDetails getProduct(String productId) {
        return circuitBreaker.executeSupplier(
            () -> bulkhead.executeSupplier(
                () -> fetchFromService(productId)
            ),
            throwable -> getFromCache(productId, throwable)
        );
    }

    private ProductDetails fetchFromService(String productId) {
        return webClient.get()
            .uri("/products/{id}", productId)
            .retrieve()
            .bodyToMono(ProductDetails.class)
            .timeout(Duration.ofSeconds(3))
            .block();
    }

    private ProductDetails getFromCache(String productId, Throwable cause) {
        ProductDetails cached = localCache.getIfPresent(productId);
        if (cached != null) {
            log.warn("Circuit breaker fallback: returning cached product {} (cause: {})",
                productId, cause.getMessage());
            return cached.withStale(true);  // mark as potentially stale
        }
        throw new ServiceUnavailableException(
            "Product catalog unavailable and no cache available for " + productId);
    }

    @Cacheable(value = "productCache", key = "#productId")
    public ProductDetails getProductWithCache(String productId) {
        return getProduct(productId);
    }
}

// Configuration
@Bean
public CircuitBreakerConfig productCatalogCbConfig() {
    return CircuitBreakerConfig.custom()
        .failureRateThreshold(50)
        .waitDurationInOpenState(Duration.ofSeconds(20))
        .slidingWindowSize(20)
        .minimumNumberOfCalls(10)
        .recordExceptions(WebClientResponseException.InternalServerError.class,
            ConnectException.class, TimeoutException.class)
        .ignoreExceptions(WebClientResponseException.NotFound.class)  // 404 = not a failure
        .build();
}
```

---

## Scenario 4: Distributed Tracing with OpenTelemetry

**Setup:** Debug a slow request that spans order-service → inventory-service → warehouse-service. Implement distributed tracing.

```java
// Spring Boot auto-configuration with OpenTelemetry agent
// -javaagent:/app/opentelemetry-javaagent.jar
// -Dotel.service.name=order-service
// -Dotel.exporter.otlp.endpoint=http://jaeger-collector:4317

// Manual span creation for business events
@Service
public class OrderService {

    @Autowired private Tracer tracer;

    @NewSpan("order.place")
    public Order placeOrder(CreateOrderRequest request) {
        Span span = tracer.currentSpan();
        span.tag("customer.id", String.valueOf(request.customerId()));
        span.tag("items.count", String.valueOf(request.items().size()));

        // Reserve inventory — automatic span created by instrumented HTTP client
        ReservationResult reservation = inventoryClient.reserve(request.items());

        if (!reservation.isSuccess()) {
            span.tag("order.status", "INVENTORY_UNAVAILABLE");
            throw new InsufficientInventoryException(reservation.getUnavailableItems());
        }

        Order order = createAndSaveOrder(request);
        span.tag("order.id", String.valueOf(order.getId()));
        span.tag("order.status", "PLACED");
        return order;
    }
}

// Propagate trace context to async tasks
@Async
public void processOrderAsync(Order order, Span parentSpan) {
    // Attach to parent span for async continuations
    try (Scope scope = parentSpan.makeCurrent()) {
        Span asyncSpan = tracer.spanBuilder("order.async-processing")
            .setParent(Context.current())
            .startSpan();
        try {
            doProcessing(order);
        } finally {
            asyncSpan.end();
        }
    }
}
```

**Trace propagation via HTTP headers:**
- W3C Trace Context: `traceparent: 00-traceId-spanId-flags`
- Spring Cloud Sleuth / Micrometer Tracing propagates automatically

---

## Scenario 5: Autoscaling Based on Custom Metrics (KEDA)

**Setup:** Scale the order processor based on Kafka consumer lag. When lag exceeds 1000 messages, scale up to handle backlog.

```yaml
# KEDA ScaledObject
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15    # check every 15 seconds
  cooldownPeriod: 300    # wait 5 minutes before scaling down
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      topic: orders
      bootstrapServers: kafka-headless:9092
      consumerGroup: order-processors
      lagThreshold: "1000"      # 1 replica per 1000 messages of lag
      activationLagThreshold: "10"  # activate (scale from 0) at 10 messages

---
# Application: handle pod scaling gracefully
@Service
public class OrderProcessor {

    @KafkaListener(topics = "orders", groupId = "order-processors",
                   concurrency = "${kafka.consumer.concurrency:3}")
    public void processOrder(ConsumerRecord<String, OrderEvent> record) {
        // Idempotent processing — KEDA may scale down mid-batch
        // Kafka at-least-once: process must be idempotent
        String idempotencyKey = "order:" + record.value().getOrderId();
        if (processedOrders.putIfAbsent(idempotencyKey, true) != null) {
            log.debug("Order {} already processed, skipping", record.value().getOrderId());
            return;
        }

        orderService.process(record.value());
    }
}
```

---

## Scenario 6: Multi-Region Deployment Strategy

**Setup:** Your service needs to be deployed in US-East and EU-West for data residency compliance. Design the deployment strategy.

```
Architecture:
  US-East:                    EU-West:
  ┌─────────────────┐        ┌─────────────────┐
  │ EKS Cluster     │        │ EKS Cluster     │
  │ order-service   │        │ order-service   │
  │ US PostgreSQL   │        │ EU PostgreSQL   │ (GDPR: EU data stays in EU)
  │ US Redis        │        │ EU Redis        │
  └─────────────────┘        └─────────────────┘
         ↑                           ↑
   Route 53 latency-based routing
   → routes EU users to EU-West
   → routes US users to US-East
```

```java
// Data residency: route DB connections based on user's region
@Component
public class RegionAwareDataSourceRouter extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        // Extract region from JWT claim or request header
        String region = RequestContextHolder.getRequestAttributes() instanceof
            ServletRequestAttributes sra
            ? extractRegion(sra.getRequest())
            : "us-east";  // default
        return region;
    }
}

// Configuration
@Bean
public DataSource dataSource() {
    RegionAwareDataSourceRouter router = new RegionAwareDataSourceRouter();
    router.setTargetDataSources(Map.of(
        "us-east", usEastDataSource(),
        "eu-west", euWestDataSource()
    ));
    router.setDefaultTargetDataSource(usEastDataSource());
    return router;
}

// Ensure EU user data never touches US database
@Service
public class OrderService {
    public Order placeOrder(CreateOrderRequest request) {
        validateRegionCompliance(request.getUser(), DataRegionContext.getCurrent());
        return orderRepository.save(buildOrder(request));
        // DataSource router ensures correct regional DB is used
    }
}
```

---

## Scenario 7: Config Management with Spring Cloud Config

**Setup:** Manage externalized configuration for 20 microservices across dev, staging, and production environments.

```
Git Repository (config-repo):
  ├── application.yml          # shared across all services
  ├── application-prod.yml     # shared prod overrides
  ├── order-service.yml        # order-service defaults
  ├── order-service-prod.yml   # order-service production
  └── inventory-service.yml    # inventory-service defaults

Config Server → microservices pull config at startup
```

```yaml
# Config Server
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
  public static void main(String[] args) { SpringApplication.run(...); }
}

# application.yml (Config Server)
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          clone-on-start: true
          default-label: main
          search-paths: '{application}'  # per-service directory
  security:
    user:
      name: config-server
      password: ${CONFIG_SERVER_PASSWORD}

---
# Client service bootstrap.yml
spring:
  application:
    name: order-service
  config:
    import: configserver:http://config-server:8888
  cloud:
    config:
      username: config-server
      password: ${CONFIG_SERVER_PASSWORD}
      fail-fast: true  # fail at startup if config server unreachable
      retry:
        max-attempts: 6
        initial-interval: 1000
```

**Dynamic refresh without restart:**
```java
@RefreshScope  // bean recreated when /actuator/refresh called
@ConfigurationProperties(prefix = "app.feature-flags")
public class FeatureFlags {
    private boolean newCheckoutEnabled;
    private int maxItemsPerOrder;
}

// Trigger refresh via Spring Cloud Bus (Kafka or RabbitMQ)
// One call to any instance refreshes all instances via message bus
```

---

## Scenario 8: Implementing a Sidecar for Log Aggregation

**Setup:** All services log to stdout (12-factor). Collect and ship logs to Elasticsearch using a Fluent Bit sidecar.

```yaml
# Pod with Fluent Bit sidecar
spec:
  containers:
  - name: order-service
    image: order-service:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app

  - name: fluent-bit
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
      readOnly: true
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"

  volumes:
  - name: log-volume
    emptyDir: {}  # shared between containers
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config

---
# Fluent Bit ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [INPUT]
        Name  tail
        Path  /var/log/app/*.log
        Parser  json

    [FILTER]
        Name  record_modifier
        Match *
        Record namespace ${NAMESPACE}
        Record pod_name ${POD_NAME}
        Record service order-service

    [OUTPUT]
        Name  es
        Match *
        Host  elasticsearch.logging.svc.cluster.local
        Port  9200
        Index order-service-logs
```

---

## Scenario 9: Pod Disruption Budget for High Availability

**Setup:** Ensure at least 2 instances of your service are always available during rolling updates and node maintenance.

```yaml
# PodDisruptionBudget — used by Kubernetes drain, rolling update
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
spec:
  minAvailable: 2  # OR use maxUnavailable: 1
  selector:
    matchLabels:
      app: order-service

---
# Anti-affinity — spread pods across nodes to avoid single-node failure
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - order-service
        topologyKey: kubernetes.io/hostname  # one pod per node

    # Prefer different availability zones
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - order-service
          topologyKey: topology.kubernetes.io/zone
```

---

## Scenario 10: Implementing Graceful Shutdown for a Kafka Consumer

**Setup:** A Kafka consumer must commit all in-flight message offsets before shutdown. Implement graceful shutdown.

```java
@Service
public class OrderKafkaConsumer implements DisposableBean {

    private final KafkaListenerEndpointRegistry registry;
    private final AtomicBoolean shuttingDown = new AtomicBoolean(false);
    private final CountDownLatch inFlight = new CountDownLatch(0);
    private final Semaphore concurrencyLimiter = new Semaphore(10);

    @KafkaListener(id = "orderConsumer", topics = "orders")
    public void consume(ConsumerRecord<String, OrderEvent> record,
                        Acknowledgment ack) throws InterruptedException {

        if (shuttingDown.get()) {
            // Don't process new messages during shutdown
            return;  // message will be redelivered to next instance
        }

        concurrencyLimiter.acquire();
        try {
            processOrder(record.value());
            ack.acknowledge();  // manual commit after successful processing
        } finally {
            concurrencyLimiter.release();
        }
    }

    @Override
    public void destroy() throws Exception {
        log.info("Kafka consumer shutting down gracefully...");
        shuttingDown.set(true);

        // Stop the listener (no new polls from Kafka)
        registry.getListenerContainer("orderConsumer").stop();

        // Wait for all in-flight messages to complete (max 30 seconds)
        boolean allDone = concurrencyLimiter.tryAcquire(10, 30, TimeUnit.SECONDS);
        if (!allDone) {
            log.warn("Shutdown timeout: some messages may be reprocessed");
        }

        log.info("Kafka consumer shutdown complete");
    }
}
```

---

## Follow-Up Questions

1. **Kubernetes resource limits:** What happens when a pod exceeds its memory limit?
   It gets OOMKilled (Out of Memory Killed) — kernel kills the container. Pod is restarted. If it OOMKills repeatedly, Kubernetes increases the `CrashLoopBackOff` wait. Set limits to prevent one pod from starving others, but set requests accurately to allow scheduling.

2. **Why not use `latest` tag in production?**
   `latest` tag makes deployments non-deterministic — two deploys may get different images. Use immutable tags (commit SHA or semantic version). `latest` also bypasses `imagePullPolicy: IfNotPresent`.

3. **ECS vs EKS vs Lambda:** How do you choose?
   Lambda: event-driven, short-lived, unpredictable traffic, pay-per-use. ECS: simpler than Kubernetes, AWS-managed control plane. EKS: complex workloads, need Kubernetes features, multi-cloud portability.

4. **Service mesh overhead:** Is Istio worth it for 5 microservices?
   No. Service mesh complexity is justified for 15+ services. For small deployments: use Spring Security for mTLS, Resilience4j for circuit breaking, Zipkin for tracing — no mesh needed.

5. **ConfigMap vs Secret rotation:** How do you handle config rotation without restart?
   Use `@RefreshScope` + Spring Cloud Bus, or mount config as volume (k8s updates volumes automatically within ~1 minute). For secrets, prefer external-secrets-operator with Vault/Secrets Manager and implement `SmartInitializingSingleton` to reload.
