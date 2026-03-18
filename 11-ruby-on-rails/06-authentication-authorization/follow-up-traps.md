# Chapter 06 — Authentication & Authorization: Follow-Up Traps

---

**Trap 1: "Devise uses `current_user` in the controller. How do you access the current user in a background job?"**

```ruby
# WRONG: current_user is a request-scoped controller method
# It reads from the session, which doesn't exist in a background job
class SendWelcomeEmailJob < ApplicationJob
  def perform
    current_user.send_welcome_email  # NoMethodError: undefined method `current_user`
  end
end

# CORRECT: Pass the user ID explicitly
class SendWelcomeEmailJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end

# Enqueue from controller:
SendWelcomeEmailJob.perform_later(current_user.id)

# Advanced pattern: Current attributes (thread-local store)
class Current < ActiveSupport::CurrentAttributes
  attribute :user
end

# Set in controller:
before_action { Current.user = current_user }

# Access in jobs — but only works if job runs synchronously in same thread
# For async jobs, ALWAYS pass user_id explicitly — don't rely on Current
```

---

**Trap 2: "You use `before_action :authenticate_user!` on ApplicationController. Your Devise PasswordsController handles `GET /users/password/edit`. Does this cause a problem?"**

```ruby
# PROBLEM: authenticate_user! redirects unauthenticated users to sign_in
# A user clicking a password reset link is NOT authenticated
# authenticate_user! runs before PasswordsController#edit
# → redirected to sign_in page instead of seeing the reset form!

# Devise's own controllers use skip_before_action internally:
# BUT if you add authenticate_user! to ApplicationController and Devise
# controllers inherit from it, you must skip it for Devise controllers.

# Fix 1: Explicitly skip for devise controllers
class ApplicationController < ActionController::Base
  before_action :authenticate_user!, unless: :devise_controller?
end

# Fix 2: Use Devise's helper method:
before_action :authenticate_user!, except: [:some_public_action]
# devise_controller? returns true when the current controller is a Devise controller

# This also affects RegistrationsController (sign up page)
# ConfirmationsController (email confirmation)
# SessionsController — Devise already handles this, but inheritance matters
```

---

**Trap 3: "Pundit's `policy_scope` is used in `index` actions. What happens if you forget to call it?"**

```ruby
# WRONG: Using unscoped query — returns ALL records regardless of authorization
def index
  @posts = Post.all  # Every post, including drafts, other users' private posts
  # Pundit::NotAuthorizedError is NOT raised — you just leak data silently
end

# Pundit can detect this with after_action:
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  after_action :verify_policy_scoped, only: :index

  # This raises Pundit::PolicyScopingNotPerformedError if policy_scope wasn't called
end

# CORRECT:
def index
  @posts = policy_scope(Post)
  # Internally calls PostPolicy::Scope.new(current_user, Post).resolve
end

# Common mistake: calling authorize without policy_scope in index
def index
  authorize Post  # Checks PostPolicy#index? — but doesn't scope the query!
  @posts = Post.all  # Still returns all posts
end

# The two are complementary:
# authorize — "can this user perform this action?"
# policy_scope — "which records can this user see?"
```

---

**Trap 4: "Strong parameters and Devise: a user sends `role: 'admin'` in the registration form. Does Devise protect against this?"**

```ruby
# Devise uses strong parameters via sanitize_parameters
# Default permitted params for sign_up: email, password, password_confirmation
# role is NOT permitted by default

# BUT: if you've customized permitted params without care:
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    # WRONG: permits ALL extra attributes
    devise_parameter_sanitizer.permit(:sign_up, keys: devise_parameter_sanitizer.sanitize(:sign_up) + [:role])

    # CORRECT: explicitly list safe attributes
    devise_parameter_sanitizer.permit(:sign_up, keys: [:first_name, :last_name])
    # Never include :role, :admin, :permissions, :account_type unless you want users to set it
  end
end

# Also dangerous: admin flags set via account_update
devise_parameter_sanitizer.permit(:account_update, keys: [:name, :avatar])
# Not :admin, :role, :permissions

# The real trap: Devise doesn't check this for you — it only sanitizes what you tell it to
# Test: POST /users with role=admin, confirm the role is NOT set
```

---

**Trap 5: "You have a Pundit policy with `show?` returning `record.user == user`. Your `update?` just calls `show?`. What's the bug in update?"**

```ruby
# Policy:
class PostPolicy < ApplicationPolicy
  def show?
    record.published? || record.user == user
  end

  def update?
    show?  # Reuses show? logic — looks clean...
  end
end

# BUG: A published post is readable by everyone (record.published?)
# But update? delegates to show?, so ANYONE can update published posts!

# update? should only allow the owner:
def update?
  record.user == user  # Ownership check only
end

# Or with admin:
def update?
  record.user == user || user.admin?
end

# The trap: DRY patterns in policies can silently grant more permissions than intended
# Each policy method should be explicit about its own logic
# Sharing methods is fine for truly identical logic, but beware of semantic differences

# Another example:
def destroy?
  update?  # Seems right...
end
# But if update? allows editors (not just owners), destroy allows editors too
# Maybe you only want owners to destroy
```

---

**Trap 6: "Session fixation — what is it and does Rails prevent it automatically?"**

