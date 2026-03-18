# Chapter 10 — Deployment & Production: Structured Answers

---

## Answer 1: Zero-Downtime Migration — Expand, Migrate, Contract

**Question:** You need to rename the `users.name` column to `users.full_name` without downtime.

```ruby
# WRONG approach (causes downtime):
class RenameNameToFullName < ActiveRecord::Migration[7.1]
  def change
    rename_column :users, :name, :full_name
  end
end
# Old code deployed: expects `name` column → SQL error after migration!

# ─────────────────────────────────────────────────────────────────────
# SAFE APPROACH: 3 deploys over 2-3 days

# ── DEPLOY 1: Expand ──────────────────────────────────────────────────
# Add new column, write to BOTH, read from old

# Migration:
class AddFullNameToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :full_name, :string
    # Don't add NOT NULL yet — old code won't write it
  end
end

# App code — update to write both columns:
class User < ApplicationRecord
  before_save :sync_full_name

  def name=(val)
    write_attribute(:name, val)
    write_attribute(:full_name, val)
  end

  def full_name=(val)
    write_attribute(:full_name, val)
    write_attribute(:name, val)
  end

  private

  def sync_full_name
    self.full_name ||= name  # Backfill if only name was set
  end
end

# ── BACKFILL (can run in background job) ──────────────────────────────
class BackfillFullNameJob < ApplicationJob
  def perform
    User.where(full_name: nil).find_each do |user|
      user.update_column(:full_name, user.name)
    end
  end
end

# ── DEPLOY 2: Switch ──────────────────────────────────────────────────
# Start reading from new column, still writing both

class User < ApplicationRecord
  def name
    full_name || read_attribute(:name)  # Read from full_name first
  end
end

# Update all code to use full_name instead of name
# All queries, reports, serializers

# ── DEPLOY 3: Contract ────────────────────────────────────────────────
# Remove old column — no code references it anymore

class RemoveNameFromUsers < ActiveRecord::Migration[7.1]
  def change
    remove_column :users, :name, :string
  end
end

# Clean up the User model:
class User < ApplicationRecord
  # Remove all the sync_full_name / backward compatibility code
end
```

---

## Answer 2: Production Puma and Rails Configuration

**Question:** Show a production-ready Puma config and Rails production settings.

```ruby
# config/puma.rb
max_threads_count = ENV.fetch('RAILS_MAX_THREADS', 5).to_i
min_threads_count = ENV.fetch('RAILS_MIN_THREADS') { max_threads_count }.to_i
threads min_threads_count, max_threads_count

# Workers: for cluster mode (multi-process)
# Set based on available RAM: each worker ~300-500MB
# Formula: (total RAM - 1GB overhead) / per-worker RAM
workers ENV.fetch('WEB_CONCURRENCY', 2).to_i

# Worker timeout: kill and restart if worker is silent for this long
# Should be > your slowest expected request
worker_timeout ENV.fetch('PUMA_WORKER_TIMEOUT', 60).to_i

# Phased restart support
preload_app!

# Port / binding
port        ENV.fetch('PORT', 3000)
environment ENV.fetch('RAILS_ENV', 'development')

# Worker lifecycle hooks for connection pool management
on_worker_boot do
  # Re-establish DB connection in each worker after fork
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end

# Graceful shutdown: drain in-flight requests before shutdown
shutdown_debug ENV.fetch('PUMA_SHUTDOWN_DEBUG', false)

# Logging
log_requests ENV.fetch('PUMA_LOG_REQUESTS', false) == 'true'

# ─────────────────────────────────────────────────────────────────────

# config/environments/production.rb
Rails.application.configure do
  # Eager load all code (Zeitwerk) — no lazy loading in production
  config.eager_load = true

  # Full error reports disabled for security
  config.consider_all_requests_local = false

  # Caching
  config.action_controller.perform_caching = true
  config.cache_store = :redis_cache_store, {
    url:            ENV['REDIS_URL'],
    expires_in:     1.hour,
    error_handler: -> (method:, returning:, exception:) {
      Sentry.capture_exception(exception)
    }
  }

  # Static files (Nginx/CDN should handle this, not Rails)
  config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?

  # Logging
  config.log_level = :info
  config.log_tags  = [:request_id, :remote_ip]
  config.logger    = ActiveSupport::Logger.new($stdout)
  config.log_formatter = ::Logger::Formatter.new

  # Assets
  config.assets.compile = false  # Never compile in production (precompile instead)
  config.assets.digest  = true   # Fingerprint filenames for cache busting

  # Security
  config.force_ssl = true  # Set to false if SSL is terminated at load balancer

  # Session
  config.session_store :cookie_store,
    key:      '_myapp_session',
    secure:   true,
    httponly: true,
    same_site: :lax

  # ActionMailer
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.perform_deliveries = true
  config.action_mailer.raise_delivery_errors = false  # Don't crash on email failure
  config.action_mailer.default_url_options = { host: ENV['APP_HOST'] }

  # Job queue
  config.active_job.queue_adapter = :sidekiq
  config.active_job.queue_name_prefix = "production"

  # Active Storage (S3)
  config.active_storage.service = :amazon

  # Strict host authorization (Rails 6+)
  config.hosts << ENV['APP_HOST']
end
```

