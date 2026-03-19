# Chapter 03 — Routing & Controllers: Structured Answers

---

## Answer 1: Full RESTful Controller Implementation

**Question:** Build a complete, production-quality `PostsController` with all RESTful actions, strong parameters, error handling, and multi-format support.

```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :find_post, only: [:show, :edit, :update, :destroy]
  before_action :require_ownership!, only: [:edit, :update, :destroy]

  def index
    @posts = Post.published
                 .includes(:author, :category, :tags)
                 .order(published_at: :desc)
                 .page(params[:page])
                 .per(params[:per_page] || 15)

    respond_to do |format|
      format.html
      format.json do
        render json: {
          posts: @posts.map { |p| post_summary(p) },
          meta: { total: @posts.total_count, page: @posts.current_page }
        }
      end
      format.rss { render layout: false }
    end
  end

  def show
    respond_to do |format|
      format.html
      format.json { render json: post_detail(@post) }
    end
  end

  def new
    @post = current_user.posts.build
  end

  def create
    @post = current_user.posts.build(post_params)

    respond_to do |format|
      if @post.save
        format.html { redirect_to @post, notice: "Post published!" }
        format.json { render json: post_detail(@post), status: :created,
                              location: @post }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: { errors: @post.errors.full_messages },
                             status: :unprocessable_entity }
      end
    end
  end

  def edit; end

  def update
    respond_to do |format|
      if @post.update(post_params)
        format.html { redirect_to @post, notice: "Post updated!" }
        format.json { render json: post_detail(@post) }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: { errors: @post.errors.full_messages },
                             status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @post.destroy
    respond_to do |format|
      format.html { redirect_to posts_url, notice: "Post deleted" }
      format.json { head :no_content }
    end
  end

  private

  def find_post
    @post = Post.find(params[:id])
  rescue ActiveRecord::RecordNotFound
    respond_to do |format|
      format.html { render file: 'public/404.html', status: :not_found }
      format.json { render json: { error: "Post not found" }, status: :not_found }
    end
  end

  def require_ownership!
    unless @post.author == current_user || current_user.admin?
      respond_to do |format|
        format.html { redirect_to @post, alert: "Not authorized" }
        format.json { render json: { error: "Forbidden" }, status: :forbidden }
      end
    end
  end

  def post_params
    params.require(:post).permit(
      :title, :body, :category_id, :published, :published_at,
      :meta_description, :featured_image,
      tag_ids: []
    )
  end

  def post_summary(post)
    {
      id: post.id,
      title: post.title,
      author: post.author.name,
      published_at: post.published_at&.iso8601,
      url: post_url(post)
    }
  end

  def post_detail(post)
    post_summary(post).merge(
      body: post.body,
      category: post.category&.name,
      tags: post.tags.map(&:name),
      updated_at: post.updated_at.iso8601
    )
  end
end
```

---

## Answer 2: Route Design for a SaaS Application

**Question:** Design the routes for a SaaS app with: public marketing pages, user accounts, a multi-tenant dashboard, and an admin panel.

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # ─── Marketing / Public ─────────────────────────────────
  root to: 'marketing#index'
  get '/pricing', to: 'marketing#pricing', as: :pricing
  get '/features', to: 'marketing#features', as: :features
  get '/about', to: 'marketing#about', as: :about
  get '/blog', to: 'blog/posts#index', as: :blog
  get '/blog/:slug', to: 'blog/posts#show', as: :blog_post

  # ─── Authentication ──────────────────────────────────────
  get    '/login',    to: 'sessions#new',     as: :login
  post   '/login',    to: 'sessions#create'
  delete '/logout',   to: 'sessions#destroy', as: :logout
  resource :registration, only: [:new, :create]
  resource :password_reset, only: [:new, :create, :edit, :update]

  # ─── User Account ────────────────────────────────────────
  scope '/account' do
    resource :profile,   only: [:show, :edit, :update]
    resource :password,  only: [:edit, :update]
    resource :billing,   only: [:show, :edit, :update]
    resources :api_keys, only: [:index, :create, :destroy]
    resource :subscription do
      post :cancel
      post :resume
    end
  end

  # ─── Multi-tenant Dashboard ──────────────────────────────
  constraints WorkspaceConstraint do
    scope '/w/:workspace_slug', as: :workspace do
      resource :dashboard, only: [:show]
      resources :projects do
        resources :tasks, shallow: true do
          member do
            patch :complete
            patch :assign
          end
        end
        resources :members, only: [:index, :create, :destroy]
      end
      resources :members
      resources :invitations, only: [:index, :create, :destroy]
      resource :settings, only: [:show, :update]
    end
  end

  # ─── Webhooks ────────────────────────────────────────────
  scope '/webhooks' do
    post '/stripe',   to: 'webhooks/stripe#receive'
    post '/sendgrid', to: 'webhooks/sendgrid#receive'
  end

  # ─── Admin Panel ─────────────────────────────────────────
  namespace :admin, constraints: AdminConstraint.new do
    root to: 'dashboard#index'
    resources :users do
      member do
        post :impersonate
        post :suspend
        post :reinstate
      end
      collection do
        get :export
      end
    end
    resources :workspaces, only: [:index, :show, :destroy]
    resources :subscriptions
    resource  :system_status, only: [:show]
  end

  # ─── API ─────────────────────────────────────────────────
  namespace :api, defaults: { format: :json } do
    namespace :v1 do
      resources :projects,  only: [:index, :show, :create, :update, :destroy]
      resources :tasks,     only: [:index, :show, :create, :update, :destroy]
      resources :users,     only: [:show, :update]
    end
  end

  # ─── Health checks ────────────────────────────────────────
  get '/health', to: 'health#check'
  get '/up',     to: proc { [200, {}, ['OK']] }
