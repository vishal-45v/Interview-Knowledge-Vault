# Chapter 01 — Rails MVC Architecture: Structured Answers

---

## Answer 1: Complete Rails Request Lifecycle

**Question:** Walk through the complete lifecycle of a `GET /users/1` request in a Rails 7 application.

**Answer:**

The lifecycle has 7 distinct phases:

### Phase 1: TCP/HTTP Layer
The browser establishes a TCP connection to the server (Puma). Puma receives the raw HTTP request and parses it into a Rack `env` hash.

### Phase 2: Rack Middleware Stack
The request passes through the middleware stack from outermost to innermost. Key middleware in order:
- `ActionDispatch::RequestId` — assigns a unique `X-Request-Id`
- `ActionDispatch::RemoteIp` — determines real client IP from `X-Forwarded-For`
- `Rack::Runtime` — adds `X-Runtime` header to response
- `ActionDispatch::Cookies` — parses Cookie header, makes `cookies[]` available
- `ActionDispatch::Session::CookieStore` — decrypts/deserializes session cookie
- `ActionDispatch::Flash` — populates flash from session

### Phase 3: Routing
`ActionDispatch::Routing::RouteSet#call` is invoked. The router matches `GET /users/1` against the route table (compiled by `config/routes.rb`). It determines `controller: 'users', action: 'show', id: '1'` and sets `env['action_dispatch.request.path_parameters']`.

### Phase 4: Controller Instantiation
Rails instantiates `UsersController`. It merges path parameters into `params`. The `before_action` filter chain runs — for example, `authenticate_user!` from Devise.

### Phase 5: Action Execution
`UsersController#show` executes. Typically: `@user = User.find(params[:id])`. ActiveRecord issues a SQL query to the database connection pool (checked out from the pool).

### Phase 6: View Rendering
Rails determines the template to render (`app/views/users/show.html.erb`). The layout wraps it. ERB is evaluated, producing an HTML string. Any partials are rendered. Fragment cache hits skip rendering.

### Phase 7: Response
Rails sets `Content-Type: text/html; charset=utf-8`, status 200, and the rendered body. The middleware stack unwinds — each middleware can add headers, compress the body, etc. Puma writes the HTTP response back to the socket.

```
Browser → Puma → Rack Middleware Stack → Router → Controller
                                                       ↓
Browser ← Puma ← Rack Middleware Stack ← View Rendering ← Model (DB)
```

---

## Answer 2: Zeitwerk Autoloading — How It Works

**Question:** Explain Zeitwerk autoloading. How does it map files to constants? What is eager loading?

**Answer:**

Zeitwerk is a code loader for Ruby that Rails uses. It implements the autoloading feature where you can reference constants (class/module names) without explicitly requiring their files.

**File-to-Constant Mapping:**

Zeitwerk builds a mapping from file paths to expected constant names at boot time:

```
app/models/user.rb              → User
app/models/admin/user.rb        → Admin::User
app/services/pdf_generator.rb   → PdfGenerator
app/jobs/send_email_job.rb      → SendEmailJob
```

Rules:
- Underscores become CamelCase: `send_email` → `SendEmail`
- Directory nesting maps to module nesting: `admin/user` → `Admin::User`
- Acronyms need explicit configuration in `config/initializers/inflections.rb`

**How Autoloading Works:**

When Ruby encounters an uninitialized constant `User`, Zeitwerk's `const_missing` hook fires. Zeitwerk checks its mapping, finds `app/models/user.rb`, requires it, and returns the constant.

```ruby
# This just works — no require needed:
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])  # User constant auto-resolved
  end
end
```

**Eager Loading:**

In production, lazy autoloading is unsafe for multi-threaded servers (two threads could try to autoload the same constant simultaneously, causing race conditions). Eager loading solves this by loading ALL constants at boot:

```ruby
# config/environments/production.rb
config.eager_load = true  # Rails.application.eager_load! is called at startup
```

This makes startup slower but eliminates per-request autoloading entirely. Every class is already loaded.

**Reloading in Development:**

```ruby
# config/environments/development.rb
config.enable_reloading = true  # Rails 7.1+ (was config.cache_classes = false)
```

The file watcher (Listen gem) detects changes. On the next request, Zeitwerk unloads changed constants and their descendants, then autoloads them fresh on next access.

---

## Answer 3: Writing a Rack Middleware

**Question:** Write a Rack middleware that logs all requests with their method, path, status code, and duration.

**Answer:**

