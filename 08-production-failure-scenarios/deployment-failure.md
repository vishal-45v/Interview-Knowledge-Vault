# Production Failure: Deployment Failure and Rollback

---

## Scenario: Bad Deployment Causes Outage

**Timeline:**
```
14:00 — Deploy v2.5.0 to production (rolling update)
14:05 — Error rate spikes from 0.1% to 35%
14:06 — PagerDuty alert: "Error rate > 10%"
14:08 — On-call identifies new deployment as cause
14:10 — Rollback to v2.4.9 initiated
14:15 — Rollback complete, error rate back to 0.1%
```

---

## What Went Wrong

```java
// v2.5.0: New code with a bug
public class OrderService {

    @Transactional
    public Order createOrder(OrderRequest request) {
        // Bug: NPE when request.getPromoCode() is null
        String discount = promoService.getDiscount(request.getPromoCode().toUpperCase());
        //                                         ^^^^ NullPointerException!
        // 40% of orders don't have a promo code → NPE on those orders
    }
}
```

---

## Immediate Response

```bash
# 1. Rollback Kubernetes deployment
kubectl rollout undo deployment/myapp

# 2. Or rollback to specific revision
kubectl rollout history deployment/myapp  # Show history
kubectl rollout undo deployment/myapp --to-revision=3

# 3. Verify rollback in progress
kubectl rollout status deployment/myapp

# 4. Verify error rate recovering
# Watch Grafana dashboard or:
watch -n 5 'curl -s http://localhost:8080/actuator/metrics/http.server.requests | jq'

# 5. For ECS:
aws ecs update-service --cluster prod --service myapp \
    --task-definition myapp:previous-revision-number \
    --force-new-deployment
```

---

## Prevention: Canary Deployment

```
Instead of deploying to ALL instances simultaneously:
  
  Phase 1: Deploy to 5% of instances
  Monitor for 10 minutes:
  - Error rate: still < 0.5%?
  - Latency: still < 200ms P95?
  - Custom metrics: order success rate?

  If all green:
  Phase 2: 25% → monitor
  Phase 3: 50% → monitor
  Phase 4: 100%

  If any metric degrades:
  → Immediately rollback the canary (affects only 5% of users)
  → No manual intervention needed if using Argo Rollouts or AWS CodeDeploy
```

---

## Automated Rollback with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      analysis:
        templates:
        - templateName: error-rate-check
        startingStep: 1
        args:
        - name: service-name
          value: myapp
      steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 25
      - pause: {duration: 10m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 1m
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          rate(http_server_requests_seconds_count{status=~"5..",job="myapp"}[5m])
          / rate(http_server_requests_seconds_count{job="myapp"}[5m])
    successCondition: result[0] < 0.01  # Error rate < 1%
```

---

## Post-Deployment Checklist

```
Before deployment:
  [ ] Tests pass in CI (unit, integration, contract tests)
  [ ] Staging deployment tested manually
  [ ] DB migration tested on staging
  [ ] Rollback plan documented and tested

During deployment:
  [ ] Monitor error rate, latency, success metrics
  [ ] Compare against baseline (same time last week)
  [ ] Be ready to rollback within 5 minutes

After deployment:
  [ ] Error rate back to baseline after 15 minutes?
  [ ] No memory leaks (heap growing?)
  [ ] No latency regression?
  [ ] Business metrics normal (orders placed, conversions)?

Rollback triggers:
  - Error rate > 5% (baseline 0.1%)
  - P95 latency > 500ms (baseline 100ms)
  - Any new exception type appearing
  - Business metric degradation (order success rate drops)
```
