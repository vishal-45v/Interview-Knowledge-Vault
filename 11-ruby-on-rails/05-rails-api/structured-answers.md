# Chapter 05 — Rails API: Structured Answers

---

## Answer 1: Complete JWT Authentication System

**Question:** Build a complete JWT authentication system with access tokens, refresh tokens, and revocation.

```ruby
# config/credentials.yml.enc (edit with: rails credentials:edit)
# jwt_access_secret: very_long_random_string_1
# jwt_refresh_secret: very_long_random_string_2

# app/services/jwt_service.rb
module JwtService
  ACCESS_SECRET  = -> { Rails.application.credentials.jwt_access_secret! }
  REFRESH_SECRET = -> { Rails.application.credentials.jwt_refresh_secret! }
  ACCESS_TTL     = 15.minutes
  REFRESH_TTL    = 30.days
  ALGORITHM      = 'HS256'

  TokenExpiredError = Class.new(StandardError)
  InvalidTokenError = Class.new(StandardError)

  def self.generate_tokens(user)
    jti = SecureRandom.uuid
    access_token  = encode_access(user_id: user.id, jti: jti)
    refresh_token = encode_refresh(user_id: user.id, jti: jti)

    RefreshToken.create!(
      user: user,
      jti: jti,
      token_digest: Digest::SHA256.hexdigest(refresh_token),
      expires_at: REFRESH_TTL.from_now
    )

    { access_token: access_token, refresh_token: refresh_token,
      expires_in: ACCESS_TTL.to_i, token_type: "Bearer" }
  end

  def self.encode_access(payload)
    JWT.encode(
      payload.merge(type: 'access', exp: ACCESS_TTL.from_now.to_i, iat: Time.current.to_i),
      ACCESS_SECRET.call, ALGORITHM
    )
  end

  def self.encode_refresh(payload)
    JWT.encode(
      payload.merge(type: 'refresh', exp: REFRESH_TTL.from_now.to_i),
      REFRESH_SECRET.call, ALGORITHM
    )
  end

  def self.decode_access(token)
    decode_token(token, ACCESS_SECRET.call, type: 'access')
  end

  def self.decode_refresh(token)
    decode_token(token, REFRESH_SECRET.call, type: 'refresh')
  end

  def self.revoke_for_user(user)
    RefreshToken.where(user: user).update_all(revoked_at: Time.current)
  end

  private_class_method def self.decode_token(token, secret, type:)
    payload = JWT.decode(token, secret, true, { algorithm: ALGORITHM }).first
    raise InvalidTokenError, "Wrong token type" unless payload['type'] == type
    HashWithIndifferentAccess.new(payload)
  rescue JWT::ExpiredSignature
    raise TokenExpiredError, "Token expired"
  rescue JWT::DecodeError => e
    raise InvalidTokenError, e.message
  end
end

# app/controllers/api/v1/auth/sessions_controller.rb
class Api::V1::Auth::SessionsController < Api::V1::BaseController
  skip_before_action :authenticate!

  def create
    user = User.find_by(email: params[:email]&.downcase)

    if user&.authenticate(params[:password])
      tokens = JwtService.generate_tokens(user)
      render json: {
        data: { tokens: tokens, user: UserSerializer.new(user).serializable_hash }
      }, status: :created
    else
      render json: { error: { code: "invalid_credentials",
                               message: "Invalid email or password" } },
             status: :unauthorized
    end
  end

  def destroy
    current_token_jti = current_payload['jti']
    RefreshToken.find_by(jti: current_token_jti)&.revoke!
    head :no_content
  end
end

# app/controllers/api/v1/auth/token_refreshes_controller.rb
class Api::V1::Auth::TokenRefreshesController < Api::V1::BaseController
  skip_before_action :authenticate!

  def create
    raw_token = params.require(:refresh_token)
    payload = JwtService.decode_refresh(raw_token)

    token_record = RefreshToken.active.find_by!(
      jti: payload['jti'],
      user_id: payload['user_id'],
      token_digest: Digest::SHA256.hexdigest(raw_token)
    )

    user = token_record.user
    # Token rotation: revoke old, issue new
    token_record.revoke!
    new_tokens = JwtService.generate_tokens(user)

    render json: { data: { tokens: new_tokens } }, status: :created
  rescue JwtService::TokenExpiredError
    render json: { error: { code: "refresh_token_expired", message: "Please log in again" } },
           status: :unauthorized
  rescue JwtService::InvalidTokenError, ActiveRecord::RecordNotFound
    render json: { error: { code: "invalid_refresh_token", message: "Invalid token" } },
           status: :unauthorized
  end
end

# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!

  private

  def authenticate!
    token = request.headers['Authorization']&.delete_prefix('Bearer ')
    payload = JwtService.decode_access(token)
    @current_user = User.find(payload[:user_id])
    @current_payload = payload
  rescue JwtService::TokenExpiredError
    render json: { error: { code: "token_expired", message: "Access token expired" } },
           status: :unauthorized
  rescue JwtService::InvalidTokenError, ActiveRecord::RecordNotFound
    render json: { error: { code: "unauthorized", message: "Authentication required" } },
           status: :unauthorized
  end

  attr_reader :current_user, :current_payload
end
```

