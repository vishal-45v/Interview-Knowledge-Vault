# Chapter 06 — Authentication & Authorization: Theory Questions

---

**Q1. What is Devise? What modules does it provide?**

Devise is a full authentication solution for Rails based on Warden. It provides 10 modules:

```ruby
class User < ApplicationRecord
  devise :database_authenticatable,   # Encrypts and stores password
         :registerable,               # Users can register/delete accounts
         :recoverable,                # Password reset via email
         :rememberable,               # Remember me cookie
         :validatable,                # Email/password validations
         :confirmable,                # Email confirmation required
         :lockable,                   # Lock account after N failed attempts
         :timeoutable,                # Session expires after inactivity
         :trackable,                  # Tracks sign_in_count, last_sign_in_at, IP
         :omniauthable                # OmniAuth integration
end
```

---

**Q2. What is `has_secure_password`? How does it work?**

```ruby
# Requires bcrypt gem
class User < ApplicationRecord
  has_secure_password
  # Adds:
  # - virtual attribute :password and :password_confirmation
  # - validates :password, length: { minimum: 72 } (bcrypt limit)
  # - validates presence of password on create
  # - BCrypt hashing stored in :password_digest column
end

# Migration required:
# add_column :users, :password_digest, :string

user = User.create!(email: "alice@example.com", password: "secret123")
user.authenticate("secret123")   # => user object (truthy)
user.authenticate("wrong")       # => false

# Rails 7.1+ added:
user = User.authenticate_by(email: "alice@example.com", password: "secret123")
# Returns nil if not found OR wrong password (constant-time, prevents timing attacks)
```

---

**Q3. What is bcrypt? Why is it used for passwords instead of SHA-256?**

bcrypt is an adaptive hashing algorithm designed for passwords. Key properties:

1. **Cost factor (work factor)**: bcrypt is intentionally slow (configurable). Default cost 12 = 4096 iterations. Raises cost as hardware gets faster.
2. **Salt built-in**: Every hash includes a random salt — same password produces different hash every time.
3. **One-way**: Cannot reverse hash to recover password.

SHA-256 is FAST by design (used for data integrity). An attacker with GPU can compute billions of SHA-256 hashes/second. bcrypt with cost 12: ~0.1 seconds per hash on modern hardware — makes brute force impractical.

```ruby
BCrypt::Password.create("secret123", cost: 12)
# => "$2a$12$randomsalt22chars...hashedvalue..."
#         ↑  ↑
#    version cost
```

---

**Q4. How do you add custom fields to Devise registration?**

```ruby
# Step 1: Add columns via migration
class AddNameToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :first_name, :string
    add_column :users, :last_name, :string
    add_column :users, :phone_number, :string
  end
end

# Step 2: Configure strong parameters in ApplicationController
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  private

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:first_name, :last_name, :phone_number])
    devise_parameter_sanitizer.permit(:account_update, keys: [:first_name, :last_name, :phone_number])
  end
end

# Step 3: Override Devise controllers for full customization:
rails generate devise:controllers users
# config/routes.rb:
devise_for :users, controllers: {
  registrations: 'users/registrations',
  sessions: 'users/sessions',
  passwords: 'users/passwords'
}
```

---

**Q5. What is Pundit? How does a policy work?**

```ruby
# gem 'pundit'

# app/policies/article_policy.rb
class ArticlePolicy < ApplicationPolicy
  def index?
    true  # Anyone can view the list
  end

  def show?
    record.published? || user_is_author?
  end

  def create?
    user.present?  # Any logged-in user
  end

  def update?
    user_is_author? || user.admin?
  end

  def destroy?
    update?
  end

  class Scope < Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(author: user))
      else
        scope.where(published: true)
      end
    end
  end

  private

  def user_is_author?
    user.present? && record.author == user
  end
end

# In controller:
class ArticlesController < ApplicationController
  include Pundit::Authorization
  after_action :verify_authorized, except: :index
  after_action :verify_policy_scoped, only: :index

  def index
    @articles = policy_scope(Article)
  end

  def show
    @article = Article.find(params[:id])
    authorize @article  # calls ArticlePolicy#show?
  end

  def update
    @article = Article.find(params[:id])
    authorize @article  # calls ArticlePolicy#update?
    @article.update!(article_params)
  end
end
```

---

**Q6. What is CanCanCan? How does it compare to Pundit?**

