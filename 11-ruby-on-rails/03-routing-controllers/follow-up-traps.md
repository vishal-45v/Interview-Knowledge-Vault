# Chapter 03 — Routing & Controllers: Follow-Up Traps

---

**Trap 1: "You said `redirect_to` stops execution. Does it actually? What if there's code after it?"**

No — `redirect_to` does NOT stop execution. It sets the response but the method continues unless you explicitly `return`.

```ruby
# BUG: code after redirect_to still runs!
def create
  unless current_user.admin?
    redirect_to root_path, alert: "Not authorized"
    # Execution continues here!
  end
  @report = Report.create!(report_params)  # Runs even after redirect!
  render json: @report  # DoubleRenderError if redirect was called above
end

# FIX:
def create
  unless current_user.admin?
    redirect_to root_path, alert: "Not authorized"
    return  # ← explicit return required!
  end
  @report = Report.create!(report_params)
  render json: @report
end

# Cleaner pattern with before_action:
before_action :require_admin!

def require_admin!
  redirect_to root_path, alert: "Not authorized" unless current_user.admin?
  # In a before_action: if redirect_to or render is called,
  # Rails halts the filter chain and does NOT call the action.
  # No explicit return needed in before_action.
end
```

---

**Trap 2: "Strong parameters have a `permit!` method. When would you use it and what's the risk?"**

```ruby
# permit! allows ALL parameters — bypasses strong parameter protection
params.require(:user).permit!
# Equivalent to returning all submitted params without filtering

# Legitimate uses:
# 1. Admin-only endpoints where any attribute is allowed
# 2. Testing/debugging (never in production code paths)
# 3. When all incoming keys are from a trusted source (API-to-API)

# Risk: if a user can reach code with permit!, they can:
# - Set role: 'admin'
# - Set stripe_customer_id: 'cus_attackertoken'
# - Set confirmed_at: Time.now (bypass email confirmation)
# - Set deleted_at: nil (restore soft-deleted records)

# Better alternative for "allow any key" scenarios:
params.require(:metadata).to_unsafe_h.slice(*AllowedMetadataKeys::FOR_ROLE[current_user.role])
```

---

**Trap 3: "You have `before_action :find_user, only: [:show, :edit, :update, :destroy]`. A new developer adds a `duplicate` action. Is it protected?"**

No — `only: [...]` means the filter runs ONLY for the listed actions. The `duplicate` action is not in the list, so `find_user` doesn't run. `@user` is nil in `duplicate`.

```ruby
# Dangerous pattern — easy to forget new actions:
before_action :find_user, only: [:show, :edit, :update, :destroy]
before_action :require_ownership, only: [:edit, :update, :destroy]

# Add duplicate: @user is nil, require_ownership doesn't run → bug!
def duplicate
  @new_user = @user.dup  # @user is nil → NoMethodError!
end

# Better pattern: use except: when most actions need the filter
before_action :find_user, except: [:index, :new, :create]
# Now duplicate is covered automatically

# Or: explicitly add to both lists when action is added
before_action :find_user, only: [:show, :edit, :update, :destroy, :duplicate]
before_action :require_ownership, only: [:edit, :update, :destroy, :duplicate]
```

---

**Trap 4: "You nest routes 3 levels deep: articles have comments, comments have reactions. Why is this a problem and what's the alternative?"**

3-level nesting generates URLs like `/articles/1/comments/2/reactions/3`, which creates:

1. **Controller complexity**: `ReactionsController` receives `params[:article_id]`, `params[:comment_id]`, and `params[:id]` — must validate the whole chain
2. **Helper ugliness**: `article_comment_reaction_path(@article, @comment, @reaction)` — fragile
3. **URL coupling**: if you later want to show a reaction without knowing the article, the URL doesn't work

