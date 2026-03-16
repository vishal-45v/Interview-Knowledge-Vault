# Prometheus & Grafana for Spring Boot

---

## Prometheus Scraping Spring Boot

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-myapp'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['myapp:8080']
    # Or use kubernetes service discovery:
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: myapp
```

---

## Key Spring Boot Metrics

```
# HTTP requests
http_server_requests_seconds_count{uri="/api/orders",method="POST",status="200"}
http_server_requests_seconds_sum
http_server_requests_seconds_bucket

# JVM
jvm_memory_used_bytes{area="heap"}
jvm_gc_pause_seconds_max
jvm_threads_live_threads

# Connection pool (HikariCP)
hikaricp_connections_active
hikaricp_connections_timeout_total

# Cache (if @Cacheable with Micrometer)
cache_gets_total{name="products",result="hit"}
cache_gets_total{name="products",result="miss"}
```

---

## Grafana Dashboard Queries (PromQL)

```
# Request rate (per second, 5 min window)
rate(http_server_requests_seconds_count{job="myapp"}[5m])

# 99th percentile latency
histogram_quantile(0.99,
  rate(http_server_requests_seconds_bucket{job="myapp"}[5m]))

# Error rate
rate(http_server_requests_seconds_count{job="myapp",status=~"5.."}[5m])
/ rate(http_server_requests_seconds_count{job="myapp"}[5m])

# JVM heap usage %
(jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) * 100

# HikariCP pool usage
hikaricp_connections_active / hikaricp_connections_max
```

---

## Alerting Rules

```yaml
# alert.rules.yml
groups:
- name: spring-boot
  rules:
  - alert: HighErrorRate
    expr: |
      rate(http_server_requests_seconds_count{status=~"5.."}[5m])
      / rate(http_server_requests_seconds_count[5m]) > 0.05
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate: {{ $value | humanizePercentage }}"

  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "p95 latency > 2s: {{ $value }}s"

  - alert: JVMHeapNearFull
    expr: |
      (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "JVM heap > 90%"
```
