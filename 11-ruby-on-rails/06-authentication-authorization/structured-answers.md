# Chapter 06 — Authentication & Authorization: Structured Answers

---

## Answer 1: Complete Pundit Authorization System with Role-Based Access

**Question:** Implement a complete Pundit-based authorization system with roles, scopes, and admin override.

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ROLES = %w[viewer editor moderator admin].freeze
  has_many :role_assignments
  has_many :roles, through: :role_assignments

  def role?(role_name)
    roles.exists?(name: role_name)
  end

  def admin?
    role?('admin')
  end
end

# app/models/role.rb
class Role < ApplicationRecord
  NAMES = %w[viewer editor moderator admin].freeze
  validates :name, inclusion: { in: NAMES }
end

# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    raise Pundit::NotAuthorizedError, "must be logged in" unless user
    @user   = user
    @record = record
  end

  def index?   = false
  def show?    = false
  def create?  = false
  def update?  = false
  def destroy? = false

  class Scope
    attr_reader :user, :scope

    def initialize(user, scope)
      raise Pundit::NotAuthorizedError, "must be logged in" unless user
      @user  = user
      @scope = scope
    end

    def resolve
      raise NotImplementedError, "#{self.class}#resolve is not implemented"
    end
  end
end

# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def index?
    true  # Everyone can see the list (scope filters what they see)
  end

  def show?
    record.published? || owner? || moderator_or_admin?
  end

  def create?
    user.role?('editor') || user.admin?
  end

  def update?
    (owner? && user.role?('editor')) || moderator_or_admin?
  end

  def destroy?
    owner? || user.admin?
  end

  def publish?
    user.role?('moderator') || user.admin?
  end

  class Scope < Scope
    def resolve
      if user.admin? || user.role?('moderator')
        scope.all
      elsif user.role?('editor')
        scope.where("published = ? OR user_id = ?", true, user.id)
      else
        scope.published
      end
    end
  end

  private

  def owner?
    record.user == user
  end

  def moderator_or_admin?
    user.role?('moderator') || user.admin?
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  rescue_from Pundit::NotAuthorizedError, with: :handle_not_authorized
  rescue_from Pundit::NotDefinedError,    with: :handle_policy_missing

  after_action :verify_authorized,     except: :index
  after_action :verify_policy_scoped,  only:   :index

  private

  def handle_not_authorized(exception)
    policy_name = exception.policy.class.to_s.underscore
    flash[:alert] = t("#{policy_name}.#{exception.query}", scope: "pundit", default: :default)
    redirect_back_or_to(root_path)
  end

  def handle_policy_missing
    render plain: "Policy not found", status: :internal_server_error
  end
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = policy_scope(Post).includes(:user).page(params[:page])
  end

  def show
    @post = Post.find(params[:id])
    authorize @post
  end

  def create
    @post = current_user.posts.build(post_params)
    authorize @post

    if @post.save
      redirect_to @post, notice: "Post created"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    @post = Post.find(params[:id])
    authorize @post

    if @post.update(post_params)
      redirect_to @post, notice: "Post updated"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def publish
    @post = Post.find(params[:id])
    authorize @post, :publish?

    @post.update!(published: true, published_at: Time.current)
    redirect_to @post, notice: "Post published"
  end

  private

  def post_params
    params.require(:post).permit(:title, :body, :category)
  end
end
```

---

## Answer 2: Complete Devise + OmniAuth Integration with Account Linking

**Question:** Implement Google OAuth login that handles both new users and linking to existing email/password accounts.

```ruby
# Gemfile:
# gem 'devise'
# gem 'omniauth-google-oauth2'
# gem 'omniauth-rails_csrf_protection'

# db/migrate/xxx_add_oauth_to_users.rb
class AddOauthToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :provider, :string
    add_column :users, :uid, :string
    add_column :users, :avatar_url, :string
    add_index :users, [:provider, :uid], unique: true
  end
end

# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable,
         :rememberable, :validatable, :confirmable, :omniauthable,
         omniauth_providers: [:google_oauth2]

  def self.from_omniauth(auth)
    # Find by provider+uid (existing OAuth account)
    existing_oauth = find_by(provider: auth.provider, uid: auth.uid)
    return existing_oauth if existing_oauth

    # Find by email (existing email/password account — link it)
    existing_email = find_by(email: auth.info.email)
    if existing_email
      existing_email.update!(
        provider:   auth.provider,
        uid:        auth.uid,
        avatar_url: auth.info.image
      )
      return existing_email
    end

    # New user via OAuth
    create!(
      provider:   auth.provider,
      uid:        auth.uid,
      email:      auth.info.email,
      name:       auth.info.name,
      avatar_url: auth.info.image,
      password:   Devise.friendly_token[0, 20],  # Random password (not used)
      confirmed_at: Time.current  # Skip email confirmation for OAuth users
    )
  end

  def oauth_user?
    provider.present? && uid.present?
  end

  def password_required?
    # OAuth users don't need a password
    super && !oauth_user?
  end
