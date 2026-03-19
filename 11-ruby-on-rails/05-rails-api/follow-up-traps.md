# Chapter 05 — Rails API: Follow-Up Traps

---

**Trap 1: "You said JWT is stateless. Does that mean you can't log out a user with JWT auth?"**

Correct — pure stateless JWT cannot be invalidated server-side. If you issue a JWT and a user "logs out," the token is still valid until expiry. This is a fundamental JWT limitation.

```ruby
# Solution 1: Short access token TTL (15 min) + refresh token blacklist
# User "logs out" → invalidate the refresh token in DB
# Access token still valid for up to 15 min (acceptable tradeoff)

# Solution 2: JTI (JWT ID) allowlist/denylist
class JwtService
  def self.encode(payload)
    jti = SecureRandom.uuid
    JtiRecord.create!(jti: jti, expires_at: 24.hours.from_now)
    JWT.encode(payload.merge(jti: jti), SECRET, ALGORITHM)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET, true, { algorithm: ALGORITHM }).first
    # Check token hasn't been revoked:
    raise InvalidTokenError unless JtiRecord.active.exists?(jti: decoded['jti'])
    decoded
  end
end

# Logout:
def destroy
  payload = JwtService.decode(current_token)
  JtiRecord.find_by(jti: payload['jti'])&.revoke!
  head :no_content
end

# This requires a DB lookup on every request — JWT is no longer "stateless"
# You've reinvented session management. Consider if JWT is even the right choice.
```

---

**Trap 2: "ActiveModel::Serializer (AMS) can cause N+1. Give a concrete example."**

```ruby
# Bug:
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name
  has_many :posts  # Posts are lazy-loaded per user!
end

# Controller with N+1:
users = User.all  # No includes
render json: users, each_serializer: UserSerializer
# SQL: SELECT * FROM users
# Then for each user: SELECT * FROM posts WHERE user_id = ?

# Fix:
users = User.includes(:posts).all
render json: users, each_serializer: UserSerializer
# AMS uses pre-loaded association if available

# Hidden N+1 in AMS — association within association:
class PostSerializer < ActiveModel::Serializer
  has_many :comments
end
class UserSerializer < ActiveModel::Serializer
  has_many :posts  # Each post triggers comments query!
end

# Fix:
users = User.includes(posts: :comments).all

# The real trap: AMS doesn't warn you about N+1
# Use Bullet gem or database query counting in tests
```

---

**Trap 3: "CORS headers are set by rack-cors middleware. But what if an OPTIONS preflight request hits a route that returns 404 because it's not defined in your routes?"**

This is a common misconfiguration. `rack-cors` must intercept the OPTIONS request BEFORE routing:

```ruby
# Config order matters:
config.middleware.insert_before 0, Rack::Cors do  # ← BEFORE ActionDispatch routing
  allow do
    origins 'https://clientapp.com'
    resource '/api/*', methods: [:get, :post, :options, :delete], headers: :any
  end
end

# If CORS middleware is added AFTER routing:
config.middleware.use Rack::Cors  # ← AFTER routing
# The router processes OPTIONS /api/users first
# No route defined for OPTIONS → 404
# rack-cors never sees the request → no CORS headers → client gets 404, not 200 with CORS headers

# Verify middleware order:
# rails middleware | grep -n 'Cors\|Static'
# Rack::Cors should appear near the top (line 1-3)
```

---

**Trap 4: "You used `render json: @user` without a serializer. What does `to_json` return by default and what's the security risk?"**

```ruby
# Default ActiveRecord to_json includes ALL attributes
user = User.find(1)
user.to_json
# → {"id":1,"name":"Alice","email":"alice@example.com",
#    "password_digest":"$2a$12$...",
#    "reset_password_token":"abc...",
#    "stripe_customer_id":"cus_xxx",
#    "admin":false}

# SECURITY RISK: leaks password_digest, tokens, internal IDs

# Rails' built-in protection:
class User < ApplicationRecord
  def as_json(options = {})
    super(options.merge(
      only: [:id, :name, :email, :created_at],
      except: [:password_digest, :reset_password_token, :stripe_customer_id]
    ))
  end
end

# But this is fragile — adding a column requires remembering to update the list
# Better: always use an explicit serializer/blueprint that lists allowed fields

# The safest approach: never use `render json: @model` without serializer in an API
render json: UserBlueprint.render(@user)  # explicit whitelist
```

---

**Trap 5: "JWT signature uses HS256 (symmetric). What's the problem with this in microservices?"**

