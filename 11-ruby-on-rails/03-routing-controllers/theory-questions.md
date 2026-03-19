# Chapter 03 — Routing & Controllers: Theory Questions

---

**Q1. What does `resources :articles` generate? List all 8 routes.**

```ruby
resources :articles
# Generates:
# GET    /articles          articles#index
# GET    /articles/new      articles#new
# POST   /articles          articles#create
# GET    /articles/:id      articles#show
# GET    /articles/:id/edit articles#edit
# PATCH  /articles/:id      articles#update
# PUT    /articles/:id      articles#update
# DELETE /articles/:id      articles#destroy
```

---

**Q2. What is the difference between `resources` and `resource` (singular)?**

```ruby
# resources :sessions — plural, generates all 8 routes with :id
# Assumes multiple records; routes include /sessions/:id

# resource :session — singular, no :id in routes (only one record per context)
# GET    /session/new   session#new
# POST   /session       session#create
# GET    /session       session#show
# GET    /session/edit  session#edit
# PATCH  /session       session#update
# DELETE /session       session#destroy
# No index action (no :id needed — implicit "current user's session")
```

---

**Q3. What is the difference between `member` routes and `collection` routes?**

```ruby
resources :posts do
  member do
    post :publish     # POST /posts/:id/publish  → posts#publish (specific post)
    get  :preview     # GET  /posts/:id/preview  → posts#preview
  end

  collection do
    get :search       # GET  /posts/search       → posts#search (no :id)
    get :archived     # GET  /posts/archived     → posts#archived
  end
end

# member: acts on ONE specific resource (has :id)
# collection: acts on the collection of resources (no :id)
```

---

**Q4. What are nested routes? When are they appropriate? What is the "shallow" option?**

```ruby
# Nested:
resources :articles do
  resources :comments  # GET /articles/:article_id/comments/:id
end

# When: comment only makes sense in context of article
# Smell: nesting deeper than 2 levels is usually wrong

# Shallow nesting (Rails recommendation):
resources :articles do
  resources :comments, shallow: true
  # GET    /articles/:article_id/comments     comments#index
  # POST   /articles/:article_id/comments     comments#create
  # GET    /articles/:article_id/comments/new comments#new
  # GET    /comments/:id                      comments#show (no article_id needed)
  # GET    /comments/:id/edit                 comments#edit
  # PATCH  /comments/:id                      comments#update
  # DELETE /comments/:id                      comments#destroy
end
```

---

**Q5. What is the difference between `namespace`, `scope`, and `scope module:`?**

```ruby
# namespace: adds path prefix AND module prefix AND named route prefix
namespace :admin do
  resources :users
  # Route: /admin/users → controller: Admin::UsersController
  # Helpers: admin_users_path
end

# scope (path only): adds path prefix, NO module change, NO route name change
scope :v1 do
  resources :users
  # Route: /v1/users → controller: UsersController (no module)
  # Helpers: users_path
end

# scope module: (module only): adds module namespace, NO path prefix
scope module: :api do
  resources :users
  # Route: /users → controller: Api::UsersController
  # Helpers: users_path
end

# scope path + module + as:
scope path: "/api/v1", module: :api_v1, as: :api_v1 do
  resources :users
  # Route: /api/v1/users → controller: ApiV1::UsersController
  # Helpers: api_v1_users_path
end
```

---

**Q6. What are route concerns? Show an example.**

```ruby
# Concerns extract shared route definitions:
concern :reviewable do
  resources :reviews
  post :upvote, on: :member
end

concern :commentable do
  resources :comments
end

resources :articles, concerns: [:reviewable, :commentable]
resources :products, concerns: [:reviewable]

# Equivalent to writing out reviews/upvote routes inside each resource
```

---

**Q7. Explain `before_action`, `around_action`, and `after_action`. In what order do they run?**

```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!        # 1. runs BEFORE action
  before_action :find_article, only: [:show, :edit, :update, :destroy]
  around_action :set_locale               # 2. wraps action
  after_action  :log_action               # 3. runs AFTER action (and around_action's post)

  private

  def authenticate_user!
    redirect_to login_path unless current_user
  end

  def find_article
    @article = Article.find(params[:id])
  end

  def set_locale
    I18n.with_locale(params[:locale] || I18n.default_locale) do
      yield  # action runs inside this block
    end
  end

  def log_action
    Rails.logger.info "#{current_user&.email} accessed #{action_name}"
  end
end

# Order: before_action → around_action(before yield) → action → around_action(after yield) → after_action
```

