# Chapter 08 — Caching: Follow-Up Traps

---

**Trap 1: "You cache `User.all` for 1 hour. A user updates their profile. What happens?"**

```ruby
# The cache is stale for up to 1 hour after the update
# Any request in that window sees old data

# Why this is especially bad:
Rails.cache.fetch("all_users", expires_in: 1.hour) { User.all.to_a }
# Returns 1000 users. User #42 changes their email.
# Everyone sees old email for up to 1 hour.
# Plus: if User.all returns 1000+ objects, that's a large serialized blob in Redis

# Fix 1: Time-based key (not great — still stale)
Rails.cache.fetch("all_users_#{Date.current}", expires_in: 1.day) { User.all.to_a }

# Fix 2: Invalidate on update
class User < ApplicationRecord
  after_commit :invalidate_all_users_cache

  private
  def invalidate_all_users_cache
    Rails.cache.delete("all_users")
  end
end

# Fix 3: Use a version key derived from latest update time
def users_cache_key
  max_updated = User.maximum(:updated_at)
  "all_users/#{max_updated.to_i}"
end
Rails.cache.fetch(users_cache_key, expires_in: 1.hour) { User.all.to_a }
# New key on every write → old key naturally expires via TTL
# Downside: SELECT MAX(updated_at) on every cache key computation

# Fix 4: Don't cache User.all — cache at the right granularity
# Cache individual users or paginated subsets
```

---

**Trap 2: "Your Russian Doll cache correctly invalidates when a post is updated. But updating a comment doesn't invalidate the post cache. Why?"**

```ruby
# Russian Doll invalidation requires explicit touch propagation
# ActiveRecord doesn't automatically touch parent on child update

# BROKEN setup:
class Post < ApplicationRecord
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post
  # No touch: true → updating comment doesn't change post.updated_at
end

# View:
# cache @post do  ← key includes @post.cache_key_with_version
#   render @post.comments  ← nested cache
# end

# When comment updated: post.updated_at unchanged → same cache key → stale!

# FIX:
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # Updates post.updated_at on comment change
end

# touch: true cascades:
# Comment → touch Post → (if post has: belongs_to :user, touch: true) → touch User

# TRAP WITHIN THE FIX:
# touch: true does UPDATE posts SET updated_at = ? WHERE id = ?
# This fires post.after_commit callbacks!
# If Post has expensive callbacks, every comment update triggers them
# Be careful about callback chains triggered by touch
```

---

**Trap 3: "You use `cache @product` in a view. The product has 10 variants (sizes/colors). A new variant is added. Does the cache invalidate?"**

```ruby
# It depends on what the cache block contains

# If the view renders @product.variants:
cache @product do
  render @product.variants  # Renders variants, but cache key is @product.cache_key
end

# Adding a variant does NOT change @product.updated_at (the product record itself)
# → cache key unchanged → stale variant list!

# Fix 1: Include variants in cache key
cache [@product, @product.variants.cache_key_for_collection] do
  render @product.variants
end
# cache_key_for_collection uses: count + maximum(:updated_at)

# Fix 2: touch: true from variant to product
class ProductVariant < ApplicationRecord
  belongs_to :product, touch: true
  # Adding/updating a variant touches the product → cache key updates
end

# Fix 3: Separate cache for variants
cache @product do
  # Product details only (no variants)
end

cache [@product, 'variants'] do
  render @product.variants  # Separate cache, same invalidation problem
end
# Still need touch: true or collection cache key
```

---

**Trap 4: "You set `Cache-Control: public, max-age=3600` on your product API endpoint. A user's price is personalized. What happens?"**

