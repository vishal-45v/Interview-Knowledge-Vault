# Networking — Scenario Questions

> 22 real-world scenarios covering network debugging, HTTP client configuration, WebSockets, and production networking issues.

---

## Scenario 1: HTTP Client Timeout Cascade Failure

**Situation:** Your microservice calls an external payment API. The payment service starts responding slowly (30+ seconds). Within minutes, your service's thread pool is exhausted and it stops responding to all requests, including health checks.

**What happened?**

```
Payment API responding slowly (30s)
         │
         ▼
RestTemplate (no timeout configured) blocks for 30s per request
         │
         ▼
Tomcat thread pool (200 threads) — all threads blocked waiting on payment API
         │
         ▼
New requests queue, then reject — "connection refused" or 503
         │
         ▼
Kubernetes readiness probe fails → pod removed from load balancer
         │
         ▼
Other pods suffer same fate → full outage
```

**Fix: Configure timeouts and circuit breaker**

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        // 1. Configure timeouts
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(3_000);   // 3s to connect
        factory.setReadTimeout(5_000);      // 5s to read response

        RestTemplate restTemplate = new RestTemplate(factory);

        // 2. Add error handler
        restTemplate.setErrorHandler(new DefaultResponseErrorHandler());
        return restTemplate;
    }

    // OR with WebClient (reactive)
    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3_000)
            .responseTimeout(Duration.ofSeconds(5))
            .doOnConnected(conn ->
                conn.addHandlerLast(new ReadTimeoutHandler(5))
                    .addHandlerLast(new WriteTimeoutHandler(3)));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

**Production rule:** Every HTTP client call must have an explicit timeout. The default for most libraries is no timeout or an extremely long one.

---

## Scenario 2: SSL Certificate Expiry Causes Production Outage

**Situation:** At 3 AM, all API calls to a partner service start failing with `javax.net.ssl.SSLHandshakeException: PKIX path building failed`. No code changes were deployed. The partner service's SSL certificate expired.

**Debugging:**

```bash
# Check certificate expiry
openssl s_client -connect partner.example.com:443 </dev/null 2>/dev/null | \
    openssl x509 -noout -dates
# notBefore=Jan  1 00:00:00 2023 GMT
# notAfter=Jan  1 00:00:00 2024 GMT   ← EXPIRED

# Verify the chain
openssl s_client -showcerts -connect partner.example.com:443 </dev/null
```

**Short-term fix (if you control the truststore):**

```java
// NEVER in production — disables all cert validation
TrustManager[] trustAllCerts = new TrustManager[]{
    new X509TrustManager() {
        public void checkClientTrusted(X509Certificate[] chain, String authType) {}
        public void checkServerTrusted(X509Certificate[] chain, String authType) {}
        public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
    }
};
// This makes your app vulnerable to MITM attacks
```

**Real fix:**

```java
// If partner sends self-signed or private CA cert, add to truststore
// keytool -importcert -alias partner -file partner.crt -keystore custom-truststore.jks

@Bean
public SSLContext customSSLContext() throws Exception {
    KeyStore trustStore = KeyStore.getInstance("JKS");
    try (InputStream is = new ClassPathResource("custom-truststore.jks").getInputStream()) {
        trustStore.load(is, "changeit".toCharArray());
    }
    TrustManagerFactory tmf = TrustManagerFactory.getInstance(
        TrustManagerFactory.getDefaultAlgorithm());
    tmf.init(trustStore);

    SSLContext sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, tmf.getTrustManagers(), null);
    return sslContext;
}
```

**Prevention:** Monitor certificate expiry with a scheduled job or tool (Prometheus ssl_cert_expiry_seconds metric).

---

## Scenario 3: Implementing WebSocket Chat in Spring Boot

**Situation:** Build a real-time chat feature where users in the same room receive messages instantly.

