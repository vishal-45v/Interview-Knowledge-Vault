# Chapter 10 — Deployment & Production: Diagram Explanations

---

## Diagram 1: Production Architecture

```
Internet
    │
    ▼
CDN (Cloudflare / CloudFront)
    │  Cache-Control: public → served from edge, no origin hit
    │  Cache miss / dynamic content:
    ▼
Load Balancer (AWS ALB / nginx)
    │  SSL termination (HTTPS → HTTP to app)
    │  Health check: GET /up → 200?
    │  Round-robin to app servers
    ├─────────────────────┐
    │                     │
    ▼                     ▼
App Server 1           App Server 2
Puma (cluster mode)    Puma (cluster mode)
  ├─ Worker 1           ├─ Worker 1
  │  ├─ Thread 1-5      │  ├─ Thread 1-5
  ├─ Worker 2           ├─ Worker 2
     └─ Thread 1-5         └─ Thread 1-5
    │                     │
    └──────────┬──────────┘
               │
       ┌───────┼───────┐
       │               │
       ▼               ▼
  PostgreSQL        Redis
  (primary DB)   (cache + sessions
                  + Sidekiq queue)
                       │
                       ▼
                Sidekiq Workers
                (background jobs)
```

---

## Diagram 2: Zero-Downtime Deploy (Phased Restart)

```
BEFORE DEPLOY:
  Load Balancer → [Old Worker A] [Old Worker B] [Old Worker C]
  All serving version V1

─────────────────────────────────────────────────────────────────────

DEPLOY STARTS (git push → Capistrano/Kamal/deploy script):
  Step 1: New code deployed to servers

  Step 2: Send SIGUSR1 to Puma (phased restart)
  Puma spawns new worker alongside old:
  Load Balancer → [Old Worker A] [Old Worker B] [Old Worker C]
                + [New Worker 1]  (version V2, accepting requests)

  Old Worker A: finish in-flight requests → exit
  Load Balancer → [Old Worker B] [Old Worker C] [New Worker 1]
                + [New Worker 2]  (version V2)

  Old Worker B: finish in-flight requests → exit
  Load Balancer → [Old Worker C] [New Worker 1] [New Worker 2]
                + [New Worker 3]  (version V2)

  Old Worker C: finish in-flight requests → exit

AFTER DEPLOY:
  Load Balancer → [New Worker 1] [New Worker 2] [New Worker 3]
  All serving version V2
  Zero requests dropped!

─────────────────────────────────────────────────────────────────────

CONSTRAINT: Old and New code run simultaneously!
  DB schema must be compatible with BOTH V1 and V2 code during transition.
  Never: deploy code that requires a column that hasn't been added yet.
  Never: drop a column that V1 code still reads.
```

---

## Diagram 3: Expand, Migrate, Contract for Column Rename

```
INITIAL STATE:
  Database: users table has column: name
  Code V1: reads/writes users.name

─────────────────────────────────────────────────────────────────────

DEPLOY 1 — EXPAND:
  Migration: ADD COLUMN users.full_name (nullable)
  Code V2: reads users.name, writes BOTH users.name AND users.full_name
  Background job: backfills full_name = name for all existing rows

  DB: name="Alice", full_name="Alice"  (both populated)
  Old code (V1): still works (reads name only)
  New code (V2): writes both (forward compatible)

─────────────────────────────────────────────────────────────────────

DEPLOY 2 — SWITCH:
  Migration: add NOT NULL constraint to full_name (all rows populated now)
  Code V3: reads users.full_name, writes BOTH (backward compatible if V2 still runs)

  DB: name="Alice", full_name="Alice"  (both still present)
  V2 code still running: reads name → still works
  V3 code reads full_name → correct

─────────────────────────────────────────────────────────────────────

DEPLOY 3 — CONTRACT:
  Code V4: reads/writes ONLY full_name (no reference to name)
  Migration: REMOVE COLUMN users.name

  DB: full_name="Alice"  (name column gone)
  V3 code: would error if still running (but it's been replaced by V4)
  V4 code: reads full_name only → works

  Total time: 3 deploys, ~2-3 days. Zero downtime.
```

---

## Diagram 4: Puma Worker + Thread Sizing

```
Server Resources: 4GB RAM, 4 CPU cores

Puma Configuration Decision:
  Goal: maximize throughput without OOM or CPU starvation

  Per worker footprint:
    Rails boot: ~300MB
    + 5 threads in flight: ~50MB each = 250MB
    Total per worker: ~550MB

  Max workers: (4GB - 500MB OS overhead) / 550MB ≈ 6 workers
  → Use 4 workers (leave headroom)

  Threads: Rails apps are I/O-bound (DB queries, external APIs)
    I/O wait: thread sleeps, CPU free for other threads
    → Higher thread count is beneficial
    → Use 5 threads per worker (balance memory vs concurrency)

  Result: 4 workers × 5 threads = 20 concurrent requests

  Database pool: RAILS_MAX_THREADS = 5 (per worker)
  Total DB connections: 4 workers × 5 pool = 20 connections per server

─────────────────────────────────────────────────────────────────────

Environment Variables:
  WEB_CONCURRENCY=4      # Workers
  RAILS_MAX_THREADS=5    # Threads per worker
  DATABASE_POOL=5        # Must match RAILS_MAX_THREADS

─────────────────────────────────────────────────────────────────────

MEMORY FORMULA:
  Available for Puma: Total RAM - OS (500MB) - Redis (100MB) - Sidekiq (300MB)
  Workers = available / per_worker_memory
  Threads = per_worker_memory / per_thread_memory (5-10 is typical)
```

---

## Diagram 5: Feature Flag Rollout Stages

```
STAGE 1: Internal testing (0.1% → admin users only)
  Flipper.enable_actor(:new_checkout, admin_user)
  → Only admin team sees new flow, 100% of users unaffected

STAGE 2: Beta users (5% of users)
  Flipper.enable_percentage_of_actors(:new_checkout, 5)
  → Deterministic: same 5% always see it, other 95% never do

STAGE 3: Gradual rollout
  10% → monitor: error rate, conversion rate, performance
  25% → still good? → continue
  50% → monitor
  100% → fully rolled out

INSTANT ROLLBACK at any stage:
  Flipper.disable(:new_checkout)  ← one line, takes effect in seconds

─────────────────────────────────────────────────────────────────────

FEATURE FLAG DECISION FLOW:

  User makes request
        │
        ▼
  Flipper.enabled?(:feature, user)
        │
        ├── YES → new behavior
        └── NO  → old behavior

  Flipper checks (in order):
  1. Is feature enabled for ALL? → enabled
  2. Is feature enabled for this SPECIFIC user? → enabled
  3. Is this user in an ENABLED GROUP? → enabled
  4. Does user fall in ENABLED PERCENTAGE? → hash(user.flipper_id) % 100 < percentage
  5. None of above → disabled
```

---
