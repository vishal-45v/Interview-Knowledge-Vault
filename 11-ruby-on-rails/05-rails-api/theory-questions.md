# Chapter 05 — Rails API: Theory Questions

---

**Q1. What is API-only Rails (`rails new --api`)? What does it change?**

```bash
rails new myapi --api
# Generates a slimmer Rails application:
# - ApplicationController inherits from ActionController::API (not ::Base)
# - Removes middleware: Cookies, Sessions, Flash, BrowserSecurity
# - Removes view-related gems (sprockets, etc.)
# - No app/assets directory
# - No app/views (except mailers by default)

# config/application.rb in API mode:
config.api_only = true
```

---

**Q2. What are the serialization options in Rails? Compare ActiveModel::Serializer, jsonapi-serializer, Blueprinter, and Jbuilder.**

```ruby
# 1. Jbuilder — DSL in .json.jbuilder views
# app/views/users/show.json.jbuilder
json.id @user.id
json.name @user.name
json.posts @user.posts do |post|
  json.id post.id
  json.title post.title
end
# Pro: familiar ERB-like syntax; Con: slow for large collections, view layer

# 2. ActiveModel::Serializer (AMS)
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name, :email
  has_many :posts
  has_one  :profile
end
render json: @user  # Automatically uses UserSerializer
# Pro: convention-based; Con: N+1 issues, slow

# 3. jsonapi-serializer (formerly fast_jsonapi by Netflix)
class UserSerializer
  include JSONAPI::Serializer
  attributes :name, :email
  has_many :posts
  belongs_to :role
end
serialized = UserSerializer.new(@user, include: [:posts]).serializable_hash
render json: serialized
# Pro: fastest, JSON:API spec compliant; Con: JSON:API format (opinionated)

# 4. Blueprinter — "blueprints" as plain classes
class UserBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :email
  field :full_name do |user, _options|
    "#{user.first_name} #{user.last_name}"
  end
  view :detailed do
    fields :phone, :address
    association :posts, blueprint: PostBlueprint
  end
end
render json: UserBlueprint.render(@user, view: :detailed)
# Pro: explicit views, composable, fast; Con: less magical
```

---

**Q3. What is JWT authentication? How do you implement it in Rails?**

JWT (JSON Web Token) consists of 3 base64-encoded parts: Header.Payload.Signature

```ruby
# gem 'jwt'

# config/initializers/jwt.rb
module JwtService
  SECRET = Rails.application.credentials.jwt_secret!
  ALGORITHM = 'HS256'
  EXPIRY = 24.hours

  def self.encode(payload)
    payload = payload.merge(exp: EXPIRY.from_now.to_i, iat: Time.current.to_i)
    JWT.encode(payload, SECRET, ALGORITHM)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET, true, { algorithm: ALGORITHM })
    HashWithIndifferentAccess.new(decoded.first)
  rescue JWT::ExpiredSignature
    raise TokenExpiredError, "Token has expired"
  rescue JWT::DecodeError => e
    raise InvalidTokenError, "Invalid token: #{e.message}"
  end
end

# Sessions controller:
class Api::V1::SessionsController < Api::V1::BaseController
  skip_before_action :authenticate!

  def create
    user = User.find_by(email: params[:email]&.downcase)
    if user&.authenticate(params[:password])
      token = JwtService.encode({ user_id: user.id, email: user.email })
      render json: {
        token: token,
        user: UserSerializer.new(user).serializable_hash,
        expires_at: JwtService::EXPIRY.from_now.iso8601
      }, status: :created
    else
      render json: { error: "Invalid credentials" }, status: :unauthorized
    end
  end
end

# Base controller authentication:
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!

  private

  def authenticate!
    header = request.headers['Authorization']
    token = header&.split(' ')&.last
    raise ActionController::InvalidAuthenticityToken unless token

    payload = JwtService.decode(token)
    @current_user = User.find(payload[:user_id])
  rescue JwtService::TokenExpiredError
    render json: { error: "Token expired", code: "token_expired" }, status: :unauthorized
  rescue JwtService::InvalidTokenError, ActiveRecord::RecordNotFound
    render json: { error: "Unauthorized" }, status: :unauthorized
  end
end
```

