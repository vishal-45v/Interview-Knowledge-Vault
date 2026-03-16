# System Design Interview Strategy

> A structured approach to answering system design questions in senior/staff engineer interviews.

---

## The REDS Framework

**R — Requirements clarification (5 minutes)**
**E — Estimation (2-3 minutes)**
**D — Design (20-25 minutes)**
**S — Scale and bottlenecks (5-10 minutes)**

---

## Step 1: Requirements Clarification

Always clarify before designing. Interviewers expect you to ask questions.

**Functional requirements:**
- What are the core features? (Write/read? Real-time? Offline?)
- Who are the users? (Consumers, businesses, internal?)
- What are the main use cases? (most common path, edge cases?)

**Non-functional requirements:**
- Scale? (users, requests per second, data size)
- Latency requirements? (< 100ms? Real-time?)
- Availability requirements? (99.9%? 99.99%?)
- Consistency requirements? (Strong? Eventual?)
- Durability? (Can data be lost? How much?)

**Sample questions for an "Uber-like" system:**
- "Are we designing for drivers and riders? Both sides?"
- "How many cities? What's the expected scale?"
- "Real-time location tracking needed?"
- "Do we need surge pricing logic in this design?"
- "What's the focus — matching, location tracking, or billing?"

---

## Step 2: Estimation (Back-of-Envelope)

Interviewers want to see you can reason about scale.

**Common formulas:**
```
Requests per second (RPS) = Daily Active Users × requests/user/day / 86,400

Storage per year = RPS × request_size × seconds_per_year
                 = RPS × size × 31,536,000

Read:Write ratio = (usually 10:1 or higher for most apps)

Network bandwidth = RPS × average_response_size

Cache size = hot data × % of traffic to cache
```

**Example — Twitter-like feed system:**
```
Daily Active Users: 100 million
Avg tweets per user: 5/day
Read:Write ratio: 100:1

Write RPS: 100M × 5 / 86,400 ≈ 6,000 tweets/sec
Read RPS: 6,000 × 100 = 600,000 reads/sec

Storage per tweet: ~300 bytes (text + metadata)
Storage per day: 6,000 × 300B × 86,400 = ~155 GB/day
Storage per year: ~56 TB/year (without media)
```

---

## Step 3: High-Level Design

Start simple, then add components as needed.

**Typical components to consider:**
- Load balancer (ALB/Nginx)
- API Gateway (authentication, rate limiting, routing)
- Application servers (stateless, horizontally scalable)
- Cache layer (Redis/Memcached)
- Database (primary/replica, sharding if needed)
- Message queue (Kafka/SQS for async processing)
- CDN (for static assets)
- Object storage (S3 for files, images)
- Search service (Elasticsearch for full-text search)

**Draw the data flow:**
1. Client → Load Balancer
2. Load Balancer → API Gateway
3. API Gateway → Service
4. Service → Cache (check first)
5. Cache miss → Database
6. Service → Message Queue (for async work)
7. Consumer → Database/Storage

---

## Step 4: Detailed Design — Focus on Key Components

Pick 1-2 critical components and go deep.

### URL Shortener (e.g., TinyURL)

```
Write flow: POST /shorten {"url": "https://long.com/abc..."}
  1. Validate URL
  2. Generate short code: MD5(url) → take 7 chars, or base62(auto-increment ID)
  3. Store: short_code → original_url in Redis (fast reads) + DB (persistence)
  4. Return: {"shortUrl": "https://tiny.ly/xK4j9pQ"}

Read flow: GET /{shortCode}
  1. Check Redis cache (hot codes)
  2. If miss, check DB
  3. Return 301 Redirect (permanent, cached by browser) or 302 (temporary)

Key design decisions:
- Base62 encoding: 0-9, a-z, A-Z = 62 chars
  7 chars = 62^7 = 3.5 trillion unique codes (enough)
- Random vs sequential codes? (sequential reveals info)
- Custom URLs? (collision check needed)
- Expiry? (TTL on cache entry, scheduled cleanup for DB)
```

### Rate Limiting Design

```
Token Bucket Algorithm:
  Each client gets a "bucket" with max N tokens
  Each request consumes 1 token
  Tokens refill at a fixed rate (e.g., 100/minute)
  If bucket empty → 429 Too Many Requests

Implementation with Redis:
  ZADD user:123:requests {timestamp} {requestId}
  ZREMRANGEBYSCORE user:123:requests 0 {windowStart}
  ZCARD user:123:requests  → current count
  If count > limit → reject

Or simpler with sliding window counter:
  INCR rate:user:123:{minute}
  EXPIRE rate:user:123:{minute} 120 (2x window to handle boundaries)
```

---

## Step 5: Scale and Bottlenecks

Identify and address:

**Database scaling:**
- Read replicas (route reads to replicas)
- Sharding (horizontal partitioning by user_id, region, etc.)
- Caching (Redis for hot data)
- Connection pooling

**Application scaling:**
- Stateless services (horizontal scaling)
- Microservices (independent scaling per service)
- Circuit breakers (prevent cascading failures)

**Common bottlenecks:**
1. Database (usually the first bottleneck)
2. Network I/O between services
3. Memory pressure (cache, heap)
4. CPU for CPU-bound operations

---

## System Design Quick-Reference Card

| Requirement | Solution |
|-------------|---------|
| Millions of users | Horizontal scaling + load balancer |
| Read-heavy | Cache (Redis), read replicas |
| Write-heavy | Sharding, queue-based async |
| Real-time updates | WebSocket, Server-Sent Events |
| Search | Elasticsearch, Solr |
| Files/images | S3, CDN |
| Distributed locks | Redis SETNX |
| Rate limiting | Redis + token bucket |
| Idempotency | Idempotency key + DB dedup |
| Ordering guarantees | Kafka (per-partition ordering) |
| High availability | Multi-AZ, circuit breaker, health checks |
| 99.99% uptime | 53 minutes downtime/year → 5+ nines of infra |