```ruby
# Anti-pattern (3 levels deep):
resources :articles do
  resources :comments do
    resources :reactions  # /articles/:article_id/comments/:comment_id/reactions/:id
  end
end

# Better: shallow + max 2 levels
resources :articles, shallow: true do
  resources :comments do
    resources :reactions, shallow: true
  end
end
# Results in:
# GET  /articles/:article_id/comments/new (create context with parent)
# GET  /comments/:id                      (individual resource needs no parent)
# GET  /comments/:comment_id/reactions/new
# GET  /reactions/:id                     (shallow: no comment_id needed)

# Alternative: flat routes with belongs_to checks in controller
resources :reactions  # GET /reactions/:id
class ReactionsController < ApplicationController
  def show
    @reaction = Reaction.find(params[:id])
    # Controller verifies ownership explicitly instead of via URL
  end
end
```

---

**Trap 5: "You said `format.js` renders a `.js.erb` file. What happens in Rails 7 with Turbo if you send a form with AJAX?"**

In Rails 7, Turbo Drive intercepts form submissions automatically. The response format is `text/vnd.turbo-stream.html` (Turbo Stream) not `application/javascript`. The old `format.js` and `.js.erb` files no longer work with Turbo by default.

```ruby
# Rails 6 with UJS (old way):
respond_to do |format|
  format.js   # renders create.js.erb
end
# create.js.erb: document.getElementById('modal').remove()

# Rails 7 with Turbo (new way):
respond_to do |format|
  format.turbo_stream do
    render turbo_stream: [
      turbo_stream.remove("modal"),
      turbo_stream.append("flash", partial: "shared/flash", locals: { msg: "Saved!" })
    ]
  end
  format.html { redirect_to root_path }
end

# Or inline:
render turbo_stream: turbo_stream.replace("post_#{@post.id}", partial: "posts/post", locals: { post: @post })
```

---

**Trap 6: "You route `namespace :api` but your API is consumed by mobile apps. What authentication issue does this create with CSRF?"**

`namespace :api` using `ActionController::Base` inherits CSRF protection. Mobile apps can't send the CSRF token (no cookie/session context), so all non-GET requests fail with 422 Unprocessable Entity.

```ruby
# Problem: ApiController inherits from ApplicationController → CSRF checked
class Api::V1::UsersController < ApplicationController
  # CSRF protection enabled → mobile POST requests fail!
end

# Solution 1: Use ActionController::API for API namespace
class Api::BaseController < ActionController::API
  # No CSRF protection — safe because no session/cookie auth
end

# Solution 2: Skip CSRF for API controllers that use token auth
class Api::BaseController < ApplicationController
  skip_before_action :verify_authenticity_token
  before_action :authenticate_api_token!
  # Safe because no session auth → CSRF doesn't apply
end

# Solution 3: Null session strategy (clears session on bad token)
class Api::BaseController < ApplicationController
  protect_from_forgery with: :null_session
end
```

---

**Trap 7: "You use `respond_to` in a base controller with a `format.json` block. What happens if a child controller renders HTML by calling `render :index` — does it return JSON or HTML?"**

`respond_to` is per-action, not inherited. A child controller's action that calls `render :index` without its own `respond_to` block will render the HTML template. Each action must declare its own `respond_to` if it needs multi-format support.

```ruby
# Base controller with respond_to doesn't affect child controller actions:
class ApplicationController < ActionController::Base
  # No inherited respond_to here
end

class ArticlesController < ApplicationController
  def index
    @articles = Article.all
    respond_to do |format|
      format.html
      format.json { render json: @articles }
    end
  end

  def search
    @articles = Article.search(params[:q])
    render :index  # Only renders HTML index template — no JSON support
    # format.json is NOT inherited from index action
  end
end
```

---

**Trap 8: "The `params` hash includes `controller` and `action` keys. Can these be overridden by user input?"**

No — `controller` and `action` are set by the router from the matched route and cannot be overridden by user-submitted parameters. They are path parameters, not request body parameters.

```ruby
# User submits: POST /articles with body: { controller: "admin/users", action: "destroy" }
# params[:controller] → "articles" (from route, not from POST body)
# params[:action] → "create" (from route, not from POST body)

# However: query string CAN overlap with other params
# GET /articles/1?id=999
# params[:id] → "1" (path parameter wins over query string)

# Merge priority (highest to lowest):
# 1. Path parameters (from routes: :id, :format)
# 2. Request body (POST form data or JSON)
# 3. Query string parameters
```