```java
// 1. Enable WebSocket with STOMP
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // In-memory message broker for /topic (broadcast) and /queue (user-specific)
        registry.enableSimpleBroker("/topic", "/queue");
        // Prefix for client-to-server messages
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOrigins("https://app.example.com")
                .withSockJS();  // SockJS fallback for older browsers
    }
}

// 2. Message controller
@Controller
public class ChatController {

    @MessageMapping("/chat/{roomId}")   // Client sends to /app/chat/{roomId}
    @SendTo("/topic/room/{roomId}")     // Broadcast to all subscribers
    public ChatMessage sendMessage(@DestinationVariable String roomId,
                                   ChatMessage message,
                                   Principal user) {
        message.setSender(user.getName());
        message.setTimestamp(Instant.now());
        return message;
    }

    // Send to specific user
    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    public void sendPrivateMessage(String userId, ChatMessage message) {
        messagingTemplate.convertAndSendToUser(userId, "/queue/private", message);
    }
}

// 3. JavaScript client
// const socket = new SockJS('/ws');
// const stompClient = Stomp.over(socket);
// stompClient.connect({}, () => {
//   stompClient.subscribe('/topic/room/123', (message) => {
//     displayMessage(JSON.parse(message.body));
//   });
//   stompClient.send('/app/chat/123', {}, JSON.stringify({text: 'Hello!'}));
// });
```

**Scaling WebSockets across multiple instances:** Replace in-memory broker with Redis Pub/Sub or a dedicated broker like RabbitMQ/ActiveMQ:

```yaml
spring:
  messaging:
    stomp:
      relay:
        host: rabbitmq.internal
        port: 61613
```

---

## Scenario 4: REST Client with Retry and Circuit Breaker

**Situation:** Your service calls an inventory API that occasionally returns 503 errors (1-2% of requests). Implement resilient retry with circuit breaker.

```java
@Service
public class InventoryClient {

    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;

    public InventoryClient(WebClient.Builder webClientBuilder,
                           CircuitBreakerRegistry registry) {
        this.webClient = webClientBuilder
            .baseUrl("https://inventory.internal")
            .build();
        this.circuitBreaker = registry.circuitBreaker("inventory");
        this.retry = Retry.of("inventory", RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryOnException(e -> e instanceof WebClientResponseException.ServiceUnavailable)
            .build());
    }

    public Mono<InventoryStatus> getStatus(String productId) {
        return webClient.get()
            .uri("/products/{id}/inventory", productId)
            .retrieve()
            .bodyToMono(InventoryStatus.class)
            // Retry on 503, with exponential backoff
            .retryWhen(reactor.util.retry.Retry
                .backoff(3, Duration.ofMillis(200))
                .filter(e -> e instanceof WebClientResponseException.ServiceUnavailable)
                .maxBackoff(Duration.ofSeconds(2)))
            // Circuit breaker
            .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
            // Fallback when circuit open
            .onErrorReturn(CallNotPermittedException.class, InventoryStatus.unknown());
    }
}
```

```yaml
# application.yml — Circuit breaker config
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50        # Open after 50% failures
        wait-duration-in-open-state: 10s  # Wait before trying again
        permitted-number-of-calls-in-half-open-state: 3
```

---

## Scenario 5: Debugging "Connection Reset by Peer"

**Situation:** Clients intermittently receive `java.net.SocketException: Connection reset` errors from your API. The errors appear randomly, not correlated with a specific endpoint.

**Common causes and diagnostics:**

```bash
# 1. Check for connection timeouts at load balancer
# AWS ALB default idle timeout: 60 seconds
# Nginx keepalive_timeout: 75 seconds (default)

# 2. Check your application's keepalive settings
# Tomcat default keepAliveTimeout: 60000ms

# 3. Check if client is using connection pool with stale connections
# Check /actuator/metrics/http.client.requests for error rates

# 4. tcpdump to capture RST packets
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' -w reset.pcap
```

**Most common cause:** Load balancer idle timeout < connection pool keepalive timeout.

```
Client pool keeps connection alive 90s
ALB closes connection after 60s idle
Client sends request on "alive" connection
RST received → "Connection reset by peer"
```

**Fix:**

```yaml
# Tomcat: set keepalive shorter than ALB timeout
server:
  tomcat:
    keep-alive-timeout: 50000    # 50s — less than ALB's 60s
    max-keep-alive-requests: 100

# Or for HTTP client pool: set shorter idle eviction
# OkHttpClient:
#   .connectionPool(new ConnectionPool(5, 50, TimeUnit.SECONDS))  # evict after 50s
```

---

