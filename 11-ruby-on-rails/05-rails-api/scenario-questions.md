# Chapter 05 — Rails API: Scenario Questions

---

**S1. A mobile app reports that after 15 minutes of inactivity, they get 401 errors. They complain about having to log in again. Design a token refresh flow.**

```ruby
# Solution: short-lived access tokens (15 min) + long-lived refresh tokens (30 days)
# Mobile stores both; uses refresh token to silently get new access tokens

# When access token expires, mobile gets 401 with code "token_expired":
# { "error": { "code": "token_expired", "message": "Token has expired" } }
# Mobile client detects this code → calls POST /api/v1/auth/refresh with refresh token
# Gets new access + refresh token → retries original request

# Rails implementation:
class Api::V1::Auth::RefreshController < Api::V1::BaseController
  skip_before_action :authenticate!

  def create
    refresh_token = params.require(:refresh_token)
    result = TokenRefreshService.new(refresh_token).call

    render json: {
      access_token: result.access_token,
      refresh_token: result.refresh_token,
      expires_in: 900  # 15 minutes in seconds
    }, status: :created
  rescue TokenRefreshService::InvalidToken => e
    render json: { error: { code: "invalid_refresh_token", message: e.message } },
           status: :unauthorized
  rescue TokenRefreshService::ExpiredToken
    render json: { error: { code: "refresh_token_expired",
                             message: "Please log in again" } },
           status: :unauthorized
  end
end
```

---

**S2. A third-party developer reports your API returns inconsistent error formats — sometimes `{ "error": "message" }`, sometimes `{ "errors": [...] }`, sometimes just an HTTP status code with empty body. Standardize the error format.**

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  include Api::ErrorHandling
end

# app/controllers/concerns/api/error_handling.rb
module Api::ErrorHandling
  extend ActiveSupport::Concern

  included do
    rescue_from StandardError,                       with: :handle_server_error
    rescue_from ActiveRecord::RecordNotFound,        with: :handle_not_found
    rescue_from ActiveRecord::RecordInvalid,         with: :handle_validation_error
    rescue_from ActionController::ParameterMissing,  with: :handle_bad_request
    rescue_from ActionController::InvalidAuthenticityToken, with: :handle_unauthorized
  end

  private

  def api_error(status:, code:, message:, details: nil)
    payload = {
      error: {
        code: code.to_s,
        message: message,
        request_id: request.request_id,
        timestamp: Time.current.iso8601
      }
    }
    payload[:error][:details] = details if details.present?
    render json: payload, status: status
  end

  def handle_not_found(e)
    api_error(status: :not_found, code: :not_found,
              message: "The requested resource was not found")
  end

  def handle_validation_error(e)
    api_error(
      status: :unprocessable_entity,
      code: :validation_failed,
      message: "Validation failed",
      details: e.record.errors.group_by_attribute
                         .transform_values { |errs| errs.map(&:message) }
    )
  end

  def handle_bad_request(e)
    api_error(status: :bad_request, code: :bad_request,
              message: "Missing required parameter: #{e.param}")
  end

  def handle_unauthorized(e)
    api_error(status: :unauthorized, code: :unauthorized,
              message: "Authentication required")
  end

  def handle_server_error(e)
    Rails.logger.error "Unhandled error: #{e.class}: #{e.message}\n#{e.backtrace.first(10).join("\n")}"
    Sentry.capture_exception(e) if defined?(Sentry)
    api_error(status: :internal_server_error, code: :server_error,
              message: Rails.env.production? ? "An unexpected error occurred" : e.message)
  end
end
```

---

**S3. You need to build an API that returns different amounts of data depending on which client calls it (mobile gets minimal data, web gets full data, admin gets everything). How do you design this?**

```ruby
# Blueprinter view-based approach:
class UserBlueprint < Blueprinter::Base
  identifier :id

  # Mobile: minimal data (bandwidth sensitive)
  view :mobile do
    fields :name, :avatar_url
    field :display_name do |user| user.first_name end
  end

  # Standard web
  view :default do
    fields :name, :email, :bio, :avatar_url, :created_at
    field :posts_count do |user| user.posts.published.count end
  end

  # Admin: all fields including sensitive
  view :admin do
    fields :name, :email, :bio, :avatar_url, :created_at
    fields :phone, :last_sign_in_at, :sign_in_count, :failed_attempts
    field :internal_notes do |user| user.admin_notes end
    association :devices, blueprint: DeviceBlueprint
  end