---

**Q4. What is CORS? How do you configure it in Rails?**

```ruby
# CORS (Cross-Origin Resource Sharing): browser security mechanism
# Blocks browser AJAX requests to different origins (different domain/port/protocol)

# gem 'rack-cors'
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'https://myapp.com', 'https://www.myapp.com',
            /\Ahttps:\/\/.*\.myapp\.com\z/  # subdomain wildcard

    # In development only:
    origins 'http://localhost:3000', 'http://localhost:5173'  # Vite dev server

    resource '/api/*',
             headers: :any,
             methods: [:get, :post, :put, :patch, :delete, :options, :head],
             credentials: true,           # Allow cookies/auth headers
             max_age: 7200,               # Cache preflight for 2 hours
             expose: %w[Authorization X-Request-Id]  # Headers JS can read
  end

  # Public read-only endpoints (no credentials needed):
  allow do
    origins '*'
    resource '/api/v1/public/*', headers: :any, methods: [:get, :head]
  end
end
```

---

**Q5. What are API versioning strategies? Compare URL versioning, header versioning, and subdomain versioning.**

```ruby
# Strategy 1: URL versioning (most common, most visible)
# GET /api/v1/users  →  GET /api/v2/users
namespace :api do
  namespace :v1 do; resources :users; end
  namespace :v2 do; resources :users; end
end
# Pro: obvious, cacheable, easy to test; Con: "v" in URL is REST purists' nightmare

# Strategy 2: Accept header versioning (HTTP-native)
# Accept: application/vnd.myapp.v2+json
class ApiVersionConstraint
  def initialize(version:)
    @version = version
  end
  def matches?(request)
    request.headers['Accept'].include?("application/vnd.myapp.v#{@version}+json")
  end
end
namespace :api do
  scope constraints: ApiVersionConstraint.new(version: 2) do
    namespace :v2 do; resources :users; end
  end
  scope constraints: ApiVersionConstraint.new(version: 1) do
    namespace :v1 do; resources :users; end
  end
end
# Pro: clean URLs, RESTful; Con: hard to test in browser, less visible

# Strategy 3: Custom header
# X-API-Version: 2
# Similar to Accept header approach

# Strategy 4: Subdomain
# v1.api.example.com  vs  v2.api.example.com
constraints subdomain: 'v2' do
  namespace :api, path: '/', constraints: { subdomain: 'v2' } do
    namespace :v2 do; resources :users; end
  end
end
```

---

**Q6. How do you implement pagination in a Rails API? Compare Kaminari and Pagy.**

```ruby
# Kaminari: full-featured, model-level, generates HTML or JSON pagination
# gem 'kaminari'
class Api::V1::PostsController < Api::V1::BaseController
  def index
    @posts = Post.published
                 .order(created_at: :desc)
                 .page(params[:page])
                 .per(params[:per_page] || 20)

    render json: {
      posts: @posts.map { |p| post_serializer(p) },
      meta: {
        current_page: @posts.current_page,
        total_pages: @posts.total_pages,
        total_count: @posts.total_count,
        per_page: @posts.limit_value
      }
    }
  end
end

# Pagy: fastest, lightest, no AR monkey-patching
# gem 'pagy'
class Api::V1::PostsController < Api::V1::BaseController
  include Pagy::Backend

  def index
    @pagy, @posts = pagy(Post.published.order(created_at: :desc),
                         items: params[:per_page] || 20)
    render json: {
      posts: @posts.map { |p| post_serializer(p) },
      meta: pagy_metadata(@pagy)  # { page:, prev:, next:, last:, count: }
    }
  end
end

# Cursor-based pagination (for real-time feeds — avoids offset issues):
def index
  @cursor = params[:cursor]
  @posts = Post.published
               .where(@cursor ? "created_at < ?".to_sym : "1=1", decode_cursor(@cursor))
               .order(created_at: :desc)
               .limit(21)

  @has_more = @posts.size > 20
  @posts = @posts.first(20)
  @next_cursor = @has_more ? encode_cursor(@posts.last.created_at) : nil

  render json: { posts: @posts, next_cursor: @next_cursor, has_more: @has_more }
end
```