## Scenario 6: Implementing Pagination with REST API

**Situation:** Your `/api/products` endpoint returns all 500K products in one call. Implement pagination.

```java
// Cursor-based pagination (preferred for large datasets)
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public ResponseEntity<PageResponse<ProductDto>> getProducts(
            @RequestParam(required = false) String cursor,  // last seen ID
            @RequestParam(defaultValue = "20") @Max(100) int limit) {

        List<ProductDto> products = productService.getProductsAfterCursor(cursor, limit + 1);

        boolean hasMore = products.size() > limit;
        if (hasMore) {
            products = products.subList(0, limit);
        }

        String nextCursor = hasMore
            ? products.get(products.size() - 1).getId()
            : null;

        return ResponseEntity.ok(new PageResponse<>(products, nextCursor, hasMore));
    }
}

// Service layer
@Transactional(readOnly = true)
public List<ProductDto> getProductsAfterCursor(String cursor, int limit) {
    if (cursor == null) {
        return productRepository.findTopNByOrderByIdAsc(limit, PageRequest.of(0, limit));
    }
    return productRepository.findByIdGreaterThanOrderByIdAsc(cursor, PageRequest.of(0, limit));
}
```

```java
// Response wrapper
public record PageResponse<T>(
    List<T> data,
    String nextCursor,
    boolean hasMore
) {}
```

**Response example:**
```json
{
  "data": [...20 products...],
  "nextCursor": "prod_abc123",
  "hasMore": true
}
```

Client fetches next page: `GET /api/products?cursor=prod_abc123&limit=20`

**Why not offset pagination?** `OFFSET 10000` makes the database scan and discard 10,000 rows. Cursor pagination always starts from an indexed position.

---

## Scenario 7: DNS Round-Robin vs Load Balancer

**Situation:** Your team is debating whether to use DNS round-robin (multiple A records) or a load balancer for distributing traffic across 3 application servers.

**DNS round-robin:**
```
api.example.com  →  10.0.0.1
api.example.com  →  10.0.0.2
api.example.com  →  10.0.0.3
```

**Problems with DNS round-robin:**
1. **No health checks** — if 10.0.0.2 goes down, clients still get that IP until TTL expires
2. **Sticky clients** — OS and browser cache DNS results; slow propagation of changes
3. **Uneven distribution** — clients with large connection pools may saturate one server
4. **No SSL termination** — each server needs its own certificate
5. **No advanced routing** — can't route `/api/users/*` to one cluster

**Load balancer advantages:**
- Health checks (remove unhealthy instances automatically)
- SSL termination (one cert at LB, HTTP inside VPC)
- Advanced routing, A/B testing
- Connection draining (graceful shutdown)
- Sticky sessions if needed
- DDoS protection, WAF integration

**When DNS round-robin is acceptable:** non-critical internal services, simple static content CDN origins, global load balancing between data centers (GeoDNS).

---

## Scenario 8: HTTP/2 with Spring Boot

**Situation:** You want to enable HTTP/2 to improve performance for your API, especially for clients making many parallel requests.

```yaml
# application.yml
server:
  http2:
    enabled: true
  ssl:
    enabled: true           # HTTP/2 requires HTTPS (in practice)
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEY_STORE_PASSWORD}
    key-store-type: PKCS12
```

**HTTP/2 benefits for APIs:**
- **Multiplexing**: browser makes 6 parallel requests over HTTP/1.1; HTTP/2 handles all over 1 connection with no queuing
- **Header compression (HPACK)**: reduces overhead for repeated headers (JWT token, content-type)
- **Server push**: proactively send related resources (rarely used in APIs)

**Testing HTTP/2:**
```bash
curl -v --http2 https://localhost:8443/api/health
# Look for: < HTTP/2 200
```

**gRPC uses HTTP/2** for its binary framing and multiplexing. If your services communicate via gRPC, HTTP/2 is already in use.

---

## Scenario 9: Implementing Rate Limiting at API Gateway

**Situation:** Your public API is being abused — some clients send 10,000 requests per minute, causing slowdowns for other users.