end

# Controller determines view based on client type:
class Api::V1::UsersController < Api::V1::BaseController
  def show
    @user = User.find(params[:id])
    view = serializer_view_for_client
    render json: UserBlueprint.render(@user, view: view)
  end

  private

  def serializer_view_for_client
    return :admin if current_user.admin? && params[:expand] == "admin"
    return :mobile if request.headers['X-Client-Type'] == 'mobile'
    :default
  end
end
```

---

**S4. Your API suddenly gets 10x traffic from a new partnership. Your database becomes a bottleneck. What API-level caching strategies do you implement?**

```ruby
# Strategy 1: HTTP caching with ETags (zero DB cost for unchanged resources)
class Api::V1::ProductsController < Api::V1::BaseController
  def show
    @product = Product.find(params[:id])
    return render json: @product if stale?(@product, public: false)
    # 304 returned if client has current version → no DB read needed for response body
  end

  def index
    fresh_when(
      etag: Product.published.maximum(:updated_at)&.to_i || 0,
      last_modified: Product.published.maximum(:updated_at) || Time.current,
      public: true
    )
    @products = Product.published.order(created_at: :desc).limit(20)
    render json: @products
  end
end

# Strategy 2: Rails.cache for expensive aggregations
def stats
  stats = Rails.cache.fetch("api_stats_#{params[:period]}", expires_in: 5.minutes) do
    {
      total_users: User.count,
      active_users: User.where(last_active_at: 1.hour.ago..).count,
      total_orders: Order.where(created_at: period_range).count,
      revenue: Order.where(created_at: period_range).sum(:total).to_f
    }
  end
  render json: stats
end

# Strategy 3: Cache serialized API responses
class Api::V1::ProductsController < Api::V1::BaseController
  def index
    cache_key = "api_v1_products_#{params.to_unsafe_h.sort.to_h.to_json}"
    cached = Rails.cache.fetch(cache_key, expires_in: 2.minutes) do
      products = Product.published.includes(:category).order(created_at: :desc).limit(50)
      ProductBlueprint.render(products)
    end
    render json: cached
  end
end
```

---

**S5. A client reports they're getting CORS errors when making preflight OPTIONS requests to your API. Diagnose and fix.**

```ruby
# Diagnosis: preflight OPTIONS request fails before reaching your routes
# Browser sends: OPTIONS /api/v1/users
# Headers: Origin, Access-Control-Request-Method, Access-Control-Request-Headers

# Fix 1: Ensure Rack::Cors runs before routing
# rack-cors must be FIRST in middleware stack:
config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'https://clientapp.com', 'http://localhost:3000'
    resource '/api/*',
             headers: :any,
             methods: [:get, :post, :put, :patch, :delete, :options, :head],
             credentials: true,
             max_age: 7200
  end
end

# Fix 2: Ensure OPTIONS routes are handled
# With rack-cors, this is automatic — it intercepts OPTIONS requests
# WITHOUT rack-cors, you need explicit route:
match '/api/*path', to: proc { [200, {}, ['']] }, via: :options

# Fix 3: Credentials require explicit origin (no wildcard):
# WRONG:
origins '*'  # With credentials: true → CORS error!
# RIGHT:
origins 'https://clientapp.com'  # Exact origins when credentials: true

# Fix 4: Custom headers must be explicitly allowed:
resource '/api/*',
         headers: :any,   # or specific: ['Authorization', 'Content-Type', 'X-API-Key']
         expose: ['Authorization', 'X-Request-Id']  # Headers JS can read from response
```

---

**S6. Build a webhook delivery system that retries failed deliveries with exponential backoff.**

```ruby
# app/models/webhook_delivery.rb
class WebhookDelivery < ApplicationRecord
  belongs_to :webhook_endpoint
  belongs_to :event

  enum :status, { pending: 0, delivered: 1, failed: 2, retrying: 3 }

  MAX_RETRIES = 5
  RETRY_DELAYS = [1.minute, 5.minutes, 30.minutes, 2.hours, 12.hours].freeze

  def retry_at
    delay = RETRY_DELAYS[attempts] || RETRY_DELAYS.last
    created_at + delay
  end
