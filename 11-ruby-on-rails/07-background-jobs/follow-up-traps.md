# Chapter 07 — Background Jobs: Follow-Up Traps

---

**Trap 1: "You say jobs should be idempotent. Your job sends an email. How do you make sending an email idempotent?"**

```ruby
# Emails are hard to make idempotent because the external system (email server) is the side effect
# You can't "unsend" an email, and you can't easily check "did I already send this?"

# Strategy 1: Track email dispatch with a unique key
class WelcomeEmailJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)

    # Check if already sent using a sent_emails tracking table
    return if EmailLog.exists?(user_id: user_id, email_type: 'welcome')

    UserMailer.welcome(user).deliver_now

    EmailLog.create!(user_id: user_id, email_type: 'welcome', sent_at: Time.current)
  end
end

# Problem: Race condition between check and create
# Fix: Use unique index + rescue

def perform(user_id)
  user = User.find(user_id)
  EmailLog.create!(user_id: user_id, email_type: 'welcome', sent_at: Time.current)
  UserMailer.welcome(user).deliver_now
rescue ActiveRecord::RecordNotUnique
  # Already sent — another worker beat us to it
  Rails.logger.info "Welcome email for user #{user_id} already sent, skipping"
end

# Strategy 2: Check for a state transition (only send on first activation)
def perform(user_id)
  user = User.find(user_id)
  return unless user.recently_activated?  # State flag cleared after first notification
  UserMailer.welcome(user).deliver_now
  user.update!(welcome_email_sent: true)
end
```

---

**Trap 2: "Sidekiq uses threads. Does this mean you can use global variables to share state between workers?"**

```ruby
# DANGEROUS: Global variables are shared across all threads
$processing_count = 0

class MyJob < ApplicationJob
  def perform(id)
    $processing_count += 1  # RACE CONDITION — multiple threads modify simultaneously
    # ...
    $processing_count -= 1
  end
end

# Thread-local storage also dangerous — persists across jobs in same thread
Thread.current[:context] = "job_#{id}"
# Next job on same thread might see stale Thread.current[:context]!

# Safe: local variables only
class MyJob < ApplicationJob
  def perform(id)
    context = "job_#{id}"  # Local to this method call — thread-safe
    # ...
  end
end

# Safe: use Rails.cache or Redis with atomic operations
class MyJob < ApplicationJob
  def perform(id)
    Redis.current.incr("processing_count")
    # ...
  ensure
    Redis.current.decr("processing_count")
  end
end

# Safe: use Thread.current ONLY within a single job's execution
# And always clear it in ensure
```

---

**Trap 3: "You pass `user` (an ActiveRecord object) as a job argument. What happens?"**

```ruby
# ActiveJob serializes arguments to JSON for storage in Redis/DB
# ActiveRecord objects use GlobalID for serialization

# When you do:
SendEmailJob.perform_later(user)
# ActiveJob calls user.to_global_id → "gid://myapp/User/42"
# Stored in Redis as: { "job_class": "SendEmailJob", "arguments": ["gid://..."] }

# When job runs:
# GlobalID resolves back to User.find(42)

# THE TRAP:
# What if the user is deleted between enqueueing and execution?
# GlobalID.locate("gid://myapp/User/42") → nil
# ActiveJob raises ActiveJob::DeserializationError

# Fix: rescue deserialization errors
class SendEmailJob < ApplicationJob
  rescue_from ActiveJob::DeserializationError do |error|
    Rails.logger.warn "Job #{self.class} failed to deserialize: #{error.message} — record may be deleted"
    # Don't retry — record is gone
  end

  def perform(user)
    user.send_notification
  end
end

# Better: pass IDs and handle nil explicitly
class SendEmailJob < ApplicationJob
  def perform(user_id)
    user = User.find_by(id: user_id)
    return if user.nil?  # User deleted — skip silently
    user.send_notification
  end
end
```

---

**Trap 4: "You enqueue a job inside a database transaction. The job starts before the transaction commits. What happens?"**