---

**Trap 9: "You set `cookies[:user_id] = current_user.id` for 'remember me'. Why is this insecure and how do you fix it?"**

Plain cookies are visible in the browser and can be tampered with. A user can change `cookies[:user_id] = 1` to `cookies[:user_id] = 2` to impersonate another user.

```ruby
# INSECURE: plain cookie
cookies[:user_id] = current_user.id
# User opens DevTools → Application → Cookies → changes value to any user ID!

# SECURE option 1: signed cookie (tamper-proof, but value still readable)
cookies.signed[:user_id] = current_user.id
# Value is signed with secret_key_base → tampering detected
# But user can still SEE their ID in the cookie (minor concern)

# SECURE option 2: encrypted cookie (tamper-proof AND hidden)
cookies.encrypted[:user_id] = current_user.id
# Encrypted with secret_key_base → value is ciphertext → can't read or tamper

# SECURE option 3: opaque token (best practice)
token = SecureRandom.urlsafe_base64(32)
RememberToken.create!(user: current_user, token: Digest::SHA256.hexdigest(token), expires_at: 30.days.from_now)
cookies.permanent.signed[:remember_token] = token
# Even if cookie is stolen, they get a random token, not the user ID
# Tokens can be revoked server-side

# On login:
remember_token = cookies.signed[:remember_token]
token_record = RememberToken.active.find_by(token: Digest::SHA256.hexdigest(remember_token))
@current_user = token_record&.user
```

---

**Trap 10: "What happens to Rails when you have two routes that match the same path but different HTTP methods — one returns HTML, one returns JSON?"**

Rails routes match on BOTH path AND HTTP method. Two routes with same path but different methods are completely separate:

```ruby
get  '/status', to: 'health#check_html'   # Returns HTML health page
post '/status', to: 'webhooks#receive'    # Receives webhook POSTs

# This is valid and routes to different actions

# The confusion: respond_to is per-action, not per-route
# If you have a single route:
get '/articles', to: 'articles#index'
# And articles#index uses respond_to { format.html; format.json }
# The SAME route handles both formats (determined by Accept header, not method)

# Real trap: options requests for CORS preflight:
# OPTIONS /api/users is NOT matched by:
# post '/api/users', to: 'api/users#create'
# You need either:
# match '/api/users', to: 'cors#preflight', via: :options
# Or the rack-cors middleware (handles it for you)
```

---

**Trap 11: "Does `before_action` in a parent controller apply to all child controllers?"**

Yes, `before_action` defined in a parent controller runs for all child controllers that inherit from it — including actions in deeply nested child controllers.

```ruby
class ApplicationController < ActionController::Base
  before_action :set_locale
  before_action :authenticate_user!  # Runs for EVERY action in the app!
end

class PublicController < ApplicationController
  skip_before_action :authenticate_user!  # Must explicitly skip for public pages
end

class Api::BaseController < ApplicationController
  skip_before_action :authenticate_user!
  before_action :authenticate_api_token!
end

# Gotcha: if you add a before_action to ApplicationController for a new feature,
# it affects ALL controllers — including admin, API, public pages.
# Test thoroughly and use skip_before_action where needed.
```

---

**Trap 12: "You use `params.permit(:tags)` but `tags` is an array. Will this work?"**

No — `permit(:tags)` only permits a scalar value for `tags`. For arrays, you must use the array syntax:

```ruby
# Bug: permits only scalar (single value):
params.permit(:title, :tags)
# POST: title=Article&tags[]=ruby&tags[]=rails
# Result: tags is nil or only first value

# Fix: explicitly permit as array:
params.permit(:title, tags: [])
# Now: tags == ["ruby", "rails"] ✓

# Nested hash in array:
params.permit(:title, items: [:id, :quantity, :price])
# Permits: { items: [{ id: 1, quantity: 2, price: "10.00" }] }

# Common gotcha with JSON API:
# Content-Type: application/json
# POST body: {"article": {"title": "Test", "tag_ids": [1, 2, 3]}}
params.permit(:title, tag_ids: [])
# Correct ✓
```