```java
// Using Bucket4j for in-memory rate limiting (single instance)
@Component
public class RateLimitingFilter implements Filter {

    // Per-client rate limit buckets
    private final Cache<String, Bucket> buckets = Caffeine.newBuilder()
        .expireAfterAccess(1, TimeUnit.HOURS)
        .build();

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String clientKey = getClientKey(request);  // API key or IP
        Bucket bucket = buckets.get(clientKey, k -> createBucket());

        if (bucket.tryConsume(1)) {
            chain.doFilter(req, res);
        } else {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.setHeader("X-RateLimit-Limit", "1000");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
        }
    }

    private Bucket createBucket() {
        Bandwidth limit = Bandwidth.classic(1000, Refill.greedy(1000, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }

    private String getClientKey(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? "key:" + apiKey : "ip:" + request.getRemoteAddr();
    }
}
```

**Distributed rate limiting with Redis (multi-instance):**
```java
// Use Redis sliding window counter
String key = "rate:" + clientKey + ":" + (System.currentTimeMillis() / 60_000);
Long count = redisTemplate.opsForValue().increment(key);
redisTemplate.expire(key, 2, TimeUnit.MINUTES);  // 2x window for boundary handling

if (count > 1000) {
    // Rate limit exceeded
}
```

---

## Scenario 10: Connection Pool Exhaustion in Production

**Situation:** Your service shows `HikariPool-1 - Connection is not available, request timed out after 30000ms` errors under load.

**Diagnosis:**

```bash
# 1. Check pool metrics via Actuator
curl http://localhost:8080/actuator/metrics/hikaricp.connections
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active

# 2. Check what's holding connections
# In PostgreSQL:
SELECT pid, state, wait_event, query_start, query
FROM pg_stat_activity
WHERE datname = 'mydb'
ORDER BY query_start;

# Look for long-running queries or connections in "idle in transaction" state
```

**Common causes:**
1. **Pool size too small** for the load
2. **Long-running transactions** holding connections
3. **Missing transaction boundaries** — connection held for entire request including external calls
4. **Slow queries** backing up the pool
5. **Connection leak** — connection obtained but not returned (exception without finally/try-with-resources)

```java
// Connection leak — WRONG
public void processOrder(Long id) {
    Connection conn = dataSource.getConnection();  // Gets connection from pool
    // ... if an exception occurs here, connection is never returned to pool
    conn.close();  // Never reached on exception!
}

// CORRECT — use try-with-resources
public void processOrder(Long id) {
    try (Connection conn = dataSource.getConnection()) {
        // Connection automatically returned to pool
    }
}

// With Spring: use @Transactional — Spring manages connection lifecycle
@Transactional
public void processOrder(Long id) {
    // Connection returned to pool after method exits (commit or rollback)
}
```

**Fix for undersized pool:**
```yaml
spring.datasource.hikari:
  maximum-pool-size: 20    # Increase (check DB max_connections first!)
  connection-timeout: 5000 # Fail fast (5s) instead of waiting 30s
```

---

## Scenario 11: Debugging "Too Many Open Files"

**Situation:** Your Java service crashes with `java.net.SocketException: Too many open files` after running for several hours.

**Cause:** File descriptor leak. Each socket (HTTP connection, DB connection, file handle) uses an OS file descriptor. Linux default per-process limit: 1024.

```bash
# Check current limits
ulimit -n              # Current soft limit
cat /proc/<PID>/limits # Per-process limits

# Count open FDs for process
ls -l /proc/<PID>/fd | wc -l

# See what FDs are open
lsof -p <PID> | head -50
lsof -p <PID> | grep "ESTABLISHED" | wc -l   # Open TCP connections
```

**Common causes in Java:**
1. Connection pool connections not being returned (leak)
2. HTTP client without connection pooling (creating new connection per request)
3. Files opened but not closed (missing close() or try-with-resources)

**Fix:**
```bash
# Increase OS limits (in /etc/security/limits.conf or systemd service)
myuser soft nofile 65536
myuser hard nofile 65536

# In Docker/Kubernetes pod spec:
# securityContext.ulimits is not standard; use initContainer or sysctl
```

**Better fix: find the leak** with heap dump analysis (look for SocketImpl instances) or add metrics tracking connection pool size.

---

## Scenario 12: Implementing Async HTTP with CompletableFuture

