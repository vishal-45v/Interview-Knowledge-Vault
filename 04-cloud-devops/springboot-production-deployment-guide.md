# Spring Boot Production Deployment Guide

---

## Production Readiness Checklist

### Application Configuration
- [ ] All secrets from environment variables or secrets manager (NOT hardcoded)
- [ ] Logging is structured JSON format
- [ ] Actuator health, metrics, prometheus endpoints exposed
- [ ] Graceful shutdown configured (server.shutdown=graceful)
- [ ] OSIV disabled (spring.jpa.open-in-view=false)
- [ ] Connection pool sized appropriately (hikari.maximum-pool-size)
- [ ] Timeouts configured (DB, HTTP client, circuit breaker)
- [ ] spring.profiles.active set per environment

### Docker/Container
- [ ] Multi-stage build used (separate build and runtime)
- [ ] Running as non-root user
- [ ] Health check defined in Dockerfile or K8s probe
- [ ] JVM container flags set (-XX:+UseContainerSupport, -XX:MaxRAMPercentage)
- [ ] Resource limits set (memory, CPU)
- [ ] Specific image tag (not :latest)

### Infrastructure
- [ ] Multiple replicas (min 2 for HA)
- [ ] Pod Disruption Budget (PDB) configured
- [ ] Horizontal Pod Autoscaler (HPA) configured
- [ ] Liveness and readiness probes configured
- [ ] Rolling update strategy configured
- [ ] Rollback mechanism in place

### Observability
- [ ] Metrics exported to Prometheus
- [ ] Logs shipped to centralized logging (ELK/CloudWatch)
- [ ] Distributed tracing enabled
- [ ] Dashboards created for key metrics
- [ ] Alerts configured for: high error rate, high latency, OOM, OOMKilled

### Security
- [ ] Container image vulnerability scanning in CI
- [ ] Secrets not in container image or environment variables in plain text
- [ ] HTTPS only (HTTP redirected)
- [ ] CORS configured correctly
- [ ] Rate limiting enabled
- [ ] Input validation on all endpoints

---

## JVM Tuning for Production

```bash
# General production JVM flags
java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/heapdump.hprof \
  -Xlog:gc*:file=/var/log/gc.log:time,pid:filecount=5,filesize=20m \
  -Djava.security.egd=file:/dev/./urandom \
  -jar app.jar
```

---

## Zero-Downtime Deployment Process

```
1. New pods start up → startupProbe waits for readiness
2. New pods pass readiness probe → added to load balancer
3. Old pods receive no new traffic → drain existing requests
4. terminationGracePeriodSeconds ensures in-flight requests complete
5. Old pods shut down gracefully (Spring shutdown hooks run)
6. New pods serving 100% of traffic
```

For Kubernetes, set:
- `server.shutdown=graceful` in Spring Boot
- `terminationGracePeriodSeconds > spring.lifecycle.timeout-per-shutdown-phase`
- `maxUnavailable: 0` in rolling update strategy
