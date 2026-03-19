# Chapter 07 — Background Jobs: Structured Answers

---

## Answer 1: Idempotent Payment Job with Stripe

**Question:** Design a job to process payments that is safe to retry and won't double-charge customers.

```ruby
# The key: use Stripe's idempotency keys
# Same idempotency key → Stripe returns same result, doesn't re-charge

class ProcessPaymentJob < ApplicationJob
  queue_as :critical
  sidekiq_options retry: 3

  def perform(order_id)
    order = Order.find_by(id: order_id)

    # Guard 1: Record deleted
    return unless order

    # Guard 2: Already processed (idempotency check)
    return if order.payment_completed?

    # Guard 3: Already attempted with same job_id (retry safety)
    # Use order_id as idempotency key (stable, not job_id which changes on retry)
    idempotency_key = "order_payment_#{order_id}_#{order.updated_at.to_i}"

    charge = Stripe::Charge.create(
      {
        amount:   order.total_cents,
        currency: 'usd',
        customer: order.user.stripe_customer_id,
        metadata: { order_id: order_id }
      },
      idempotency_key: idempotency_key  # Stripe deduplicates on this key
    )

    order.update!(
      payment_status:    'completed',
      stripe_charge_id:  charge.id,
      paid_at:           Time.current
    )

    OrderMailer.receipt(order).deliver_later

  rescue Stripe::CardError => e
    # Card declined — don't retry, mark as failed
    order.update!(payment_status: 'failed', payment_error: e.message)
    discard_job(e)  # Don't retry card declines

  rescue Stripe::IdempotencyError => e
    # Different params with same key — log and investigate
    Rails.logger.error "Idempotency conflict for order #{order_id}: #{e.message}"
    raise  # Retry will use same key and same params — should succeed

  rescue Stripe::RateLimitError, Stripe::APIConnectionError => e
    # Transient errors — retry with backoff
    raise  # Let Sidekiq retry with exponential backoff
  end
end

# config: Order model state machine
class Order < ApplicationRecord
  enum payment_status: { pending: 0, processing: 1, completed: 2, failed: 3 }

  def payment_completed?
    completed?
  end
end
```

---

## Answer 2: Transactional Outbox Pattern

**Question:** Implement the outbox pattern to guarantee job enqueueing when the database transaction commits.

```ruby
# Problem: enqueue_later inside transaction may run before commit
# Solution: store "pending jobs" in DB as part of same transaction,
# then a separate process reads and enqueues them after commit

# db/migrate/xxx_create_outbox_messages.rb
class CreateOutboxMessages < ActiveRecord::Migration[7.1]
  def change
    create_table :outbox_messages do |t|
      t.string  :job_class, null: false
      t.jsonb   :arguments, null: false, default: []
      t.string  :queue,     default: 'default'
      t.integer :priority,  default: 0
      t.datetime :process_after
      t.datetime :processed_at
      t.datetime :failed_at
      t.string  :error_message
      t.integer :attempts, default: 0
      t.timestamps
    end
    add_index :outbox_messages, :processed_at
    add_index :outbox_messages, :created_at
  end
end

# app/models/outbox_message.rb
class OutboxMessage < ApplicationRecord
  scope :pending, -> { where(processed_at: nil, failed_at: nil).where("process_after IS NULL OR process_after <= ?", Time.current) }

  def self.enqueue(job_class, *arguments, wait: nil)
    create!(
      job_class:     job_class.to_s,
      arguments:     arguments,
      process_after: wait ? wait.from_now : nil
    )
  end

  def dispatch!
    job_class.constantize.perform_later(*arguments)
    update!(processed_at: Time.current)
  rescue => e
    update!(failed_at: Time.current, error_message: e.message, attempts: attempts + 1)
  end
end

# Usage in models/services (inside transactions):
class UserRegistrationService
  def call(params)
    ActiveRecord::Base.transaction do
      user = User.create!(params)
      # Enqueued as part of same transaction — if transaction rolls back, no outbox entry
      OutboxMessage.enqueue(WelcomeEmailJob, user.id)
      OutboxMessage.enqueue(CreateDefaultWorkspaceJob, user.id)
    end
    # Only AFTER transaction commits, outbox entries exist
  end
end

# app/jobs/outbox_processor_job.rb — runs on a schedule
class OutboxProcessorJob < ApplicationJob
  queue_as :outbox

  def perform
    OutboxMessage.pending.order(:created_at).limit(100).each do |message|
      message.dispatch!
    end
    # Re-schedule itself
    OutboxProcessorJob.set(wait: 5.seconds).perform_later if OutboxMessage.pending.exists?
  end
end

# Alternative: use after_commit hook + direct enqueueing (simpler, good enough for most cases)
class User < ApplicationRecord
  after_create_commit :enqueue_welcome_job

  private

  def enqueue_welcome_job
    WelcomeEmailJob.perform_later(id)
  end
end
```