```ruby
# gem 'cancancan'

# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new  # Guest user

    if user.admin?
      can :manage, :all  # Admins can do everything
    elsif user.editor?
      can :read, :all
      can :manage, Article, author_id: user.id
      can :manage, Comment, user_id: user.id
      cannot :delete, User  # Override: editors can't delete users
    else
      # Regular user
      can :read, Article, published: true
      can :create, [Comment, Article]
      can [:update, :destroy], Article, author_id: user.id
      can [:update, :destroy], Comment, user_id: user.id
    end
  end
end

# In controller:
class ArticlesController < ApplicationController
  load_and_authorize_resource  # Automatically loads @article and calls authorize!

  def show
    # @article loaded and authorized automatically
  end
end

# Manual check:
authorize! :update, @article  # raises CanCan::AccessDenied if not allowed
can? :update, @article        # returns boolean
cannot? :delete, User         # boolean
```

---

**Q7. What is OAuth? How does OmniAuth work with Rails?**

```ruby
# OAuth 2.0: authorization protocol where user grants access to their data
# on one service (Google) to another service (your app)

# gem 'omniauth'
# gem 'omniauth-google-oauth2'
# gem 'omniauth-github'
# gem 'omniauth-rails_csrf_protection'

# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
           Rails.application.credentials.google.client_id,
           Rails.application.credentials.google.client_secret,
           scope: 'email profile',
           prompt: 'select_account'

  provider :github,
           Rails.application.credentials.github.client_id,
           Rails.application.credentials.github.client_secret,
           scope: 'user:email'
end

# config/routes.rb:
get '/auth/:provider/callback', to: 'omniauth_callbacks#create'
get '/auth/failure', to: 'omniauth_callbacks#failure'

# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < ApplicationController
  def create
    auth = request.env['omniauth.auth']
    @user = User.from_omniauth(auth)

    if @user.persisted?
      sign_in @user
      redirect_to root_path, notice: "Signed in via #{auth.provider.titleize}"
    else
      redirect_to new_registration_path, alert: "Could not sign you in"
    end
  end

  def failure
    redirect_to root_path, alert: "Authentication failed: #{params[:message]}"
  end
end

# User model:
class User < ApplicationRecord
  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email     = auth.info.email
      user.name      = auth.info.name
      user.avatar_url = auth.info.image
      user.password  = SecureRandom.hex  # dummy password for OAuth users
    end
  end
end
```

---

**Q8. What is two-factor authentication (2FA)? How do you implement TOTP in Rails?**

```ruby
# TOTP: Time-based One-Time Password (RFC 6238)
# Used by: Google Authenticator, Authy, 1Password

# gem 'rotp'  # Ruby One-Time Password
# gem 'rqrcode'  # For QR code generation

class User < ApplicationRecord
  before_create :generate_otp_secret

  def otp_provisioning_uri
    ROTP::TOTP.new(otp_secret).provisioning_uri(email, issuer: "MyApp")
  end

  def verify_otp(code)
    totp = ROTP::TOTP.new(otp_secret)
    # verify with 30-second drift tolerance:
    totp.verify(code.to_s.delete(" "), drift_behind: 15, drift_ahead: 15)
  end

  def enable_2fa!
    update!(otp_required: true)
  end

  private

  def generate_otp_secret
    self.otp_secret = ROTP::Base32.random
  end
end

# 2FA Setup Controller:
class TwoFactorController < ApplicationController
  def new
    @qr_code = RQRCode::QRCode.new(current_user.otp_provisioning_uri)
    @svg = @qr_code.as_svg(module_size: 4)
  end

  def create
    if current_user.verify_otp(params[:code])
      current_user.enable_2fa!
      # Generate and store backup codes:
      backup_codes = 10.times.map { SecureRandom.hex(5).upcase }
      current_user.update!(backup_codes: backup_codes.map { BCrypt::Password.create(_1) })
      flash[:notice] = "2FA enabled. Save these backup codes: #{backup_codes.join(', ')}"
      redirect_to profile_path
    else
      flash[:alert] = "Invalid code. Please try again."
      render :new
    end
  end
end

# 2FA verification in sessions controller:
class SessionsController < ApplicationController
  def create
    user = User.authenticate_by(email: params[:email], password: params[:password])
    if user
      if user.otp_required?
        session[:pending_user_id] = user.id
        redirect_to two_factor_verify_path
      else
        sign_in(user)
        redirect_to root_path
      end
    else
      flash[:alert] = "Invalid credentials"
      render :new
    end
  end
end
```

---

**Q9. What is role-based access control (RBAC)? How do you implement it in Rails?**

