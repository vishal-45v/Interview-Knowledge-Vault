# Chapter 07 — Background Jobs: Diagram Explanations

---

## Diagram 1: Job Lifecycle — From Enqueue to Completion

```
Rails App                    Redis                    Sidekiq Worker
    │                           │                           │
    │  MyJob.perform_later(id)   │                           │
    │──────────────────────────▶│                           │
    │                           │  LPUSH queue:default      │
    │                           │  { job_class: "MyJob",    │
    │                           │    args: [id],            │
    │                           │    jid: "abc123",         │
    │                           │    enqueued_at: ... }     │
    │                           │                           │
    │                           │  Sidekiq::Fetcher polls   │
    │                           │◀─────────────────────────│
    │                           │  BRPOP queue:default 2    │
    │                           │  → job payload            │
    │                           │──────────────────────────▶│
    │                           │                           │  MyJob.new.perform(id)
    │                           │                           │  success → done
    │                           │                           │  failure → retry

─────────────────────────────────────────────────────────────────────

JOB STATES:

  Enqueued          Scheduled         Running         Dead
  ┌─────────┐       ┌──────────┐      ┌─────────┐    ┌─────────┐
  │ queue:  │       │ zset:    │      │ workers │    │ zset:   │
  │ default │       │ schedule │      │ hash    │    │ dead    │
  │ (FIFO)  │       │ (sorted  │      │ (in     │    │ (after  │
  │         │       │ by time) │      │ flight) │    │ 25 fail)│
  └─────────┘       └──────────┘      └─────────┘    └─────────┘
       ▲                  │                │               ▲
       │                  │ time arrives   │ exception     │
       │                  ▼                │               │
  perform_later      moved to queue   retry with backoff ──┘
  perform_in         for execution    (up to 25 times)
```

---

## Diagram 2: Sidekiq Retry Backoff Schedule

```
Attempt    Wait Time (approx)    Total Time from First Failure
─────────  ──────────────────    ─────────────────────────────
1 (fail)   immediate
2          25 sec                25 seconds
3          75 sec                1.7 minutes
4          3 min                 5 minutes
5          6 min                 11 minutes
6          11 min                22 minutes
10         1.7 hours             ~4 hours
15         ~12 hours             ~2 days
20         ~2.4 days             ~7 days
25         ~7 days               ~21 days → DEAD QUEUE

Formula: (retry_count ** 4) + 15 + rand(10) * (retry_count + 1)

─────────────────────────────────────────────────────────────────────

RETRY CONFIGURATION:

class MyJob < ApplicationJob
  sidekiq_options retry: 5          # Max 5 retries (not 25)

  # Or with custom handling:
  sidekiq_retries_exhausted do |job, exception|
    # Called when max retries exceeded (just before dead queue)
    Bugsnag.notify(exception)
    AdminMailer.job_failed(job).deliver_now
  end
end

# Discard without retrying:
rescue_from ActiveRecord::RecordNotFound, with: :discard_job
```

---

## Diagram 3: Queue Priority and Worker Assignment

```
Sidekiq Process (config/sidekiq.yml):
  queues:
    - [critical, 10]    ← checked 10x more often
    - [default, 3]      ← checked 3x more often
    - [low, 1]          ← checked 1x

Thread allocation with 10 workers, 3 queues:

  Each worker polls queues in order of weight, randomly:
  Iteration 1: try critical (10/14 chance), try default (3/14), try low (1/14)

  Real effect with 10 workers, constant work in all queues:
  ┌──────────────────────────────────────────────────────────┐
  │  ~7 workers handle critical                              │
  │  ~2 workers handle default                               │
  │  ~1 worker handles low                                   │
  └──────────────────────────────────────────────────────────┘

  WARNING: If critical is always saturated:
  → low priority jobs may NEVER run (starvation)
  → Solution: separate Sidekiq processes per queue type

  Process 1: queues: [critical]         (5 workers)
  Process 2: queues: [default, low]     (10 workers)
```

---

## Diagram 4: Transactional Outbox vs Direct Enqueueing

```
DIRECT ENQUEUEING (UNSAFE):

  Rails Transaction:
  ┌─────────────────────────────────────────────────┐
  │  user = User.create!                             │
  │  MyJob.perform_later(user.id)  ←── enqueued NOW │
  │                                                  │
  │  [something fails here]                          │
  │  ROLLBACK → user never committed to DB!          │
  └─────────────────────────────────────────────────┘
  Job runs: User.find(user.id) → RecordNotFound ✗

─────────────────────────────────────────────────────────────────────

OUTBOX PATTERN (SAFE):

  Rails Transaction:
  ┌──────────────────────────────────────────────────────┐
  │  user = User.create!                                  │
  │  OutboxMessage.create!(job: MyJob, args: [user.id])   │
  │  ← both in SAME transaction                          │
  │                                                      │
  │  if failure → both rolled back → no orphan job       │
  │  if success → both committed                         │
  └──────────────────────────────────────────────────────┘

  OutboxProcessorJob (runs every 5 seconds):
  OutboxMessage.pending.each do |msg|
    msg.job_class.perform_later(*msg.arguments)  # enqueue to Redis/Sidekiq
    msg.update!(processed_at: Time.current)
  end

─────────────────────────────────────────────────────────────────────

SIMPLER ALTERNATIVE (usually sufficient):

  class User < ApplicationRecord
    after_create_commit -> { MyJob.perform_later(id) }
    # after_create_commit fires AFTER transaction commits
    # Not inside the transaction
  end
```

---

## Diagram 5: Sidekiq Threading Model and DB Connections

```
OS Process: Sidekiq
  ├── Main Thread (Sidekiq heartbeat, scheduler)
  ├── Thread 1  ─── Job A ─── DB Connection 1
  ├── Thread 2  ─── Job B ─── DB Connection 2
  ├── Thread 3  ─── Job C ─── DB Connection 3
  ├── Thread 4  ─── waiting ─ waiting for DB connection
  ├── Thread 5  ─── waiting ─ waiting for DB connection
  ├── Thread 6  ─── Job D ─── DB Connection 4
  ├── Thread 7  ─── Job E ─── DB Connection 5  ← pool exhausted
  ├── Thread 8  ─── waiting ─ ActiveRecord::ConnectionTimeoutError!
  ├── Thread 9  ─── Job F ─── waiting for connection...
  └── Thread 10 ─── Job G ─── waiting for connection...

CONFIGURATION FIX:

  # config/database.yml
  production:
    pool: 12  # >= Sidekiq concurrency (10) + 2 overhead

  # config/sidekiq.yml
  :concurrency: 10

  # Environment check:
  ActiveRecord::Base.connection_pool.size
  # Should be >= Sidekiq.options[:concurrency]

  # PgBouncer (transaction mode):
  # 100 app "connections" → 20 real DB connections
  # app_pool: 100, pgbouncer_pool: 20
```

---