**Situation:** Your order API needs to call 3 services in parallel (inventory, pricing, user) and combine results. Sequential calls take 300ms × 3 = 900ms. Parallel: ~300ms.

```java
@Service
public class OrderService {

    @Autowired private InventoryClient inventoryClient;
    @Autowired private PricingClient pricingClient;
    @Autowired private UserClient userClient;

    public OrderDetails getOrderDetails(String orderId) throws Exception {
        // Fire all 3 requests in parallel
        CompletableFuture<InventoryStatus> inventoryFuture =
            CompletableFuture.supplyAsync(
                () -> inventoryClient.getStatus(orderId), executor);

        CompletableFuture<PricingInfo> pricingFuture =
            CompletableFuture.supplyAsync(
                () -> pricingClient.getPrice(orderId), executor);

        CompletableFuture<UserInfo> userFuture =
            CompletableFuture.supplyAsync(
                () -> userClient.getUser(orderId), executor);

        // Wait for all to complete
        CompletableFuture.allOf(inventoryFuture, pricingFuture, userFuture).join();

        return new OrderDetails(
            inventoryFuture.get(),
            pricingFuture.get(),
            userFuture.get()
        );
    }
}

// With WebClient (reactive approach — better for high throughput)
public Mono<OrderDetails> getOrderDetailsReactive(String orderId) {
    Mono<InventoryStatus> inventory = inventoryClient.getStatus(orderId);
    Mono<PricingInfo> pricing = pricingClient.getPrice(orderId);
    Mono<UserInfo> user = userClient.getUser(orderId);

    return Mono.zip(inventory, pricing, user)
               .map(tuple -> new OrderDetails(tuple.getT1(), tuple.getT2(), tuple.getT3()));
}
```

---

## Scenario 13: Health Check Endpoint Design

**Situation:** Design health check endpoints for a Spring Boot microservice deployed in Kubernetes.

```java
// Spring Boot Actuator provides /actuator/health out of the box
// Customize the health aggregation
@Configuration
public class HealthConfig {

    // Custom health indicator for a dependency
    @Component
    public static class ExternalApiHealthIndicator implements HealthIndicator {

        @Autowired private ExternalApiClient client;

        @Override
        public Health health() {
            try {
                boolean ok = client.ping();
                if (ok) {
                    return Health.up()
                                 .withDetail("response_time_ms", client.getLastLatency())
                                 .build();
                }
                return Health.down().withDetail("reason", "Ping returned false").build();
            } catch (Exception e) {
                return Health.down(e).build();
            }
        }
    }
}
```

```yaml
# application.yml
management:
  endpoint:
    health:
      show-details: when-authorized   # Don't expose internals publicly
      show-components: when-authorized
      probes:
        enabled: true   # Enable /actuator/health/liveness and /readiness
  health:
    db:
      enabled: true
    redis:
      enabled: true
    diskspace:
      enabled: true
```

```yaml
# Kubernetes probe configuration
livenessProbe:
  httpGet:
    path: /actuator/health/liveness    # Is the JVM alive?
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness   # Can we serve traffic?
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

Liveness vs readiness:
- **Liveness**: "Is the app alive or deadlocked?" Failure = restart pod
- **Readiness**: "Is the app ready for traffic?" Failure = remove from load balancer, don't restart

---

## Scenario 14: Implementing Request Correlation IDs

**Situation:** In a microservices environment, you need to trace a request across 5 services. Each service logs differently, making it impossible to correlate logs.

```java
// 1. Filter to generate/propagate correlation ID
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    public static final String CORRELATION_ID_MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // Add to MDC so all log statements include it automatically
        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
        // Add to response so clients can reference it
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY);  // MUST clean up to prevent leaks
        }
    }
}

// 2. Propagate to downstream HTTP calls
@Bean
public RestTemplate restTemplate() {
    RestTemplate rt = new RestTemplate();
    rt.getInterceptors().add((request, body, execution) -> {
        String correlationId = MDC.get(CorrelationIdFilter.CORRELATION_ID_MDC_KEY);
        if (correlationId != null) {
            request.getHeaders().set(
                CorrelationIdFilter.CORRELATION_ID_HEADER, correlationId);
        }
        return execution.execute(request, body);
    });
    return rt;
}

