# Chapter 03 — Routing & Controllers: Scenario Questions

---

**S1. A developer reports that after submitting a form, clicking the browser's back button re-submits the form. Users are accidentally creating duplicate orders. How do you fix this?**

```ruby
# Problem: POST → render (not redirect) means the browser's back button
# re-submits the POST request, creating duplicate records.

# Fix: Use the Post/Redirect/Get (PRG) pattern:
def create
  @order = Order.new(order_params)
  if @order.save
    redirect_to @order, notice: "Order placed!"  # Redirect after POST
    # Browser's back button now goes to GET /orders/:id — no resubmission
  else
    render :new, status: :unprocessable_entity  # Re-render (NOT redirect)
    # Error case: stay at the form with validation errors
  end
end

# Additional protection: idempotency key
def create
  idempotency_key = params[:idempotency_key] || SecureRandom.uuid

  @order = Order.find_by(idempotency_key: idempotency_key) ||
           Order.create!(order_params.merge(idempotency_key: idempotency_key))

  redirect_to @order
rescue ActiveRecord::RecordInvalid => e
  render :new, status: :unprocessable_entity
end
```

---

**S2. You need to build an admin area at `/admin/` that requires admin authentication, uses a separate layout, and has its own base controller. Design the routing and controller hierarchy.**

```ruby
# config/routes.rb:
Rails.application.routes.draw do
  namespace :admin do
    root to: 'dashboard#index'
    resources :users
    resources :products
    resources :orders, only: [:index, :show, :update]
    resource  :settings
  end
end

# app/controllers/admin/base_controller.rb:
module Admin
  class BaseController < ApplicationController
    layout 'admin'
    before_action :require_admin!

    private

    def require_admin!
      unless current_user&.admin?
        redirect_to root_path, alert: "Not authorized"
      end
    end
  end
end

# app/controllers/admin/users_controller.rb:
module Admin
  class UsersController < BaseController
    def index
      @users = User.order(created_at: :desc).page(params[:page]).per(50)
    end

    def update
      @user = User.find(params[:id])
      if @user.update(admin_user_params)
        redirect_to admin_users_path, notice: "Updated"
      else
        render :edit
      end
    end

    private

    def admin_user_params
      # Admin can update more fields than regular users
      params.require(:user).permit(:name, :email, :role, :active, :notes)
    end
  end
end
```

---

**S3. Your `before_action :authenticate_user!` is redirecting API requests to the login page HTML. How do you fix this for API clients?**

```ruby
class ApplicationController < ActionController::Base
  before_action :authenticate_user!

  private

  def authenticate_user!
    return if current_user

    respond_to do |format|
      format.html { redirect_to login_path, alert: "Please log in" }
      format.json { render json: { error: "Unauthorized" }, status: :unauthorized }
      format.any  { head :unauthorized }
    end
  end
end

# Or separate controllers entirely:
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    before_action :authenticate_api_token!

    private

    def authenticate_api_token!
      token = request.headers['Authorization']&.split(' ')&.last
      @current_user = ApiToken.active.find_by!(token: token)&.user
    rescue ActiveRecord::RecordNotFound
      render json: { error: "Invalid or expired token" }, status: :unauthorized
    end
  end
end
```

---

**S4. You need to implement a search feature at `/search` that can search across Articles, Products, and Users. Design the routing and controller.**

```ruby
# config/routes.rb:
Rails.application.routes.draw do
  get '/search', to: 'search#index', as: :search
  # OR as a resource on collections:
  resources :articles do
    collection { get :search }
  end
end

# app/controllers/search_controller.rb:
class SearchController < ApplicationController
  SEARCHABLE_TYPES = %w[Article Product User].freeze

  def index
    @query = params[:q]&.strip
    @type  = params[:type]

    if @query.blank?
      @results = {}
      return
    end

    types_to_search = if @type.in?(SEARCHABLE_TYPES)
                        [@type]
                      else
                        SEARCHABLE_TYPES
                      end

    @results = types_to_search.index_with do |type|
      type.constantize.search(@query).limit(10)
    end

    respond_to do |format|
      format.html
      format.json do
        render json: {
          query: @query,
          results: @results.transform_values { |records| records.map(&:as_json_search) }
        }
      end
    end
  end
end

# Model concern:
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      where("#{table_name}.searchable_content ILIKE ?", "%#{query}%")
    }
  end
end
```

---

**S5. Strong parameters are working for simple attributes, but you need to permit a nested `address_attributes` hash with dynamic keys for international addresses (different countries have different address fields). How do you handle this?**