---

**Q8. What are strong parameters? Why were they introduced?**

Strong parameters (introduced in Rails 4) protect against mass assignment vulnerabilities. Without them, an attacker could POST `user[admin]=true` to escalate privileges.

```ruby
# Vulnerable (old attr_accessible / attr_protected approach was fragile):
def create
  @user = User.new(params[:user])  # Unsafe — params[:user] could contain any key!
end

# Safe with strong parameters:
def create
  @user = User.new(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email, :password, :password_confirmation)
  # :admin is not permitted — even if submitted, it's ignored
end

# Nested attributes:
def post_params
  params.require(:post).permit(
    :title, :body,
    category_ids: [],           # array of IDs
    tags_attributes: [:id, :name, :_destroy],  # nested attributes
    metadata: {}                # permit all keys in a hash (use carefully!)
  )
end
```

---

**Q9. What is `respond_to` and when do you use it?**

```ruby
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])

    respond_to do |format|
      format.html                      # render show.html.erb
      format.json { render json: @article }
      format.xml  { render xml: @article }
      format.pdf  { send_data generate_pdf(@article), filename: "article.pdf", type: "application/pdf" }
    end
  end
end

# The format is determined by:
# 1. URL extension: /articles/1.json
# 2. Accept header: Accept: application/json
# 3. Format param: /articles/1?format=json
```

---

**Q10. What is the difference between `redirect_to` and `render`?**

```ruby
# render: returns a response with the current action context (no new request)
def create
  @user = User.new(user_params)
  if @user.save
    redirect_to @user  # sends 302 Redirect → browser makes GET /users/:id
  else
    render :new        # returns new.html.erb content with current @user (errors visible)
    # No redirect → URL stays at POST /users
    # @user has error messages — template can display them
  end
end

# Key difference:
# redirect_to → browser sends NEW HTTP request (clean URL, no data from current request)
# render → returns HTML content for THIS request (URL doesn't change)

# Common mistake: calling render and then redirect_to → DoubleRenderError
```

---

**Q11. What is `flash` and how does it persist between requests?**

```ruby
# flash stores messages that persist for ONE redirect and then disappear
def create
  if @user.save
    flash[:notice] = "Account created successfully!"
    redirect_to root_path
    # On the next request (root_path), flash[:notice] is available
    # After that request, flash is cleared
  end
end

# flash.now: available ONLY in the current request (for renders, not redirects)
def create
  if @user.save
    redirect_to root_path, notice: "Created!"
  else
    flash.now[:alert] = "Please fix the errors"
    render :new  # flash.now message available in this render
  end
end

# flash.keep: keeps flash messages for one more request
# flash.discard: throws away flash before it's displayed

# In views:
# <% if flash[:notice] %><div class="notice"><%= flash[:notice] %></div><% end %>
```

---

**Q12. What are cookies vs sessions in Rails? What are the security implications?**

```ruby
# Cookies: stored in browser, sent with every request, limited to 4KB
cookies[:user_id] = 1                          # plain (readable by JS)
cookies.signed[:user_id] = 1                   # signed (tamper-proof, still readable)
cookies.encrypted[:user_id] = 1                # encrypted (tamper-proof AND hidden)
cookies.permanent.encrypted[:user_id] = 1      # expires far future
cookies[:user_id] = { value: 1, expires: 1.year, httponly: true, secure: true }

# Sessions: typically stored as encrypted cookie (CookieStore)
session[:user_id] = @user.id   # encrypted, can't be read by JS (httponly)
session.clear                   # logout

# Security flags:
# httponly: true — JavaScript can't access (prevents XSS cookie theft)
# secure: true — HTTPS only (prevents HTTP interception)
# same_site: :strict — prevents CSRF (cookie not sent cross-site)

# Session security configuration:
Rails.application.config.session_store :cookie_store,
  key: '_myapp_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 30.days
```

---

**Q13. What is `params` in Rails? How does it merge path, query string, and POST body parameters?**

