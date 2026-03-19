# High-Level Architecture Questions

---

## Design a URL Shortener (TinyURL)

**Scale:** 100M URLs shortened/day, 1B redirects/day, 5 years retention

**Estimation:**
```
Write RPS: 100M / 86400 ≈ 1,200/sec
Read RPS: 1B / 86400 ≈ 11,600/sec → ~10:1 read:write ratio
Storage: 100M/day × 365 × 5 = 182B URLs × 500B each ≈ 91TB over 5 years
```

**Design:**
```
Client → Load Balancer → URL Shortener Service → Redis Cache → Write to DB
                                                → DB (MySQL/Cassandra)

Short code generation:
  Option 1: Hash(originalUrl) → take first 7 chars of MD5
  Option 2: Auto-increment ID → Base62 encode (a-z, A-Z, 0-9)
  Option 3: Pre-generated pool of codes → assign from pool

Redirect:
  GET /{code} → check Redis cache → if miss, check DB
              → 301 Redirect (permanent, browser caches)
              → 302 Redirect (temporary, analytics tracking)
```

---

## Design Instagram-Like Feed

**Scale:** 500M users, 100M photos/day, 5B feed views/day

**Fan-out strategies:**
```
Push (Fan-out on write):
  User posts → immediately write to all followers' feeds
  Read: O(1) — feed pre-computed
  Write: O(followers) — expensive for celebrities with 10M followers

Pull (Fan-out on read):
  User requests feed → fetch posts from all followees
  Read: O(followees) — expensive at read time
  Write: O(1)

Hybrid (Instagram approach):
  Normal users (< 1M followers): Push model
  Celebrities (> 1M followers): Pull model — their posts fetched at read time
```

---

## Design a Notification System

**Requirements:** Send push, email, SMS notifications at scale (100M/day)

```
Architecture:
  Event Sources (Order placed, payment received, etc.)
       │
       ▼
  Notification Service API
  (validates, enriches, persists notification)
       │
       ▼
  Kafka Topic: notification-requests
       │
       ├── Email Consumer → SendGrid/SES
       ├── SMS Consumer → Twilio
       ├── Push Consumer → FCM/APNs
       └── Webhook Consumer → HTTP calls

Key features:
  - Retry with exponential backoff
  - Dead letter queue for failed notifications
  - Rate limiting per user (don't spam)
  - User preferences (opt-out per channel)
  - Notification log for auditing
```

---

## Design a Distributed Cache

**Requirements:** Low latency, high availability, horizontal scalability

```
Architecture:
  Client → Cache Cluster (Redis Cluster)
         → Consistent hashing to shard key
         → Node 1 (master) + 2 replicas
         → Node 2 (master) + 2 replicas
         → Node 3 (master) + 2 replicas

Key design decisions:
  Eviction: LRU (Least Recently Used) for general cache
  Consistency: Redis uses async replication (AP)
  Failure: Automatic failover (sentinel/cluster mode)
  Partitioning: Consistent hashing (minimal key movement on node add/remove)
```