```ruby
# Option 1: Enumerate known fields
def user_params
  params.require(:user).permit(
    :name, :email,
    address_attributes: [
      :id,
      :line1, :line2,         # US/UK
      :city, :state,          # US
      :county,                # UK
      :postal_code,
      :country_code,
      :prefecture,            # Japan
      :ward,                  # Japan
      :_destroy
    ]
  )
end

# Option 2: Permit the entire hash (controlled risk — use only for known-safe scenarios)
def user_params
  base = params.require(:user).permit(:name, :email)
  base.merge(
    address_attributes: params.dig(:user, :address_attributes)
                               &.permit!
                               &.to_h || {}
  )
end

# Option 3: Custom strong params sanitizer
def address_params
  allowed_keys = AddressFieldRegistry.keys_for(params.dig(:user, :address_attributes, :country_code))
  params.require(:user)
        .require(:address_attributes)
        .permit(*allowed_keys, :country_code, :_destroy)
end
```

---

**S6. You have a RESTful API that needs to support both `/api/v1/users` and `/api/v2/users` with different response formats. How do you set up the routing and controllers?**

```ruby
# config/routes.rb:
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:index, :show, :create, :update, :destroy]
    end

    namespace :v2 do
      resources :users, only: [:index, :show, :create, :update, :destroy]
    end
  end
end

# app/controllers/api/base_controller.rb:
module Api
  class BaseController < ActionController::API
    before_action :authenticate!
    rescue_from ActiveRecord::RecordNotFound, with: :record_not_found

    private

    def authenticate!
      # shared auth logic
    end

    def record_not_found
      render json: { error: "Not found" }, status: :not_found
    end
  end
end

# app/controllers/api/v1/users_controller.rb:
module Api
  module V1
    class UsersController < Api::BaseController
      def index
        @users = User.all.page(params[:page])
        render json: { users: @users.map(&:as_json_v1), meta: pagination_meta(@users) }
      end
    end
  end
end

# app/controllers/api/v2/users_controller.rb:
module Api
  module V2
    class UsersController < Api::BaseController
      def index
        @users = User.all.page(params[:page])
        # V2: uses serializer, includes relationships, different format
        render json: UserSerializer.new(@users, { params: { version: 2 } }).serializable_hash
      end
    end
  end
end
```

---

**S7. A user submits a form. Your `create` action calls a service. The service can fail with different error types. How do you handle each type and return appropriate responses?**

```ruby
class OrdersController < ApplicationController
  def create
    service = OrderCreationService.new(order_params, user: current_user)

    case result = service.call
    in { success: true, order: order }
      respond_to do |format|
        format.html { redirect_to order, notice: "Order ##{order.number} created!" }
        format.json { render json: order, status: :created }
      end
    in { success: false, reason: :insufficient_inventory }
      respond_to do |format|
        format.html do
          flash.now[:alert] = "Some items are out of stock"
          @order = service.order
          render :new, status: :unprocessable_entity
        end
        format.json { render json: { error: "Insufficient inventory" }, status: :conflict }
      end
    in { success: false, errors: errors }
      respond_to do |format|
        format.html do
          flash.now[:alert] = errors.to_sentence
          @order = service.order
          render :new, status: :unprocessable_entity
        end
        format.json { render json: { errors: errors }, status: :unprocessable_entity }
      end
    end
  rescue OrderCreationService::PaymentFailed => e
    respond_to do |format|
      format.html { redirect_to new_order_path, alert: "Payment failed: #{e.message}" }
      format.json { render json: { error: e.message }, status: :payment_required }
    end
  end
end
```

---

**S8. How do you implement file downloads through a Rails controller — including serving private files that require authentication?**

```ruby
class DocumentsController < ApplicationController
  before_action :authenticate_user!
  before_action :find_document
  before_action :authorize_access!

  # Option 1: Send file directly (for small files or non-cloud storage)
  def download
    send_file @document.file_path,
              filename: @document.filename,
              type: @document.content_type,
              disposition: 'attachment'
  end

  # Option 2: Generate presigned URL (for S3/cloud storage — recommended)
  def download
    presigned_url = @document.file.url(expires_in: 5.minutes)
    redirect_to presigned_url, allow_other_host: true
  end

  # Option 3: Stream file from S3 through Rails (for audit logging or DRM)
  def download
    response.headers['Content-Type'] = @document.content_type
    response.headers['Content-Disposition'] = "attachment; filename=#{@document.filename}"
    response.headers['Content-Length'] = @document.file_size

    @document.file.stream do |chunk|
      response.stream.write(chunk)
    end
  ensure
    response.stream.close
  end

  # Option 4: X-Accel-Redirect (nginx offloads serving, Rails controls auth)
  def download
    response.headers['X-Accel-Redirect'] = "/private_files/#{@document.s3_key}"
    response.headers['Content-Disposition'] = "attachment; filename=#{@document.filename}"
    head :ok
  end

  private

  def find_document
    @document = Document.find(params[:id])
  end

  def authorize_access!
    unless @document.accessible_by?(current_user)
      render file: 'public/403.html', status: :forbidden
    end
  end
end
```