// 3. Logback pattern to include correlationId
// <pattern>%d{ISO8601} [%thread] [%X{correlationId}] %-5level %logger - %msg%n</pattern>
```

---

## Scenario 15: HTTP Caching with ETags

**Situation:** Your `/api/products/{id}` endpoint is called 1000 times per minute but the data rarely changes. Implement HTTP caching to reduce server load.

```java
@GetMapping("/api/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id,
                                           WebRequest request) {
    Product product = productService.findById(id);

    // Generate ETag from product content hash
    String etag = "\"" + Integer.toHexString(product.hashCode()) + "\"";

    // 304 Not Modified if client has fresh copy
    if (request.checkNotModified(etag, product.getUpdatedAt().toEpochMilli())) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();
    }

    return ResponseEntity.ok()
        .eTag(etag)
        .lastModified(product.getUpdatedAt())
        .cacheControl(CacheControl.maxAge(30, TimeUnit.MINUTES)
                                  .cachePublic())
        .body(product);
}
```

**How it works:**
1. First request: server returns 200 with `ETag: "abc123"` and `Last-Modified: Thu, 01 Jan 2024`
2. Client caches the response
3. Second request: client sends `If-None-Match: "abc123"` and `If-Modified-Since: Thu, 01 Jan 2024`
4. If unchanged: server returns 304 (no body) — saves bandwidth
5. If changed: server returns 200 with new body and new ETag

**Cache-Control directives:**
- `max-age=300` — cache for 5 minutes
- `no-cache` — cache but must revalidate with server before use
- `no-store` — never cache (sensitive data)
- `private` — only cache in browser (not shared proxies)
- `public` — can cache in shared CDN/proxy

---

## Scenario 16: WebSocket at Scale — Horizontal Scaling Problem

**Situation:** Your chat app works fine on one instance but fails when you scale to 3 instances. Users on different instances can't see each other's messages.

**Problem:** WebSocket connections are stateful — each user is connected to a specific instance. When User A (connected to instance 1) sends a message to User B (connected to instance 2), instance 1 doesn't know how to reach instance 2's WebSocket connection.

**Solution: Redis Pub/Sub as message bus**

```java
@Configuration
public class WebSocketScalingConfig {

    @Bean
    public MessageListenerAdapter messageListener(ChatBroadcastService service) {
        return new MessageListenerAdapter(service, "onMessage");
    }

    @Bean
    public RedisMessageListenerContainer redisContainer(
            RedisConnectionFactory factory,
            MessageListenerAdapter listener) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listener, new PatternTopic("chat:*"));
        return container;
    }
}

@Service
public class ChatBroadcastService {

    @Autowired private SimpMessagingTemplate wsTemplate;
    @Autowired private StringRedisTemplate redisTemplate;

    // When message arrives via Redis Pub/Sub (published by any instance)
    public void onMessage(String message, String channel) {
        String roomId = channel.replace("chat:", "");
        wsTemplate.convertAndSend("/topic/room/" + roomId, message);
    }

    // When user sends message, publish to Redis so ALL instances broadcast it
    public void broadcastMessage(String roomId, ChatMessage message) {
        String payload = objectMapper.writeValueAsString(message);
        redisTemplate.convertAndSend("chat:" + roomId, payload);
    }
}
```

**Alternative: Use a dedicated STOMP broker (RabbitMQ/ActiveMQ)** — Spring WebSocket has native support, simpler for STOMP-based apps.

---

## Scenario 17: gRPC vs REST — Choosing the Right Protocol

**Situation:** You're designing internal service-to-service communication between microservices. The team is debating gRPC vs REST+JSON.

**REST+JSON:**
- Human-readable, easy to debug with curl/Postman
- Firewall-friendly (port 443/80)
- Wide ecosystem support
- Versioning can be tricky
- Slightly larger payload than binary

**gRPC:**
- Protocol Buffers — binary, 3-10x smaller payload, faster serialization
- Strongly typed contracts (`.proto` files as source of truth)
- Code generation for client/server stubs
- HTTP/2 multiplexing built-in
- Bidirectional streaming support
- Harder to debug (need gRPCurl or Postman gRPC)
- Less familiar to some developers

**Choose gRPC when:**
- High-throughput internal services (thousands of calls/second between services)
- Streaming use cases (server streaming, client streaming, bidirectional)
- Polyglot microservices (proto is language-agnostic)
- Strong contract enforcement needed

**Choose REST when:**
- Public APIs (browsers, external clients)
- Event-driven (webhooks)
- Simple CRUD with standard HTTP semantics
- Team more comfortable with HTTP debugging tools

```java
// gRPC Spring Boot example
// Proto definition:
// service OrderService {
//   rpc CreateOrder(CreateOrderRequest) returns (OrderResponse);
//   rpc StreamOrderUpdates(OrderId) returns (stream OrderUpdate);
// }