end

# app/controllers/users/omniauth_callbacks_controller.rb
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    @user = User.from_omniauth(request.env["omniauth.auth"])

    if @user.persisted?
      flash[:notice] = "Signed in with Google"
      sign_in_and_redirect @user, event: :authentication
    else
      session["devise.google_data"] = request.env["omniauth.auth"].except("extra")
      redirect_to new_user_registration_url, alert: @user.errors.full_messages.join("\n")
    end
  rescue ActiveRecord::RecordInvalid => e
    flash[:alert] = "Could not sign in: #{e.message}"
    redirect_to new_user_session_path
  end

  def failure
    redirect_to root_path, alert: "Authentication failed: #{failure_message}"
  end
end

# config/initializers/devise.rb
Devise.setup do |config|
  config.omniauth :google_oauth2,
    Rails.application.credentials.dig(:google, :client_id),
    Rails.application.credentials.dig(:google, :client_secret),
    scope: 'email,profile',
    prompt: 'select_account',
    access_type: 'online'
end
```

---

## Answer 3: TOTP Two-Factor Authentication

**Question:** Implement TOTP-based 2FA with backup codes and "remember this device" functionality.

```ruby
# Gemfile: gem 'rotp', gem 'rqrcode'

# db/migrate/xxx_add_2fa_to_users.rb
class Add2faToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :otp_secret, :string
    add_column :users, :otp_enabled, :boolean, default: false, null: false
    add_column :users, :otp_backup_codes, :string, array: true, default: []
  end
end

# app/models/user.rb
class User < ApplicationRecord
  encrypts :otp_secret  # Rails 7 ActiveRecord encryption

  def generate_otp_secret!
    update!(otp_secret: ROTP::Base32.random)
  end

  def otp_provisioning_uri
    ROTP::TOTP.new(otp_secret, issuer: "MyApp").provisioning_uri(email)
  end

  def verify_otp(code)
    totp = ROTP::TOTP.new(otp_secret)
    totp.verify(code, drift_behind: 15, drift_ahead: 15)
  end

  def generate_backup_codes!
    codes = Array.new(10) { SecureRandom.hex(5).upcase }
    update!(otp_backup_codes: codes.map { |c| BCrypt::Password.create(c) })
    codes  # Return plain codes once — never stored plain
  end

  def use_backup_code!(code)
    valid_index = otp_backup_codes.find_index { |hashed| BCrypt::Password.new(hashed) == code }
    return false unless valid_index

    remaining = otp_backup_codes.dup
    remaining.delete_at(valid_index)
    update!(otp_backup_codes: remaining)
    true
  end

  def otp_required?
    otp_enabled?
  end
end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :require_two_factor!

  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      if user.otp_required?
        session[:pending_user_id] = user.id
        session[:pending_user_expires] = 10.minutes.from_now.to_i
        redirect_to new_two_factor_path
      else
        complete_sign_in(user)
      end
    else
      flash.now[:alert] = "Invalid credentials"
      render :new, status: :unprocessable_entity
    end
  end
end

# app/controllers/two_factors_controller.rb
class TwoFactorsController < ApplicationController
  skip_before_action :require_two_factor!
  before_action :load_pending_user

  def create
    code = params[:otp_attempt]&.gsub(/\s/, '')

    if @user.verify_otp(code) || @user.use_backup_code!(code)
      session.delete(:pending_user_id)
      session.delete(:pending_user_expires)

      if params[:remember_device] == '1'
        cookies.signed[:device_trust] = {
          value: { user_id: @user.id, token: generate_device_token(@user) }.to_json,
          expires: 30.days,
          httponly: true,
          secure: Rails.env.production?
        }
      end

      complete_sign_in(@user)
    else
      flash.now[:alert] = "Invalid code"
      render :new, status: :unprocessable_entity
    end
  end

  private

  def load_pending_user
    user_id = session[:pending_user_id]
    expires = session[:pending_user_expires]

    unless user_id && expires && Time.current.to_i < expires.to_i
      session.delete(:pending_user_id)
      redirect_to sign_in_path, alert: "Session expired"
      return
    end

    @user = User.find_by(id: user_id)
    redirect_to sign_in_path unless @user
  end

  def generate_device_token(user)
    # Store hashed version in DB for verification
    token = SecureRandom.hex(32)
    user.trusted_devices.create!(
      token_digest: Digest::SHA256.hexdigest(token),
      expires_at: 30.days.from_now
    )
    token
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :require_two_factor!

  private

  def require_two_factor!
    return unless current_user&.otp_required?
    return if device_trusted?
    return if session[:two_factor_verified] == current_user.id

    redirect_to new_two_factor_path
  end

  def device_trusted?
    return false unless cookies.signed[:device_trust]
    data = JSON.parse(cookies.signed[:device_trust], symbolize_names: true)
    return false unless data[:user_id] == current_user&.id

    current_user.trusted_devices.active
                .exists?(token_digest: Digest::SHA256.hexdigest(data[:token]))
  rescue JSON::ParserError
    false
  end