---

**Q7. What are HTTP status codes used in Rails APIs? List key ones.**

```ruby
# 2xx — Success
render json: @user, status: :ok           # 200
render json: @user, status: :created      # 201 (POST creating resource)
head :no_content                           # 204 (DELETE successful)
head :accepted                             # 202 (async operation accepted)

# 3xx — Redirect
redirect_to @new_resource, status: :moved_permanently  # 301
redirect_to @resource, status: :see_other              # 303 (after POST)

# 4xx — Client Error
render json: { error: "Not found" }, status: :not_found          # 404
render json: { error: "Unauthorized" }, status: :unauthorized     # 401 (no credentials)
render json: { error: "Forbidden" }, status: :forbidden           # 403 (wrong credentials)
render json: { errors: [...] }, status: :unprocessable_entity     # 422 (validation errors)
render json: { error: "Bad request" }, status: :bad_request       # 400 (malformed request)
render json: { error: "Conflict" }, status: :conflict             # 409 (duplicate resource)
render json: { error: "Too many requests" }, status: :too_many_requests  # 429 (rate limited)

# 5xx — Server Error
render json: { error: "Server error" }, status: :internal_server_error   # 500
render json: { error: "Unavailable" }, status: :service_unavailable      # 503
```

---

**Q8. What is `rescue_from` in API controllers? Show a comprehensive error handling setup.**

```ruby
class Api::BaseController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound do |e|
    render json: error_body(e.message, :not_found), status: :not_found
  end

  rescue_from ActiveRecord::RecordInvalid do |e|
    render json: {
      error: "Validation failed",
      details: e.record.errors.as_json
    }, status: :unprocessable_entity
  end

  rescue_from ActionController::ParameterMissing do |e|
    render json: error_body("Missing parameter: #{e.param}", :bad_request), status: :bad_request
  end

  rescue_from Pundit::NotAuthorizedError do
    render json: error_body("Not authorized", :forbidden), status: :forbidden
  end

  rescue_from ActiveRecord::RecordNotUnique do
    render json: error_body("Resource already exists", :conflict), status: :conflict
  end

  private

  def error_body(message, code = :error)
    {
      error: {
        message: message,
        code: code,
        request_id: request.request_id
      }
    }
  end
end
```

---

**Q9. What is content negotiation in Rails? How does `respond_to` work in API context?**

```ruby
# Content negotiation: client specifies format via Accept header
# Rails matches format to respond_to blocks

class Api::V1::ReportsController < Api::V1::BaseController
  def show
    @report = Report.find(params[:id])

    respond_to do |format|
      format.json { render json: ReportSerializer.new(@report).serializable_hash }
      format.csv  { send_data @report.to_csv, filename: "report_#{@report.id}.csv", type: :csv }
      format.pdf  { send_data @report.to_pdf, filename: "report_#{@report.id}.pdf", type: :pdf }
    end
  end
end

# Client request examples:
# Accept: application/json  → format.json
# Accept: text/csv          → format.csv
# GET /reports/1.json       → format.json (URL extension)
# GET /reports/1.csv        → format.csv

# Default format in routes:
namespace :api, defaults: { format: :json } do
  resources :reports
end
# Now all api routes default to JSON without needing .json extension
```

---

**Q10. What is rate limiting in APIs? How do you implement it with `rack-attack`?**