---

## Answer 2: Blueprinter Serialization with Multiple Views

**Question:** Implement a comprehensive serialization layer using Blueprinter with multiple views, computed fields, and associations.

```ruby
# app/blueprints/user_blueprint.rb
class UserBlueprint < Blueprinter::Base
  identifier :id

  # Shared across all views
  fields :name, :email
  field :created_at do |user| user.created_at.iso8601 end

  # Default view
  field :bio
  field :avatar_url do |user, opts|
    opts[:cdn_host] ? "#{opts[:cdn_host]}#{user.avatar_path}" : user.avatar_url
  end
  field :posts_count do |user| user.posts.published.count end

  # Minimal (for list views, mobile)
  view :minimal do
    field :avatar_url do |user| user.avatar_thumbnail_url end
  end

  # Detailed (for profile pages)
  view :detailed do
    include_view :default
    fields :website, :location, :twitter_handle
    field :follower_count do |user| user.followers.count end
    field :following_count do |user| user.following.count end
    association :recent_posts, blueprint: PostBlueprint, view: :card do |user|
      user.posts.published.order(published_at: :desc).limit(5)
    end
    association :profile, blueprint: ProfileBlueprint
  end

  # Admin (adds sensitive/internal fields)
  view :admin do
    include_view :detailed
    fields :last_sign_in_at, :sign_in_count, :failed_attempts, :locked_at
    fields :stripe_customer_id, :subscription_status
    field :admin_notes
    field :internal_id do |user| "USR-#{user.id.to_s.rjust(8, '0')}" end
  end
end

# app/blueprints/post_blueprint.rb
class PostBlueprint < Blueprinter::Base
  identifier :id
  fields :title, :slug
  field :published_at do |post| post.published_at&.iso8601 end

  view :card do
    field :excerpt do |post| truncate(post.body, length: 200) end
    field :cover_image_url
    association :author, blueprint: UserBlueprint, view: :minimal
  end

  view :full do
    include_view :card
    field :body
    fields :views_count, :likes_count
    association :tags, blueprint: TagBlueprint
    association :comments, blueprint: CommentBlueprint do |post|
      post.comments.approved.order(created_at: :desc).limit(20)
    end
  end
end

# Controller usage:
class Api::V1::UsersController < Api::V1::BaseController
  def show
    @user = User.includes(posts: :tags, profile: nil).find(params[:id])
    view = current_user.admin? ? :admin : :detailed
    render json: UserBlueprint.render(@user, view: view, cdn_host: cdn_host)
  end

  def index
    @users = User.active.page(params[:page]).per(20)
    render json: UserBlueprint.render(@users, view: :minimal)
  end

  private

  def cdn_host
    Rails.env.production? ? "https://cdn.example.com" : nil
  end
end
```

---

## Answer 3: Comprehensive CORS Configuration

**Question:** Configure CORS for a production app with multiple allowed origins (prod, staging, mobile app scheme) and security considerations.

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  # Production web origins
  allow do
    origins(
      'https://myapp.com',
      'https://www.myapp.com',
      'https://app.myapp.com',
      # Staging:
      'https://staging.myapp.com',
      # Dynamic subdomain pattern:
      /\Ahttps:\/\/[a-z0-9\-]+\.myapp\.com\z/
    )

    resource '/api/*',
             headers: :any,
             methods: [:get, :post, :put, :patch, :delete, :options, :head],
             credentials: true,
             max_age: 7200,
             expose: ['Authorization', 'X-Request-Id', 'X-RateLimit-Remaining']
  end

  # Mobile apps using custom URL scheme (Capacitor/Electron)
  allow do
    origins 'capacitor://localhost', 'ionic://localhost', 'http://localhost'
    resource '/api/*', headers: :any, methods: [:get, :post, :put, :patch, :delete, :options]
  end

  # Public API endpoints (no credentials, CDN-cacheable)
  allow do
    origins '*'
    resource '/api/v1/public/*',
             headers: ['Accept', 'Accept-Language', 'Content-Type'],
             methods: [:get, :head, :options],
             credentials: false,
             max_age: 86400  # 24h preflight cache for public endpoints
  end

  # Development overrides (loaded from environment-specific initializer)
  if Rails.env.development?
    allow do
      origins 'http://localhost:3000', 'http://localhost:5173',
              'http://localhost:8080', 'http://127.0.0.1:*'
      resource '*', headers: :any, methods: [:get, :post, :put, :patch, :delete, :options]
    end
  end
end