```ruby
# Cache-Control: public means ANY cache (CDN, proxy, browser) can cache this response
# If the response contains personalized data:
# User A requests product → CDN caches { price: $99.00, user_tier: "premium" }
# User B requests same product → CDN serves cached response → User B sees $99 price!

# FIX: Use Cache-Control: private for personalized responses
response.headers['Cache-Control'] = 'private, max-age=300'
# private = only the browser cache (not CDN or shared proxies) may cache this

# Or: separate endpoints for public vs personalized data
# GET /api/v1/products/1           → public data, Cache-Control: public
# GET /api/v1/products/1/pricing   → personalized, Cache-Control: private

# In Rails:
# For public API data:
expires_in 1.hour, public: true   # Sets Cache-Control: public, max-age=3600

# For user-specific data:
expires_in 5.minutes, public: false  # Sets Cache-Control: private, max-age=300

# Also: if you use Vary header
response.headers['Vary'] = 'Authorization'
# Tells CDN to cache DIFFERENT responses for different Authorization header values
# But CDN may not respect Vary correctly for Authorization
```

---

**Trap 5: "You cache a count with `Rails.cache.fetch('user_count') { User.count }`. You delete a user. The count shows 101 when there are actually 100 users. How do you fix this?"**

```ruby
# Time-based expiry is the lazy fix (stale for up to TTL):
Rails.cache.fetch('user_count', expires_in: 1.minute) { User.count }
# Stale for up to 1 minute — acceptable?

# Active invalidation on change:
class User < ApplicationRecord
  after_commit :invalidate_user_count_cache

  private

  def invalidate_user_count_cache
    Rails.cache.delete('user_count')
  end
end

# Better: use a version-based key
def self.cached_count
  Rails.cache.fetch("users/count/#{User.maximum(:updated_at)&.to_i}") do
    User.count
  end
end
# But: SELECT MAX(updated_at) on every count read — only worthwhile if count query is very slow

# Best for simple counts: use counter_cache
# (DB-level, always accurate, no cache invalidation needed)
class User < ApplicationRecord
  belongs_to :organization, counter_cache: :users_count
end
# Organization has users_count column, auto-incremented/decremented

# Or: use Rails.cache.increment for simple counters
Rails.cache.increment('user_count')   # on create
Rails.cache.decrement('user_count')   # on destroy
# But risk: cache miss → wrong baseline → reinitialize periodically
```

---

**Trap 6: "You have `stale?(@post)` in your controller. A POST to `/posts/1/like` increments the like count but doesn't update `post.updated_at`. The ETag doesn't change. The client's cached version shows wrong like count."**

```ruby
# stale? uses: cache_key_with_version → includes updated_at
# If updated_at doesn't change, ETag doesn't change
# Client sends If-None-Match: old_etag → server returns 304 Not Modified → client shows stale data

# FIX 1: Touch the post when likes change
class Like < ApplicationRecord
  belongs_to :post, touch: true  # post.updated_at updated → ETag changes
end

# FIX 2: Include the like count in the ETag
def show
  @post = Post.find(params[:id])
  if stale?(etag: [@post, @post.likes_count], last_modified: @post.updated_at)
    render json: PostBlueprint.render(@post)
  end
end

# FIX 3: Don't use ETags for frequently-changing counters
# Use short max-age instead:
expires_in 30.seconds, public: false
# Client re-validates every 30 seconds regardless

# The real lesson: ETag-based caching works well for data where
# updated_at accurately reflects all changes. For denormalized counts
# or computed fields, you need to ensure the ETag reflects those too.
```

---

**Trap 7: "You use `:file_store` for caching in development. After clearing cache with `Rails.cache.clear`, the app is still serving cached responses. Why?"**

```ruby
# FileStore stores cache in tmp/cache/ (or configured path)
# Rails.cache.clear deletes all files in that directory
# BUT: if you're using HTTP caching headers (Expires, Cache-Control) too,
# the BROWSER has its own cache independent of Rails.cache

# Scenario:
# 1. Server sets: Cache-Control: max-age=3600
# 2. Browser caches the response locally for 1 hour
# 3. You clear Rails.cache (FileStore) on the server
# 4. Browser sends request to server — WAIT, browser doesn't send it!
#    It uses its own cached version for another hour!

# Debug: in browser devtools, disable cache (Network tab → Disable cache)
# Or: force-reload (Ctrl+Shift+R / Cmd+Shift+R)

# Fix for development: set short or zero cache headers in development
config/environments/development.rb:
config.action_controller.perform_caching = false  # Disable Rails fragment caching in dev

# Or use cache-busting query param during development
# Or use :null_store in development:
config.cache_store = :null_store
# Appears to cache but always misses — shows behavior without actually caching
```