---

**S9. You need to rate-limit an API endpoint to 100 requests per user per hour. How do you implement this without external gems?**

```ruby
# Using Rails.cache (backed by Redis):
class Api::BaseController < ActionController::API
  before_action :enforce_rate_limit

  RATE_LIMIT = 100
  RATE_PERIOD = 1.hour

  private

  def enforce_rate_limit
    return unless current_user  # skip for unauthenticated (handle separately)

    cache_key = "rate_limit:#{current_user.id}:#{Time.now.beginning_of_hour.to_i}"
    count = Rails.cache.increment(cache_key, 1, expires_in: RATE_PERIOD)

    # Set TTL on first hit (increment returns the new value)
    response.headers['X-RateLimit-Limit']     = RATE_LIMIT.to_s
    response.headers['X-RateLimit-Remaining'] = [RATE_LIMIT - count, 0].max.to_s
    response.headers['X-RateLimit-Reset']     = (Time.now.beginning_of_hour + RATE_PERIOD).to_i.to_s

    if count > RATE_LIMIT
      render json: {
        error: "Rate limit exceeded. Try again at #{Time.now.beginning_of_hour + RATE_PERIOD}"
      }, status: :too_many_requests
    end
  end
end

# Or use the rack-attack gem (production-grade):
# config/initializers/rack_attack.rb
Rack::Attack.throttle("api/user", limit: 100, period: 3600) do |request|
  request.env['current_user_id'] if request.path.start_with?('/api/')
end
```

---

**S10. You have a multi-step form (wizard) for user registration. How do you manage state between steps without saving incomplete records to the database?**

```ruby
# Option 1: Session-based wizard
class RegistrationsController < ApplicationController
  STEPS = %w[personal contact payment review].freeze

  def show
    @step = current_step
    @registration = RegistrationForm.new(session[:registration_data] || {})
  end

  def update
    step_params = send("#{current_step}_params")
    data = (session[:registration_data] || {}).merge(step_params)
    session[:registration_data] = data

    if step_valid?(current_step, data)
      if current_step == STEPS.last
        finalize_registration(data)
      else
        redirect_to registration_path(step: next_step)
      end
    else
      @registration = RegistrationForm.new(data)
      flash.now[:alert] = "Please fix the errors"
      render :show
    end
  end

  private

  def current_step
    params[:step].presence_in(STEPS) || STEPS.first
  end

  def next_step
    STEPS[STEPS.index(current_step) + 1]
  end

  def finalize_registration(data)
    user = User.create!(data.slice("name", "email", "password"))
    Profile.create!(user: user, **data.slice("phone", "address"))
    session.delete(:registration_data)
    sign_in(user)
    redirect_to dashboard_path, notice: "Welcome!"
  end
end

# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :phone, :string
  attribute :current_step, :string

  validates :name, :email, presence: true, if: -> { current_step == 'personal' }
  validates :phone, presence: true, if: -> { current_step == 'contact' }
end
```

---

**S11. How do you implement request logging middleware that masks sensitive parameters (like passwords, credit card numbers)?**

```ruby
# config/application.rb - built-in Rails filtering:
config.filter_parameters += [:password, :credit_card_number, :cvv, :ssn,
                              :authorization, :token, :secret, :_key,
                              :crypt, :salt, :certificate, :otp, :ssn]

# This filters params in Rails logs automatically:
# Parameters: {"email"=>"alice@example.com", "password"=>"[FILTERED]"}

# Custom middleware for request/response body logging:
class AuditLogMiddleware
  SENSITIVE_KEYS = %w[password credit_card cvv token secret].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)
    status, headers, body = @app.call(env)

    log_request(request, status)
    [status, headers, body]
  end

  private

  def log_request(request, status)
    filtered_params = deep_filter(request.filtered_parameters)
    Rails.logger.info({
      method: request.method,
      path: request.path,
      status: status,
      user_id: request.env['current_user_id'],
      params: filtered_params,
      ip: request.remote_ip
    }.to_json)
  end

  def deep_filter(hash)
    hash.transform_values do |v|
      if SENSITIVE_KEYS.any? { |key| hash.key?(key) }
        "[FILTERED]"
      elsif v.is_a?(Hash)
        deep_filter(v)
      else
        v
      end
    end
  end
end
```

---

**S12. Implement a controller action that handles bulk operations (updating multiple records at once) safely.**