@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    @Override
    public void createOrder(CreateOrderRequest request,
                            StreamObserver<OrderResponse> responseObserver) {
        Order order = orderService.create(request);
        responseObserver.onNext(OrderResponse.newBuilder()
            .setOrderId(order.getId())
            .setStatus("CREATED")
            .build());
        responseObserver.onCompleted();
    }
}
```

---

## Scenario 18: Handling Slow Network in Spring Boot — Graceful Shutdown

**Situation:** During deployment, Kubernetes sends SIGTERM to your pod. Active HTTP connections (some with slow clients on 4G) are still in progress. How do you ensure no requests are dropped?

```yaml
# application.yml
server:
  shutdown: graceful        # Wait for active requests to complete

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # Max wait before force shutdown
```

```java
// Custom graceful shutdown behavior
@Component
public class GracefulShutdownHandler {

    @EventListener(ContextClosedEvent.class)
    public void onApplicationClose() {
        log.info("Application shutting down — draining connections");
        // Stop accepting new work (mark as not-ready)
        // Kubernetes readiness probe will stop sending traffic
    }
}
```

**Shutdown sequence with Kubernetes:**
```
1. Kubernetes sends SIGTERM to pod
2. Pod's readiness probe starts failing (or endpoint returns 503)
   → K8s removes pod from Service endpoints (stops new traffic)
3. Spring Boot starts graceful shutdown period (30s)
4. Active requests complete (up to 30s)
5. Application context closes, beans destroyed
6. JVM exits
7. If JVM still alive after terminationGracePeriodSeconds (60s), K8s sends SIGKILL
```

```yaml
# Kubernetes deployment
spec:
  terminationGracePeriodSeconds: 60   # Must be > spring.lifecycle.timeout-per-shutdown-phase
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]  # Give load balancer time to deregister
```

---

## Scenario 19: Debugging Network Latency Between Services

**Situation:** Service A calling Service B shows P99 latency of 800ms. Service B's own P99 is 50ms. Where are the extra 750ms coming from?

**Systematic diagnosis:**

```bash
# 1. Check DNS resolution time
time nslookup service-b.namespace.svc.cluster.local
# If >10ms, there's a DNS issue (ndots config, DNS cache)

# 2. Check TCP connection time
curl -w "\ntime_namelookup: %{time_namelookup}\ntime_connect: %{time_connect}\ntime_appconnect: %{time_appconnect}\ntime_ttfb: %{time_starttransfer}\ntime_total: %{time_total}\n" \
     -o /dev/null -s http://service-b:8080/api/test

# 3. Check if connection pooling is working
# If time_connect is large every time: no connection reuse (no keep-alive)
# If time_namelookup is large: DNS caching issue

# 4. Check thread pool — are requests queuing?
curl http://service-a:8080/actuator/metrics/executor.queue.remaining
```

**Common causes of added latency:**
- DNS lookup per request (not caching, ndots=5 causing 5 DNS queries)
- No connection pooling (new TCP+TLS handshake per request)
- Thread pool queueing (insufficient parallelism)
- Garbage collection pauses (check GC logs)
- Serialization overhead (large Jackson objects)

```yaml
# Fix DNS caching in JVM (default cache is 30s, but negative cache is infinite in some JVMs)
# In JVM options:
# -Dsun.net.inetaddr.ttl=30
# -Dsun.net.inetaddr.negative.ttl=10

# Kubernetes: configure ndots to reduce DNS queries
# In pod spec:
dnsConfig:
  options:
    - name: ndots
      value: "2"   # Default is 5 — causes many unnecessary lookups