HS256 uses a shared secret — every service that needs to verify JWTs must have the secret key. If any service is compromised, all services are compromised.

```ruby
# Problem with HS256 in microservices:
# Service A generates tokens with SECRET="mysecret"
# Service B needs to verify → must also have SECRET="mysecret"
# 10 services = 10 copies of the same secret = 10 attack surfaces

# Solution: RS256 (asymmetric - RSA)
# Auth service: generates tokens with PRIVATE KEY (kept secret)
# All other services: verify with PUBLIC KEY (safe to distribute)

# Generate RSA key pair:
# openssl genrsa -out private.pem 2048
# openssl rsa -in private.pem -pubout -out public.pem

# Auth service:
private_key = OpenSSL::PKey::RSA.new(File.read('private.pem'))
JWT.encode(payload, private_key, 'RS256')

# Other services (only need public key):
public_key = OpenSSL::PKey::RSA.new(File.read('public.pem'))
JWT.decode(token, public_key, true, { algorithm: 'RS256' })

# ES256 (ECDSA) is even better — smaller signatures, faster verification
```

---

**Trap 6: "You added rate limiting per IP. Why might this cause problems for legitimate users?"**

```ruby
# Problem: Multiple users behind the same NAT/proxy share an IP
# Corporate office: 500 employees → all outbound requests come from one IP
# If rate limit is 100/minute per IP → office gets rate-limited after 100 requests
# Legitimate users blocked

# Problem 2: Load balancers and CDNs
# Your app sees the LB's IP, not the user's IP
# All traffic appears to come from one IP → rate limit hits immediately

# Fix: Rate limit by user/token, not IP
throttle('api/authenticated', limit: 100, period: 1.minute) do |req|
  # Use auth token as rate limit key when authenticated
  req.env['HTTP_AUTHORIZATION']&.split(' ')&.last
end

# Fall back to IP for unauthenticated requests:
throttle('api/unauthenticated', limit: 20, period: 1.minute) do |req|
  req.ip unless req.env['HTTP_AUTHORIZATION'].present?
end

# For proxy/LB: use X-Forwarded-For correctly
# Rails' ActionDispatch::RemoteIp middleware handles this
# Rack::Attack can be configured to use the real IP:
Rack::Attack.blocklist_ip("1.2.3.4")  # Block specific IP from X-Forwarded-For
```

---

**Trap 7: "You return `{ errors: record.errors.full_messages }`. Why is this poor API design?"**

```ruby
# Problem: full_messages are human-readable and tied to locale
# Not machine-readable → client can't reliably parse which field has which error
# "Name can't be blank" vs "Email is already taken" — client doesn't know which field

# Client needs to display errors next to the right form field
# But they have to string-match to guess which field → fragile

# Poor API error response:
{ "errors": ["Name can't be blank", "Email is already taken"] }
# Client: how do I know "Email is already taken" is about the email field?

# Better: structured errors
{ "errors": {
    "name": ["can't be blank"],
    "email": ["is already taken", "must be a valid email"]
  }
}

# Even better: include error codes for i18n on client side
{ "errors": [
    { "field": "name", "code": "blank", "message": "can't be blank" },
    { "field": "email", "code": "taken", "message": "is already taken" }
  ]
}

# Implementation:
record.errors.each_with_object({}) do |error, hash|
  (hash[error.attribute] ||= []) << { code: error.type, message: error.message }
end
```

---

**Trap 8: "You namespace your API at `/api/v1/`. A client sends a request to `/api/v1/users` with `Content-Type: application/json` but no `Accept` header. What format does Rails respond with?"**

Without an `Accept` header, the client sends no preference. Rails defaults to the format of the URL or `text/html` if nothing matches.

```ruby
# Route default matters:
namespace :api, defaults: { format: :json } do
  namespace :v1 do
    resources :users
  end
end
# With defaults: { format: :json }, Rails uses JSON format
# Without defaults, Rails falls through to text/html (or whatever is first in respond_to)

# In your controller:
def index
  respond_to do |format|
    format.json { render json: @users }
    format.html { render :index }  # This runs if Accept: text/html or no Accept header
  end
end

# In an API-only app (ActionController::API), no HTML rendering — always responds with JSON
# Because there's no format.html block, Rails returns JSON by default

# Best practice: set defaults: { format: :json } in routes
# AND: use ActionController::API for API controllers (removes HTML rendering entirely)
```

---