```ruby
# lib/middleware/request_logger.rb
class RequestLogger
  def initialize(app, logger: Rails.logger)
    @app    = app
    @logger = logger
  end

  def call(env)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    status, headers, body = @app.call(env)

    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)

    request = Rack::Request.new(env)
    @logger.info(
      "#{request.request_method} #{request.path} → #{status} (#{duration_ms}ms)"
    )

    [status, headers, body]
  rescue StandardError => e
    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)
    @logger.error "#{env['REQUEST_METHOD']} #{env['PATH_INFO']} → ERROR (#{duration_ms}ms): #{e.message}"
    raise
  end
end

# config/application.rb
config.middleware.use RequestLogger
# Or with custom logger:
config.middleware.use RequestLogger, logger: Logger.new($stdout)
```

Key design decisions:
- `Process.clock_gettime(CLOCK_MONOTONIC)` is immune to system clock changes (NTP adjustments, daylight saving), unlike `Time.now`
- `rescue StandardError => e; raise` ensures the exception propagates while still logging the error
- The middleware yields `[status, headers, body]` unchanged — pure observation, no mutation

---

## Answer 4: Convention Over Configuration — 5 Concrete Examples

**Question:** Give 5 concrete examples of Convention over Configuration in Rails.

**Answer:**

1. **Model ↔ Table naming:**
   ```ruby
   class Order < ApplicationRecord; end
   # Rails assumes table name: "orders" (pluralized, underscored)
   # No configuration needed unless you want to override:
   # self.table_name = "purchase_orders"
   ```

2. **Controller ↔ View path:**
   ```ruby
   class ProductsController < ApplicationController
     def index
       # Rails looks for app/views/products/index.html.erb
       # No render call needed
     end
   end
   ```

3. **Foreign keys in associations:**
   ```ruby
   class Comment < ApplicationRecord
     belongs_to :post
     # Rails assumes foreign key: post_id column on comments table
     # Override: belongs_to :post, foreign_key: :article_id
   end
   ```

4. **Join table naming:**
   ```ruby
   class Article < ApplicationRecord
     has_and_belongs_to_many :tags
     # Rails assumes join table: "articles_tags" (alphabetical order, pluralized)
   end
   ```

5. **Primary key:**
   ```ruby
   # Rails assumes every table has an id integer/bigint primary key
   # No configuration needed for standard CRUD
   # Override per model: self.primary_key = "uuid"
   ```

---

## Answer 5: Middleware Order Matters — Example

**Question:** Give a concrete example where middleware ORDER in the stack causes a bug.

**Answer:**

```ruby
# Bug scenario: Authentication middleware runs BEFORE session is loaded

# WRONG order:
config.middleware.use AuthenticationMiddleware   # runs first
# ...
# ActionDispatch::Session::CookieStore             # runs later — session not ready yet!

# In AuthenticationMiddleware#call:
def call(env)
  session = env['rack.session']  # nil! CookieStore hasn't run yet
  # ...
end

# CORRECT — insert AFTER session middleware:
config.middleware.insert_after ActionDispatch::Session::CookieStore, AuthenticationMiddleware

# Another real example: CORS middleware must run before routing
# If routing rejects the request first, the CORS headers are never set,
# causing preflight OPTIONS requests to fail with 404
config.middleware.insert_before ActionDispatch::Routing::RouteSet, Rack::Cors do
  allow { origins '*'; resource '*', headers: :any, methods: [:options, :get, :post] }
end
```

---

## Answer 6: Development vs Production Differences

**Question:** A new Rails developer asks "why does my app behave differently in development vs production?" Explain the key differences.

**Answer:**

```ruby
# config/environments/development.rb — KEY settings:
config.enable_reloading = true          # Code reloads on each request
config.eager_load = false               # Classes loaded on demand
config.consider_all_requests_local = true  # Full error pages shown
config.action_controller.perform_caching = false  # No fragment caching
config.active_record.migration_error = :page_load # Show pending migration banner
config.assets.debug = true              # Serve assets uncompiled, individually
config.log_level = :debug               # Verbose SQL and all log levels

# config/environments/production.rb — KEY settings:
config.eager_load = true                # ALL classes loaded at boot
config.consider_all_requests_local = false  # Generic 500 page for errors
config.force_ssl = true                 # HTTPS only
config.action_controller.perform_caching = true  # Fragment caching enabled
config.assets.compile = false          # Assets must be precompiled (rails assets:precompile)
config.log_level = :info               # Less verbose
config.action_mailer.perform_deliveries = true   # Actually send emails
```

The most dangerous pitfall: an app "works in development" because:
- Autoloading silently fixes missing requires
- Database is local and fast (no N+1 performance issues visible)
- No caching means always fresh data
- Full error pages hide the true nature of exceptions

---

## Answer 7: CSRF Protection — How It Works End-to-End

**Question:** Explain CSRF and how Rails protects against it.

**Answer:**

**The Attack:** A malicious site embeds `<form action="https://bank.com/transfer" method="POST">`. If a logged-in user visits the malicious site, the form submits with their cookies (including session cookie), performing actions on their behalf.