```ruby
# gem 'rack-attack'
# config/initializers/rack_attack.rb

class Rack::Attack
  # Allow health checks unconditionally
  safelist('allow health check') do |req|
    req.path == '/up' || req.path == '/health'
  end

  # Throttle: 300 requests per 5 minutes per IP
  throttle('general/ip', limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?('/assets')
  end

  # Throttle: API endpoints — 100 per minute per token
  throttle('api/token', limit: 100, period: 1.minute) do |req|
    if req.path.start_with?('/api/')
      req.env['HTTP_AUTHORIZATION']&.split(' ')&.last
    end
  end

  # Throttle: Login attempts — 5 per 20 seconds per email
  throttle('api/login', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/sessions' && req.post?
      req.params['email']&.downcase&.gsub(/\s+/, '')
    end
  end

  # Blocklist: explicitly blocked IPs
  blocklist('block bad actors') do |req|
    BlockedIp.exists?(ip: req.ip)  # Store bad IPs in DB
  end

  # Custom response for throttled requests:
  self.throttled_responder = lambda do |req|
    retry_after = (req.env['rack.attack.match_data'] || {})[:period]
    [
      429,
      {
        'Content-Type' => 'application/json',
        'Retry-After' => retry_after.to_s
      },
      [{ error: 'Too many requests', retry_after: retry_after }.to_json]
    ]
  end
end

# Redis as backend (recommended):
Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(url: ENV['REDIS_URL'])
```

---

**Q11. What are the differences between Jbuilder, blueprinter, and building hash manually?**

```ruby
# 1. Manual hash (simplest, most explicit)
render json: {
  id: @user.id,
  name: @user.name,
  email: @user.email,
  posts_count: @user.posts.count  # N+1 if not preloaded!
}

# 2. Jbuilder (DSL-based, readable for complex structures)
# app/views/users/show.json.jbuilder
json.id @user.id
json.name @user.name
json.email @user.email
json.created_at @user.created_at.iso8601
json.posts @user.posts do |post|
  json.id post.id
  json.title post.title
  json.published post.published?
end
# Pro: readable hierarchy; Con: slower, uses view layer

# 3. Blueprinter (explicit, view-based, reusable)
class UserBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :email
  field :created_at do |user| user.created_at.iso8601 end

  view :with_posts do
    association :posts, blueprint: PostBlueprint
  end
end
render json: UserBlueprint.render(@user, view: :with_posts)

# 4. jsonapi-serializer (JSON:API standard, fastest)
class UserSerializer
  include JSONAPI::Serializer
  attributes :name, :email
  attribute :created_at do |user| user.created_at.iso8601 end
  has_many :posts
end
render json: UserSerializer.new(@users, include: ['posts']).serializable_hash
```

---

**Q12. What is JWT token refresh? How do you implement access + refresh token pattern?**

```ruby
class JwtService
  ACCESS_SECRET  = Rails.application.credentials.jwt_access_secret!
  REFRESH_SECRET = Rails.application.credentials.jwt_refresh_secret!
  ACCESS_TTL     = 15.minutes
  REFRESH_TTL    = 30.days

  def self.generate_tokens(user)
    access_token = encode_access(user)
    refresh_token = encode_refresh(user)
    # Store refresh token hash in DB for revocation:
    user.refresh_tokens.create!(
      token_hash: Digest::SHA256.hexdigest(refresh_token),
      expires_at: REFRESH_TTL.from_now
    )
    { access_token: access_token, refresh_token: refresh_token }
  end

  def self.encode_access(user)
    JWT.encode({ user_id: user.id, type: 'access', exp: ACCESS_TTL.from_now.to_i },
               ACCESS_SECRET, 'HS256')
  end

  def self.encode_refresh(user)
    JWT.encode({ user_id: user.id, type: 'refresh', exp: REFRESH_TTL.from_now.to_i },
               REFRESH_SECRET, 'HS256')
  end

  def self.decode_refresh(token)
    payload = JWT.decode(token, REFRESH_SECRET, true, { algorithm: 'HS256' }).first
    raise InvalidTokenError unless payload['type'] == 'refresh'
    payload
  rescue JWT::ExpiredSignature, JWT::DecodeError => e
    raise InvalidTokenError, e.message
  end
end

# Refresh endpoint:
class Api::V1::TokenRefreshController < Api::V1::BaseController
  skip_before_action :authenticate!

  def create
    refresh_token = params[:refresh_token]
    payload = JwtService.decode_refresh(refresh_token)

    # Check DB for non-revoked token:
    token_record = RefreshToken.active.find_by!(
      user_id: payload['user_id'],
      token_hash: Digest::SHA256.hexdigest(refresh_token)
    )

    user = token_record.user
    # Rotate: invalidate old refresh token, issue new pair
    token_record.invalidate!
    new_tokens = JwtService.generate_tokens(user)

    render json: new_tokens, status: :created
  rescue JwtService::InvalidTokenError, ActiveRecord::RecordNotFound
    render json: { error: "Invalid or expired refresh token" }, status: :unauthorized
  end
end
```