```ruby
# Simple enum roles:
class User < ApplicationRecord
  enum :role, { guest: 0, member: 1, editor: 2, admin: 3, super_admin: 4 }

  ROLE_HIERARCHY = { super_admin: 4, admin: 3, editor: 2, member: 1, guest: 0 }.freeze

  def at_least?(required_role)
    ROLE_HIERARCHY[role.to_sym] >= ROLE_HIERARCHY[required_role]
  end
end

# Multiple roles with a join table (more flexible):
class User < ApplicationRecord
  has_many :user_roles
  has_many :roles, through: :user_roles

  def has_role?(role_name, resource: nil)
    roles.exists?(name: role_name, resource_type: resource&.class&.name, resource_id: resource&.id)
  end

  def add_role!(role_name, resource: nil)
    role = Role.find_or_create_by!(
      name: role_name,
      resource_type: resource&.class&.name,
      resource_id: resource&.id
    )
    roles << role unless roles.include?(role)
  end
end

class Role < ApplicationRecord
  has_many :user_roles
  has_many :users, through: :user_roles
  belongs_to :resource, polymorphic: true, optional: true
end

# Usage:
user.add_role!(:admin)
user.add_role!(:editor, resource: workspace)  # resource-scoped role

# Pundit integration:
class ArticlePolicy < ApplicationPolicy
  def update?
    user.has_role?(:admin) ||
    user.has_role?(:editor, resource: record.workspace)
  end
end
```

---

**Q10. What is session fixation and how does Rails prevent it?**

Session fixation: attacker sets a known session ID before the user logs in, then after login uses that ID (now associated with the logged-in user).

```ruby
# Attack:
# 1. Attacker: GET /login → gets session cookie: _session_id=KNOWN123
# 2. Attacker tricks victim into using KNOWN123 (e.g., via URL: /login?session=KNOWN123)
# 3. Victim logs in → session KNOWN123 now has user_id = victim
# 4. Attacker uses KNOWN123 → now authenticated as victim!

# Prevention: reset session on login (Rails best practice):
class SessionsController < ApplicationController
  def create
    user = User.authenticate_by(email: params[:email], password: params[:password])
    if user
      reset_session  # ← CRITICAL: generates NEW session ID, discards old session data
      session[:user_id] = user.id
      redirect_to root_path
    end
  end
end

# Devise does this automatically via the :database_authenticatable module
# reset_session is called before setting session[:current_user_id]
```

---

**Q11. What is Devise's `current_user` helper? How does it work under the hood?**

```ruby
# Devise adds current_user (and current_admin, etc.) as helper methods
# to ApplicationController via Devise::Controllers::Helpers

# Equivalent implementation:
def current_user
  @current_user ||= warden.authenticate(scope: :user)
end

# Warden (underlying auth framework):
# - Checks session for :user scope
# - Calls configured strategies (database_authenticatable checks session[:current_user_id])
# - Returns deserialized user from session data

# user_signed_in? helper:
def user_signed_in?
  !!current_user
end

# authenticate_user! helper (used as before_action):
def authenticate_user!
  unless current_user
    flash[:alert] = "You need to sign in"
    redirect_to new_user_session_path
  end
end
```

---

**Q12. What is OAuth token storage? Where should you store OAuth access tokens?**

```ruby
# Never store OAuth access tokens in plain text in the DB
# They can be used to access the user's third-party account

# Option 1: Encrypt at column level (Rails 7.1+ ActiveRecord encryption)
class OauthCredential < ApplicationRecord
  encrypts :access_token, deterministic: false  # AES-256-GCM
  encrypts :refresh_token, deterministic: false
  belongs_to :user
end

# Option 2: Rails credentials + per-record encryption
class User < ApplicationRecord
  def store_oauth_tokens(access_token:, refresh_token:, expires_at:)
    key = Rails.application.credentials.oauth_encryption_key
    encrypted_access = ActiveSupport::MessageEncryptor.new(key).encrypt_and_sign(access_token)
    update!(
      oauth_access_token_encrypted: encrypted_access,
      oauth_refresh_token_encrypted: encrypt_token(refresh_token),
      oauth_token_expires_at: expires_at
    )
  end
end

# Option 3: Store in separate encrypted table with short expiry:
class OauthToken < ApplicationRecord
  encrypts :access_token, :refresh_token  # Rails 7.1+ encryption
  belongs_to :user
  scope :valid, -> { where("expires_at > ?", Time.current) }

  def expired?
    expires_at < Time.current
  end

  def refresh!
    # Call OAuth provider to refresh using refresh_token
    new_tokens = OAuthClient.refresh(refresh_token: refresh_token)
    update!(
      access_token: new_tokens.access_token,
      expires_at: new_tokens.expires_at
    )
  end
end
```