```ruby
class ProductsController < ApplicationController
  def bulk_update
    operation = params.require(:operation)
    product_ids = params.require(:product_ids)

    case operation
    when "activate"
      count = Product.where(id: product_ids).update_all(status: :active, updated_at: Time.current)
      redirect_to products_path, notice: "#{count} products activated"
    when "deactivate"
      count = Product.where(id: product_ids).update_all(status: :inactive, updated_at: Time.current)
      redirect_to products_path, notice: "#{count} products deactivated"
    when "delete"
      products = Product.where(id: product_ids)
      destroyed_count = 0
      products.each do |product|
        product.destroy
        destroyed_count += 1
      end
      redirect_to products_path, notice: "#{destroyed_count} products deleted"
    else
      redirect_to products_path, alert: "Unknown operation"
    end
  rescue ActionController::ParameterMissing => e
    redirect_to products_path, alert: "Missing required parameters: #{e.param}"
  end

  # For async bulk operations (large datasets):
  def bulk_export
    product_ids = params[:product_ids] || Product.pluck(:id)
    job = BulkExportJob.perform_later(product_ids, current_user.id)
    render json: { job_id: job.job_id, status: "queued" }
  end
end
```

---

**S13. How do you implement a public API alongside a web UI in a single Rails app, sharing authentication but with different response formats?**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Web UI
  devise_for :users
  resources :dashboard
  resources :products

  # API (versioned, separate namespace)
  namespace :api, defaults: { format: :json } do
    namespace :v1 do
      devise_scope :user do
        post 'sessions', to: 'sessions#create'
        delete 'sessions', to: 'sessions#destroy'
      end
      resources :products, only: [:index, :show]
      resources :orders
    end
  end
end

# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods

      before_action :authenticate_with_token!

      private

      def authenticate_with_token!
        authenticate_or_request_with_http_token do |token, _options|
          @current_user = User.find_by(api_token: Digest::SHA256.hexdigest(token))
        end
      end
    end
  end
end

# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < BaseController
      def index
        products = Product.published.page(params[:page])
        render json: {
          products: products.map { |p| serialize_product(p) },
          pagination: pagination_meta(products)
        }
      end

      private

      def serialize_product(product)
        {
          id: product.id,
          name: product.name,
          price: product.price.to_s,
          available: product.available?,
          image_url: product.image.url
        }
      end
    end
  end
end
```

---

**S14. How would you implement server-side pagination with proper URL-based state (page number in URL, not session)?**

```ruby
# config/routes.rb
resources :articles  # GET /articles?page=2&per_page=20

# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  ALLOWED_PER_PAGE = [10, 20, 50, 100].freeze
  DEFAULT_PER_PAGE = 20

  def index
    @page     = [params[:page].to_i, 1].max
    @per_page = params[:per_page].to_i.in?(ALLOWED_PER_PAGE) ? params[:per_page].to_i : DEFAULT_PER_PAGE
    @sort     = params[:sort].presence_in(%w[title created_at updated_at]) || "created_at"
    @dir      = params[:dir] == "asc" ? :asc : :desc

    @articles = Article.published
                       .order(@sort => @dir)
                       .offset((@page - 1) * @per_page)
                       .limit(@per_page + 1)  # +1 to detect "has more"

    @has_more = @articles.size > @per_page
    @articles = @articles.first(@per_page)

    respond_to do |format|
      format.html
      format.json do
        render json: {
          articles: @articles,
          pagination: {
            page: @page,
            per_page: @per_page,
            has_more: @has_more,
            next_page: @has_more ? @page + 1 : nil,
            prev_page: @page > 1 ? @page - 1 : nil
          }
        }
      end
    end
  end
end
```

---

**S15. A controller action is 150 lines long with complex business logic. How do you refactor it?**

```ruby
# Before: fat controller (bad)
class CheckoutsController < ApplicationController
  def create
    @cart = Cart.find(params[:cart_id])
    # 20 lines of validation
    # 30 lines of payment processing
    # 15 lines of inventory management
    # 20 lines of email notifications
    # 10 lines of loyalty points calculation
    # 15 lines of analytics tracking
    # 40 lines of error handling
  end
end

# After: thin controller (good)
class CheckoutsController < ApplicationController
  def create
    result = Checkout::ProcessOrderCommand.new(
      cart_id:        params[:cart_id],
      user:           current_user,
      payment_params: payment_params,
      shipping_params: shipping_params
    ).execute

    if result.success?
      redirect_to order_confirmation_path(result.order), notice: "Order placed!"
    else
      flash.now[:errors] = result.errors
      @cart = result.cart
      render :new, status: :unprocessable_entity
    end
  end

  private

  def payment_params
    params.require(:payment).permit(:stripe_token, :billing_address, :save_card)
  end

  def shipping_params
    params.require(:shipping).permit(:name, :address, :city, :postal_code, :country)
  end
end

# The actual work in app/commands/checkout/process_order_command.rb
# Testable independently of HTTP layer
# Controller only handles: parameter parsing, auth, routing the response
```

