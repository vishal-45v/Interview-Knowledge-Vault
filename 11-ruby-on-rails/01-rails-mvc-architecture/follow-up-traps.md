# Chapter 01 — Rails MVC Architecture: Follow-Up Traps

These questions are designed to probe shallow understanding. A candidate who memorized definitions will stumble here.

---

**Trap 1: "You said Zeitwerk maps filenames to constants. What happens if I have a file `app/models/html_parser.rb`? What constant does Zeitwerk expect?"**

The trap: Many candidates say `HtmlParser`. The correct answer is `HtmlParser` — Zeitwerk uses `String#camelize` from ActiveSupport which does a simple `gsub(/(?:^|_)(.)/) { $1.upcase }`. So `html_parser` → `HtmlParser`. However, if you name your class `HTMLParser` (all caps acronym), Zeitwerk will raise a `NameError` because it expects `HtmlParser`.

```ruby
# Zeitwerk expects:
# app/models/html_parser.rb → class HtmlParser
# app/models/xml_document.rb → class XmlDocument

# This FAILS with Zeitwerk:
# app/models/html_parser.rb containing class HTMLParser

# Fix: use inflector
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym 'HTML'
end
# Now Zeitwerk expects HTMLParser for html_parser.rb
```

---

**Trap 2: "You mentioned eager_load is false in development to speed up startup. So code reloading is free? What's the actual cost?"**

The trap: Candidates assume reloading has no cost. In reality, every request in development that triggers a reload must re-require changed files, which involves filesystem polling (the Listen gem watching files), class unloading (removing constants), and re-evaluation. With large apps this can add 100–500ms per-request in development.

Also: Spring preloader caches code between runs (for `rails console`, `rails test`). Forgetting to `spring stop` after changing initializers or Gemfile causes stale behaviour.

---

**Trap 3: "You said Rack middleware wraps the request. Can you add middleware that only runs for a specific route?"**

The trap: Candidates say "no, middleware runs for everything." While it's technically true that Rack middleware runs for all requests, you can implement route-based behaviour inside the middleware:

```ruby
class AdminOnlyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    if env['PATH_INFO'].start_with?('/admin')
      # do admin-specific things
    end
    @app.call(env)
  end
end
```

The better Rails answer: Use `before_action` in controllers for route-specific logic. Middleware is for cross-cutting concerns.

---

**Trap 4: "If `config.cache_classes = false` in development, does that mean every class is reloaded on every request?"**

No — only changed classes are reloaded. The file watcher (Listen) detects which `.rb` files changed since the last request. Zeitwerk then unloads only those constants and their dependencies. Unchanged classes remain loaded. This is called "selective reloading."

The subtlety: If class A depends on class B and B changes, A must also be reloaded. Zeitwerk tracks dependencies to handle this.

---

**Trap 5: "You mentioned `ActionController::API` omits cookies and sessions. But a candidate says their API controllers use `current_user` which comes from a Devise session. Is that possible?"**

Yes, but only if the API controller inherits from `ApplicationController < ActionController::Base` (not `ActionController::API`) or if they manually include session modules:

```ruby
# Including session support in an API controller:
class Api::BaseController < ActionController::API
  include ActionController::Cookies
  include ActionController::RequestForgeryProtection
end
```

However, this is unusual. Most API applications use token auth to avoid session overhead. The trap reveals whether the candidate understands what `ActionController::API` actually strips out.

---

**Trap 6: "The router matches routes in order. What happens if I define two routes that could match the same URL?"**

The first matching route wins. Rails routes are matched top-to-bottom in `config/routes.rb`:

```ruby
# config/routes.rb
get '/users/new', to: 'users#new'   # matches /users/new
resources :users                     # also generates GET /users/new → users#new

# If resources :users comes FIRST, it still works because resources generates
# named patterns, and /users/new is its own separate route entry.
# The real trap: custom routes vs resourceful routes for the same path.

get '/products/:id', to: 'products#legacy_show'  # matches /products/123
resources :products                               # also matches /products/123

# Result: /products/123 → products#legacy_show (first match wins)
# Fix: put resources :products before the custom route
```