```ruby
# params is an ActionController::Parameters object
# Merges: path parameters + query string parameters + request body (form/JSON)

# GET /articles/1?search=ruby&sort=desc with path: { id: '1' }
# params == { id: '1', search: 'ruby', sort: 'desc', controller: 'articles', action: 'show' }

# POST /articles with body: { article: { title: "Test" } }
# params[:article][:title] == "Test"

# Priority: path params > POST body params > query string params
# (path params can't be overridden by body)

# Type coercion in routes:
get '/users/:id', to: 'users#show', constraints: { id: /\d+/ }
# params[:id] is always a String — coerce explicitly:
User.find(params[:id].to_i)  # or params.require(:id)
User.find(params[:id])       # ActiveRecord handles "1" → 1 automatically
```

---

**Q14. What is `ActionController::API` and what does it not include compared to `ActionController::Base`?**

`ActionController::API` is a lightweight base class that removes browser-centric features:

Removed from API mode:
- CSRF protection (`verify_authenticity_token`)
- Cookie management
- Session management
- Flash
- View rendering (ERB/HAML templates)
- Asset helpers
- Browser-specific request formats

Kept in API mode:
- `before_action`, `around_action`, `after_action`
- Strong parameters
- `respond_to`
- Rendering JSON/XML
- HTTP Basic auth
- Token authentication

---

**Q15. What are route constraints? Give examples of regex, proc, and object constraints.**

```ruby
# Regex constraint — matches path segment format
get '/articles/:id', to: 'articles#show', constraints: { id: /\d{4,}/ }

# Proc constraint — evaluated at request time
get '/beta', to: 'beta#index', constraints: ->(request) {
  request.remote_ip.start_with?("192.168.")
}

# Object constraint — class with matches? method
class AdminConstraint
  def matches?(request)
    user_id = request.session[:user_id]
    user_id && User.find_by(id: user_id)&.admin?
  end
end

namespace :admin, constraints: AdminConstraint.new do
  resources :users
  resources :reports
end

# Subdomain constraint:
constraints subdomain: "api" do
  namespace :api do
    resources :v1
  end
end
```

---

**Q16. How do you handle exceptions globally in a Rails controller?**

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :handle_invalid
  rescue_from ActionController::ParameterMissing, with: :handle_bad_request
  rescue_from Pundit::NotAuthorizedError, with: :handle_unauthorized

  private

  def handle_not_found(exception)
    respond_to do |format|
      format.html { render file: "public/404.html", status: :not_found }
      format.json { render json: { error: "Resource not found" }, status: :not_found }
    end
  end

  def handle_invalid(exception)
    respond_to do |format|
      format.json { render json: { errors: exception.record.errors.full_messages }, status: :unprocessable_entity }
    end
  end

  def handle_bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end

  def handle_unauthorized
    render json: { error: "Not authorized" }, status: :forbidden
  end
end
```

---

**Q17. What are `only:` and `except:` options on `before_action`?**

```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!
  before_action :find_article, only: [:show, :edit, :update, :destroy]
  before_action :check_ownership, except: [:index, :show, :new, :create]

  # only: filter applies ONLY to listed actions
  # except: filter applies to ALL actions EXCEPT listed ones
  # These are inverse — choose whichever is more concise
end
```

---

**Q18. What is `redirect_to` with various redirect types? What HTTP status codes are used?**

```ruby
redirect_to @article                    # 302 Found (default)
redirect_to articles_path               # 302 Found
redirect_to "https://example.com"       # 302 Found
redirect_to :back                       # 302 to HTTP Referer (Rails 4)
redirect_back(fallback_location: root_path)  # Rails 5+ version of :back

redirect_to @article, status: :moved_permanently  # 301 Permanent
redirect_to @article, status: 301                 # same as above
redirect_to @article, status: :see_other          # 303 (POST → GET)
redirect_to @article, status: :temporary_redirect # 307 (preserves HTTP method)

# With flash:
redirect_to @article, notice: "Saved!"   # sets flash[:notice]
redirect_to @article, alert: "Error!"    # sets flash[:alert]
```

---

**Q19. Explain `format.js` in a Rails controller. How does it work with Turbo?**

```ruby
# Legacy (Rails 5/6 with UJS):
def create
  @comment = Comment.new(comment_params)
  @comment.save

  respond_to do |format|
    format.html { redirect_to @post }
    format.js   # renders app/views/comments/create.js.erb
  end
end
# create.js.erb:
# document.getElementById('comments').insertAdjacentHTML('beforeend', '<%= j render @comment %>')

# Modern (Rails 7 with Turbo):
def create
  @comment = Comment.create!(comment_params)
  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: turbo_stream.append("comments", @comment)
    end
    format.html { redirect_to @post }
  end
