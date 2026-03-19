# Chapter 01 — Rails MVC Architecture: Scenario Questions

---

**S1. A junior developer reports that a change they made to a model is not being reflected when they refresh the browser in development. The server log shows the old code being executed. What could cause this and how do you diagnose it?**

Consider: Zeitwerk not watching the file (non-standard path), `config.cache_classes = true` accidentally set in development, a syntax error preventing file load, Spring (preloader) caching old code, or a constant defined in an initializer (initializers don't reload).

---

**S2. You need to add request timing (log the time taken by each request) to your Rails app. You want this to work for ALL requests, even those that throw exceptions. How do you implement this as Rack middleware?**

```ruby
class RequestTimer
  def initialize(app)
    @app = app
  end

  def call(env)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    Rails.logger.info "Request took #{(duration * 1000).round(2)}ms"
    [status, headers, body]
  end
end

# config/application.rb
config.middleware.use RequestTimer
```

Follow-up: Why use `Process.clock_gettime` instead of `Time.now`?

---

**S3. Your Rails app is deployed and users report that after a deploy, some requests get a 500 error for about 30 seconds, then everything stabilizes. What Rails mechanism might cause this and how do you fix it?**

This is likely `config.active_record.migration_error = :page_load` or code/database schema mismatch during rolling deploys. Also could be Zeitwerk loading issues, or Puma workers warming up. Consider: zero-downtime deploy strategy, `maintenance_tasks` gem, blue-green deployments.

---

**S4. A request to `GET /users/1` is returning a 404, but you confirmed the route exists in `config/routes.rb` with `resources :users`. Walking through the Rails stack, where would you check to diagnose this?**

Steps:
1. `rails routes | grep users` — confirm route is there
2. Check controller: `UsersController#show` exists?
3. Check `before_action` — is one redirecting early?
4. Check `User.find(params[:id])` — is record 1 missing? (404 from ActiveRecord::RecordNotFound)
5. Check mounted engines or route constraints that might intercept

---

**S5. You want to serve static files from a custom directory `/data/uploads` that is outside the Rails `public/` folder. How do you configure this in Rails using Rack middleware?**

```ruby
# config/application.rb
config.middleware.insert_before ActionDispatch::Static,
  Rack::Static,
  urls: ["/uploads"],
  root: "/data"

# Or use X-Accel-Redirect / X-Sendfile with nginx:
config.action_dispatch.x_sendfile_header = "X-Accel-Redirect"
```

---

**S6. Your application has both a web UI and a JSON API. The web UI needs session-based auth, the web API needs JWT auth. How do you architect this in a single Rails application?**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Web UI (full stack)
  scope module: :web do
    resources :dashboard
    resource :session
  end

  # API namespace (API-only module)
  namespace :api do
    namespace :v1 do
      resources :users
      resources :products
    end
  end
end

# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    before_action :authenticate_token!

    private

    def authenticate_token!
      token = request.headers['Authorization']&.split(' ')&.last
      @current_user = JwtService.decode(token)
    rescue JWT::DecodeError
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
end

# app/controllers/web/base_controller.rb
module Web
  class BaseController < ApplicationController
    before_action :authenticate_user!
  end
end
```

---

**S7. You're tasked with adding a feature flag system to Rails. All feature flags should be checked in milliseconds, not hit the database on every request. How do you implement this?**

```ruby
# Use Rails.cache backed by Redis for in-memory flag checks
class FeatureFlag
  CACHE_TTL = 5.minutes

  def self.enabled?(flag_name, user: nil)
    flags = Rails.cache.fetch("feature_flags", expires_in: CACHE_TTL) do
      FeatureFlagRecord.all.index_by(&:name)
    end

    flag = flags[flag_name.to_s]
    return false unless flag&.enabled?

    return true unless flag.user_percentage < 100
    user.present? && (user.id % 100) < flag.user_percentage
  end
end
```

---

**S8. Your Rails app was running fine but after upgrading from Rails 6 to Rails 7, you get `NameError: uninitialized constant` errors for classes that worked before. What are the likely causes?**

Causes:
1. Classic → Zeitwerk autoloader migration: files must follow strict naming conventions (`app/models/admin/user.rb` → `Admin::User`, not `User`)
2. Circular dependencies that Classic autoloader accidentally resolved
3. Constants defined in `config/initializers/` that reference application constants (initializers run before app code is autoloaded)
4. Missing `# frozen_string_literal: true` in some contexts
5. Concerns or modules with non-standard file locations

```ruby
# Fix: check with Zeitwerk itself
Rails.application.reloader.check!
# or
Rails.autoloaders.main.check!
```

---

**S9. A senior developer says "never put business logic in controllers." You're building a checkout feature. How do you structure the code following Rails best practices?**

```ruby
# app/services/checkout_service.rb
class CheckoutService
  include ActiveModel::Model

  attr_reader :order, :payment_result

  def initialize(cart:, user:, payment_params:)
    @cart = cart
    @user = user
    @payment_params = payment_params
  end

  def call
    ActiveRecord::Base.transaction do
      @order = Order.create!(user: @user, total: @cart.total)
      @cart.items.each { |item| order.line_items.create!(item.attributes) }
      @payment_result = PaymentGateway.charge(@payment_params.merge(amount: order.total))
      raise ActiveRecord::Rollback unless @payment_result.success?
      order.paid!
    end
    @payment_result.success?
  end
end

# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  def create
    service = CheckoutService.new(
      cart: current_cart,
      user: current_user,
      payment_params: payment_params
    )

    if service.call
      redirect_to order_path(service.order), notice: "Order placed!"
    else
      flash[:error] = "Payment failed"
      render :new
    end
  end
end
```

---

**S10. Your Rails app's test suite runs Zeitwerk's eager load check but fails with a `Zeitwerk::NameError`. How do you diagnose and fix it?**

```ruby
# In spec/rails_helper.rb or a dedicated spec:
RSpec.describe "Zeitwerk" do
  it "eager loads all files without error" do
    expect { Rails.application.eager_load! }.not_to raise_error
  end
end

# Diagnose: run in console
Rails.application.eager_load!
# Zeitwerk will show exactly which file/constant has the mismatch

# Common causes:
# - app/models/admin_user.rb containing class Admin::User (should be in app/models/admin/user.rb)
# - app/services/pdf_generator.rb containing class PDFGenerator (Zeitwerk maps to PdfGenerator)
```

---

**S11. You need to handle a webhook from a third-party service. The webhook requires raw request body for HMAC signature verification, but Rails' middleware parses the body before your controller sees it. How do you solve this?**

```ruby
# config/application.rb
# Rails 7.x: skip parsing for webhook endpoint
config.middleware.insert_before ActionDispatch::PermissionPolicy::Middleware,
  Rack::RawBody

# Or use a middleware to cache the raw body:
class RawBodyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    if env['PATH_INFO'] == '/webhooks/stripe'
      env['RAW_POST_DATA'] = env['rack.input'].read
      env['rack.input'] = StringIO.new(env['RAW_POST_DATA'])
    end
    @app.call(env)
  end
end

# In the controller:
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def stripe
    payload = request.raw_post
    sig_header = request.env['HTTP_STRIPE_SIGNATURE']

    event = Stripe::Webhook.construct_event(payload, sig_header, ENV['STRIPE_WEBHOOK_SECRET'])
    # process event...
    head :ok
  rescue Stripe::SignatureVerificationError
    head :bad_request
  end
end
```

---

**S12. Your production Rails app is slow to boot (45+ seconds). How do you diagnose and reduce boot time?**

```bash
# Profile boot time
RAILS_ENV=production time bundle exec rails runner "puts 'done'"

# Use bootsnap (usually already in Gemfile)
# gem 'bootsnap', require: false (in config/boot.rb: require 'bootsnap/setup')

# Check initializers:
# Add timing to each initializer temporarily
# config/initializers files — look for ones hitting DB or making HTTP calls at boot

# Check eager_load: large apps load 1000s of files
# Use rails stats to see file counts
```

```ruby
# config/boot.rb — ensure bootsnap is set up
require 'bootsnap/setup'

# Audit heavy initializers:
# config/initializers/some_service.rb
# BAD: making HTTP call at boot
REMOTE_CONFIG = HTTParty.get("https://config-service.example.com/config")

# GOOD: lazy load
class RemoteConfig
  def self.fetch
    @config ||= HTTParty.get("https://config-service.example.com/config")
  end
end
```

---

**S13. Explain how you would implement a multi-tenant Rails application using subdomain-based routing (`company1.app.com`, `company2.app.com`).**

```ruby
# config/routes.rb
constraints SubdomainConstraint do
  scope module: :tenanted do
    resources :projects
    resources :users
  end
end

# lib/subdomain_constraint.rb
class SubdomainConstraint
  EXCLUDED = %w[www api admin].freeze

  def self.matches?(request)
    request.subdomain.present? && !EXCLUDED.include?(request.subdomain)
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_tenant

  private

  def set_current_tenant
    subdomain = request.subdomain
    @current_tenant = Account.find_by!(subdomain: subdomain)
    ActsAsTenant.current_tenant = @current_tenant
  rescue ActiveRecord::RecordNotFound
    render file: 'public/404.html', status: :not_found
  end
end
```

---

**S14. A developer on your team accidentally committed `config.consider_all_requests_local = true` to the production environment file. What security risk does this create?**

`consider_all_requests_local = true` causes Rails to render the full exception backtrace (including file paths, local variables, and source code) in the browser error page instead of a generic 500 page. This leaks application structure, gem paths, database queries, and potentially sensitive variable values to any user who triggers an error. Immediate mitigation: revert the commit, redeploy, and rotate any credentials visible in error pages.

---

**S15. You need to add a Content Security Policy (CSP) header to your Rails application. How do you configure it using Rails' built-in DSL?**

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https

  # Allow Turbo/Stimulus inline scripts
  policy.script_src_attr :unsafe_inline

  # Nonce-based CSP for inline scripts
  policy.script_src :self, :https, -> { "'nonce-#{request.content_security_policy_nonce}'" }
end

# Enable nonce generation
Rails.application.config.content_security_policy_nonce_generator =
  -> (request) { SecureRandom.base64(16) }

Rails.application.config.content_security_policy_nonce_directives = %w[script-src]

# In views, use the nonce:
# <%= javascript_tag nonce: true do %>
#   console.log("secure inline script");
# <% end %>
```