end

# app/jobs/deliver_webhook_job.rb
class DeliverWebhookJob < ApplicationJob
  queue_as :webhooks

  def perform(delivery_id)
    delivery = WebhookDelivery.find(delivery_id)
    endpoint = delivery.webhook_endpoint

    response = HTTP.timeout(5)
                   .headers(
                     "Content-Type" => "application/json",
                     "X-Webhook-ID" => delivery.id.to_s,
                     "X-Webhook-Signature" => compute_signature(delivery),
                     "User-Agent" => "MyApp-Webhook/1.0"
                   )
                   .post(endpoint.url, json: delivery.event.payload)

    if response.status.success?
      delivery.update!(
        status: :delivered,
        response_code: response.status.code,
        delivered_at: Time.current
      )
    else
      handle_failure(delivery, response.status.code, response.body.to_s)
    end
  rescue HTTP::TimeoutError, HTTP::ConnectionError => e
    handle_failure(delivery, nil, e.message)
  end

  private

  def handle_failure(delivery, code, body)
    new_attempts = delivery.attempts + 1
    delivery.update!(
      attempts: new_attempts,
      last_error: "#{code}: #{body.truncate(500)}",
      response_code: code
    )

    if new_attempts < WebhookDelivery::MAX_RETRIES
      delivery.update!(status: :retrying)
      DeliverWebhookJob.set(wait: delivery.retry_at - Time.current)
                       .perform_later(delivery.id)
    else
      delivery.update!(status: :failed)
      WebhookFailureNotifier.new(delivery).notify
    end
  end

  def compute_signature(delivery)
    secret = delivery.webhook_endpoint.secret
    payload = delivery.event.payload.to_json
    "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, payload)}"
  end
end
```

---

**S7. You need to add API versioning to an existing app that currently has no versioning. The existing controllers are at `app/controllers/api/`. How do you migrate without breaking existing clients?**

```ruby
# Strategy: "API V1 is what we have now" — promote existing to V1 without changing URLs

# Step 1: Move controllers to api/v1/ namespace
# app/controllers/api/v1/users_controller.rb → promoted from api/users_controller.rb

# Step 2: Keep old routes working (backward compatibility):
# config/routes.rb
namespace :api do
  # Old routes (no version) → forward to V1 controllers:
  resources :users, controller: 'v1/users', only: [:index, :show, :create, :update, :destroy]

  # New versioned routes:
  namespace :v1 do
    resources :users
  end

  # V2 with new features:
  namespace :v2 do
    resources :users  # Different controller: api/v2/users_controller.rb
  end
end

# Step 3: Base controllers:
class Api::BaseController < ActionController::API
  # Shared auth, error handling
end

class Api::V1::BaseController < Api::BaseController
  # V1-specific behavior
end

class Api::V2::BaseController < Api::BaseController
  # V2-specific behavior (e.g., different pagination format)
end

# Step 4: V2 inherits from V1 with overrides (DRY):
class Api::V2::UsersController < Api::V1::UsersController
  # Override only what changed in V2:
  def show
    @user = User.find(params[:id])
    render json: UserV2Blueprint.render(@user)  # Different format
  end
end
```

---

**S8. Implement a batch API endpoint that allows clients to submit multiple operations in a single HTTP request.**

```ruby
# POST /api/v1/batch
# Body: { "operations": [{ "method": "GET", "path": "/api/v1/users/1" }, ...] }