---

## Answer 3: Feature Flags with Flipper

**Question:** Implement feature flags for a gradual rollout.

```ruby
# Gemfile: gem 'flipper', gem 'flipper-redis', gem 'flipper-ui'

# config/initializers/flipper.rb
require 'flipper'
require 'flipper/adapters/redis'

Flipper.configure do |config|
  config.adapter do
    redis = Redis.new(url: ENV['REDIS_URL'])
    Flipper::Adapters::Redis.new(redis)
  end
end

# Define features:
# Run once in console or via migration:
# Flipper.add(:new_checkout)
# Flipper.add(:beta_dashboard)
# Flipper.add(:dark_mode)

# config/routes.rb (mount admin UI)
mount Flipper::UI.app(Flipper) => '/admin/features', constraints: AdminConstraint.new

# app/controllers/application_controller.rb
def new_checkout_enabled?
  Flipper.enabled?(:new_checkout, current_user)
end
helper_method :new_checkout_enabled?

# Gradual rollout:
# 10% of users:
Flipper.enable_percentage_of_actors(:new_checkout, 10)
# → deterministic: user 42 always gets same result (hash of user.id % 100)

# Specific users (beta testers):
Flipper.enable_actor(:new_checkout, admin_user)

# Specific group:
Flipper.register(:beta_users) do |actor|
  actor.respond_to?(:beta?) && actor.beta?
end
Flipper.enable_group(:new_checkout, :beta_users)

# Everyone:
Flipper.enable(:new_checkout)

# Disable instantly:
Flipper.disable(:new_checkout)

# In views:
# <% if flipper[:new_checkout].enabled?(current_user) %>
#   <%= render 'new_checkout_button' %>
# <% else %>
#   <%= render 'classic_checkout_button' %>
# <% end %>

# In controllers:
class CheckoutsController < ApplicationController
  before_action :ensure_new_checkout_enabled, only: [:new_flow]

  private

  def ensure_new_checkout_enabled
    redirect_to classic_checkout_path unless Flipper.enabled?(:new_checkout, current_user)
  end
end

# Make users Flipper actors by including the module:
class User < ApplicationRecord
  include Flipper::Identifier
  # Gives each user a unique flipper_id: "User;42"
end
```

---

## Answer 4: Structured Logging for Production

**Question:** Set up structured JSON logging that integrates with log aggregation systems.

```ruby
# Gemfile: gem 'lograge', gem 'logstash-event'

# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.base_controller_class = ['ActionController::Base', 'ActionController::API']

  config.lograge.custom_options = lambda do |event|
    {
      host:        event.payload[:headers]['Host'],
      request_id:  event.payload[:headers]['X-Request-Id'],
      user_id:     event.payload[:current_user_id],
      tenant_id:   event.payload[:tenant_id],
      params:      event.payload[:params].except('controller', 'action', 'format'),
      exception:   event.payload[:exception]&.first,
    }.compact
  end

  # Output as JSON
  config.lograge.formatter = Lograge::Formatters::Json.new
end

# config/environments/production.rb
config.log_level = :info
config.logger = ActiveSupport::Logger.new($stdout).tap do |logger|
  logger.formatter = proc do |severity, time, progname, msg|
    {
      timestamp: time.utc.iso8601(3),
      level:     severity,
      message:   msg,
      app:       'myapp',
      env:       Rails.env
    }.to_json + "\n"
  end
end

# Adding request context to all logs (tagged logging):
config.log_tags = [
  :request_id,
  -> (req) { req.env['HTTP_X_FORWARDED_FOR'] || req.remote_ip }
]

# app/controllers/application_controller.rb
around_action :set_logging_context

def set_logging_context
  payload = {
    current_user_id: current_user&.id,
    tenant_id:       current_tenant&.id
  }
  Rails.logger.tagged(payload.map { |k, v| "#{k}=#{v}" }) { yield }
end

# Custom log entry in code:
Rails.logger.info({
  event:   'payment_processed',
  user_id: user.id,
  amount:  charge.amount,
  source:  'stripe'
}.to_json)

# Output (goes to stdout → log aggregator picks up):
# {"timestamp":"2024-01-15T12:00:00.123Z","level":"INFO","message":"{\"event\":\"payment_processed\"...}","app":"myapp","env":"production"}
```