---

## Answer 3: Large-Scale Email Job with Batching and Progress

**Question:** Design a system to send newsletters to 500,000 users with progress tracking.

```ruby
# app/jobs/newsletter_campaign_job.rb
class NewsletterCampaignJob < ApplicationJob
  queue_as :newsletters
  BATCH_SIZE = 1000

  def perform(campaign_id, offset = 0)
    campaign = EmailCampaign.find(campaign_id)
    return if campaign.cancelled?

    total    = campaign.recipients.count
    batch    = campaign.recipients.offset(offset).limit(BATCH_SIZE)

    return if batch.empty?

    batch.each do |user|
      NewsletterEmailJob.perform_later(campaign_id, user.id)
    end

    # Update progress
    processed = [offset + BATCH_SIZE, total].min
    campaign.update_columns(
      jobs_enqueued: processed,
      progress_pct:  (processed.to_f / total * 100).round
    )

    # Enqueue next batch
    if offset + BATCH_SIZE < total
      NewsletterCampaignJob.perform_later(campaign_id, offset + BATCH_SIZE)
    else
      campaign.update!(status: 'sending', all_enqueued_at: Time.current)
    end
  end
end

# app/jobs/newsletter_email_job.rb
class NewsletterEmailJob < ApplicationJob
  queue_as :newsletters
  sidekiq_options retry: 3

  def perform(campaign_id, user_id)
    campaign = EmailCampaign.find_by(id: campaign_id)
    return unless campaign&.sending?

    user = User.find_by(id: user_id)
    return unless user&.subscribed?

    # Idempotency: don't send twice
    return if EmailDelivery.exists?(campaign_id: campaign_id, user_id: user_id)

    CampaignMailer.newsletter(user, campaign).deliver_now

    EmailDelivery.create!(
      campaign_id: campaign_id,
      user_id:     user_id,
      delivered_at: Time.current
    )

    # Increment delivered counter atomically
    EmailCampaign.where(id: campaign_id).update_all("delivered_count = delivered_count + 1")
  end
end

# db model to track progress
class EmailCampaign < ApplicationRecord
  enum status: { draft: 0, sending: 1, sent: 2, cancelled: 3 }

  def progress
    return 0 if recipients_count == 0
    (delivered_count.to_f / recipients_count * 100).round
  end
end

# API endpoint to poll progress:
# GET /api/v1/campaigns/123/status
# → { status: "sending", delivered: 250000, total: 500000, progress: 50 }
```

---

## Answer 4: Job with Resumable Progress (CSV Import)

**Question:** Implement a large CSV import job that can resume from where it left off after being killed.

