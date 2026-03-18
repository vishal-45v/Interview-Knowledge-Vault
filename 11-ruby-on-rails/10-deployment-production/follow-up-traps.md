# Chapter 10 — Deployment & Production: Follow-Up Traps

---

**Trap 1: "You run `rails db:migrate` after deploying the new code. But old app servers are still running and reading the new schema. What happens?"**

```ruby
# The deploy order matters enormously:
# WRONG ORDER:
# 1. Deploy new code (adds references to new column "user_tier")
# 2. Run db:migrate (adds "user_tier" column)
# Problem: Between steps 1 and 2, new code running → SELECT user_tier → column doesn't exist yet → 500!

# ALSO WRONG ORDER:
# 1. Run db:migrate (adds "user_tier" column)
# 2. Deploy new code
# Problem: If migration is DESTRUCTIVE (drops a column), old code running after migration → 500!

# THE SAFE PATTERN: "Expand, Migrate, Contract"
# Phase 1 (expand): Add new column, keep old column. Both work simultaneously.
#   - db:migrate adds: user_tier column (nullable)
#   - Old code: ignores user_tier
#   - New code: reads user_tier (falls back to computing from old column if nil)
# Phase 2: Deploy new code. Old and new work with same schema.
# Phase 3 (contract): Remove old column in a LATER deploy.
#   - After confirming no code references old column
#   - db:migrate drops old column

# The key insight: schema changes must be backward compatible with the PREVIOUS version of code
# Never: add NOT NULL column without a default in a live migration
# Never: remove a column still referenced by deployed code
# Never: rename a column in a single migration (old code breaks immediately)
```

---

**Trap 2: "You have `config.force_ssl = true` in production.rb. Your app is behind an AWS load balancer that terminates SSL. What happens?"**

```ruby
# SSL termination at LB: browser → HTTPS → LB → HTTP → Rails app
# Rails sees HTTP (not HTTPS) connection
# force_ssl: "this request is not HTTPS → redirect to HTTPS"
# Rails redirects to https:// → browser goes back to LB → LB sends HTTP to Rails → redirect loop!

# Fix: configure trusted proxy / X-Forwarded-Proto header
# LB adds: X-Forwarded-Proto: https
# Rails reads this with:
config.action_dispatch.trusted_proxies = [
  "127.0.0.1",
  IPAddr.new("10.0.0.0/8"),      # AWS VPC CIDR
  IPAddr.new("172.16.0.0/12"),   # AWS VPC CIDR range
]

# With trusted proxy, Rails reads X-Forwarded-Proto: https
# → request.ssl? returns true → force_ssl doesn't redirect
# → HSTS headers are still set

# Alternative: handle SSL entirely at the LB/CDN level
# Set force_ssl: false in Rails, enforce HTTPS at infrastructure level
# Simpler but loses HSTS header from Rails (set it in Nginx instead)

# Verify: request.ssl?, request.headers['X-Forwarded-Proto']
# In Rails console (production): ActionDispatch::Request.new(env).ssl?
```

---

**Trap 3: "Your app uses `Rails.cache` with `:memory_store`. You deploy 3 Puma workers. What's the problem?"**

```ruby
# Each Puma worker is a SEPARATE OS process
# :memory_store = in-memory hash, private to each process
# 3 workers = 3 separate memory stores, no sharing

# Problems:
# 1. Cache misses: worker 1 caches "users_count=100". Worker 2 has empty cache → queries DB
# 2. Cache inconsistency: worker 1 invalidates "users_count". Worker 2's cache still has old value!
# 3. Memory multiplication: same cached data stored 3× (one per worker)

# This is why :memory_store is only suitable for:
# - Single-process dev/test environments
# - Data that's process-local by design (singleton per process)

# Production cache stores (shared across processes and servers):
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
# All workers hit the same Redis instance
# Cache invalidation: one worker invalidates → all workers see the change

# Exception: in-process caching for truly immutable data
# memoization with @instance_variable is fine (cleared on each request)
```

---

**Trap 4: "You rotate `SECRET_KEY_BASE`. What breaks and how severe is it?"**