class Api::V1::BatchController < Api::V1::BaseController
  MAX_OPERATIONS = 20

  def create
    operations = params.require(:operations)

    if operations.size > MAX_OPERATIONS
      return render json: { error: "Too many operations (max #{MAX_OPERATIONS})" },
                    status: :bad_request
    end

    results = operations.map.with_index do |op, index|
      execute_operation(op, index)
    end

    render json: { results: results }
  end

  private

  def execute_operation(op, index)
    method = op[:method]&.upcase
    path   = op[:path]
    body   = op[:body]

    unless method.in?(%w[GET POST PATCH PUT DELETE])
      return { index: index, status: 400, body: { error: "Invalid method" } }
    end

    # Build a sub-request using Rack::MockRequest
    env = Rack::MockRequest.env_for(
      path,
      method: method,
      input: body.to_json,
      "CONTENT_TYPE" => "application/json",
      "HTTP_AUTHORIZATION" => request.headers["Authorization"]
    )

    status, headers, response_body = Rails.application.call(env)
    body_str = response_body.body rescue response_body.join

    {
      index: index,
      status: status,
      headers: headers.slice("Content-Type"),
      body: JSON.parse(body_str)
    }
  rescue JSON::ParserError => e
    { index: index, status: 500, body: { error: "Invalid response from sub-operation" } }
  end
end
```

---

**S9. Your API serialization is creating N+1 queries when returning users with their posts. Debug and fix it.**

```ruby
# Bug in controller:
def index
  @users = User.all  # Missing includes!
  render json: UserBlueprint.render(@users, view: :with_posts)
end

# Bug in blueprint:
class UserBlueprint < Blueprinter::Base
  view :with_posts do
    association :posts, blueprint: PostBlueprint
    # For each user: SELECT * FROM posts WHERE user_id = ? (N queries!)
  end
end

# Fix 1: Add includes in controller
def index
  @users = User.includes(:posts, :profile).all
  render json: UserBlueprint.render(@users, view: :with_posts)
end

# Fix 2: For jsonapi-serializer:
class UserSerializer
  include JSONAPI::Serializer
  has_many :posts
end
# Controller:
@users = User.includes(:posts).all
render json: UserSerializer.new(@users, include: [:posts]).serializable_hash
# jsonapi-serializer automatically uses pre-loaded associations if available

# Fix 3: Detect N+1 during testing:
# gem 'bullet' in development
config.after_initialize do
  Bullet.enable = true
  Bullet.raise = true  # Raise in tests, not just log
end

# RSpec:
describe "GET /api/v1/users" do
  it "does not cause N+1 queries" do
    create_list(:user, 5, :with_posts)
    expect {
      get '/api/v1/users', headers: auth_headers
    }.to make_database_queries(count: 1..3)  # Expect at most 3 queries
  end
end
```

---

**S10. Design a GraphQL-like sparse fieldsets feature for your REST API.**

```ruby
# Client can request only specific fields:
# GET /api/v1/users/1?fields=id,name,email
# GET /api/v1/users?fields=id,name&include=posts&post_fields=id,title

class Api::V1::UsersController < Api::V1::BaseController
  def show
    @user = User.find(params[:id])
    render json: UserBlueprint.render(@user,
                                      view: :default,
                                      options: { fields: requested_fields })
  end

  def index
    includes = []
    includes << :posts if requested_include?(:posts)
    includes << :profile if requested_include?(:profile)

    @users = User.includes(*includes).all
    render json: UserBlueprint.render(@users,
                                      view: :default,
                                      options: { fields: requested_fields })
  end

  private

  def requested_fields
    return nil unless params[:fields].present?
    params[:fields].split(",").map(&:strip).map(&:to_sym)
  end

  def requested_include?(resource)
    params[:include]&.split(",")&.include?(resource.to_s)
  end
end

# Blueprint that supports sparse fields:
class UserBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :email, :bio, :created_at

  view :default do
    # Fields will be filtered by options[:fields] if present
  end

  association :posts, blueprint: PostBlueprint, if: ->(_field_name, _user, options) {
    options[:fields].nil? || options[:fields].include?(:posts)
  }
end

# Validate fields to prevent column enumeration:
ALLOWED_FIELDS = {
  users: %w[id name email bio avatar_url created_at],
  posts: %w[id title published_at status]
}.freeze

def requested_fields
  return nil unless params[:fields].present?
  params[:fields].split(",").map(&:strip) & ALLOWED_FIELDS[:users]  # Intersection — whitelist