---

**Trap 7: "Rails request lifecycle — at what point are before_actions executed relative to the controller action? Can a before_action prevent the action from running?"**

`before_action` filters run before the action method. A `before_action` can halt the filter chain and prevent the action from running by calling `render` or `redirect_to`. Once either is called, Rails sets the response and the filter chain stops.

The trap: candidates say "raise an exception stops it." While that's true, it's not the idiomatic approach. Also: calling `return` alone in a `before_action` does NOT stop the filter chain — you must call `render` or `redirect_to`.

```ruby
# WRONG: return alone doesn't stop action execution
before_action :check_admin
def check_admin
  return unless current_user.admin? # action STILL runs!
end

# CORRECT: render or redirect stops the chain
before_action :check_admin
def check_admin
  redirect_to root_path, alert: "Not authorized" unless current_user.admin?
end
```

---

**Trap 8: "You mentioned the asset pipeline fingerprints files. What is the fingerprint based on? Is it a timestamp?"**

No — it's an MD5 hash of the file's contents. This means: if two deploys have the same file content, they get the same fingerprint (cache hit). If content changes even slightly, the fingerprint changes (cache miss forces re-download). A timestamp would invalidate cache on every deploy regardless of content changes.

```
# Without fingerprint:
application.js

# With fingerprint:
application-abc12345def67890abc12345def67890.js
```

---

**Trap 9: "A middleware in your stack raises an exception. What happens to the request?**

If middleware raises an unhandled exception, it propagates up through the middleware stack. Each middleware that used a `begin/rescue` block can catch it. `ActionDispatch::ShowExceptions` middleware catches exceptions and renders error pages. In production, it uses `config.exceptions_app` (or `ActionDispatch::PublicExceptions`).

The trap: candidates assume the controller's `rescue_from` handles middleware exceptions. It doesn't — `rescue_from` only handles exceptions in the controller and its filters.

---

**Trap 10: "Why does Rails add a `_method` hidden field to forms instead of just using JavaScript to change the HTTP method?"**

Because HTML forms fundamentally only support GET and POST. Without JavaScript, PUT/PATCH/DELETE forms would fail. The `_method` hidden field plus `Rack::MethodOverride` middleware provides a graceful fallback that works even with JavaScript disabled. This is a progressive enhancement pattern.

---

**Trap 11: "You said Zeitwerk eagerly loads all files in production. Does that include files in `lib/`?"**

Only if explicitly configured. By default, `lib/` is NOT in `eager_load_paths`. Only `app/` subdirectories are autoloaded by default.

```ruby
# To add lib to autoloading:
config.autoload_lib(ignore: %w[assets tasks])

# Rails 7.1+ shorthand:
# This adds lib to autoload_paths AND eager_load_paths, ignoring non-Ruby files
```

Files in `lib/tasks/` (Rake tasks) and `lib/assets/` should usually be ignored.

---

**Trap 12: "Your Rails app is behind a load balancer that terminates SSL. The app logs show requests coming in as HTTP. How does `config.force_ssl` interact with this setup?"**

`config.force_ssl = true` adds `ActionDispatch::SSL` middleware which checks `request.ssl?`. Behind an SSL-terminating load balancer, `request.ssl?` returns `false` (because the request to Rails IS HTTP), so Rails would redirect to HTTPS, creating a redirect loop.

Fix options:
1. Disable `config.force_ssl` and handle HTTPS redirect at the load balancer
2. Configure `ActionDispatch::SSL` to trust the `X-Forwarded-Proto` header:

```ruby
config.force_ssl = true
# Ensure your load balancer sends X-Forwarded-Proto: https
# ActionDispatch::SSL reads this header automatically
```

3. Use `config.assume_ssl = true` (Rails 7.2+) which tells Rails to treat all requests as HTTPS.