```ruby
# THE BUG:
ActiveRecord::Base.transaction do
  user = User.create!(email: params[:email])
  SendWelcomeEmailJob.perform_later(user.id)
  # Job enqueued here...
  # Job runs here (if using async adapter or if Redis is fast)...
  # Job: User.find(user_id) → ActiveRecord::RecordNotFound!
  # BECAUSE the transaction hasn't committed yet!
  raise "something failed"  # Transaction rolls back
  # User never committed, but job might have already run!
end

# Fix: use after_commit callback or the after_commit_everywhere gem
class User < ApplicationRecord
  after_create_commit do
    SendWelcomeEmailJob.perform_later(id)
  end
  # after_create_commit fires AFTER the transaction commits
  # NOT inside the transaction
end

# Fix 2: Use perform_later outside the transaction
user = nil
ActiveRecord::Base.transaction do
  user = User.create!(email: params[:email])
end
SendWelcomeEmailJob.perform_later(user.id)  # Only if transaction succeeded

# Fix 3: Transactional outbox pattern (write job to DB table in same transaction)
# See structured-answers.md for full implementation
```

---

**Trap 5: "Your job fails and Sidekiq retries it. The job has a side effect: it increments a counter in Redis. After 3 retries, the counter is incremented 3 times. How do you fix this?"**

```ruby
# The problem: side effects accumulate on retry
class UpdateStatsJob < ApplicationJob
  def perform(user_id, event)
    user = User.find(user_id)
    # Do DB work...
    user.save!
    Redis.current.incr("user:#{user_id}:events")  # Runs on EVERY attempt!
  end
end

# Fix 1: Move Redis update after all failure-prone code
class UpdateStatsJob < ApplicationJob
  def perform(user_id, event)
    user = User.find(user_id)
    user.process_event(event)
    user.save!
    # Only reach here if DB work succeeded
    Redis.current.incr("user:#{user_id}:events")
  end
end
# But still runs multiple times if Redis fails and job retries

# Fix 2: Use a job execution ID to deduplicate
class UpdateStatsJob < ApplicationJob
  def perform(user_id, event)
    execution_key = "job_executed:#{job_id}"
    # Redis SET with NX (only set if not exists)
    already_ran = !Redis.current.set(execution_key, 1, nx: true, ex: 3600)
    return if already_ran

    user = User.find(user_id)
    user.process_event(event)
    user.save!
    Redis.current.incr("user:#{user_id}:events")
  end
end

# Fix 3: Design around idempotent operations (SET vs INCR)
# Store the actual state, not a delta
Redis.current.set("user:#{user_id}:event_count", user.reload.events.count)
```

---

**Trap 6: "You have 10 Sidekiq workers each with 10 threads = 100 concurrent DB connections. Your DB pool is set to 5. What happens?"**

```ruby
# Each thread needs its own DB connection from the pool
# 100 threads competing for 5 connections → 95 threads waiting
# After timeout (5 seconds by default) → ActiveRecord::ConnectionTimeoutError

# config/database.yml — wrong default:
production:
  pool: 5  # Only 5 connections!

# Fix: pool size should be >= threads per process
# Sidekiq default: 10 threads per worker
# Formula: pool_size = sidekiq_concurrency + 2 (extra for web requests in same process)

production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

# Sidekiq initializer:
Sidekiq.configure_server do |config|
  config.redis = { url: ENV['REDIS_URL'] }
  # Set DB pool to match concurrency
  ActiveRecord::Base.connection_pool.disconnect!
  ActiveRecord::Base.establish_connection(
    ActiveRecord::Base.configurations.find_db_config('production').configuration_hash
      .merge(pool: config.options[:concurrency] + 2)
  )
end

# Environment variable approach:
# DATABASE_POOL=12 (for 10 Sidekiq threads + 2 overhead)
# RAILS_MAX_THREADS=10 (Puma threads for web)
# Set DB pool: [DATABASE_POOL, RAILS_MAX_THREADS].max
```

---

**Trap 7: "You use `after_save` callback to enqueue a job. The user updates their profile 5 times in quick succession. 5 jobs run concurrently. Is this a problem?"**

```ruby
# Depends on the job. If the job is idempotent (reads current state),
# 5 concurrent runs = 5 reads of current state = same result (OK but wasteful)

# If the job is stateful (uses the state at enqueue time), it's a problem:
class SyncProfileJob < ApplicationJob
  def perform(user_id, snapshot_data)
    # Uses data from when job was enqueued
    ExternalAPI.sync(snapshot_data)  # 5 different snapshots, all sent → external data chaos
  end
end

# Pattern 1: Debouncing — use unique jobs (sidekiq-unique-jobs gem)
class SyncProfileJob < ApplicationJob
  sidekiq_options unique: :until_executed, unique_args: ->(args) { [args.first] }
  # Only ONE job per user_id in queue at a time
end

# Pattern 2: "Soft debounce" — schedule job in future
after_save do
  SyncProfileJob.set(wait: 5.seconds).perform_later(id)
  # If 5 updates happen in 5 seconds, only the last job survives
  # (not truly guaranteed without unique jobs)
end

# Pattern 3: Read current state in job (don't pass snapshot)
class SyncProfileJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    ExternalAPI.sync(user.to_sync_hash)  # Always reads latest state
    # All 5 jobs sync the same current state → no net harm (idempotent)
  end
end
```

