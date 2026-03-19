# Chapter 08 — Caching: Structured Answers

---

## Answer 1: Complete Russian Doll Caching with Touch Chain

**Question:** Implement Russian Doll caching for a blog with posts, comments, and authors. Include the complete touch chain and explain the invalidation flow.

```ruby
# Models with touch chain:
class User < ApplicationRecord
  has_many :posts
  has_many :comments
  # No touch needed here — User is the root
end

class Post < ApplicationRecord
  belongs_to :user, touch: true   # Post change → touches User
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post, touch: true   # Comment change → touches Post → touches User
end

# When a comment is updated:
# 1. comment.updated_at = Time.current
# 2. touch triggers: post.updated_at = Time.current
# 3. touch cascades: user.updated_at = Time.current
# → ALL cache keys containing these objects are now different

# app/views/users/show.html.erb
<%= cache @user do %>
  <%= render 'user_profile', user: @user %>

  <% @user.posts.includes(comments: :user).each do |post| %>
    <%= cache post do %>
      <%= render 'post_header', post: post %>

      <% post.comments.each do |comment| %>
        <%= cache comment do %>
          <%= render 'comment', comment: comment %>
        <% end %>
      <% end %>
    <% end %>
  <% end %>
<% end %>

# Cache key structure:
# Outer: "views/users/42-20240115120000/abc123def456"
#          namespace / id-updated_at / template_digest
# Middle: "views/posts/99-20240115130000/789xyz"
# Inner:  "views/comments/500-20240115140000/def456"

# Cache hit/miss on comment edit:
# - Comment 500 updated → comment key changes → comment cache MISS
# - Post 99 touched → post key changes → post cache MISS
# - User 42 touched → user key changes → user cache MISS
# → Full re-render on next request, new cache written

# Cache hit on unrelated data:
# - Comment on Post 100 updated → only Post 100 chain invalidated
# - Post 99, User 42 caches: still valid → served from cache!

# Config to enable caching:
# config/environments/production.rb
config.action_controller.perform_caching = true
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  expires_in: 1.hour  # Default TTL for all fragment caches
}
```

---

## Answer 2: Low-Level Caching with Cache Stampede Prevention

**Question:** Implement caching for an expensive aggregation query with cache stampede prevention.

```ruby
# app/services/dashboard_stats_service.rb
class DashboardStatsService
  CACHE_KEY     = 'admin/dashboard/stats'
  CACHE_TTL     = 15.minutes
  STALE_TTL     = 5.minutes  # How long after expiry to serve stale

  def self.fetch
    # race_condition_ttl: when cache expires, only ONE request recomputes
    # Other concurrent requests get the stale value for race_condition_ttl more seconds
    Rails.cache.fetch(
      CACHE_KEY,
      expires_in:         CACHE_TTL,
      race_condition_ttl: STALE_TTL  # Serve stale data for 5 min while recomputing
    ) do
      compute_stats
    end
  end

  def self.invalidate!
    Rails.cache.delete(CACHE_KEY)
  end

  private

  def self.compute_stats
    # This is expensive — runs once per 15 minutes
    {
      total_users:        User.count,
      new_users_today:    User.where("created_at > ?", Date.current.beginning_of_day).count,
      total_revenue:      Order.completed.sum(:total_amount),
      monthly_revenue:    Order.completed.where("created_at > ?", 1.month.ago).sum(:total_amount),
      top_products:       Product.joins(:order_items)
                                 .group('products.id')
                                 .order('SUM(order_items.quantity) DESC')
                                 .limit(10)
                                 .pluck(:name, :id),
      computed_at:        Time.current
    }
  end
end

# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < Admin::BaseController
  def show
    @stats = DashboardStatsService.fetch
    @cache_age = Time.current - @stats[:computed_at] if @stats[:computed_at]
  end
end

# Background job to proactively warm/refresh cache before expiry:
class RefreshDashboardStatsJob < ApplicationJob
  queue_as :low

  def perform
    DashboardStatsService.invalidate!
    DashboardStatsService.fetch  # Re-populates cache
  end
end

# Schedule every 14 minutes (just before 15-min TTL):
# config/schedule.rb (whenever gem) or cron:
# every 14.minutes { RefreshDashboardStatsJob.perform_later }
```

---

## Answer 3: HTTP Caching with ETags

**Question:** Implement ETag-based HTTP caching for an API endpoint correctly.