end
```

---

**S11. A client integrates with your API and says they need a "meta" field in every response with the API version and response time. Implement this as middleware.**

```ruby
# app/middleware/api_meta_middleware.rb
class ApiMetaMiddleware
  API_VERSION = "1.0.3"

  def initialize(app)
    @app = app
  end

  def call(env)
    return @app.call(env) unless api_request?(env)

    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round(2)

    if json_response?(headers)
      inject_meta(status, headers, body, duration_ms)
    else
      [status, headers, body]
    end
  end

  private

  def api_request?(env)
    env['PATH_INFO'].start_with?('/api/')
  end

  def json_response?(headers)
    headers['Content-Type']&.include?('application/json')
  end

  def inject_meta(status, headers, body, duration_ms)
    original_body = body.body rescue body.join rescue ""
    parsed = JSON.parse(original_body)

    enriched = parsed.merge(
      "_meta" => {
        "api_version" => API_VERSION,
        "response_time_ms" => duration_ms,
        "timestamp" => Time.current.iso8601
      }
    )

    new_body = enriched.to_json
    headers['Content-Length'] = new_body.bytesize.to_s
    [status, headers, [new_body]]
  rescue JSON::ParserError
    [status, headers, body]  # Non-JSON response — leave unchanged
  end
end

# config/application.rb:
config.middleware.use ApiMetaMiddleware
```

---

**S12. How do you test a Rails API comprehensively? Show a complete request spec.**

```ruby
# spec/requests/api/v1/users_spec.rb
require 'rails_helper'

RSpec.describe "GET /api/v1/users", type: :request do
  let(:user) { create(:user) }
  let(:admin) { create(:user, :admin) }
  let(:token) { JwtService.encode(user_id: user.id) }
  let(:admin_token) { JwtService.encode(user_id: admin.id) }
  let(:auth_headers) { { "Authorization" => "Bearer #{token}" } }
  let(:admin_headers) { { "Authorization" => "Bearer #{admin_token}" } }

  describe "GET /api/v1/users" do
    context "when authenticated" do
      before { create_list(:user, 5) }

      it "returns 200 with paginated users" do
        get "/api/v1/users", headers: auth_headers

        expect(response).to have_http_status(:ok)
        json = response.parsed_body
        expect(json["data"]).to be_an(Array)
        expect(json["meta"]["total"]).to eq(6)  # 5 + current user
      end

      it "paginates correctly" do
        get "/api/v1/users?page=1&per_page=3", headers: auth_headers
        expect(response.parsed_body["data"].length).to eq(3)
      end

      it "does not cause N+1 queries" do
        create_list(:user, 10)
        expect {
          get "/api/v1/users", headers: auth_headers
        }.to make_database_queries(count: 1..4)
      end
    end

    context "when unauthenticated" do
      it "returns 401" do
        get "/api/v1/users"
        expect(response).to have_http_status(:unauthorized)
        expect(response.parsed_body["error"]["code"]).to eq("unauthorized")
      end
    end
  end

  describe "GET /api/v1/users/:id" do
    it "returns the user" do
      get "/api/v1/users/#{user.id}", headers: auth_headers

      expect(response).to have_http_status(:ok)
      json = response.parsed_body["data"]
      expect(json["id"]).to eq(user.id.to_s)
      expect(json["attributes"]["email"]).to eq(user.email)
      expect(json["attributes"]).not_to have_key("password_digest")
    end

    it "returns 404 for missing user" do
      get "/api/v1/users/999999", headers: auth_headers
      expect(response).to have_http_status(:not_found)
    end
  end

  describe "POST /api/v1/users" do
    let(:valid_params) do
      { user: { name: "Alice", email: "alice@example.com", password: "password123!" } }
    end

    it "creates a user" do
      expect {
        post "/api/v1/users", params: valid_params.to_json,
             headers: admin_headers.merge("Content-Type" => "application/json")
      }.to change(User, :count).by(1)

      expect(response).to have_http_status(:created)
      expect(response.parsed_body["data"]["attributes"]["email"]).to eq("alice@example.com")
    end

    it "returns validation errors" do
      post "/api/v1/users",
           params: { user: { email: "invalid" } }.to_json,
           headers: admin_headers.merge("Content-Type" => "application/json")

      expect(response).to have_http_status(:unprocessable_entity)
      errors = response.parsed_body["error"]["details"]
      expect(errors["email"]).to include("is invalid")
    end
  end