end
```

---

**Q20. What is `current_user` in Rails? How is it typically implemented?**

```ruby
# Devise provides current_user automatically when you include :devise module

# Manual implementation using a session:
class ApplicationController < ActionController::Base
  helper_method :current_user

  private

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  def authenticate_user!
    redirect_to login_path unless current_user
  end

  def signed_in?
    current_user.present?
  end
end

# JWT-based (API):
class Api::BaseController < ActionController::API
  private

  def current_user
    @current_user ||= begin
      token = request.headers['Authorization']&.split(' ')&.last
      payload = JWT.decode(token, Rails.application.credentials.jwt_secret!).first
      User.find(payload['user_id'])
    rescue JWT::DecodeError, ActiveRecord::RecordNotFound
      nil
    end
  end
end
```

---

**Q21. What is the `config/routes.rb` `draw` method? How do you split a large routes file?**

```ruby
# config/routes.rb — main file:
Rails.application.routes.draw do
  draw :admin    # loads config/routes/admin.rb
  draw :api      # loads config/routes/api.rb
  draw :web      # loads config/routes/web.rb

  root to: 'home#index'
end

# config/routes/admin.rb:
namespace :admin do
  resources :users
  resources :reports
  resource :dashboard
end

# config/routes/api.rb:
namespace :api do
  namespace :v1 do
    resources :users, only: [:index, :show, :create, :update]
    resources :products
  end
end
```

---

**Q22. How do you add custom format types to Rails routing?**

```ruby
# Register custom MIME types in config/initializers/mime_types.rb:
Mime::Type.register "application/pdf", :pdf
Mime::Type.register "text/csv", :csv
Mime::Type.register "application/vnd.ms-excel", :xls

# In controller:
def index
  @users = User.all
  respond_to do |format|
    format.html
    format.csv  { send_data generate_csv(@users), filename: "users.csv", type: :csv }
    format.pdf  { send_data generate_pdf(@users), filename: "users.pdf", type: :pdf }
  end
end
```

---

**Q23. What is `ActionController::Live` and Server-Sent Events (SSE) in Rails?**

```ruby
class NotificationsController < ApplicationController
  include ActionController::Live

  def stream
    response.headers['Content-Type'] = 'text/event-stream'
    response.headers['Last-Modified'] = Time.now.httpdate

    sse = SSE.new(response.stream, retry: 300, event: "notification")

    begin
      10.times do |i|
        sse.write({ message: "Notification #{i}", time: Time.now })
        sleep 1
      end
    rescue ActionController::Live::ClientDisconnected
      Rails.logger.info "Client disconnected from SSE stream"
    ensure
      sse.close
    end
  end
end
```

---

**Q24. What is `ActionController::Parameters#to_unsafe_h`? When would you use it?**

```ruby
# params.to_unsafe_h converts strong parameters to a plain hash, bypassing permit
# Use only when:
# - Building dynamic queries where you genuinely don't know keys ahead of time
# - Interacting with libraries that expect Hash, not ActionController::Parameters
# - Logging/debugging (never for mass assignment!)

def admin_report_params
  params.require(:filters).to_unsafe_h  # Admin-only: any filter allowed
end

# Safe alternatives that don't bypass security:
params.permit!.to_h          # permits everything (similar danger)
params[:user].to_h           # doesn't filter — same danger if used for mass assignment
params.require(:user).permit(:name, :email).to_h  # correct approach
```

---

**Q25. What are route helpers and how do you use them in controllers, views, and mailers?**

```ruby
# Route helpers generated by resources :articles:
# articles_path         → /articles
# article_path(@article) → /articles/42
# new_article_path      → /articles/new
# edit_article_path(@article) → /articles/42/edit
# articles_url          → http://example.com/articles (full URL)

# In controllers:
redirect_to article_path(@article)
redirect_to new_article_path

# In views:
link_to "Articles", articles_path
link_to @article.title, article_path(@article)
button_to "Delete", article_path(@article), method: :delete

# In mailers (must use _url, not _path):
class ArticleMailer < ApplicationMailer
  def published(article)
    @article = article
    @url = article_url(@article)  # full URL — email client needs host
    mail to: @article.author.email, subject: "Your article is live!"
  end
end

# Helper aliases (both work):
article_path(@article)        # same as:
article_path(id: @article.id) # Rails extracts id automatically via to_param
```