```ruby
# app/controllers/api/v1/products_controller.rb
class Api::V1::ProductsController < Api::V1::BaseController
  # PUBLIC resource — same for all users
  def show
    @product = Product.includes(:category, :images).find(params[:id])

    # stale? checks If-None-Match (ETag) and If-Modified-Since headers
    # Returns true if client has stale data → we should render
    # Returns false if client has fresh data → returns 304 Not Modified automatically
    if stale?(@product, public: true)
      render json: ProductBlueprint.render(@product)
    end
    # If stale? returns false, Rails already set 304 and we don't need to render
  end

  # USER-SPECIFIC resource
  def show_with_pricing
    @product = Product.find(params[:id])

    if stale?(
      etag:          [@product, current_user.pricing_tier],  # User-specific ETag
      last_modified: @product.updated_at,
      public:        false  # Private cache (browser only, not CDN)
    )
      render json: {
        product: ProductBlueprint.render(@product),
        price:   current_user.price_for(@product)
      }
    end
  end

  # COLLECTION with pagination
  def index
    @products = Product.active.order(created_at: :desc).page(params[:page]).per(20)
    collection_etag = [
      @products.map(&:cache_key_with_version),
      params[:page]
    ]

    if stale?(etag: collection_etag, public: true)
      render json: {
        data: ProductBlueprint.render(@products),
        meta: { total: @products.total_count, page: @products.current_page }
      }
    end
  end
end

# Custom ETag with Rack::ETag middleware:
# Rails automatically adds ETag header from the response body MD5 if not set manually
# Manual stale? is more efficient — avoids rendering the response if client has it

# Testing HTTP caching:
# curl -v https://api.example.com/products/1
# → 200 OK, ETag: "abc123", Cache-Control: public, max-age=0, must-revalidate

# curl -v -H 'If-None-Match: "abc123"' https://api.example.com/products/1
# → 304 Not Modified (no body — bandwidth saved!)
```

---

## Answer 4: Conditional Caching for Mixed Public/Private Pages

**Question:** Cache a product page that has both public and user-specific content without leaking personalized data.

```ruby
# Strategy: cache the public shell, inject personalization via JS/Turbo

# Option 1: Fragment caching with conditional logic
# app/views/products/show.html.erb

<%# Public content — same for all users — cache aggressively %>
<%= cache @product, expires_in: 30.minutes do %>
  <h1><%= @product.name %></h1>
  <p><%= @product.description %></p>
  <%= render @product.images %>
  <%= render 'specs', product: @product %>
<% end %>

<%# Private/personalized content — NOT cached %>
<div id="pricing" data-product-id="<%= @product.id %>">
  <% if current_user %>
    <p>Your price: <%= number_to_currency(current_user.price_for(@product)) %></p>
    <%= button_to "Add to Cart", cart_items_path, params: { product_id: @product.id } %>
  <% else %>
    <p>Price: <%= number_to_currency(@product.base_price) %></p>
    <%= link_to "Sign in to see your price", sign_in_path %>
  <% end %>
</div>

# Option 2: ESI (Edge Side Includes) — CDN fetches fragments separately
# Cache the main page, but mark pricing fragment with ESI include
# <esi:include src="/api/products/1/pricing" />
# CDN fetches and stitches the personalized part

# Option 3: Fragment caching with user-scoped keys
<%= cache [@product, current_user&.id, current_user&.pricing_tier] do %>
  <%= render 'personalized_section', product: @product, user: current_user %>
<% end %>
# Separate cache entry per user tier — efficient if few tiers (guest, basic, premium)

# Option 4: Vary header (tells caches to vary by Authorization)
response.headers['Vary'] = 'Authorization'
# CDN caches separate responses for each Authorization value
# Can explode cache storage if many unique tokens
# Better: vary by a stable user attribute (pricing_tier, not token)

# app/controllers/products_controller.rb
def show
  @product = Product.includes(:images, :category, :specs).find(params[:id])

  # Set public cache headers for the page shell
  # Note: if you use cookies/session, browser won't cache with public anyway
  fresh_when @product, public: !current_user
end
```

---

## Answer 5: Multi-Layer Caching Architecture

**Question:** Design a complete caching strategy for a high-traffic e-commerce site.

```ruby
# Layer 1: HTTP caching at CDN (Cloudflare/Fastly)
# Public product pages: Cache-Control: public, max-age=300, stale-while-revalidate=60
# Product images: Cache-Control: public, max-age=31536000, immutable (content-hash filenames)
# API (public): Cache-Control: public, max-age=60
# API (private/authenticated): Cache-Control: private, no-store

# Layer 2: HTTP caching at browser
# Same headers apply — browser respects Cache-Control
# Conditional GET with ETag for API clients

# Layer 3: Rails fragment caching (Redis)
# Product cards, navigation menus, category lists
config.cache_store = :redis_cache_store, {
  url:            ENV['REDIS_CACHE_URL'],
  expires_in:     1.hour,
  namespace:      "#{Rails.env}:v1",  # namespace includes version for cache busting on deploy
  error_handler:  -> (method:, returning:, exception:) { Bugsnag.notify(exception) }
}

# Layer 4: Low-level caching (Rails.cache.fetch)
# Expensive aggregations, external API responses, computed values

# Layer 5: Database query caching (built-in to Rails)
# Automatic — same query in same request returns cached result
# Clear with: ActiveRecord::Base.uncached { ... } when needed

# Cache key namespacing for zero-downtime deploys:
# config/initializers/cache_version.rb
Rails.application.config.cache_key_version = ENV.fetch('CACHE_VERSION', Rails.application.config.version)

# In views:
cache [@product, Rails.application.config.cache_key_version] do
  # New deploy → new version → all caches automatically invalidated
end

# Or: configure namespace in cache store:
config.cache_store = :redis_cache_store, {
  namespace: "myapp:#{ENV['HEROKU_RELEASE_VERSION'] || 'dev'}"
}

# Monitoring cache effectiveness:
# Track: cache hit rate (should be > 80%), cache memory usage, key count
# Alert on: hit rate < 60%, memory near limit, key count explosion
Rails.cache.stats  # For Redis cache store
```

---
