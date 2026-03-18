# Chapter 01 — Rails MVC Architecture: Theory Questions

---

## MVC Pattern & Request Lifecycle

**Q1. Describe the full Rails MVC request lifecycle from the moment a browser sends a GET request to when the HTML response is returned.**

Expected depth: router → middleware stack → ActionDispatch → controller instantiation → before_actions → action method → model interaction → view rendering → response.

---

**Q2. What is Rack and what role does it play in Rails?**

Describe the Rack specification (`call(env)` returning `[status, headers, body]`), how Rails is a Rack application, and why this matters for middleware composition.

---

**Q3. What is ActionDispatch and how does it relate to Rack?**

ActionDispatch is Rails' request/response handling layer. Explain `ActionDispatch::Request`, `ActionDispatch::Response`, routing integration, and its middleware components.

---

**Q4. Walk through what happens inside `config/routes.rb` — how does Rails match an incoming URL to a controller action?**

Cover `ActionDispatch::Routing::RouteSet`, route recognition, route generation, the `routes.rb` DSL, and what `rails routes` shows.

---

**Q5. What is the Rack middleware stack in a default Rails 7 application? Name at least 6 middleware and explain what each does.**

Example: `Rack::Sendfile`, `ActionDispatch::Static`, `ActionDispatch::Executor`, `ActiveSupport::Cache::Strategy::LocalCache::Middleware`, `Rack::Runtime`, `ActionDispatch::RequestId`, `ActionDispatch::RemoteIp`, `Sprockets::Rails::QuietAssets`, `ActionDispatch::Reloader`, `ActionDispatch::Callbacks`, `ActiveRecord::Migration::CheckPending`, `ActionDispatch::Cookies`, `ActionDispatch::Session::CookieStore`, `ActionDispatch::Flash`, `ActionDispatch::ContentSecurityPolicy::Middleware`, `Rack::MethodOverride`, `ActionDispatch::Head`, `Rack::ConditionalGet`, `Rack::ETag`, `Rack::TempfileReaper`.

---

**Q6. How do you add custom Rack middleware to a Rails application? What are the differences between `config.middleware.use`, `config.middleware.insert_before`, and `config.middleware.insert_after`?**

```ruby
# config/application.rb
config.middleware.use MyCustomMiddleware
config.middleware.insert_before ActionDispatch::Static, AnotherMiddleware
config.middleware.insert_after Rack::Runtime, TimingMiddleware
```

---

**Q7. What is Convention over Configuration (CoC) in Rails? Give 5 specific examples.**

Examples: file naming → class naming, table naming → model naming, `app/models/user.rb` contains `User`, `app/controllers/users_controller.rb` contains `UsersController`, foreign keys (`user_id`), join tables (`articles_tags` alphabetical order).

---

**Q8. Explain Zeitwerk, Rails 7's autoloader. How does it differ from the Classic autoloader?**

Cover: file → constant mapping, `Kernel#require` not needed for app code, eager loading in production, reloading in development, namespace handling with modules.

---

**Q9. What is eager loading in Rails? Why is `config.eager_load = true` set only in production?**

Explain that eager loading loads all Ruby files at boot (via `eager_load_paths`), prevents autoloading issues in multi-threaded environments, is slow at startup (acceptable in production, annoying in development).

---

**Q10. What files does `config/application.rb` configure? What is the difference between `config/application.rb`, `config/environment.rb`, and `config/environments/production.rb`?**

- `application.rb`: application-wide config loaded in all environments
- `environment.rb`: requires `application.rb`, calls `Rails.application.initialize!`
- `environments/production.rb`: environment-specific overrides

---

**Q11. What are Rails initializers? How do they differ from `config/application.rb` settings? In what order do initializers run?**

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*', headers: :any, methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

Initializers run after the app is configured but before it handles requests. They run alphabetically within `config/initializers/`.

---

**Q12. What are the three Rails environments (development, test, production) and what are the key differences in their configurations?**

| Setting | Development | Test | Production |
|---------|-------------|------|------------|
| `config.cache_classes` | false | true | true |
| `config.eager_load` | false | false | true |
| `config.consider_all_requests_local` | true | true | false |
| `config.action_mailer.perform_deliveries` | false | false | true |

---

**Q13. What is `config.cache_classes` (or `config.enable_reloading` in Rails 7.1+)? How does code reloading work in development?**

In development, Rails watches files for changes using `ActiveSupport::FileWatcher` (Listen or Evented). When a file changes, Zeitwerk unloads the constant and re-requires the file on the next request.

---

**Q14. Describe the Rails asset pipeline. What problem does it solve and what are the key components?**

Covers: Sprockets (asset compilation, concatenation, fingerprinting), asset helper methods (`asset_path`, `javascript_include_tag`, `stylesheet_link_tag`), manifest files, fingerprinting for cache busting, public/assets vs app/assets.