---

**Q13. What is Devise Confirmable? How does email confirmation work?**

```ruby
# Devise :confirmable module:
# - Sends confirmation email on registration
# - User must click link before logging in
# - Has confirmation_token column (stored hashed)
# - confirmed_at column (nil until confirmed)

class User < ApplicationRecord
  devise :database_authenticatable, :confirmable
end

# Migration adds:
# t.string   :confirmation_token
# t.datetime :confirmed_at
# t.datetime :confirmation_sent_at
# t.string   :unconfirmed_email

# Allow unconfirmed users to sign in for N days:
config.allow_unconfirmed_access_for = 2.days

# Re-send confirmation email:
user.send_confirmation_instructions

# Confirm programmatically (e.g., admin confirms user):
user.confirm

# Custom confirmation email:
# app/views/devise/mailer/confirmation_instructions.html.erb
# Receives: @token, @resource (the user)
# URL: user_confirmation_url(confirmation_token: @token)
```

---

**Q14. How do you implement API token authentication (personal access tokens) for Rails?**

```ruby
class PersonalAccessToken < ApplicationRecord
  belongs_to :user
  has_secure_token :token_value, length: 32  # Generates unique token

  scope :active, -> { where(revoked_at: nil).where("expires_at IS NULL OR expires_at > ?", Time.current) }

  before_create :hash_token

  def self.authenticate(raw_token)
    prefix, secret = raw_token.to_s.split("_", 2)
    return nil unless prefix == "pat" && secret.present?

    token = active.find_by(token_prefix: prefix + "_" + secret[0..7])
    return nil unless token&.secure_compare(secret)

    token.touch(:last_used_at)
    token
  end

  def to_bearer
    "#{token_prefix}#{raw_secret}"
  end

  private

  def hash_token
    raw = SecureRandom.urlsafe_base64(32)
    self.token_prefix = "pat_#{raw[0..7]}"
    self.token_digest = BCrypt::Password.create(raw)
    @raw_secret = raw  # Shown once to user
  end

  def secure_compare(secret)
    BCrypt::Password.new(token_digest).is_password?(secret)
  end
end

# Show token once after creation:
class PersonalAccessTokensController < ApplicationController
  def create
    @token = current_user.personal_access_tokens.create!(
      name: params[:name],
      scopes: params[:scopes] || [],
      expires_at: params[:expires_in].present? ? params[:expires_in].to_i.days.from_now : nil
    )
    render json: {
      token: @token.to_bearer,  # Show the raw token ONCE
      message: "Store this token securely — it won't be shown again"
    }, status: :created
  end
end
```

---

**Q15. What is Pundit's `policy_scope` vs `authorize`? When do you use each?**

```ruby
# authorize: checks if current user can perform an action on a SPECIFIC record
def show
  @article = Article.find(params[:id])
  authorize @article  # Calls ArticlePolicy#show? with @article as record
  # If policy returns false: raises Pundit::NotAuthorizedError
end

# policy_scope: returns a FILTERED COLLECTION the user is allowed to see
def index
  @articles = policy_scope(Article)
  # Calls ArticlePolicy::Scope#resolve
  # Returns: Article.all for admin, Article.published for regular users
end

# The Scope class:
class ArticlePolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(published: true)
          .or(scope.where(author: user))
      end
    end
  end
end

# VERIFY these are called (safety net):
after_action :verify_authorized, except: :index    # ensures authorize was called
after_action :verify_policy_scoped, only: :index   # ensures policy_scope was called

# Skip when needed:
skip_authorization  # in action
skip_policy_scope   # in action
```

---

**Q16. How does Devise handle password resets securely?**

```ruby
# Devise :recoverable module
# 1. User requests reset: POST /users/password (email param)
# 2. Devise generates a reset_password_token (random, hashed before storage)
# 3. Email sent with raw token in URL: /users/password/edit?reset_password_token=RAW
# 4. User submits new password with the raw token
# 5. Devise finds user by hashing the token, validates expiry, sets new password

# Token security:
# - Stored as HASH in DB (SHA256) — raw token only in email
# - Expires after: config.reset_password_within = 6.hours
# - Single-use: cleared after use

# The token in URL:
# /users/password/edit?reset_password_token=<raw_token>
# Rails + Devise = User.find_by(reset_password_token: Devise.token_generator.digest(User, :reset_password_token, raw_token))

# Custom expiry:
class User < ApplicationRecord
  devise :recoverable
  # In config/initializers/devise.rb:
  # config.reset_password_within = 2.hours
end

# Override reset password email:
# app/views/devise/mailer/reset_password_instructions.html.erb
```