```

---

## Scenario 20: Implementing API Gateway Pattern

**Situation:** You have 5 microservices. Clients need to call all 5 for the main page. Implement an API gateway to aggregate and reduce client round trips.

```java
// Simple aggregation in Spring Boot API Gateway
@RestController
@RequestMapping("/api/dashboard")
public class DashboardAggregationController {

    @GetMapping("/{userId}")
    public DashboardResponse getDashboard(@PathVariable String userId) {
        // Fan out to 3 services in parallel
        CompletableFuture<UserProfile> profile =
            CompletableFuture.supplyAsync(() -> userService.getProfile(userId));
        CompletableFuture<List<Order>> orders =
            CompletableFuture.supplyAsync(() -> orderService.getRecentOrders(userId, 5));
        CompletableFuture<List<Notification>> notifications =
            CompletableFuture.supplyAsync(() -> notificationService.getUnread(userId));

        CompletableFuture.allOf(profile, orders, notifications).join();

        return new DashboardResponse(
            profile.join(),
            orders.join(),
            notifications.join()
        );
    }
}
```

**With Spring Cloud Gateway (proxy-based):**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service    # Load balanced via service discovery
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Header, true
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

---

## Scenario 21: TCP vs HTTP for Internal Service Communication

**Situation:** A team member suggests using raw TCP sockets instead of HTTP for internal microservice calls to "reduce overhead." Evaluate this proposal.

**HTTP overhead analysis:**

For a typical REST request:
```
HTTP/1.1 headers: ~500-2000 bytes
JSON payload: varies
Total overhead vs binary TCP: ~2-5%
```

**Verdict: The overhead argument is usually wrong in practice because:**

1. **Network latency dominates**: a single round trip in the same datacenter is ~0.5ms. Header overhead at Gbps speeds adds microseconds.

2. **Connection pooling eliminates handshake cost**: HTTP keep-alive or HTTP/2 reuses connections. The 3-way handshake is a one-time cost.

3. **What you lose with raw TCP:**
   - No standard request/response correlation
   - No standard error codes
   - Must implement framing (message boundaries)
   - No load balancer support without custom protocol
   - No distributed tracing support
   - No standard health check semantics
   - Every team member must learn your custom protocol

4. **Better alternatives to raw HTTP/1.1 when performance matters:**
   - HTTP/2 — multiplexing, header compression
   - gRPC — HTTP/2 + Protocol Buffers (30-40% size reduction, faster serialization)
   - QUIC — for unstable networks

**Recommendation:** Use gRPC for high-throughput internal communication. It gives binary efficiency while maintaining standard tooling and observability.

---

## Scenario 22: Load Balancer Health Check Configuration

**Situation:** After a deployment, the load balancer routes traffic to a pod before Spring Boot has finished starting up, causing errors. Fix the health check configuration.

**Problem:** Default health check is too aggressive — pod gets traffic before `/actuator/health` is ready.

```yaml
# Kubernetes probe settings
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20    # Wait 20s before first check (JVM startup time)
  periodSeconds: 5           # Check every 5s
  failureThreshold: 3        # 3 consecutive failures before marking not-ready
  successThreshold: 1        # 1 success to mark ready

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60    # More lenient — don't restart prematurely
  periodSeconds: 10
  failureThreshold: 3
```

```yaml
# Spring Boot startup probe (K8s 1.16+) — replaces long initialDelaySeconds
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30       # Allow 30 × 10s = 5 minutes for startup
  periodSeconds: 10
```

**Spring Boot readiness state can be programmatically controlled:**

```java
@Component
public class ApplicationReadinessListener {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        // Spring Boot automatically sets readiness to ACCEPTING_TRAFFIC
        // But you can manually control it:
        // eventPublisher.publishEvent(new AvailabilityChangeEvent<>(
        //     this, ReadinessState.REFUSING_TRAFFIC));
    }

    // During graceful shutdown, set refusing traffic early
    @EventListener(ContextClosedEvent.class)
    public void onShutdown() {
        eventPublisher.publishEvent(new AvailabilityChangeEvent<>(
            this, ReadinessState.REFUSING_TRAFFIC));
    }
}
```