---

## Answer 5: Production Deployment Checklist

**Question:** What does a complete production readiness checklist look like for a Rails app?

```ruby
# ── Security ──────────────────────────────────────────────────────────
# [ ] config.force_ssl = true (or handled at load balancer)
# [ ] SECRET_KEY_BASE set from environment variable, not hardcoded
# [ ] Sensitive credentials in Rails credentials or environment variables
# [ ] Content Security Policy configured
# [ ] CORS restricted to known origins
# [ ] Rate limiting (rack-attack) configured
# [ ] SQL injection protection: use parameterized queries (ActiveRecord handles this)
# [ ] Mass assignment protection: strong parameters in all controllers
# [ ] User input sanitized in views (Rails auto-escapes, but verify html_safe usage)
# [ ] Security headers: X-Frame-Options, X-Content-Type-Options, etc.

# config/environments/production.rb additions:
config.action_dispatch.default_headers = {
  'X-Frame-Options'        => 'SAMEORIGIN',
  'X-XSS-Protection'       => '0',  # Let CSP handle this (browser XSS auditor deprecated)
  'X-Content-Type-Options' => 'nosniff',
  'X-Download-Options'     => 'noopen',
  'X-Permitted-Cross-Domain-Policies' => 'none',
  'Referrer-Policy'        => 'strict-origin-when-cross-origin'
}

# ── Performance ────────────────────────────────────────────────────────
# [ ] config.eager_load = true
# [ ] Assets precompiled: rails assets:precompile
# [ ] Database connection pool sized correctly
# [ ] Redis configured for sessions and caching
# [ ] Background job adapter configured (Sidekiq)
# [ ] DB indexes on all foreign keys and frequently queried columns
# [ ] N+1 queries detected with Bullet in development

# ── Operations ─────────────────────────────────────────────────────────
# [ ] Error tracking: Sentry/Bugsnag configured with DSN
# [ ] APM: New Relic / Datadog / Scout configured
# [ ] Log aggregation: logs output to stdout (for Docker) or to file
# [ ] Health check endpoint: GET /up → 200 OK (no auth required)
# [ ] Uptime monitoring: external check every 60 seconds
# [ ] Database backups: automated daily with point-in-time recovery
# [ ] Alerting: PagerDuty/OpsGenie for critical errors

# ── Deployment ─────────────────────────────────────────────────────────
# [ ] Zero-downtime deploys configured (Puma phased restart or blue-green)
# [ ] Release phase or deploy hooks for db:migrate
# [ ] Rollback procedure documented and tested
# [ ] Environment variables documented (not the values — the NAMES)
# [ ] Seed data for essential lookup tables

# Health check controller (never remove this):
class HealthController < ActionController::API
  def show
    checks = {
      status:   'ok',
      database: database_healthy?,
      redis:    redis_healthy?,
      jobs:     sidekiq_healthy?,
      version:  ENV['APP_VERSION'] || 'unknown'
    }
    status = checks[:database] && checks[:redis] ? :ok : :service_unavailable
    render json: checks, status: status
  end

  private

  def database_healthy?
    ActiveRecord::Base.connection.execute('SELECT 1')
    true
  rescue => e
    false
  end

  def redis_healthy?
    Redis.current.ping == 'PONG'
  rescue => e
    false
  end

  def sidekiq_healthy?
    Sidekiq::Stats.new.workers_size > 0
  rescue => e
    false
  end
end
```

---