```ruby
# SECRET_KEY_BASE is used to sign:
# 1. Cookie store (session data) — signed with MessageVerifier
# 2. Signed cookies (cookies.signed[:remember_token])
# 3. Encrypted cookies (cookies.encrypted[:data])
# 4. ActiveSupport::MessageEncryptor (used for mailer unsubscribe tokens, etc.)
# 5. ActiveSupport::MessageVerifier (used for signed URLs, etc.)

# When rotated, ALL existing signed/encrypted values become invalid:
# → Every logged-in user gets logged out (session invalid)
# → Remember-me cookies stop working
# → Password reset links stop working (if using signed tokens)
# → Unsubscribe links in emails stop working

# HOW TO ROTATE WITHOUT TOTAL LOGOUT:
# Step 1: Keep old secret as a "rotation" secret while new secret is primary
config.secret_key_base = ENV['NEW_SECRET_KEY_BASE']
config.action_dispatch.cookies_rotations.tap do |cookies|
  cookies.rotate :signed, ENV['OLD_SECRET_KEY_BASE']
  cookies.rotate :encrypted, ENV['OLD_SECRET_KEY_BASE']
end
# New tokens use new secret, old tokens still validated against old secret during transition
# After all sessions expire naturally (session TTL), remove old rotation

# Step 2: Force logout for sensitive rotation (security incident):
# Increment a global session version in Redis
# Each request: if session[:version] != current_version → force logout
```

---

**Trap 5: "You add an index to a large production table. The migration runs. What happens to the table during the index creation?"**

```ruby
# Default PostgreSQL index creation: LOCKS THE TABLE
# While building index: reads and writes blocked → app returns errors!

# Example with 10M row table: index build takes 60-300 seconds
# → 1-5 minutes of downtime!

# Fix: CONCURRENTLY index creation
class AddIndexToUsersEmail < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!  # Required for CONCURRENTLY

  def change
    add_index :users, :email,
              unique: true,
              algorithm: :concurrently  # No table lock!
  end
end

# What CONCURRENTLY does:
# - Builds index without blocking reads/writes
# - Takes longer (2-3× longer than standard)
# - Requires disable_ddl_transaction! (can't be in a transaction)
# - If interrupted, leaves an INVALID index (must be removed manually)

# Check for invalid indexes after migration:
# SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';
# Or: SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;

# What about adding columns concurrently?
# In PostgreSQL 11+: ADD COLUMN with a volatile DEFAULT also requires special handling
# Use: add_column + change_column_default + update in batches + change_column null: false
```

---

**Trap 6: "Your app returns 500 in production, but logs show the error. Sentry never receives the notification. Why?"**

```ruby
# Several reasons Sentry might not receive errors:

# 1. Sentry not initialized before error occurs
config/initializers/sentry.rb:
Sentry.init do |config|
  config.dsn = ENV['SENTRY_DSN']
end
# If ENV['SENTRY_DSN'] is nil → Sentry.init silently disables itself
# Fix: validate DSN is set: raise unless ENV['SENTRY_DSN']

# 2. Error is rescued before Sentry sees it
rescue_from StandardError do |e|
  render json: { error: e.message }, status: 500
  # Sentry's Rack middleware never sees a raised exception — it was rescued!
end
# Fix: report manually in the rescue block:
rescue_from StandardError do |e|
  Sentry.capture_exception(e)
  render json: { error: e.message }, status: 500
end

# 3. Sentry async in a test environment
# Some configurations buffer exceptions and send asynchronously
# In test/development, the buffer may not flush

# 4. Network issues (Sentry server unreachable)
# Errors → async transport queue → process exits before flush
# Fix: set transport_options timeout

# 5. Error rate limiting (Sentry's free tier)
# Too many errors → Sentry throttles inbound events
```

---

**Trap 7: "You configure Puma with 4 workers and 5 threads. What's the formula for required database connections?"**