---

**Q13. What is an API error response format standard? Show a consistent error envelope.**

```ruby
# JSONAPI error format:
{
  "errors": [
    {
      "id": "request-uuid-here",
      "status": "422",
      "code": "validation_error",
      "title": "Validation Error",
      "detail": "Name can't be blank",
      "source": { "pointer": "/data/attributes/name" }
    }
  ]
}

# Simpler custom format (widely used):
{
  "error": {
    "code": "validation_failed",
    "message": "Request validation failed",
    "details": {
      "name": ["can't be blank"],
      "email": ["is already taken", "must be a valid email"]
    },
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T12:00:00Z"
  }
}

# Implementation:
module Api
  module ErrorFormatter
    def render_error(message:, code:, status:, details: nil)
      payload = {
        error: {
          code: code,
          message: message,
          request_id: request.request_id,
          timestamp: Time.current.iso8601
        }
      }
      payload[:error][:details] = details if details
      render json: payload, status: status
    end

    def render_validation_errors(record)
      render_error(
        message: "Validation failed",
        code: :validation_failed,
        status: :unprocessable_entity,
        details: record.errors.group_by_attribute
                       .transform_values { |errs| errs.map(&:message) }
      )
    end
  end
end
```

---

**Q14. What is API authentication with API keys? How do you implement it securely?**

```ruby
# API key generation and storage:
class ApiKey < ApplicationRecord
  belongs_to :user

  before_create :generate_key_pair
  scope :active, -> { where(revoked_at: nil).where("expires_at IS NULL OR expires_at > ?", Time.current) }

  def self.authenticate(raw_key)
    # Split key into ID + secret: "key_id.secret_portion"
    key_id, secret = raw_key&.split('.', 2)
    return nil unless key_id && secret

    api_key = active.find_by(key_id: key_id)
    return nil unless api_key&.authenticate_secret(secret)

    api_key.touch(:last_used_at)
    api_key
  end

  def authenticate_secret(plain_secret)
    BCrypt::Password.new(secret_digest).is_password?(plain_secret)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  private

  def generate_key_pair
    self.key_id = SecureRandom.hex(8)  # Public identifier
    secret = SecureRandom.hex(32)       # Secret portion (shown once to user)
    self.secret_digest = BCrypt::Password.create(secret)
    self.raw_key = "#{key_id}.#{secret}"  # Virtual attr shown once to user
  end
end

# Controller authentication:
class Api::BaseController < ActionController::API
  before_action :authenticate_with_api_key!

  private

  def authenticate_with_api_key!
    raw_key = request.headers['X-API-Key'] || bearer_token
    @api_key = ApiKey.authenticate(raw_key)
    @current_user = @api_key&.user

    unless @current_user
      render json: { error: "Invalid or missing API key" }, status: :unauthorized
    end
  end

  def bearer_token
    request.headers['Authorization']&.delete_prefix('Bearer ')
  end
end
```

---