```ruby
# db/migrate/xxx_create_import_jobs.rb
class CreateImportJobs < ActiveRecord::Migration[7.1]
  def change
    create_table :import_jobs do |t|
      t.references :user, null: false
      t.string  :file_key,      null: false  # S3 key or blob ID
      t.integer :total_rows
      t.integer :processed_rows, default: 0
      t.integer :failed_rows,    default: 0
      t.jsonb   :errors,         default: []
      t.string  :status,         default: 'pending'
      t.timestamps
    end
  end
end

# app/jobs/import_products_job.rb
class ImportProductsJob < ApplicationJob
  queue_as :imports
  BATCH_SIZE = 500

  def perform(import_job_id)
    import_job = ImportJob.find(import_job_id)
    import_job.update!(status: 'processing')

    file = ActiveStorage::Blob.find_by(key: import_job.file_key)
    file.open do |tempfile|
      csv = CSV.new(tempfile, headers: true)

      # Skip already-processed rows (resume point)
      import_job.processed_rows.times { csv.shift }

      batch     = []
      row_index = import_job.processed_rows

      csv.each do |row|
        row_index += 1
        batch << parse_row(row, row_index)

        if batch.size >= BATCH_SIZE
          process_batch(import_job, batch, row_index)
          batch = []
        end
      end

      # Process remaining rows
      process_batch(import_job, batch, row_index) unless batch.empty?
    end

    import_job.update!(
      status:     'completed',
      total_rows: import_job.processed_rows
    )

  rescue => e
    import_job.update!(status: 'paused', last_error: e.message)
    raise  # Let Sidekiq retry — will resume from processed_rows
  end

  private

  def process_batch(import_job, rows, current_row)
    Product.transaction do
      rows.each do |row|
        begin
          Product.upsert(row, unique_by: :sku)
        rescue ActiveRecord::RecordInvalid => e
          import_job.errors << { row: row[:row_num], error: e.message }
        end
      end

      import_job.update_columns(
        processed_rows: current_row,
        status:         'processing'
      )
    end
  end

  def parse_row(row, index)
    {
      row_num:    index,
      sku:        row['sku']&.strip,
      name:       row['name']&.strip,
      price:      row['price']&.to_f,
      updated_at: Time.current,
      created_at: Time.current
    }
  end
end
```

---

## Answer 5: Rate-Limited External API Job

**Question:** Implement a job that respects external API rate limits without busy-waiting.

```ruby
# Approach: use Sidekiq middleware + Redis token bucket

# app/sidekiq/rate_limit_middleware.rb
class RateLimitMiddleware
  def call(worker, job, queue, redis_pool)
    rate_key = job['rate_limit_key']

    if rate_key
      allowed = redis_pool.with do |redis|
        # Sliding window rate limit
        window_start = (Time.current.to_f - 60).to_s
        pipe_results = redis.pipelined do |pipe|
          pipe.zremrangebyscore(rate_key, '-inf', window_start)
          pipe.zcard(rate_key)
          pipe.zadd(rate_key, Time.current.to_f, "#{Time.current.to_f}:#{SecureRandom.hex(4)}")
          pipe.expire(rate_key, 120)
        end
        pipe_results[1] < job['rate_limit_count'].to_i
      end

      unless allowed
        # Re-enqueue after the rate limit window resets
        wait_time = rand(30..60).seconds
        worker.class.set(wait: wait_time).perform_in(wait_time, *job['args'])
        return  # Don't yield — job not executed
      end
    end

    yield
  end
end

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add RateLimitMiddleware
  end
end

# app/jobs/sync_from_external_api_job.rb
class SyncFromExternalApiJob < ApplicationJob
  queue_as :external_sync

  # Custom Sidekiq options for rate limiting
  sidekiq_options(
    'rate_limit_key'   => 'external_api:rate_limit',
    'rate_limit_count' => 90  # 90 per minute (leave headroom below 100 limit)
  )

  def perform(resource_id)
    response = ExternalApiClient.fetch(resource_id)
    ExternalResource.upsert(response.to_h, unique_by: :external_id)
  rescue ExternalApiClient::RateLimitError => e
    # API told us we're rate limited — back off
    retry_at = e.retry_after || 60.seconds.from_now
    self.class.set(wait_until: retry_at).perform_later(resource_id)
    discard_job(e)  # Don't let Sidekiq retry normally
  end
end

# Simpler alternative: use sidekiq-throttled gem
class SyncFromExternalApiJob < ApplicationJob
  include Sidekiq::Throttled::Worker

  sidekiq_throttle(
    threshold: { limit: 90, period: 1.minute }
  )

  def perform(resource_id)
    ExternalApiClient.fetch(resource_id)
  end
end
```

---
