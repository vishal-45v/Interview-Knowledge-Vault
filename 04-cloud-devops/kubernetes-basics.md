# Kubernetes Basics for Java Backend Engineers

---

## Core Kubernetes Concepts

```
Cluster: Group of nodes running Kubernetes
  └── Control Plane (master): API server, scheduler, etcd, controller-manager
  └── Worker Nodes: Run containerized workloads

Pod: Smallest deployable unit
  └── One or more containers sharing network/storage
  └── Ephemeral — can be killed/replaced

Deployment: Manages multiple pod replicas with rolling updates

Service: Stable network endpoint for accessing pods
  └── ClusterIP: Internal only
  └── NodePort: Exposed on each node's port
  └── LoadBalancer: External load balancer (cloud providers)
  └── Ingress: HTTP/HTTPS routing (preferred for web apps)

ConfigMap: Non-sensitive configuration
Secret: Sensitive configuration (base64-encoded, or external secrets operator)

HorizontalPodAutoscaler: Auto-scale based on CPU/memory/custom metrics
PersistentVolumeClaim: Request for storage
```

---

## Spring Boot Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
    version: "1.2.3"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Create 1 extra pod during update
      maxUnavailable: 0   # Never reduce below desired count during update
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.2.3
        ports:
        - containerPort: 8080
        
        # Resource requests and limits
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"      # 0.25 CPU cores
          limits:
            memory: "1Gi"
            cpu: "1000m"     # 1 CPU core
        
        # Environment from ConfigMap and Secrets
        envFrom:
        - configMapRef:
            name: myapp-config
        env:
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
        
        # Startup probe (fails until app is ready to accept traffic)
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 30  # 30 * 10s = 5 minutes for startup
          periodSeconds: 10
        
        # Liveness probe (restart if fails)
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 15
          failureThreshold: 3
        
        # Readiness probe (stop sending traffic if fails)
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 3
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
```

---

## Service and Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
  tls:
  - hosts:
    - api.mycompany.com
    secretName: myapp-tls
```

---

## Spring Boot Actuator Integration with Kubernetes

```yaml
# application.yml — configure probes
management:
  endpoint:
    health:
      probes:
        enabled: true        # Enable liveness/readiness probes
      show-details: always
  health:
    livenessState:
      enabled: true          # /actuator/health/liveness
    readinessState:
      enabled: true          # /actuator/health/readiness
```

**Liveness vs Readiness:**
- Liveness: "Is the app alive? Should Kubernetes restart it?"
  → Fails if the app is in a bad state (deadlock, OOM)
- Readiness: "Is the app ready to handle requests?"
  → Fails if app is starting up, warming cache, or temporarily busy

---

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up when average CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## ConfigMap and Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  LOGGING_LEVEL_ROOT: "INFO"
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  db-password: "my-secret-password"  # base64-encoded in storage
  jwt-secret: "my-jwt-signing-key"
```

In production, use External Secrets Operator to sync secrets from AWS Secrets Manager, HashiCorp Vault, etc.

---

## Graceful Shutdown in Kubernetes

```yaml
# application.yml
server:
  shutdown: graceful  # Wait for in-flight requests to complete
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Max 30s to finish

# Kubernetes sends SIGTERM → Spring starts graceful shutdown → existing requests finish
# terminationGracePeriodSeconds in pod spec must be > timeout-per-shutdown-phase
```