end
```

---

**S13. Implement idiomatic API response helpers that keep controllers thin.**

```ruby
# app/controllers/concerns/api/response_helpers.rb
module Api::ResponseHelpers
  extend ActiveSupport::Concern

  private

  def render_resource(resource, serializer:, status: :ok, **options)
    render json: serializer.new(resource, options).serializable_hash, status: status
  end

  def render_collection(collection, serializer:, meta: {}, **options)
    render json: {
      data: serializer.new(collection, options).serializable_hash[:data],
      meta: collection_meta(collection).merge(meta)
    }
  end

  def render_created(resource, serializer:, location: nil)
    render_resource(resource, serializer: serializer, status: :created)
  end

  def render_no_content
    head :no_content
  end

  def render_validation_errors(resource)
    render json: {
      error: {
        code: "validation_failed",
        message: "Validation failed",
        details: resource.errors.group_by_attribute.transform_values { |e| e.map(&:message) },
        request_id: request.request_id
      }
    }, status: :unprocessable_entity
  end

  def collection_meta(collection)
    return {} unless collection.respond_to?(:current_page)
    {
      page: collection.current_page,
      per_page: collection.limit_value,
      total: collection.total_count,
      total_pages: collection.total_pages
    }
  end
end

# Usage in controllers:
class Api::V1::UsersController < Api::V1::BaseController
  include Api::ResponseHelpers

  def index
    @users = User.all.page(params[:page])
    render_collection(@users, serializer: UserSerializer)
  end

  def show
    render_resource(@user, serializer: UserSerializer)
  end

  def create
    @user = User.new(user_params)
    if @user.save
      render_created(@user, serializer: UserSerializer)
    else
      render_validation_errors(@user)
    end
  end

  def destroy
    @user.destroy
    render_no_content
  end
end
```

---

**S14. Your API needs to support both JSON and MessagePack for performance-sensitive clients. How do you add MessagePack support?**

```ruby
# gem 'msgpack'

# config/initializers/mime_types.rb
Mime::Type.register "application/msgpack", :msgpack

# config/application.rb
config.middleware.insert_before ActionDispatch::Static,
  Rack::Parser,
  content_types: {
    'application/msgpack' => proc { |body| MessagePack.unpack(body) }
  }

# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  before_action :set_content_type

  def render_api(data, status: :ok)
    respond_to do |format|
      format.json    { render json: data, status: status }
      format.msgpack { render body: MessagePack.pack(data), status: status,
                              content_type: 'application/msgpack' }
    end
  end

  private

  def set_content_type
    # Parse msgpack request body
    if request.content_type == 'application/msgpack'
      raw = request.raw_post
      @parsed_msgpack = MessagePack.unpack(raw) rescue nil
    end
  end
end

# Usage:
def index
  @users = User.all
  render_api({ users: @users.as_json })
end
```

---

**S15. How do you implement API monitoring and alerting for slow endpoints?**

```ruby
# config/initializers/api_monitoring.rb
ActiveSupport::Notifications.subscribe("process_action.action_controller") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  payload = event.payload

  # Skip non-API requests
  next unless payload[:path]&.start_with?('/api/')

  duration_ms = event.duration

  # Log slow requests
  if duration_ms > 500
    Rails.logger.warn({
      event: "slow_api_request",
      path: payload[:path],
      method: payload[:method],
      controller: payload[:controller],
      action: payload[:action],
      status: payload[:status],
      duration_ms: duration_ms.round(2),
      db_runtime_ms: payload[:db_runtime]&.round(2),
      view_runtime_ms: payload[:view_runtime]&.round(2),
      request_id: payload[:request].request_id
    }.to_json)
  end

  # StatsD/Prometheus metrics:
  if defined?(Statsd)
    Statsd.timing("api.request.#{payload[:controller]}.#{payload[:action]}", duration_ms)
    Statsd.increment("api.requests", tags: ["status:#{payload[:status]}"])
  end

  # Alert on high error rates (alert if > 5 consecutive 500s):
  if payload[:status] >= 500
    Rails.cache.increment("api_500_count", 1, expires_in: 1.minute)
    count = Rails.cache.read("api_500_count").to_i
    if count >= 5
      PagerDuty.trigger("High API error rate: #{count} 500s in 1 minute")
    end
  end
end
```