**Trap 9: "You use `before_action :authenticate!` in your API base controller, but your health check endpoint `GET /health` is in the same controller hierarchy. What happens?"**

```ruby
# Problem: health check endpoint requires authentication
# Load balancer or monitoring service doesn't have a token → 401 Unauthorized
# Load balancer marks the server as unhealthy → removed from pool!

# Common scenario:
class Api::BaseController < ActionController::API
  before_action :authenticate!
end

class Api::HealthController < Api::BaseController
  def check
    render json: { status: "ok" }  # Never reached — authenticate! redirects first!
  end
end

# Fix 1: Skip authenticate for health check
class Api::HealthController < Api::BaseController
  skip_before_action :authenticate!

  def check
    render json: { status: "ok", database: db_connected?, redis: redis_connected? }
  end
end

# Fix 2: Put health check in a completely separate, non-API controller
class HealthController < ActionController::API
  def check
    render json: { status: "ok" }
  end
end
# routes.rb: get '/health', to: 'health#check' (outside :api namespace)

# Fix 3: Simple Rack endpoint (no controller overhead)
get '/up', to: proc { [200, {}, ['OK']] }
```

---

**Trap 10: "You return HTTP 200 for all API responses and put the actual status in the JSON body: `{ success: false, status: 422 }`. What's wrong with this?"**

```ruby
# Common pattern in bad APIs:
render json: { success: false, status: 422, message: "Validation failed" }, status: 200
# HTTP STATUS IS 200 — but the payload says 422?!

# Problems:
# 1. HTTP caching breaks — proxies cache 200 responses; your "error" gets cached
# 2. Monitoring tools (APM, log aggregators) report no errors — everything is 200!
# 3. Clients can't use standard HTTP error handling (try/catch on 4xx/5xx)
# 4. Browser dev tools show no red requests — hard to debug
# 5. Load balancers mark server as healthy even if all requests are failing
# 6. Rate limiting middleware counts successful responses only

# Correct: HTTP status code IS the status
render json: { error: "Validation failed", details: @user.errors }, status: 422
# HTTP 422 → proxies don't cache, monitoring detects errors, clients handle correctly

# The ONLY legitimate "always 200" patterns:
# - Batch operations where some succeed and some fail (use 207 Multi-Status)
# - Queued/async operations that are accepted but not yet processed (202 Accepted)
```

---

**Trap 11: "You mentioned `stale?` for HTTP caching. Does ETag caching work when the response includes the current user's data?"**

No — ETag-based caching with `public: true` caches the same response for all users. If the response includes user-specific data, you must either:

```ruby
# WRONG: public caching with user data
if stale?(@product, public: true)
  render json: { product: @product, can_edit: current_user.can?(:edit, @product) }
end
# → Cache is shared between users → User A sees User B's permissions!

# CORRECT: private caching (per-user):
if stale?(etag: [@product, current_user.id],  # User-specific ETag
          last_modified: @product.updated_at,
          public: false)  # Private cache (browser only, not CDN)
  render json: { product: @product, can_edit: current_user.can?(:edit, @product) }
end

# CORRECT: Separate user-specific and public data:
# Public endpoint (no user data):
# GET /api/v1/products/1 → cached publicly
# User-specific endpoint:
# GET /api/v1/users/me/products/1 → private cache only
```

---

**Trap 12: "You add JSON:API format (using jsonapi-serializer). A client from your team says 'this is too complex.' What are the tradeoffs?"**

```ruby
# JSON:API output:
{
  "data": {
    "id": "1",
    "type": "user",
    "attributes": {
      "name": "Alice",
      "email": "alice@example.com"
    },
    "relationships": {
      "posts": {
        "data": [{ "id": "1", "type": "post" }, { "id": "2", "type": "post" }]
      }
    }
  },
  "included": [
    { "id": "1", "type": "post", "attributes": { "title": "Hello" } }
  ]
}

# vs Simple JSON:
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com",
  "posts": [{ "id": 1, "title": "Hello" }]
}

# JSON:API benefits:
# - Standardized → client libraries (jsonapi-resources, etc.)
# - Normalized data (no duplication in included)
# - Sparse fieldsets standard: ?fields[user]=name,email
# - Relationship linking standard

# JSON:API costs:
# - More verbose → larger payloads
# - More complex client parsing
# - Less intuitive for humans reading the API

# Decision: use JSON:API for complex B2B APIs with many relationships
# Use simple JSON for consumer apps, mobile apps where simplicity matters
```