**Q15. What is API documentation? How do you auto-generate it from Rails?**

```ruby
# Option 1: rswag (Swagger/OpenAPI from RSpec tests)
# gem 'rswag'
# spec/swagger_helper.rb + spec/requests/api/v1/users_spec.rb

RSpec.describe 'Users API' do
  path '/api/v1/users/{id}' do
    get 'Retrieves a user' do
      tags 'Users'
      produces 'application/json'
      parameter name: :id, in: :path, type: :integer, required: true
      parameter name: :Authorization, in: :header, type: :string, required: true

      response '200', 'user found' do
        schema type: :object,
               properties: {
                 id:    { type: :integer },
                 name:  { type: :string },
                 email: { type: :string }
               }
        let(:id) { create(:user).id }
        run_test!
      end

      response '404', 'user not found' do
        let(:id) { 0 }
        run_test!
      end
    end
  end
end

# Option 2: OpenAPI YAML manually:
# config/api_doc.yaml or docs/openapi.yaml
# Served by: GET /api-docs (using rswag-ui)

# Option 3: apipie gem (annotation-based):
api :GET, '/users/:id', 'Returns a user'
param :id, Integer, required: true, desc: 'User ID'
returns code: 200, desc: 'A user object' do
  property :id, Integer
  property :name, String
end
def show
  @user = User.find(params[:id])
  render json: @user
end
```

---

**Q16. How do you handle API errors from external services gracefully?**

```ruby
class ExternalApiService
  class ServiceUnavailableError < StandardError; end
  class TimeoutError < StandardError; end
  class RateLimitError < StandardError; end

  def self.fetch_user(external_id)
    response = Faraday.new(url: "https://api.external.com").get("/users/#{external_id}") do |req|
      req.options.timeout = 5
      req.headers['Authorization'] = "Bearer #{Rails.application.credentials.external_api_key!}"
    end

    case response.status
    when 200
      JSON.parse(response.body)
    when 429
      raise RateLimitError, "Rate limited. Retry-After: #{response.headers['Retry-After']}"
    when 503, 502, 504
      raise ServiceUnavailableError, "External API unavailable (#{response.status})"
    else
      raise "Unexpected status: #{response.status}"
    end
  rescue Faraday::TimeoutError
    raise TimeoutError, "External API timed out after 5s"
  rescue Faraday::ConnectionFailed => e
    raise ServiceUnavailableError, "Cannot connect to external API: #{e.message}"
  end
end

# In controller:
rescue_from ExternalApiService::ServiceUnavailableError do |e|
  render json: { error: "External service unavailable", code: "external_unavailable" },
         status: :service_unavailable
end

rescue_from ExternalApiService::RateLimitError do |e|
  render json: { error: "Rate limit exceeded on external service" },
         status: :service_unavailable
end
```

---

**Q17. What is the difference between `render json:` vs Jbuilder vs serializer for performance?**

```ruby
# Benchmark order (fastest to slowest for 1000 objects):
# 1. Manual hash + to_json:         ~5ms
# 2. Blueprinter:                   ~8ms
# 3. jsonapi-serializer:            ~10ms
# 4. ActiveModel::Serializer (AMS): ~50ms (loads each serializer individually)
# 5. Jbuilder:                      ~60ms (ERB overhead)

# Memory allocation also matters — AMS and Jbuilder create more Ruby objects

# For high-traffic APIs, consider:
class Api::V1::UsersController < Api::V1::BaseController
  def index
    @users = User.select(:id, :name, :email).limit(100)  # Restrict columns

    # Option A: Manual (fastest, most verbose):
    render json: @users.map { |u| { id: u.id, name: u.name, email: u.email } }

    # Option B: Blueprinter (fast + maintainable):
    render json: UserBlueprint.render_as_hash(@users)

    # Option C: Oj gem for faster JSON serialization:
    render json: Oj.dump(@users.as_json, mode: :compat)
  end
end
```

---