end
```

---

## Answer 4: Personal Access Tokens API

**Question:** Implement GitHub-style personal access tokens with scopes, display-once behavior, and audit logging.

```ruby
# db/migrate/xxx_create_personal_access_tokens.rb
class CreatePersonalAccessTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :personal_access_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string :name, null: false
      t.string :token_digest, null: false
      t.string :token_prefix, null: false  # First 8 chars for identification
      t.string :scopes, array: true, default: []
      t.datetime :last_used_at
      t.inet :last_used_ip
      t.datetime :expires_at
      t.datetime :revoked_at
      t.timestamps
    end
    add_index :personal_access_tokens, :token_digest, unique: true
    add_index :personal_access_tokens, :token_prefix
  end
end

# app/models/personal_access_token.rb
class PersonalAccessToken < ApplicationRecord
  SCOPES = %w[read:user write:user read:posts write:posts admin].freeze
  TOKEN_PREFIX = "pat_"

  belongs_to :user
  validates :name, presence: true
  validates :scopes, presence: true
  validate  :scopes_must_be_valid

  scope :active, -> { where(revoked_at: nil).where("expires_at IS NULL OR expires_at > ?", Time.current) }

  def self.generate!(user:, name:, scopes:, expires_in: nil)
    raw_token = "#{TOKEN_PREFIX}#{SecureRandom.hex(32)}"

    token = create!(
      user:         user,
      name:         name,
      scopes:       scopes,
      token_digest: Digest::SHA256.hexdigest(raw_token),
      token_prefix: raw_token.first(12),
      expires_at:   expires_in ? expires_in.from_now : nil
    )

    # Return both — raw_token shown once and NOT stored
    [token, raw_token]
  end

  def self.authenticate(raw_token)
    return nil unless raw_token&.start_with?(TOKEN_PREFIX)

    digest = Digest::SHA256.hexdigest(raw_token)
    token  = active.find_by(token_digest: digest)
    return nil unless token

    token.touch_usage
    token
  end

  def touch_usage
    update_columns(last_used_at: Time.current, last_used_ip: Current.request&.remote_ip)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  def has_scope?(scope)
    scopes.include?(scope) || scopes.include?('admin')
  end

  def active?
    revoked_at.nil? && (expires_at.nil? || expires_at > Time.current)
  end

  private

  def scopes_must_be_valid
    invalid = scopes - SCOPES
    errors.add(:scopes, "contains invalid scopes: #{invalid.join(', ')}") if invalid.any?
  end
end

# app/controllers/api/v1/personal_access_tokens_controller.rb
class Api::V1::PersonalAccessTokensController < Api::V1::BaseController
  def index
    @tokens = current_user.personal_access_tokens.active.order(created_at: :desc)
    render json: @tokens.map { |t|
      { id: t.id, name: t.name, scopes: t.scopes, prefix: t.token_prefix,
        last_used_at: t.last_used_at, expires_at: t.expires_at }
    }
  end

  def create
    token_params = params.require(:token).permit(:name, :expires_in, scopes: [])
    expires_in   = token_params[:expires_in].present? ? token_params[:expires_in].to_i.days : nil

    token, raw_token = PersonalAccessToken.generate!(
      user:       current_user,
      name:       token_params[:name],
      scopes:     token_params[:scopes] || [],
      expires_in: expires_in
    )

    render json: {
      id:         token.id,
      token:      raw_token,  # Shown ONCE — never retrievable again
      name:       token.name,
      scopes:     token.scopes,
      expires_at: token.expires_at,
      message:    "Save this token — it will not be shown again"
    }, status: :created
  rescue ActiveRecord::RecordInvalid => e
    render json: { error: e.message }, status: :unprocessable_entity
  end

  def destroy
    token = current_user.personal_access_tokens.find(params[:id])
    token.revoke!
    head :no_content
  end