```ruby
# Session fixation attack:
# 1. Attacker visits your site, gets session ID: session_id=ATTACKER_KNOWN_ID
# 2. Attacker tricks victim into using that session ID (via URL, injection)
# 3. Victim logs in — session now has authenticated user
# 4. Attacker uses the SAME session ID — they're now logged in as victim!

# Rails protection: reset_session on privilege escalation
# Devise calls reset_session automatically on sign_in

# If building custom auth:
def create
  user = User.authenticate(params[:email], params[:password])
  if user
    reset_session        # ← CRITICAL: generates new session ID, copies data
    session[:user_id] = user.id
    redirect_to root_path
  end
end

# WITHOUT reset_session:
def create  # VULNERABLE
  session[:user_id] = user.id
  # Attacker with pre-planted session ID is now authenticated!
end

# Devise's SignIn flow:
# warden.set_user(user) internally calls env['rack.session'].clear and regenerates

# Also reset on logout to prevent reuse:
def destroy
  reset_session  # Clear all session data AND generate new session ID
  redirect_to root_path
end
```

---

**Trap 7: "You add `skip_before_action :verify_authenticity_token` to your API controller. Is this safe?"**

```ruby
# Context: CSRF tokens are browser-specific (form + cookie pattern)
# JWT-authenticated APIs don't use cookies for auth → CSRF attacks don't apply
# Because the attacker's malicious form can't forge the Authorization header

# SAFE to skip CSRF for:
class Api::BaseController < ActionController::API
  # ActionController::API skips CSRF verification by default
  # No need to skip — it's already not included
end

# If inheriting from ActionController::Base:
class Api::BaseController < ActionController::Base
  skip_before_action :verify_authenticity_token
  # ONLY safe if API is authenticated via Authorization header (not cookies)
end

# DANGEROUS to skip CSRF for:
# APIs that use cookie-based sessions for auth
# If you skip CSRF but use cookie auth → cross-site form submissions work → CSRF vulnerability

# The real trap:
class Api::V1::BaseController < ApplicationController  # inherits ActionController::Base
  skip_before_action :verify_authenticity_token
  before_action :authenticate_from_cookie!  # Uses session/cookie auth!
  # Now you've removed CSRF protection from cookie-authenticated endpoints — VULNERABILITY!
end

# Rule: skip CSRF protection ONLY when authentication doesn't use cookies
```

---

**Trap 8: "Devise's `authenticate_user!` redirects to the sign-in page on failure. Your API controller inherits from ApplicationController which has Devise helpers. What happens when an API client gets a 302 redirect?"**

```ruby
# Problem: API clients don't follow redirects for auth — they expect 401 Unauthorized
# Devise redirects browser requests to sign_in page (302)
# API client follows redirect → gets HTML sign_in page → tries to parse as JSON → error

# Fix 1: Override Devise's failure response
class CustomAuthFailure < Devise::FailureApp
  def respond
    if request.format == :json || request.path.start_with?('/api/')
      json_error_response
    else
      super
    end
  end

  def json_error_response
    self.status = :unauthorized
    self.content_type = 'application/json'
    self.response_body = { error: { code: 'unauthorized', message: 'Authentication required' } }.to_json
  end
end

# config/initializers/devise.rb
Devise.setup do |config|
  config.warden do |manager|
    manager.failure_app = CustomAuthFailure
  end
end

# Fix 2: Use separate authentication for API (don't use authenticate_user!)
class Api::BaseController < ActionController::API
  before_action :authenticate_from_token!  # Custom, returns 401 not redirect
end
```

---

**Trap 9: "A user has the `editor` role. Your policy says editors can update posts. The user updates a post via `PATCH /posts/1`. Does Pundit check that the post exists before checking the policy?"**

```ruby
# Pundit does NOT check existence — it receives whatever you pass as the record
# If Post.find(params[:id]) raises ActiveRecord::RecordNotFound before pundit runs:
# → Rails renders 404, Pundit never called

# But if you use find_by (returns nil):
def update
  @post = Post.find_by(id: params[:id])  # Returns nil if not found
  authorize @post  # PostPolicy.new(current_user, nil)
  # Policy: def update? = record.editors.include?(user)
  # nil.editors → NoMethodError!
end

# Or worse — policy allows nil:
def update?
  record&.editors&.include?(user)  # returns nil (falsy) — authorization fails
  # But error message is confusing
end

# Best practice: use find (raises 404) not find_by when record must exist
def update
  @post = Post.find(params[:id])  # 404 if not found
  authorize @post                  # Policy runs on real record
end

# rescue_from in ApplicationController handles RecordNotFound → 404
rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
```

---

**Trap 10: "You implement Pundit in a multi-tenant app. Your PostPolicy#show? checks `record.organization == user.organization`. Is this enough?"**

```ruby
# Partial protection — prevents cross-org data viewing via show
# But what about the index scope?

class PostPolicy < ApplicationPolicy
  def show?
    record.organization == user.organization
  end
end

# PostPolicy::Scope MISSING or wrong:
class Scope < Scope
  def resolve
    scope.all  # RETURNS ALL POSTS FROM ALL ORGANIZATIONS!
  end
end

# Or common mistake:
class Scope < Scope
  def resolve
    scope.where(organization: user.organization)
    # But if the user navigates to GET /posts/999 (another org's post)
    # The index is scoped, but show? also needs to check organization
  end
end

# The real trap: you need BOTH:
# 1. policy_scope in index — filters the collection
# 2. authorize in show/update/destroy — checks the specific record

# Missing either one leaks data:
# Missing scope → index shows all orgs' records
# Missing authorize → direct URL access bypasses tenant isolation

# Also check: are sub-resources (comments on posts) also scoped?
# @post.comments.all — doesn't scope by organization!
# Must use: policy_scope(@post.comments)
```

---
