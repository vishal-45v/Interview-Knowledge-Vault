# Chapter 08 — Caching: Diagram Explanations

---

## Diagram 1: Rails Caching Layers

```
Request Path (from user to database):

  User / Browser
      │
      ▼
  CDN / Reverse Proxy (Cloudflare, Nginx)
  ← Cache-Control: public, max-age=300
  ← Vary, ETag headers
      │  cache miss / first request
      ▼
  Rails Application
      │
      ├── HTTP Cache check (stale? / fresh_when)
      │   → 304 Not Modified (no body sent)
      │
      ├── Fragment Cache (Redis / Memcache)
      │   cache @product do ... end
      │   → cache hit: skip rendering
      │
      ├── Low-level Cache (Rails.cache.fetch)
      │   → expensive query result stored in Redis
      │
      ├── ActiveRecord Query Cache (in-memory, per request)
      │   → same SQL in same request returns memoized result
      │
      └── Database (PostgreSQL)
          → slowest layer, queried when all caches miss

─────────────────────────────────────────────────────────────────────

CACHE STORE OPTIONS:
┌─────────────────────┬──────────────────┬────────────────────────┐
│ Store               │ Use When         │ Notes                  │
├─────────────────────┼──────────────────┼────────────────────────┤
│ :memory_store       │ Single process   │ Lost on restart        │
│                     │ dev/test         │                        │
├─────────────────────┼──────────────────┼────────────────────────┤
│ :file_store         │ Dev, small apps  │ Slow, survives restart │
│                     │                  │ Bad for multi-server   │
├─────────────────────┼──────────────────┼────────────────────────┤
│ :redis_cache_store  │ Production       │ Shared across servers  │
│                     │                  │ TTL, LRU eviction      │
├─────────────────────┼──────────────────┼────────────────────────┤
│ :mem_cache_store    │ Production       │ Memcached, no TTL on   │
│                     │                  │ key iteration          │
├─────────────────────┼──────────────────┼────────────────────────┤
│ :null_store         │ Testing          │ No actual caching      │
│                     │                  │ Cache-code testing     │
└─────────────────────┴──────────────────┴────────────────────────┘
```

---

## Diagram 2: Russian Doll Cache Hierarchy

```
User Page (outer cache key: users/42-T3)
├── User header (included in outer)
└── Posts list
    ├── Post #99 (middle cache key: posts/99-T2)
    │   ├── Post header
    │   └── Comments
    │       ├── Comment #500 (inner key: comments/500-T1)
    │       ├── Comment #501 (inner key: comments/501-T0)
    │       └── Comment #502 (inner key: comments/502-T0)
    └── Post #100 (middle cache key: posts/100-T0)
        └── Comments
            └── Comment #600 (inner key: comments/600-T0)

─────────────────────────────────────────────────────────────────────

WHEN COMMENT #500 IS UPDATED:

  1. comment_500.updated_at = new timestamp T4
     → cache key: comments/500-T4 (MISS — build new cache)

  2. touch: true on Comment → post_99.updated_at = new timestamp T5
     → cache key: posts/99-T5 (MISS — rebuild with new comment #500 cache)

  3. touch: true on Post → user_42.updated_at = new timestamp T6
     → cache key: users/42-T6 (MISS — rebuild with new post #99 cache)

  RESULT:
  ✓ Comment #500 cache: rebuilt
  ✓ Post #99 cache: rebuilt
  ✓ User #42 page cache: rebuilt
  ✓ Comment #501, #502: STILL VALID (untouched)
  ✓ Post #100 and Comment #600: STILL VALID

  Only affected nodes are rebuilt. Everything else: instant cache hit.
```

---

## Diagram 3: ETag / HTTP Cache Flow

```
FIRST REQUEST (cache empty):

  Client                              Server
    │  GET /api/products/1             │
    │─────────────────────────────────▶│
    │                                  │  compute response
    │                                  │  stale? → true → render
    │◀───────────────────────────────── │
    │  200 OK                          │
    │  ETag: "abc123"                  │
    │  Cache-Control: public, max-age=0│
    │  Last-Modified: Mon Jan 15       │
    │  { product data... }             │

─────────────────────────────────────────────────────────────────────

SUBSEQUENT REQUEST (client has cached ETag):

  Client                              Server
    │  GET /api/products/1             │
    │  If-None-Match: "abc123"         │
    │  If-Modified-Since: Mon Jan 15   │
    │─────────────────────────────────▶│
    │                                  │  stale?(@product)
    │                                  │  check: product.cache_key == "abc123"?
    │                                  │  → YES, same ETag
    │◀───────────────────────────────── │
    │  304 Not Modified                │
    │  (no body — saves bandwidth!)    │

─────────────────────────────────────────────────────────────────────

AFTER PRODUCT UPDATE:

  Client                              Server
    │  GET /api/products/1             │
    │  If-None-Match: "abc123"         │
    │─────────────────────────────────▶│
    │                                  │  product.updated_at changed
    │                                  │  new cache key → "def456"
    │                                  │  stale? → true → render
    │◀───────────────────────────────── │
    │  200 OK                          │
    │  ETag: "def456"  ← new ETag      │
    │  { updated product data... }     │
```

---

## Diagram 4: Cache Stampede and Prevention

```
STAMPEDE (no protection):

  T=0:00  Cache is populated: key="data", value={...}
  T=0:59  Cache expires (TTL = 60s)
  T=1:00  50 concurrent requests arrive

  Thread 1: cache.fetch("data") → MISS → start expensive_query (10 seconds)
  Thread 2: cache.fetch("data") → MISS → start expensive_query (10 seconds)
  ...
  Thread 50: cache.fetch("data") → MISS → start expensive_query (10 seconds)
  → 50 simultaneous expensive queries → DB on fire!

─────────────────────────────────────────────────────────────────────

WITH race_condition_ttl:

  cache.fetch("data", expires_in: 60.seconds, race_condition_ttl: 10.seconds)

  T=1:00  Cache is "expired" but race_condition_ttl extends it 10 more seconds
  Thread 1: cache.fetch → MISS (exclusive lock acquired) → start expensive_query
  Thread 2: cache.fetch → STALE HIT → returns old value (lock exists)
  Thread 3-50: same as Thread 2 → all return old value

  T=1:10  Thread 1 finishes → writes new value to cache
  Thread 51+: cache.fetch → HIT → returns new value

  DB load: 1 query instead of 50. App degrades gracefully.

─────────────────────────────────────────────────────────────────────

CACHE KEY FLOW:

  Rails.cache.fetch("expensive_data", expires_in: 15.minutes) { compute() }

  Redis key: "myapp:production:expensive_data"
             ──────────────  ─────────────────
             namespace        your key
  Value: marshalled Ruby object (uses Marshal.dump by default)
  TTL: 900 seconds (15 * 60)
```

---