end

# app/controllers/api/v1/base_controller.rb — authenticate via PAT or JWT
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!

  private

  def authenticate!
    auth_header = request.headers['Authorization']

    if auth_header&.start_with?('Bearer pat_')
      authenticate_with_pat(auth_header.delete_prefix('Bearer '))
    else
      authenticate_with_jwt(auth_header&.delete_prefix('Bearer '))
    end
  end

  def authenticate_with_pat(raw_token)
    @auth_token = PersonalAccessToken.authenticate(raw_token)

    unless @auth_token
      render json: { error: { code: 'unauthorized', message: 'Invalid or expired token' } },
             status: :unauthorized
      return
    end

    @current_user = @auth_token.user
  end

  def require_scope!(scope)
    return if @auth_token.nil?  # JWT users have full access
    return if @auth_token.has_scope?(scope)

    render json: { error: { code: 'insufficient_scope', message: "Requires #{scope} scope" } },
           status: :forbidden
  end
end
```

---

## Answer 5: User Impersonation System

**Question:** Implement a secure admin impersonation feature with audit logging.

```ruby
# db/migrate/xxx_create_impersonation_logs.rb
class CreateImpersonationLogs < ActiveRecord::Migration[7.1]
  def change
    create_table :impersonation_logs do |t|
      t.references :admin, null: false, foreign_key: { to_table: :users }
      t.references :target_user, null: false, foreign_key: { to_table: :users }
      t.string :admin_ip, null: false
      t.datetime :started_at, null: false
      t.datetime :ended_at
      t.timestamps
    end
  end
end

# app/models/impersonation_log.rb
class ImpersonationLog < ApplicationRecord
  belongs_to :admin, class_name: 'User'
  belongs_to :target_user, class_name: 'User'

  scope :active, -> { where(ended_at: nil) }

  def end!
    update!(ended_at: Time.current)
  end
end

# app/controllers/admin/impersonations_controller.rb
class Admin::ImpersonationsController < Admin::BaseController
  before_action :require_super_admin!
  before_action :prevent_nested_impersonation!

  def create
    target = User.find(params[:user_id])

    # Log the impersonation
    log = ImpersonationLog.create!(
      admin:       current_user,
      target_user: target,
      admin_ip:    request.remote_ip,
      started_at:  Time.current
    )

    # Store original admin identity in session
    session[:impersonating_user_id]   = target.id
    session[:original_admin_id]       = current_user.id
    session[:impersonation_log_id]    = log.id

    Rails.logger.warn "[IMPERSONATION] Admin #{current_user.email} (#{current_user.id}) " \
                      "started impersonating User #{target.email} (#{target.id}) " \
                      "from IP #{request.remote_ip}"

    redirect_to root_path, notice: "You are now impersonating #{target.email}"
  end

  def destroy
    log_id = session[:impersonation_log_id]
    ImpersonationLog.find_by(id: log_id)&.end!

    impersonated_email = User.find_by(id: session[:impersonating_user_id])&.email

    session.delete(:impersonating_user_id)
    session.delete(:impersonation_log_id)
    # session[:original_admin_id] remains — we're back to being the admin

    Rails.logger.info "[IMPERSONATION] Admin #{current_user.email} ended impersonation of #{impersonated_email}"

    redirect_to admin_path, notice: "Impersonation ended"
  end

  private

  def require_super_admin!
    unless current_user.role?('super_admin')
      redirect_to admin_path, alert: "Super admin access required"
    end
  end

  def prevent_nested_impersonation!
    if session[:impersonating_user_id].present?
      redirect_to admin_path, alert: "Cannot impersonate while already impersonating"
    end
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  helper_method :current_user, :impersonating?

  private

  def current_user
    @current_user ||= if session[:impersonating_user_id]
      User.find_by(id: session[:impersonating_user_id])
    else
      User.find_by(id: session[:user_id])
    end
  end

  def real_admin
    return nil unless session[:original_admin_id]
    User.find_by(id: session[:original_admin_id])
  end

  def impersonating?
    session[:impersonating_user_id].present?
  end
end

# app/views/layouts/application.html.erb (impersonation banner)
# <% if impersonating? %>
#   <div class="impersonation-banner">
#     Impersonating: <%= current_user.email %>
#     <%= button_to "End Impersonation", admin_impersonation_path, method: :delete, class: "btn btn-danger" %>
#   </div>
# <% end %>
```

---