---

**Q17. What is Warden? How does it relate to Devise?**

Warden is a Rack-based authentication framework that Devise is built on. It provides:
- Authentication strategies (pluggable — can add custom strategies)
- Session serialization/deserialization
- The `warden` object accessible in Rack env
- Failure app for handling auth failures

```ruby
# Devise implements Warden strategies:
class Devise::Strategies::DatabaseAuthenticatable < Devise::Strategies::Authenticatable
  def authenticate!
    resource = password.present? && mapping.to.find_for_database_authentication(authentication_hash)
    if validate(resource) { resource.valid_password?(password) }
      success!(resource)
    else
      fail!(I18n.t("devise.failure.#{resource ? 'invalid' : 'not_found_in_database'}", ...))
    end
  end
end

# Direct Warden access:
env['warden'].authenticate(:password)  # runs authentication strategy
env['warden'].user                     # current authenticated user
env['warden'].authenticated?           # boolean
env['warden'].logout                   # logs out user
```

---

**Q18. What is `protect_from_forgery` and when should you skip it in API controllers?**

```ruby
# protect_from_forgery: Rails CSRF protection (in ApplicationController by default)

# For API endpoints using token auth (JWT, API keys):
# - No session/cookie auth → CSRF doesn't apply
# - Request comes from non-browser client → no CSRF token available

# Skip for API controllers:
class Api::BaseController < ActionController::API
  # ActionController::API doesn't include CSRF protection by default
  # No need to skip
end

# Or if inheriting from ActionController::Base:
class Api::BaseController < ActionController::Base
  skip_before_action :verify_authenticity_token
  # Safe because we authenticate via header token, not session
end

# Or use null_session strategy (doesn't raise, just clears session):
class Api::BaseController < ActionController::Base
  protect_from_forgery with: :null_session
end

# The danger: if you skip CSRF for a controller that DOES use session auth,
# you open that endpoint to CSRF attacks
```

---

**Q19. What is Devise's Lockable module? How does brute-force protection work?**

```ruby
# Devise :lockable — lock account after N failed sign-in attempts

class User < ApplicationRecord
  devise :lockable,
         lock_strategy: :failed_attempts,  # lock after X failed attempts
         unlock_strategy: :email           # send unlock email (or :time, :both, :none)
end

# config/initializers/devise.rb:
config.maximum_attempts = 5           # Lock after 5 failed attempts
config.lock_keys = [:failed_sign_in_count]
config.unlock_in = 1.hour             # Auto-unlock after 1 hour (if :time strategy)

# Migration adds:
# t.integer  :failed_attempts, default: 0
# t.string   :unlock_token
# t.datetime :locked_at

# When locked: redirect to new_user_unlock_path with alert
# User can request unlock email

# In API context:
def create_session
  user = User.find_by(email: params[:email])
  if user&.access_locked?
    render json: { error: "Account locked. Check your email to unlock." }, status: :forbidden
  elsif user&.authenticate(params[:password])
    user.unlock_access! if user.access_locked?  # reset failed attempts
    # sign in...
  else
    user&.increment_failed_attempts
    render json: { error: "Invalid credentials" }, status: :unauthorized
  end
end
```

---

**Q20. How do you implement "sign in as another user" (impersonation) feature for admins?**

```ruby
class Admin::ImpersonationsController < Admin::BaseController
  def create
    @user = User.find(params[:user_id])
    authorize @user, :impersonate?

    session[:admin_id] = current_user.id
    session[:user_id] = @user.id

    redirect_to root_path, notice: "Viewing as #{@user.email}"
  end

  def destroy
    admin_id = session.delete(:admin_id)
    session[:user_id] = admin_id
    redirect_to admin_users_path, notice: "Returned to admin account"
  end
end

class ApplicationController < ActionController::Base
  helper_method :current_user, :impersonating?

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  def impersonating?
    session[:admin_id].present?
  end

  private

  def actual_current_admin
    session[:admin_id].present? ? User.find(session[:admin_id]) : current_user
  end
end

# Policy:
class UserPolicy < ApplicationPolicy
  def impersonate?
    # Only super admins can impersonate
    # Cannot impersonate another admin
    user.super_admin? && !record.admin?
  end
end
```

