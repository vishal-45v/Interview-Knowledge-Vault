# JVM Memory in Docker Containers

---

## JVM Memory Areas to Consider in Containers

```
Container memory limit: 1GB
JVM process memory usage:
  Heap (controlled by Xmx):    ~600MB (MaxRAMPercentage=75 of ~800MB usable)
  Metaspace:                   ~100MB (class definitions, grows as classes load)
  Code cache (JIT compiled):   ~50MB
  Thread stacks:               ~1MB per thread (250 threads = 250MB)
  Off-heap (DirectByteBuffer): varies with NIO usage
  Native libraries:            ~20MB

DANGER: If total JVM usage > container limit → OOMKilled by kernel
Container is killed, not just OutOfMemoryError!
```

---

## Recommended JVM Flags for Containers

```dockerfile
ENTRYPOINT ["java", \
    # Container awareness
    "-XX:+UseContainerSupport", \          # Respect cgroups limits (default Java 11+)
    "-XX:MaxRAMPercentage=75.0", \          # Heap = 75% of available container RAM
    "-XX:InitialRAMPercentage=50.0", \      # Start heap at 50%
    \
    # GC tuning
    "-XX:+UseG1GC", \                       # G1GC (default for most cases)
    "-XX:MaxGCPauseMillis=200", \           # Target max pause time
    \
    # Crash handling
    "-XX:+ExitOnOutOfMemoryError", \        # Kill process on OOM (let k8s restart)
    "-XX:+HeapDumpOnOutOfMemoryError", \    # Capture heap dump
    "-XX:HeapDumpPath=/tmp/heapdump.hprof", \
    \
    # Security
    "-Djava.security.egd=file:/dev/./urandom", \  # Faster random for SSL
    \
    "-jar", "app.jar"]
```

---

## Memory Sizing Guide

| Container RAM | MaxRAMPercentage=75 | Heap size | Suitable for |
|---------------|---------------------|-----------|--------------|
| 512MB | 75% | ~384MB | Small services, low traffic |
| 1GB | 75% | ~768MB | Typical microservice |
| 2GB | 75% | ~1.5GB | Medium load services |
| 4GB | 75% | ~3GB | High load, complex processing |

---

## Detecting Memory Issues in Containers

```bash
# Check if container was OOMKilled
kubectl describe pod myapp-pod | grep -i oom

# Docker stats
docker stats myapp --no-stream

# JVM heap usage via JMX or Actuator
curl http://localhost:8080/actuator/metrics/jvm.memory.used
curl http://localhost:8080/actuator/metrics/jvm.memory.max
```