---

**Q15. What is Propshaft? How does it differ from Sprockets?**

Propshaft is Rails' modern asset pipeline (used with Rails 7+ when you don't need compilation). It does NOT compile/transpile — it only digests (fingerprints) and copies assets. No SASS compilation. Designed for use with import maps or jsbundling-rails.

---

**Q16. What is the difference between Webpacker, import maps, jsbundling-rails, and esbuild in the Rails ecosystem?**

- **Webpacker**: Webpack wrapped for Rails (deprecated, removed as default in Rails 7)
- **Import maps**: Native browser ES modules, no bundling needed, default in Rails 7
- **jsbundling-rails**: Glue gem for external bundlers (esbuild, rollup, webpack)
- **esbuild**: Very fast JS bundler, used via jsbundling-rails

---

**Q17. What is ActionController::Base vs ActionController::API? What features does ActionController::API omit?**

`ActionController::API` omits: cookie management, flash, session, view rendering, CSRF protection, `before_action` filter for authenticity token. Used for API-only applications.

---

**Q18. How does Rails routing handle the `format` segment (`.json`, `.html`, `.xml`)? What is content negotiation in Rails?**

The `format` can come from the URL extension, the `Accept` header, or explicit params. `respond_to` blocks handle different formats:

```ruby
def show
  @user = User.find(params[:id])
  respond_to do |format|
    format.html
    format.json { render json: @user }
    format.xml  { render xml: @user }
  end
end
```

---

**Q19. What is `ActionDispatch::Session::CookieStore`? How does Rails store session data? What are the alternatives?**

CookieStore signs and optionally encrypts the session hash and stores it in the browser cookie. Limited to 4KB. Alternatives: `ActionDispatch::Session::CacheStore` (in Redis/Memcached), `ActionDispatch::Session::ActiveRecordStore` (database).

---

**Q20. What is CSRF protection in Rails? How does `protect_from_forgery` work?**

Rails generates an authenticity token stored in the session and embeds it in forms. On non-GET requests, it validates the submitted token against the session token. In Rails 7, the default is `protect_from_forgery with: :exception`.

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

---

**Q21. Explain the Rails boot sequence — from `bundle exec rails server` to the first request being handled.**

1. Bundler loads gems
2. `config/boot.rb` sets up Bundler
3. `config/application.rb` loads Rails framework + config
4. `config/environment.rb` calls `Rails.application.initialize!`
5. Initializers run (alphabetically)
6. Zeitwerk sets up autoload paths
7. Middleware stack is assembled
8. Puma starts listening on port 3000

---

**Q22. What is `config.autoload_paths` vs `config.eager_load_paths`? Can you add custom directories to them?**

```ruby
# config/application.rb
config.autoload_paths << Rails.root.join('lib')
config.eager_load_paths << Rails.root.join('lib')
```

`autoload_paths`: Zeitwerk watches these for constant resolution.
`eager_load_paths`: All `.rb` files here are required at boot in production.

---

**Q23. What happens when you call `render` twice in a controller action? What error does it produce?**

`AbstractController::DoubleRenderError: Render and/or redirect was called multiple times in this action.`

```ruby
# Bug:
def show
  render plain: "Hello"
  render plain: "World"  # => DoubleRenderError
end

# Fix:
def show
  return render plain: "Hello" if some_condition
  render plain: "World"
end
```

---

**Q24. What is `ActionDispatch::RemoteIp` middleware and why does it matter for security?**

It determines the client's real IP from `X-Forwarded-For` and `REMOTE_ADDR`. Important for IP-based rate limiting, geolocation, and security. Malicious clients can spoof `X-Forwarded-For`; the middleware has trusted proxy configuration.

---

**Q25. What is the difference between `config.force_ssl` and adding SSL at the load balancer level?**

`config.force_ssl = true` adds `ActionDispatch::SSL` middleware which redirects HTTP → HTTPS and sets HSTS headers. At the load balancer level, SSL is terminated before reaching Rails. Both approaches are valid; the middleware approach also sets `Strict-Transport-Security`.

---

**Q26. What is `Rack::MethodOverride` and why is it needed?**

HTML forms only support GET and POST. `Rack::MethodOverride` checks for a `_method` hidden field (or `X-HTTP-Method-Override` header) and overrides `REQUEST_METHOD` with the value (PUT, PATCH, DELETE). Rails' `form_with` and `form_tag` add this automatically for non-GET/POST forms.

---

**Q27. How does the Rails router generate URL helpers? What is the difference between `_path` and `_url` helpers?**

`_path` generates a relative path (`/users/1`), `_url` generates a full URL (`http://example.com/users/1`). URL helpers are generated by `ActionDispatch::Routing::RouteSet` based on `config/routes.rb` named routes. Always use `_url` in mailers.