**Q18. What is an ETag and how does Rails use it for API caching?**

```ruby
# ETag: response header that clients use for conditional requests
# Client: "I have version abc123 of this resource. Give me the new version only if it changed."
# Server: "It hasn't changed → 304 Not Modified (no body)"

class Api::V1::ProductsController < Api::V1::BaseController
  def show
    @product = Product.find(params[:id])

    # Option 1: Rails automatic (uses etag + last_modified):
    if stale?(@product)
      render json: @product
    end
    # If @product.updated_at matches Last-Modified header → 304 returned, no render

    # Option 2: Custom ETag:
    if stale?(etag: "#{@product.cache_key}-#{current_user.id}",
              last_modified: @product.updated_at)
      render json: ProductSerializer.new(@product).serializable_hash
    end

    # Option 3: Collection ETag:
    if stale?(last_modified: Product.maximum(:updated_at), public: true)
      @products = Product.all
      render json: @products
    end
  end
end
```

---

**Q19. How do you implement search/filtering in an API endpoint?**

```ruby
class Api::V1::ProductsController < Api::V1::BaseController
  def index
    @products = Product.all
    @products = apply_filters(@products)
    @products = apply_sort(@products)
    @products = apply_pagination(@products)
    render json: @products
  end

  private

  def apply_filters(scope)
    # Category filter
    scope = scope.where(category_id: params[:category_id]) if params[:category_id].present?

    # Price range
    scope = scope.where("price >= ?", params[:min_price]) if params[:min_price].present?
    scope = scope.where("price <= ?", params[:max_price]) if params[:max_price].present?

    # Status filter (validate against enum)
    if params[:status].present?
      status = params[:status].presence_in(Product.statuses.keys)
      scope = scope.where(status: status) if status
    end

    # Text search (avoid SQL injection with parameterized query)
    if params[:q].present?
      query = "%#{ActiveRecord::Base.sanitize_sql_like(params[:q])}%"
      scope = scope.where("name ILIKE ? OR description ILIKE ?", query, query)
    end

    # Date filter
    scope = scope.where(created_at: params[:since]..) if params[:since].present?

    scope
  end

  def apply_sort(scope)
    allowed_sorts = %w[name price created_at updated_at]
    sort_col = params[:sort]&.presence_in(allowed_sorts) || "created_at"
    sort_dir = params[:order] == "asc" ? :asc : :desc
    scope.order(sort_col => sort_dir)
  end

  def apply_pagination(scope)
    page = [params[:page].to_i, 1].max
    per  = [[params[:per_page].to_i, 1].max, 100].min
    scope.offset((page - 1) * per).limit(per)
  end
end
```

---

**Q20. What are the best practices for Rails API design in terms of response structure?**

```ruby
# Consistent envelope:
{
  "data": { ... },            # For single resources
  "data": [ ... ],            # For collections
  "meta": {                   # Pagination, counts, etc.
    "total": 150,
    "page": 2,
    "per_page": 20,
    "total_pages": 8
  },
  "links": {                  # Pagination links (HATEOAS-lite)
    "self": "/api/v1/users?page=2",
    "prev": "/api/v1/users?page=1",
    "next": "/api/v1/users?page=3"
  }
}

# Error envelope (always the same shape, regardless of error type):
{
  "error": {
    "code": "validation_failed",
    "message": "Human-readable message",
    "details": { ... },        # Optional field-level errors
    "request_id": "req_uuid"   # For debugging
  }
}

# Versioning in response:
{
  "api_version": "1.0",
  "data": { ... }
}

# Never mix success and error shapes:
# WRONG: { "success": true, "data": { ... } } vs { "success": false, "error": "..." }
# RIGHT: HTTP status codes convey success/failure, body is consistent

# Include resource IDs and type info:
{
  "id": "42",           # Always string (supports UUIDs, numeric)
  "type": "user",       # Resource type (useful for client-side data normalization)
  "attributes": { ... },
  "relationships": { ... }
}
```