**Rails' Defense:**

```ruby
# ApplicationController — enabled by default:
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

On form load:
1. Rails generates a random 128-bit token and stores it in the session
2. The token is embedded in every form as a hidden field (`authenticity_token`) and as a meta tag for AJAX

```html
<!-- Generated by Rails in every form: -->
<input type="hidden" name="authenticity_token"
       value="abc123...very_long_token...xyz">

<!-- In HTML head (for AJAX): -->
<meta name="csrf-token" content="abc123...very_long_token...xyz">
```

On form submission:
1. Rails reads the submitted `authenticity_token`
2. Compares it with the session's stored token using `ActiveSupport::SecurityUtils.secure_compare` (constant-time comparison to prevent timing attacks)
3. Mismatch → `ActionController::InvalidAuthenticityToken` → `with: :exception` raises, OR `with: :null_session` clears the session, OR `with: :reset_session` resets it

```ruby
# For API endpoints that use JWT instead of sessions:
class Api::BaseController < ActionController::Base
  skip_before_action :verify_authenticity_token  # Safe because no session/cookie auth
end
# Or use ActionController::API which doesn't include forgery protection at all
```

---

## Answer 8: ActionDispatch::Static and Asset Serving

**Question:** How does Rails serve static assets? When should you let Rails serve them vs nginx?

**Answer:**

```ruby
# config/environments/production.rb
config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?
# Or in Rails 7:
config.public_file_server.enabled = true
```

`ActionDispatch::Static` middleware checks if a request path matches a file in `public/`. If it does, it serves the file directly from Ruby — bypassing the entire Rails stack. This is simple but slow.

**Why nginx is better for static files:**

nginx can serve files at ~10x the throughput with 10x less memory because:
- No Ruby process involved
- OS-level `sendfile()` syscall (zero-copy transfer)
- Built-in gzip encoding
- `Cache-Control` headers configured per directory

```nginx
# nginx config for Rails static files:
location ^~ /assets/ {
  gzip_static on;
  expires 1y;
  add_header Cache-Control public;
  add_header ETag "";
  break;
}
```

Rule: In production with nginx, set `config.public_file_server.enabled = false`. Let nginx handle `/assets/` and `/packs/`. Only enable Rails static serving on Heroku or platforms where you can't configure nginx.

---

## Answer 9: Rack Application Interface

**Question:** Implement a minimal Rack application (not using Rails) that responds to `GET /health` with `{ "status": "ok" }`.

**Answer:**

```ruby
# config.ru — Rack application file

require 'json'

class HealthApp
  HEALTH_RESPONSE = [
    200,
    { 'Content-Type' => 'application/json' },
    [JSON.generate({ status: 'ok', time: Time.now.iso8601 })]
  ].freeze

  NOT_FOUND_RESPONSE = [
    404,
    { 'Content-Type' => 'application/json' },
    [JSON.generate({ error: 'Not Found' })]
  ].freeze

  def call(env)
    request = Rack::Request.new(env)

    if request.get? && request.path == '/health'
      # Build a fresh response each time (not frozen — Time changes)
      [200, { 'Content-Type' => 'application/json' },
       [JSON.generate({ status: 'ok', time: Time.now.iso8601 })]]
    else
      NOT_FOUND_RESPONSE
    end
  end
end

run HealthApp.new
```

This is the Rack contract: `call(env)` returns `[Integer, Hash, Array]` — status, headers, body. The body must respond to `each`. Rails is a Rack app with this same interface, just layered with middleware and routing on top.

---

## Answer 10: Configuring Multiple Environments

**Question:** You need a `staging` environment that behaves like production but with full error pages. How do you set it up?

**Answer:**

```bash
# Create the environment file:
touch config/environments/staging.rb
```

```ruby
# config/environments/staging.rb
require "active_support/core_ext/integer/time"

Rails.application.configure do
  # Behave like production:
  config.cache_classes = true
  config.eager_load = true
  config.log_level = :info
  config.force_ssl = true
  config.assets.compile = false
  config.action_controller.perform_caching = true
  config.action_mailer.perform_deliveries = true
  config.active_record.migration_error = :page_load

  # But show full error pages (unlike production):
  config.consider_all_requests_local = true  # Full error backtrace

  # Use staging-specific credentials:
  # RAILS_MASTER_KEY should point to staging key
end
```

```yaml
# config/database.yml
staging:
  <<: *default
  database: myapp_staging
  url: <%= ENV['DATABASE_URL'] %>
```

```bash
# Run with staging environment:
RAILS_ENV=staging bundle exec rails server
RAILS_ENV=staging bundle exec rails db:migrate
```

Important: Add `staging` to the `RAILS_ENV` values your credentials file supports, or use a separate `config/credentials/staging.yml.enc`.