# Test that CORS headers are correct:
# curl -v -H "Origin: https://myapp.com" \
#         -H "Access-Control-Request-Method: POST" \
#         -H "Access-Control-Request-Headers: Authorization, Content-Type" \
#         -X OPTIONS https://api.myapp.com/api/v1/users
# Expected: 200 with Access-Control-Allow-Origin, Access-Control-Allow-Methods
```

---

## Answer 4: API Versioning Strategy — Complete Implementation

**Question:** Implement a complete API versioning strategy that allows coexistence of V1 and V2 with shared business logic.

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api, defaults: { format: :json } do
    # V1 routes
    namespace :v1 do
      post 'auth/sessions',        to: 'auth/sessions#create'
      delete 'auth/sessions',       to: 'auth/sessions#destroy'
      post 'auth/token_refreshes',  to: 'auth/token_refreshes#create'
      resources :users, only: [:index, :show, :create, :update]
      resources :posts, only: [:index, :show, :create, :update, :destroy]
    end

    # V2 routes (new features)
    namespace :v2 do
      post 'auth/sessions',         to: 'auth/sessions#create'
      resources :users,             only: [:index, :show, :create, :update, :destroy]
      resources :posts,             only: [:index, :show, :create, :update, :destroy] do
        resources :reactions,       only: [:create, :destroy]
      end
      resources :notifications,     only: [:index, :update]
    end
  end
end

# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    include Api::ErrorHandling
    include Api::ResponseHelpers
    before_action :authenticate!

    protected

    def current_user
      @current_user
    end
  end
end

# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < Api::BaseController
      # V1-specific behavior: legacy pagination format
      def pagination_meta(collection)
        { total: collection.total_count, page: collection.current_page,
          per_page: collection.limit_value, total_pages: collection.total_pages }
      end
    end
  end
end

# app/controllers/api/v2/base_controller.rb
module Api
  module V2
    class BaseController < Api::BaseController
      # V2: cursor-based pagination, different format
      def pagination_meta(collection, cursor)
        { next_cursor: cursor, has_more: collection.size > per_page }
      end
    end
  end
end

# V2 users controller inherits from V1 for unchanged actions:
module Api
  module V2
    class UsersController < Api::V1::UsersController
      # Only override what changed in V2:
      def show
        @user = User.find(params[:id])
        render json: UserV2Blueprint.render(@user)  # Different serialization
      end

      def destroy
        # V2 adds this action (V1 doesn't have it)
        @user = User.find(params[:id])
        authorize @user, :destroy?
        @user.destroy!
        render_no_content
      end
    end
  end
end
```

---

## Answer 5: Pagination — Offset vs Cursor

**Question:** Implement both offset-based and cursor-based pagination, explain the tradeoffs.

```ruby
# OFFSET PAGINATION (Kaminari or manual)
class Api::V1::PostsController < Api::V1::BaseController
  def index
    @posts = Post.published
                 .order(created_at: :desc)
                 .page(params[:page])
                 .per(per_page)

    render json: {
      data: PostBlueprint.render_as_hash(@posts),
      meta: {
        current_page: @posts.current_page,
        total_pages: @posts.total_pages,
        total_count: @posts.total_count,
        per_page: @posts.limit_value
      },
      links: {
        self:  posts_url(page: @posts.current_page, per_page: per_page),
        prev:  @posts.current_page > 1 ? posts_url(page: @posts.prev_page, per_page: per_page) : nil,
        next:  @posts.next_page ? posts_url(page: @posts.next_page, per_page: per_page) : nil,
        first: posts_url(page: 1, per_page: per_page),
        last:  posts_url(page: @posts.total_pages, per_page: per_page)
      }.compact
    }
  end
end

# CURSOR PAGINATION (for real-time feeds)
# Pros: stable (new items don't shift page boundaries), consistent performance
# Cons: can't jump to arbitrary page, client must store cursor
class Api::V2::PostsController < Api::V2::BaseController
  CURSOR_TTL = 24.hours

  def index
    cursor_data = decode_cursor(params[:cursor])
    per = per_page

    @posts = Post.published.order(id: :desc)

    if cursor_data
      @posts = @posts.where("id < ?", cursor_data[:last_id])
    end

    @posts = @posts.limit(per + 1).to_a
    has_more = @posts.size > per
    @posts = @posts.first(per)

    next_cursor = has_more ? encode_cursor(last_id: @posts.last.id) : nil
    prev_cursor = cursor_data ? encode_cursor(last_id: cursor_data[:first_id]) : nil

    render json: {
      data: PostBlueprint.render_as_hash(@posts),
      meta: { has_more: has_more, per_page: per },
      cursors: {
        next: next_cursor,
        prev: prev_cursor
      }.compact
    }
  end

  private

  def encode_cursor(data)
    Base64.urlsafe_encode64(data.to_json, padding: false)
  end

  def decode_cursor(cursor_str)
    return nil unless cursor_str.present?
    JSON.parse(Base64.urlsafe_decode64(cursor_str), symbolize_names: true)
  rescue ArgumentError, JSON::ParserError
    nil
  end

  def per_page
    [[params[:per_page].to_i, 1].max, 100].min.presence || 20
  end
end
```