end
```

---

## Answer 3: Around Action for Timing and Context

**Question:** Implement an `around_action` that sets a request context (current user in a thread-safe way) and logs timing.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :set_request_context
  around_action :log_request_duration

  private

  def set_request_context
    # Thread-safe current user via Current attributes (Rails 5.2+)
    Current.user = current_user
    Current.request_id = request.request_id
    Current.ip_address = request.remote_ip

    yield

    # Cleanup: Clear after request (important for thread reuse in Puma)
    Current.reset
  end

  def log_request_duration
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    begin
      yield
    ensure
      duration = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round(2)
      Rails.logger.info "#{request.method} #{request.path} completed in #{duration}ms " \
                        "[user=#{current_user&.id}] [#{response.status}]"
    end
  end
end

# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :request_id, :ip_address

  resets { user = nil }
end

# Usage in models, mailers, jobs:
class AuditLog < ApplicationRecord
  before_create :set_audit_context

  private

  def set_audit_context
    self.performed_by ||= Current.user&.id
    self.ip_address   ||= Current.ip_address
  end
end
```

---

## Answer 4: Custom Routing Constraints

**Question:** Implement routing constraints for: (1) API version via header, (2) mobile user agent, (3) beta users only.

```ruby
# (1) API version via Accept header
class ApiVersionConstraint
  def initialize(version:, default: false)
    @version = version
    @default = default
  end

  def matches?(request)
    @default || request.headers['Accept'].include?("application/vnd.myapp.v#{@version}+json")
  end
end

namespace :api do
  scope constraints: ApiVersionConstraint.new(version: 2) do
    namespace :v2 do
      resources :users
    end
  end

  scope constraints: ApiVersionConstraint.new(version: 1, default: true) do
    namespace :v1 do
      resources :users
    end
  end
end

# (2) Mobile user agent
class MobileConstraint
  MOBILE_AGENTS = /Android|BlackBerry|iPhone|iPad|iPod|Opera Mini|IEMobile|WPDesktop/i

  def self.matches?(request)
    request.user_agent =~ MOBILE_AGENTS
  end
end

constraints MobileConstraint do
  root to: 'mobile#index', as: :mobile_root
  resources :products, controller: 'mobile/products'
end

# (3) Beta users only
class BetaConstraint
  def matches?(request)
    user_id = request.session[:user_id]
    return false unless user_id
    User.beta.exists?(id: user_id)
  end
end

namespace :beta, constraints: BetaConstraint.new do
  resources :new_features
  resource  :new_dashboard
end
```

---

## Answer 5: Comprehensive Error Handling in ApplicationController

**Question:** Set up comprehensive error handling in ApplicationController for both HTML and JSON clients.

```ruby
class ApplicationController < ActionController::Base
  # Catch ActiveRecord errors
  rescue_from ActiveRecord::RecordNotFound do |exception|
    error_response(
      status: :not_found,
      message: "#{exception.model} not found",
      html_template: '404'
    )
  end

  rescue_from ActiveRecord::RecordInvalid do |exception|
    error_response(
      status: :unprocessable_entity,
      message: "Validation failed",
      errors: exception.record.errors.full_messages
    )
  end

  # Authorization errors
  rescue_from Pundit::NotAuthorizedError do |exception|
    error_response(
      status: :forbidden,
      message: "You are not authorized to perform this action"
    )
  end

  # Parameter errors
  rescue_from ActionController::ParameterMissing do |exception|
    error_response(
      status: :bad_request,
      message: "Missing required parameter: #{exception.param}"
    )
  end

  rescue_from ActionController::UnpermittedParameters do |exception|
    error_response(
      status: :bad_request,
      message: "Unexpected parameters: #{exception.params.join(', ')}"
    )
  end

  # CSRF
  rescue_from ActionController::InvalidAuthenticityToken do
    error_response(
      status: :unprocessable_entity,
      message: "Invalid authenticity token"
    )
  end

  private

  def error_response(status:, message:, errors: nil, html_template: nil)
    respond_to do |format|
      format.html do
        if html_template
          render file: Rails.root.join("public/#{html_template}.html"),
                 status: status, layout: false
        else
          flash[:alert] = message
          redirect_back(fallback_location: root_path)
        end
      end
      format.json do
        payload = { error: message }
        payload[:errors] = errors if errors
        render json: payload, status: status
      end
    end
  end
end
```