---

**Trap 8: "You cache `render partial: 'post', collection: @posts` with collection caching. You add a new column to the posts table. Does the cache auto-invalidate?"**

```ruby
# Collection caching:
render partial: 'post', collection: @posts, cached: true
# Rails generates cache keys per post: "posts/1-20240115120000" (id + updated_at)
# Adding a new column does NOT change updated_at on existing records
# → ALL cached partials are still served with old HTML (missing new column data)

# Fix 1: Touch all records after adding the column (migration)
class AddViewCountToPosts < ActiveRecord::Migration[7.1]
  def up
    add_column :posts, :view_count, :integer, default: 0
    Post.update_all(updated_at: Time.current)  # Force cache invalidation
  end
end

# Fix 2: Include template digest in cache key (Rails does this automatically for partials!)
# Rails digests the partial template and includes it in the cache key:
# "posts/1-20240115120000/abc123" (abc123 is digest of _post.html.erb)
# If you change the partial template, the digest changes → automatic invalidation!

# CAVEAT: Rails only digests the immediate partial, not sub-partials
# If _post.html.erb calls render 'author', changes to _author.html.erb
# don't invalidate _post cache!

# Fix: explicitly include sub-partials in cache digest:
# _post.html.erb
# <%# Template dependency: posts/author %>
render 'author', author: post.author
```

---

**Trap 9: "Your Redis cache store is running on a single instance. Redis goes down. What happens to your Rails app?"**

```ruby
# By default: Rails.cache operations raise errors when Redis is unreachable
# → Your views crash if they use cache blocks
# → Your controllers crash if they call Rails.cache.fetch

# Rails cache store has an error_handler option:
config.cache_store = :redis_cache_store, {
  url:           ENV['REDIS_URL'],
  error_handler: -> (method:, returning:, exception:) {
    # Log but don't raise — return nil/false and continue without cache
    Rails.logger.error "Redis cache error in #{method}: #{exception.message}"
    Bugsnag.notify(exception)
  }
}

# With error_handler: cache misses silently → cache block re-executes (degrades to no-cache)
# App still works, just slower (no caching)

# For high availability: use Redis Sentinel or Redis Cluster
config.cache_store = :redis_cache_store, {
  url: ["redis://sentinel1:26379", "redis://sentinel2:26379"],
  role: "master",
  sentinel: true
}

# Alternative: add read-replica for reads
config.cache_store = :redis_cache_store, {
  url:      ENV['REDIS_PRIMARY_URL'],
  read_url: ENV['REDIS_REPLICA_URL']
}
```

---

**Trap 10: "You cache an expensive query result. The query returns an ActiveRecord::Relation. What gets stored in the cache?"**

```ruby
# ActiveRecord::Relation is LAZY — it doesn't execute the query until iterated
# When you store a Relation in cache:
Rails.cache.write("posts", Post.published.order(:created_at))
# This serializes the Relation OBJECT, not the results!
# When read back: the Relation may execute against the CURRENT state of the DB
# → Not truly cached! Or worse: serialization error

# The bug:
Rails.cache.fetch("posts") { Post.published.order(:created_at) }
# fetch stores the Relation, reads back the Relation
# Iterating the cached Relation hits the DB — you never cached the results!

# FIX: Force evaluation before caching
Rails.cache.fetch("posts", expires_in: 5.minutes) do
  Post.published.order(:created_at).to_a  # .to_a forces query execution
end

# Or:
Rails.cache.fetch("posts", expires_in: 5.minutes) do
  Post.published.order(:created_at).load  # .load also forces execution
end

# For individual records:
Rails.cache.fetch("post_#{id}") { Post.find(id) }  # Returns a Post object, not a Relation — fine
```

---
