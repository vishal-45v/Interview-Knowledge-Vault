# Logging, Monitoring & Distributed Tracing

---

## Structured Logging with Logback and JSON

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="prod">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"service":"myapp","env":"prod"}</customFields>
      </encoder>
    </appender>
  </springProfile>
  
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

This produces JSON logs like:
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order processed",
  "service": "myapp",
  "env": "prod",
  "correlationId": "abc-123",
  "userId": "42",
  "orderId": "789"
}
```

---

## MDC for Correlation IDs

```java
@Component
public class CorrelationFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) 
            throws IOException, ServletException {
        String correlationId = Optional.ofNullable(((HttpServletRequest)req)
            .getHeader("X-Correlation-ID")).orElse(UUID.randomUUID().toString());
        
        MDC.put("correlationId", correlationId);
        ((HttpServletResponse)res).setHeader("X-Correlation-ID", correlationId);
        
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();  // Critical: prevent memory leaks in thread pools
        }
    }
}
```

---

## Spring Boot Actuator + Micrometer

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true  # Track p50, p95, p99 latencies
      percentiles:
        http.server.requests: 0.5,0.9,0.95,0.99
```

```java
// Custom business metrics
@Service
public class OrderService {

    private final Counter ordersCreated;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry meterRegistry) {
        this.ordersCreated = meterRegistry.counter("orders.created",
            "status", "success");
        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .register(meterRegistry);
    }

    public Order processOrder(OrderRequest request) {
        return orderProcessingTime.record(() -> {
            Order order = doProcessOrder(request);
            ordersCreated.increment();
            return order;
        });
    }
}
```

---

## Distributed Tracing with Spring Boot Actuator + OpenTelemetry

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # Sample 10% of requests in production

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
```

Spring Boot automatically:
- Generates trace IDs for each request
- Propagates trace context to downstream services (via HTTP headers)
- Includes trace IDs in log lines

---

## ELK Stack for Log Aggregation

```
Application (Docker)
    │ JSON logs to stdout
    ▼
Filebeat (log shipper)
    │ ships to
    ▼
Logstash (parse, filter, enrich)
    │ sends to
    ▼
Elasticsearch (store, index)
    │
    ▼
Kibana (search, visualize, dashboard)
```

Query logs in Kibana:
- All errors in last hour: `level:ERROR AND @timestamp:[now-1h TO now]`
- Specific correlation: `correlationId:"abc-123-xyz"`
- Slow requests: `duration_ms:>1000`