---

## Answer 6: Session Security Implementation

**Question:** Implement secure session management with remember-me, session fixation protection, and concurrent session limiting.

```ruby
class SessionsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]

  def new
    redirect_to root_path if current_user
  end

  def create
    user = User.find_by(email: params[:email]&.downcase)

    if user&.authenticate(params[:password])
      # Session fixation protection: reset session BEFORE setting user_id
      reset_session
      session[:user_id] = user.id
      session[:logged_in_at] = Time.current.to_i

      # Handle concurrent session limiting
      SessionManager.new(user).enforce_session_limit!(session.id)

      # Remember me cookie
      if params[:remember_me] == "1"
        remember_token = user.generate_remember_token
        cookies.permanent.encrypted[:remember_token] = {
          value: remember_token,
          httponly: true,
          secure: Rails.env.production?,
          same_site: :lax
        }
      end

      redirect_to after_login_path(user), notice: "Welcome back, #{user.name}!"
    else
      # Prevent user enumeration — same message for unknown email and wrong password
      flash.now[:alert] = "Invalid email or password"
      @email = params[:email]  # Preserve email in form
      render :new, status: :unauthorized
    end
  end

  def destroy
    user = current_user
    reset_session
    cookies.delete(:remember_token)
    user&.invalidate_remember_token!
    redirect_to login_path, notice: "Signed out successfully"
  end

  private

  def after_login_path(user)
    stored_path = session.delete(:return_to)
    if stored_path&.start_with?("/") && !stored_path.start_with?("//")
      stored_path
    elsif user.admin?
      admin_root_path
    else
      dashboard_path
    end
  end
end
```

---

## Answer 7: REST Resource with Custom Actions

**Question:** Implement a `SubscriptionsController` with standard CRUD plus custom actions for cancellation, pause, and resume.

```ruby
# Routes:
# resources :subscriptions do
#   member do
#     post :cancel
#     post :pause
#     post :resume
#     post :change_plan
#   end
# end

class SubscriptionsController < ApplicationController
  before_action :authenticate_user!
  before_action :find_subscription

  def show
    respond_to do |format|
      format.html
      format.json { render json: subscription_details(@subscription) }
    end
  end

  def cancel
    result = SubscriptionCancellationService.new(@subscription).call(
      reason: params[:cancellation_reason],
      immediate: params[:immediate] == "true"
    )

    if result.success?
      respond_to do |format|
        format.html { redirect_to @subscription, notice: "Subscription cancelled" }
        format.json { render json: { status: "cancelled", ends_at: result.ends_at } }
      end
    else
      respond_to do |format|
        format.html { redirect_to @subscription, alert: result.error }
        format.json { render json: { error: result.error }, status: :unprocessable_entity }
      end
    end
  end

  def pause
    if @subscription.can_pause?
      @subscription.pause!(
        pause_until: params[:resume_on].present? ? Date.parse(params[:resume_on]) : nil
      )
      respond_to do |format|
        format.html { redirect_to @subscription, notice: "Subscription paused" }
        format.json { render json: { status: "paused" } }
      end
    else
      respond_to do |format|
        format.html { redirect_to @subscription, alert: "This subscription cannot be paused" }
        format.json { render json: { error: "Cannot pause" }, status: :conflict }
      end
    end
  end

  def resume
    if @subscription.paused?
      @subscription.resume!
      respond_to do |format|
        format.html { redirect_to @subscription, notice: "Subscription resumed" }
        format.json { render json: { status: "active" } }
      end
    else
      respond_to do |format|
        format.html { redirect_to @subscription, alert: "Subscription is not paused" }
        format.json { render json: { error: "Not paused" }, status: :conflict }
      end
    end
  end

  def change_plan
    result = PlanChangeService.new(@subscription, params[:plan_id]).call
    redirect_to @subscription,
                notice: result.success? ? "Plan changed!" : result.error
  end

  private

  def find_subscription
    @subscription = current_user.subscription
    unless @subscription
      redirect_to new_subscription_path, alert: "No active subscription found"
    end
  end

  def subscription_details(sub)
    {
      id: sub.id,
      plan: sub.plan_name,
      status: sub.status,
      current_period_end: sub.current_period_end&.iso8601,
      cancel_at_period_end: sub.cancel_at_period_end,
      amount: sub.amount.format
    }
  end
end
```