---

**Trap 8: "You configure `queue_as :critical` in your ActiveJob subclass, but you also have `sidekiq_options queue: :default` in the job. Which wins?"**

```ruby
# This depends on which adapter and configuration you're using

# ActiveJob queue_as sets the queue name for ActiveJob serialization
class ImportantJob < ApplicationJob
  queue_as :critical
end

# If the job also includes Sidekiq::Worker:
class ImportantJob < ApplicationJob
  include Sidekiq::Worker
  sidekiq_options queue: :default  # This wins for Sidekiq-specific config
  queue_as :critical               # This sets ActiveJob's queue name
end

# When using ActiveJob adapter (ActiveJob::QueueAdapters::SidekiqAdapter):
# queue_as is translated to Sidekiq's queue option
# Direct sidekiq_options OVERRIDE queue_as

# Safe practice: use EITHER ActiveJob (queue_as) OR Sidekiq::Worker directly, not both

# Using only ActiveJob:
class ImportantJob < ApplicationJob
  queue_as :critical
  sidekiq_options retry: 3  # Only use for non-queue options
end

# Verify which queue a job uses:
ImportantJob.queue_name  # → "critical"
```

---

**Trap 9: "A job calls `User.all.each` to process every user. What happens with 1 million users?"**

```ruby
# User.all.each loads ALL 1 million User records into memory at once
# → Potential out-of-memory crash
# → Job takes very long, Sidekiq timeout kills it
# → On retry: starts from scratch with ALL users again

# Fix: Use find_each for memory-efficient batching
class ProcessAllUsersJob < ApplicationJob
  def perform
    User.find_each(batch_size: 1000) do |user|
      # Processes 1000 users at a time from DB, GC'd after each batch
      process_user(user)
    end
  end
end

# Even better: delegate to per-user jobs
class ProcessAllUsersJob < ApplicationJob
  def perform
    User.find_each(batch_size: 1000) do |user|
      ProcessUserJob.perform_later(user.id)
    end
  end
end

# Even better with cursor for resumability:
class ProcessAllUsersJob < ApplicationJob
  def perform(last_processed_id = 0)
    batch = User.where("id > ?", last_processed_id).order(:id).limit(1000)
    return if batch.empty?

    batch.each { |user| ProcessUserJob.perform_later(user.id) }

    # Enqueue next batch
    ProcessAllUsersJob.perform_later(batch.last.id)
  end
end
```

---

**Trap 10: "You use `perform_now` in a test and it works fine. But in production with Sidekiq, the job fails. What might cause this difference?"**

```ruby
# Several causes:

# 1. Argument serialization — perform_now passes Ruby objects directly
#    Sidekiq serializes to JSON and back
class MyJob < ApplicationJob
  def perform(options)
    options[:callback].call  # Works with perform_now (Ruby Proc)
    # FAILS with Sidekiq — Proc can't be serialized to JSON!
  end
end
# Fix: only pass JSON-serializable arguments (strings, numbers, arrays, hashes)

# 2. Environment differences
#    perform_now runs in the Rails web process (with all gems, initializers)
#    Sidekiq runs in a SEPARATE process — check config/initializers run in Sidekiq

# 3. Eager loading
#    Web process may have autoloaded the constant
#    Sidekiq process may not have loaded it yet (in development)

# 4. Timezone/time
#    perform_now: uses test time stubs (Timecop, travel_to)
#    Sidekiq: uses real time

# 5. Transaction context
#    perform_now runs in the same thread/transaction as the caller
#    Sidekiq runs in a completely separate process — no shared transaction

# Best practice: in tests, use sidekiq/testing inline mode:
require 'sidekiq/testing'
Sidekiq::Testing.inline!  # Jobs execute immediately inline but go through full Sidekiq pipeline
# Or use perform_enqueued_jobs in Rails test helpers
```

---