```ruby
# Each Puma thread can hold one DB connection at a time
# Required connections = workers × threads_per_worker

# Your setup: 4 workers × 5 threads = 20 connections per app server
# On 3 app servers: 60 connections total

# config/puma.rb
workers ENV.fetch('WEB_CONCURRENCY') { 2 }
threads_count = ENV.fetch('RAILS_MAX_THREADS') { 5 }
threads threads_count, threads_count

# config/database.yml
production:
  pool: <%= ENV.fetch('RAILS_MAX_THREADS') { 5 } %>
  # This is per-worker! Total = workers × pool
  # 4 workers × 5 pool = 20 connections per server

# TRAP: people set pool to cover the server total, not per worker
# Wrong: pool: 20 (thinking total for 4 workers)
# → Each worker gets pool=20, total = 4×20 = 80 connections!

# Right: pool = threads per worker (NOT total)
# Puma worker isolation: each worker process has its own connection pool
# Pool size per worker = threads per worker (+ maybe 1-2 extra for background threads)

# The extra overhead:
# Sidekiq running in same process: add sidekiq concurrency to the pool
# Database maintenance tasks: add a few extra
# Recommended: pool = RAILS_MAX_THREADS + 2
```

---

**Trap 8: "You use `before_action :set_locale` to set the locale from params. On production, you notice locale-specific content leaking between requests. Why?"**

```ruby
# I18n.locale is a THREAD-LOCAL variable in Rails
# Thread pools: threads are reused across requests in Puma
# If you SET locale but never RESET it, the next request on the same thread inherits it!

# WRONG:
class ApplicationController < ActionController::Base
  before_action :set_locale

  private

  def set_locale
    I18n.locale = params[:locale] || I18n.default_locale
    # Sets it for this request, but if the after_action doesn't reset...
  end
end

# If around_action resets it properly:
def set_locale
  I18n.with_locale(params[:locale] || I18n.default_locale) do
    yield  # locale is set for the duration, then automatically reset
  end
end
# This is the safest pattern — guaranteed reset even on exceptions

# Similar issue with Current attributes (Rails 7.1+)
# Current attributes are reset between requests automatically
# (Rails calls Current.reset in Rack middleware after each request)

# Thread-local state sources:
# I18n.locale, Time.zone (use around_action with time zone too!)
# Thread.current[:anything] you set manually
# ActiveSupport::CurrentAttributes → auto-reset (safe)
```

---

**Trap 9: "Your Rails app is deployed on Heroku. You run `heroku run rails db:migrate`. What are the risks?"**

```ruby
# heroku run spawns a ONE-OFF dyno — a fresh instance that:
# - Runs alongside your web dynos
# - Runs the SAME code as the current slug
# - Has access to the same database
# - Is NOT the same as a web dyno

# Risk 1: Migration runs against live production database
# Web dynos still serving traffic → migration may lock tables → requests time out

# Risk 2: Migration runs NEW code's migration against OLD code's app
# If deploying code + migration together:
# - Old code: still running on web dynos
# - Migration: runs new code's schema changes
# - Old code + new schema = ActiveRecord::StatementInvalid!

# Risk 3: Heroku restart during migration
# Heroku one-off dynos can be killed
# Interrupted migration = partial schema change = broken database!

# Heroku-safe deploy procedure:
# 1. Enable maintenance mode: heroku maintenance:on
# 2. Deploy new slug: git push heroku main
# 3. Run migration: heroku run rails db:migrate
# 4. Disable maintenance mode: heroku maintenance:off

# Or: use Heroku Release Phase
# Procfile:
# release: bundle exec rails db:migrate
# web: bundle exec puma -C config/puma.rb
# Release phase runs BEFORE new dynos start → safer ordering
```

---

**Trap 10: "Your Content Security Policy blocks your Stripe.js from loading. How do you fix it without disabling CSP?"**

```ruby
# stripe.js loads from https://js.stripe.com
# Default strict CSP: script-src 'self' → blocks external scripts

# WRONG fix: script-src '*' or script-src 'unsafe-inline'
# Both defeat the purpose of CSP

# CORRECT fix: whitelist Stripe's domains specifically
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https, "https://js.stripe.com"
  policy.style_src   :self, :https, :unsafe_inline  # Stripe requires unsafe-inline for style
  policy.connect_src :self, :https, "https://api.stripe.com"
  policy.frame_src   "https://js.stripe.com"  # Stripe uses iframes for 3DS
end

# For inline scripts (Stripe.js initialization), use nonces:
Rails.application.config.content_security_policy_nonce_generator = ->(_request) {
  SecureRandom.base64(16)
}

# In your view:
# <script nonce="<%= content_security_policy_nonce %>">
#   Stripe.init('<%= @publishable_key %>');
# </script>

# This avoids 'unsafe-inline' while allowing your specific inline script
# The nonce is unique per request → attacker can't predict it
```

---