---

## Answer 8: Strong Parameters Edge Cases

**Question:** Demonstrate strong parameter handling for: array of IDs, nested attributes, file uploads, and JSON blobs.

```ruby
class ProjectsController < ApplicationController
  def create
    @project = Project.new(project_params)
    # ...
  end

  private

  def project_params
    params.require(:project).permit(
      # Simple scalar
      :name, :description, :status, :visibility,

      # Date/time (comes as string, AR coerces)
      :deadline,

      # File upload (ActionDispatch::Http::UploadedFile object)
      :cover_image, :attachment,

      # Array of IDs
      member_ids: [],
      tag_ids: [],

      # Nested attributes (has_many)
      tasks_attributes: [
        :id, :title, :description, :due_date, :assignee_id,
        :position, :_destroy  # _destroy allows nested deletion
      ],

      # Nested attributes (has_one)
      settings_attributes: [
        :id, :notifications_enabled, :visibility, :timezone
      ],

      # JSON blob / dynamic keys (CAREFULLY permitted)
      # Only allow a whitelist of top-level keys inside metadata:
      metadata: [:source, :utm_campaign, :referrer]
      # NEVER: metadata: {} unless you control all keys
    )
  end

  # Handling file upload separately:
  def handle_cover_image
    if params[:project][:cover_image].present?
      upload = params[:project][:cover_image]
      # Validate file type
      unless upload.content_type.in?(%w[image/jpeg image/png image/webp])
        raise ArgumentError, "Invalid file type: #{upload.content_type}"
      end
      # Validate size
      if upload.size > 5.megabytes
        raise ArgumentError, "File too large (max 5MB)"
      end
    end
  end
end
```

---

## Answer 9: Before Action Filter Inheritance and Overriding

**Question:** Demonstrate how to properly manage filter chains across a controller hierarchy with skip and prepend.

```ruby
class ApplicationController < ActionController::Base
  before_action :set_locale
  before_action :authenticate_user!
  before_action :track_visit
end

class PublicController < ApplicationController
  # Public pages don't require authentication:
  skip_before_action :authenticate_user!

  # Public pages should still track visits and set locale:
  # (set_locale and track_visit still run)
end

class ApiController < ActionController::API
  # API inherits from ActionController::API, not ApplicationController
  # So ApplicationController filters don't apply
  before_action :authenticate_api_token!
end

class AdminController < ApplicationController
  # Admin DOES require auth (inherited from ApplicationController)
  # But prepend a check that runs FIRST:
  prepend_before_action :require_not_impersonating!

  private

  def require_not_impersonating!
    if session[:impersonating]
      redirect_to stop_impersonation_path, alert: "Admin actions unavailable while impersonating"
    end
  end
end

class Admin::ReportsController < AdminController
  # Some admin reports are public (downloadable links with token):
  skip_before_action :authenticate_user!, only: [:download]
  before_action :verify_download_token, only: [:download]
end
```

---

## Answer 10: Handling Concurrent Requests Safely

**Question:** You have an API endpoint that creates a unique resource. How do you prevent duplicates from rapid simultaneous requests?

```ruby
class Api::V1::OrdersController < Api::V1::BaseController
  def create
    # Strategy 1: Idempotency key (client provides unique key)
    idempotency_key = request.headers['Idempotency-Key']

    if idempotency_key.present?
      existing = Order.find_by(idempotency_key: idempotency_key, user: current_user)
      if existing
        render json: existing, status: :ok  # Return existing order, 200 not 201
        return
      end
    end

    # Strategy 2: Database unique constraint + rescue
    @order = current_user.orders.build(order_params)
    @order.idempotency_key = idempotency_key if idempotency_key.present?

    if @order.save
      render json: @order, status: :created
    else
      render json: { errors: @order.errors.full_messages }, status: :unprocessable_entity
    end
  rescue ActiveRecord::RecordNotUnique
    # Another request with same idempotency key committed first
    @order = Order.find_by!(idempotency_key: idempotency_key, user: current_user)
    render json: @order, status: :ok
  end
end

# Migration for idempotency key:
class AddIdempotencyKeyToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :idempotency_key, :string
    add_index :orders, [:user_id, :idempotency_key], unique: true,
              where: "idempotency_key IS NOT NULL"
  end
end
```

