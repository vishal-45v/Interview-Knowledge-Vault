# Chapter 02 — Active Record: Scenario Questions

---

**S1. You open the Rails server logs and see this pattern for a single page load:**

```sql
SELECT * FROM posts LIMIT 20
SELECT * FROM users WHERE id = 1
SELECT * FROM users WHERE id = 2
SELECT * FROM users WHERE id = 1
SELECT * FROM users WHERE id = 3
...
```

What is this problem called, how did it happen, and how do you fix it?

```ruby
# The Bug:
def index
  @posts = Post.limit(20)
end
# View:
# posts.each { |post| post.user.name }

# Fix A: includes (choose preload or eager_load automatically)
@posts = Post.includes(:user).limit(20)

# Fix B: eager_load (single JOIN query — good when you need to filter/sort by user)
@posts = Post.eager_load(:user).limit(20)

# Verify with Bullet gem or check_for_n_plus_one in specs
```

---

**S2. A migration has already been run in production. A developer realizes there's a bug in it — they added the wrong column name. How do you fix this?**

```ruby
# WRONG approach: edit the existing migration file
# Running db:migrate again won't re-run already-migrated files

# CORRECT approach: write a NEW migration
rails generate migration FixColumnNameInProducts

# New migration:
class FixColumnNameInProducts < ActiveRecord::Migration[7.1]
  def change
    rename_column :products, :priice, :price  # fix the typo
  end
end

# In development only (if not deployed yet):
rails db:rollback  # reverts last migration
# edit the original migration
rails db:migrate   # re-runs it
# BUT: never rollback if the migration has run in production
```

---

**S3. You have a `Post` model with a `published` boolean. You want to ensure no two published posts have the same `slug`. Unpublished posts CAN have duplicate slugs. How do you implement this uniqueness validation?**

```ruby
class Post < ApplicationRecord
  validates :slug, uniqueness: { scope: :published,
                                  conditions: -> { where(published: true) },
                                  message: "has already been taken for published posts" }

  # Or at the database level (preferred for race conditions):
  # Add a unique partial index:
  # add_index :posts, :slug, unique: true, where: "published = true"

  # The DB-level constraint:
  class CreatePartialUniqueIndex < ActiveRecord::Migration[7.1]
    def change
      add_index :posts, :slug, unique: true,
                where: "published = true",
                name: "index_posts_on_slug_where_published"
    end
  end
end
```

---

**S4. A user reports that their account balance went negative during a flash sale where thousands of users were purchasing simultaneously. There are no app errors. What happened and how do you fix it?**

```ruby
# The Bug: classic TOCTOU (Time of Check to Time of Use) race condition
# Two threads both read balance: 50, both check 50 >= 30, both deduct 30
# Result: balance = -10

# Thread 1:                    # Thread 2:
# account.balance # => 50      # account.balance # => 50
# 50 >= 30? # => true          # 50 >= 30? # => true
# account.balance -= 30        # account.balance -= 30
# account.save  # balance=20   # account.save  # balance=20 ... then -10!

# Fix 1: Pessimistic locking (FOR UPDATE)
def purchase(amount)
  Account.transaction do
    account = Account.lock.find(id)  # SELECT ... FOR UPDATE
    raise InsufficientFunds if account.balance < amount
    account.update!(balance: account.balance - amount)
  end
end

# Fix 2: Optimistic locking (detect conflict, retry)
def purchase(amount)
  retries = 0
  begin
    Account.transaction do
      raise InsufficientFunds if account.balance < amount
      account.update!(balance: account.balance - amount)
    end
  rescue ActiveRecord::StaleObjectError
    retries += 1
    retry if retries < 3
    raise
  end
end

# Fix 3: SQL-level atomic decrement with constraint
Account.where(id: id)
       .where("balance >= ?", amount)
       .update_all("balance = balance - #{amount}")
# Returns 0 rows updated if balance insufficient — check count
```

---

**S5. You have a `User` model with a `before_destroy` callback that emails an admin. You also have `has_many :posts, dependent: :destroy`. What is the order of operations when you call `user.destroy`?**

```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  before_destroy :notify_admin
  after_destroy :log_deletion
end

# Order of operations:
# 1. before_destroy callback runs (notify_admin)
# 2. dependent: :destroy — each post is loaded and post.destroy is called
#    (this fires all Post's callbacks for each post)
# 3. SQL: DELETE FROM users WHERE id = ?
# 4. after_destroy callback runs (log_deletion)
# 5. (If inside transaction) after_commit runs

# GOTCHA: if notify_admin raises an exception, NO deletion happens
# GOTCHA: if dependent: :destroy is on a sub-association of post,
#          the chain cascades recursively before the user DELETE
```

---

**S6. A team member added `default_scope { order(created_at: :desc) }` to the `Message` model six months ago. Now you're trying to paginate with Kaminari and getting wrong results. What's happening?**

```ruby
# Bug: default_scope applies ORDER BY created_at DESC to ALL queries
# Kaminari's OFFSET-based pagination requires consistent ordering
# but if default_scope ORDER conflicts with Kaminari's ORDER, results are wrong

# Also: count queries with ORDER BY are slower (PostgreSQL may not use index for count)

# Diagnose:
Message.all.to_sql
# => "SELECT "messages".* FROM "messages" ORDER BY "messages"."created_at" DESC"
# That ORDER is ALWAYS there, even when you don't want it

# Fix option 1: Remove the default_scope, use explicit scope
class Message < ApplicationRecord
  # Remove: default_scope { order(created_at: :desc) }
  scope :chronological, -> { order(created_at: :asc) }
  scope :newest_first,  -> { order(created_at: :desc) }
end

# Fix option 2: Override with unscoped in specific contexts
Message.unscoped.where(user_id: 1).order(created_at: :asc)
```

---

**S7. You need to import 100,000 product records from a CSV file into the database as fast as possible. What approach do you take?**

```ruby
require 'csv'

# Approach 1: insert_all (fastest — single SQL statement per batch)
batch_size = 1000
now = Time.current

CSV.foreach("products.csv", headers: true).each_slice(batch_size) do |batch|
  rows = batch.map do |row|
    {
      name: row["name"],
      price: row["price"].to_f,
      sku: row["sku"],
      created_at: now,
      updated_at: now
    }
  end
  Product.insert_all(rows)  # Skips validations + callbacks — intentional here
end

# Approach 2: COPY command (PostgreSQL only, fastest possible)
# Uses psql COPY protocol — fastest for bulk loading
require 'pg'
conn = ActiveRecord::Base.connection.raw_connection
conn.copy_data("COPY products (name, price, sku) FROM STDIN WITH CSV") do
  CSV.foreach("products.csv", headers: true) do |row|
    conn.put_copy_data("#{row['name']},#{row['price']},#{row['sku']}\n")
  end
end

# Benchmarks (rough): 100k records
# Product.create! loop: ~120 seconds
# insert_all in batches of 1000: ~3 seconds
# COPY: ~0.5 seconds
```

---

**S8. You have a polymorphic association `has_many :events, as: :eventable`. When you run `Event.includes(:eventable).all`, you get N+1 queries for one of the eventable types. Why?**

```ruby
# Polymorphic includes limitation:
# Rails needs to know all types in the result to batch the queries
# If your events have eventable_type: ['Post', 'Video', 'Article'],
# Rails runs:
#   SELECT * FROM posts WHERE id IN (...)
#   SELECT * FROM videos WHERE id IN (...)
#   SELECT * FROM articles WHERE id IN (...)
# That's fine — 3 queries, not N+1

# The N+1 scenario: when a type isn't in the eager_load scope
# or when using .joins instead of .includes for a sub-association

# Example causing N+1:
Event.includes(:eventable).where(eventable_type: "Post").map do |event|
  event.eventable.comments.first  # N+1! comments not included
end

# Fix:
Event.includes(eventable: :comments)
     .where(eventable_type: "Post")

# Another gotcha: polymorphic includes can't do JOIN for filtering
# (can't JOIN posts AND videos in same query)
Event.joins(:eventable)  # FAILS — can't join a polymorphic association
# Must use subquery or filter per type
```

---

**S9. Your application has a `User` model and an `AdminUser` model that need to share most attributes (email, name, password) but have different behaviour and roles. How do you model this?**

```ruby
# Option 1: STI (simplest — same table)
class User < ApplicationRecord
  # type column required
end
class AdminUser < User
  has_many :audit_logs
  def admin? = true
end
# Tradeoff: all in users table; NULL columns for admin-only attributes

# Option 2: Separate tables with shared concern (no inheritance)
module Authenticatable
  extend ActiveSupport::Concern
  included do
    has_secure_password
    validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  end

  def authenticate_session(password)
    authenticate(password)
  end
end

class User < ApplicationRecord
  include Authenticatable
  has_many :orders
end
class AdminUser < ApplicationRecord
  include Authenticatable
  has_many :audit_logs
  belongs_to :department
end
# Tradeoff: duplicate tables, more migrations; better isolation

# Option 3: Role-based on single User model (usually best)
class User < ApplicationRecord
  enum :role, { customer: 0, admin: 1, super_admin: 2 }

  def admin? = role.in?(%w[admin super_admin])
end
# Simplest; use Pundit policies for behaviour differences
```

---

**S10. A complex query is taking 800ms. You run EXPLAIN ANALYZE and see a sequential scan on the `orders` table. How do you optimize it?**

```ruby
# Step 1: Get the SQL
scope = Order.where(status: :pending, created_at: 30.days.ago..)
             .includes(:user)
             .order(created_at: :desc)

puts scope.to_sql
# SELECT orders.* FROM orders WHERE status = 2 AND created_at >= '...' ORDER BY created_at DESC

# Step 2: Identify missing index
# Migration to add composite index:
class AddIndexToOrders < ActiveRecord::Migration[7.1]
  def change
    # Composite index for common query pattern
    add_index :orders, [:status, :created_at],
              name: "index_orders_on_status_and_created_at"
    # If only filtering by status:
    add_index :orders, :status
  end
end

# Step 3: For enum — ensure you're comparing integer not string
Order.where(status: Order.statuses[:pending])  # compares integer 0
# vs
Order.where(status: :pending)  # Rails translates to integer automatically

# Step 4: Avoid SELECT * when you only need a few columns
Order.where(status: :pending).select(:id, :user_id, :total, :created_at)

# Step 5: Check N+1 from includes
Order.where(status: :pending).includes(:user).each do |order|
  # order.user already loaded — no extra queries
  puts order.user.email
end
```

---

**S11. How do you write a custom validator that can be reused across multiple models?**

```ruby
# app/validators/phone_number_validator.rb
class PhoneNumberValidator < ActiveModel::EachValidator
  PHONE_REGEX = /\A[\+]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{3}[-\s\.]?[0-9]{4,6}\z/

  def validate_each(record, attribute, value)
    unless value =~ PHONE_REGEX
      record.errors.add(attribute, options[:message] || "is not a valid phone number")
    end
  end
end

# Usage across models:
class User < ApplicationRecord
  validates :mobile, phone_number: true
end

class Contact < ApplicationRecord
  validates :phone, phone_number: { message: "must be a 10-digit US number" }
end

# Or as a class validator with validate method:
class PastDateValidator < ActiveModel::Validator
  def validate(record)
    if record.birth_date.present? && record.birth_date > Date.today
      record.errors.add(:birth_date, "must be in the past")
    end
  end
end

class Profile < ApplicationRecord
  validates_with PastDateValidator
end
```

---

**S12. You notice that a `before_save` callback is causing issues in your test suite — it makes HTTP calls to a third-party API. How do you structure this to be testable?**

```ruby
# Problem code:
class User < ApplicationRecord
  before_save :sync_to_crm

  private

  def sync_to_crm
    CrmClient.new.update_contact(email: email, name: name)
  end
end

# Solution 1: Use after_commit + job (recommended)
class User < ApplicationRecord
  after_commit :enqueue_crm_sync, on: [:create, :update]

  private

  def enqueue_crm_sync
    CrmSyncJob.perform_later(id)
  end
end
# Test: job is enqueued, not executed inline

# Solution 2: Inject the CRM client for testability
class User < ApplicationRecord
  after_commit :sync_to_crm, on: [:create, :update]

  def sync_to_crm(client: CrmClient.new)
    return unless email_previously_changed? || name_previously_changed?
    client.update_contact(email: email, name: name)
  end
end
# Test: user.sync_to_crm(client: double('CrmClient', update_contact: true))

# Solution 3: Skip in test environment
class User < ApplicationRecord
  after_save :sync_to_crm, unless: -> { Rails.env.test? }
end
# Anti-pattern: test environment should mirror production behaviour
```

---

**S13. Explain how you would implement soft deletes (paranoia) without the `paranoia` gem.**

```ruby
# Migration:
add_column :users, :deleted_at, :datetime, index: true

# Model:
class User < ApplicationRecord
  # Default scope to exclude deleted records
  default_scope { where(deleted_at: nil) }

  def soft_delete
    update_column(:deleted_at, Time.current)
  end

  def restore
    update_column(:deleted_at, nil)
  end

  def deleted?
    deleted_at.present?
  end

  # Override destroy to soft-delete instead
  def destroy
    soft_delete
  end

  # Access deleted records:
  # User.unscoped.where.not(deleted_at: nil)
  # User.unscoped.all  — all including deleted

  # Unique index with soft delete:
  # add_index :users, :email, unique: true, where: "deleted_at IS NULL"
end

# Better approach with a concern:
module SoftDeletable
  extend ActiveSupport::Concern

  included do
    default_scope { where(deleted_at: nil) }
    scope :only_deleted,    -> { unscoped.where.not(deleted_at: nil) }
    scope :with_deleted,    -> { unscoped }
  end

  def soft_delete!
    update!(deleted_at: Time.current)
  end

  def restore!
    update!(deleted_at: nil)
  end

  def deleted? = deleted_at.present?
end
```

---

**S14. You have a model with many callbacks that are making it hard to understand what happens on save. How do you refactor?**

```ruby
# Messy model:
class Order < ApplicationRecord
  before_validation :normalize_address
  after_validation  :geocode_address
  before_save       :calculate_tax
  before_save       :apply_discount
  before_create     :generate_order_number
  after_create      :notify_warehouse
  after_create      :send_confirmation_email
  after_update      :notify_changes
  after_commit      :sync_to_erp
  after_destroy     :cleanup_files
end

# Refactored using Service Object:
class OrderCreationService
  def initialize(params, user:)
    @params = params
    @user = user
  end

  def call
    ActiveRecord::Base.transaction do
      order = Order.new(normalized_params)
      order.order_number = generate_order_number
      order.tax = TaxCalculator.new(order).calculate
      order.apply_coupon(@params[:coupon_code])
      order.save!
      OrderMailer.confirmation(order).deliver_later
      WarehouseNotifier.new(order).notify
      order
    end
  end

  private

  def normalized_params
    @params.merge(address: AddressNormalizer.call(@params[:address]))
  end

  def generate_order_number
    "ORD-#{SecureRandom.hex(6).upcase}"
  end
end

# Keep only truly model-intrinsic callbacks:
class Order < ApplicationRecord
  before_save :set_status_timestamps
  after_commit :sync_to_erp  # External sync always after commit

  private
  def set_status_timestamps
    self.shipped_at = Time.current if status_changed? && shipped?
  end
end
```

---

**S15. A migration adds a column to a large table (50 million rows) in production. What problems can occur and how do you handle it?**

```ruby
# Problem: ALTER TABLE on a large table can lock the table for minutes
# In PostgreSQL: adding a column with NOT NULL + DEFAULT causes full table rewrite

# BAD approach — causes table lock:
class AddStatusToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :status, :integer, null: false, default: 0
    # On PostgreSQL < 11: full table rewrite!
  end
end

# GOOD approach — use strong_migrations gem or manual steps:

# Step 1: Add nullable column first (instant, no lock)
class AddStatusToOrders < ActiveRecord::Migration[7.1]
  def change
    add_column :orders, :status, :integer
  end
end

# Step 2: Backfill in batches (separate deploy)
Order.find_in_batches(batch_size: 10_000) do |batch|
  Order.where(id: batch.map(&:id)).update_all(status: 0)
end

# Step 3: Add NOT NULL constraint + default (after backfill)
class AddNotNullToOrderStatus < ActiveRecord::Migration[7.1]
  def change
    change_column_null :orders, :status, false
    change_column_default :orders, :status, 0
  end
end

# PostgreSQL 11+: adding column with constant default is instant (no table rewrite)
# Use strong_migrations gem to catch these issues automatically

# gem 'strong_migrations'
# It raises errors for dangerous migrations and suggests safe alternatives
```

---

**S16. How do you handle a has_many relationship where you need to load ONLY the latest record per user?**

```ruby
# Scenario: User has_many orders; you want the most recent order per user

# Approach 1: has_one with custom order
class User < ApplicationRecord
  has_one :latest_order, -> { order(created_at: :desc) }, class_name: "Order"
end

User.includes(:latest_order).map { |u| [u.name, u.latest_order&.total] }

# Approach 2: Custom SQL with DISTINCT ON (PostgreSQL)
class User < ApplicationRecord
  has_one :latest_order,
          -> { from(Order.select("DISTINCT ON (user_id) *").order("user_id, created_at DESC"), :orders) },
          class_name: "Order"
end

# Approach 3: Window function query (most flexible)
latest_orders = Order
  .select("DISTINCT ON (user_id) *")
  .order("user_id, created_at DESC")

users_with_orders = User
  .joins("JOIN (#{latest_orders.to_sql}) latest_orders ON users.id = latest_orders.user_id")
  .select("users.*, latest_orders.total AS latest_order_total")

# Approach 4: Lateral join (PostgreSQL 9.3+)
# SELECT users.*, latest.* FROM users
# LEFT JOIN LATERAL (
#   SELECT * FROM orders WHERE orders.user_id = users.id
#   ORDER BY created_at DESC LIMIT 1
# ) latest ON true
```

---

**S17. You're debugging and find that `user.save` returns false but `user.errors` is empty. What are the possible causes?**

```ruby
# Possible causes:
# 1. A callback returns false (halts callback chain in older Rails)
# In Rails 5+: callbacks must explicitly throw(:abort) to halt
before_save :check_something
def check_something
  throw(:abort) if some_condition  # halts save, no error added
  # Returning false from callback in Rails <= 4 halted; Rails 5+ it doesn't
end

# 2. Database constraint violation caught silently
# If you rescue ActiveRecord::StatementInvalid in a callback and don't add errors

# 3. Custom validate method that doesn't add errors correctly
def validate_something
  # BUG: returns false but forgets to add error
  false  # No error added! save returns false but errors is empty
end
# Fix:
def validate_something
  errors.add(:base, "Something went wrong") if some_invalid_condition
end

# 4. before_validation callback calls throw(:abort)

# Diagnosis:
user.valid?                          # run validations, check errors
user.errors.full_messages            # see all errors
ActiveRecord::Base.logger = Logger.new(STDOUT)  # see SQL
# Wrap in transaction and check:
User.transaction do
  user.save
  raise ActiveRecord::Rollback  # see what SQL ran
end
```

---

**S18. Implement a versioning/audit system for a model using callbacks.**

```ruby
# Migration for versions table:
create_table :versions do |t|
  t.string  :item_type, null: false
  t.bigint  :item_id, null: false
  t.string  :event, null: false     # 'create', 'update', 'destroy'
  t.jsonb   :object_changes         # what changed
  t.bigint  :whodunnit              # user_id who made change
  t.timestamps
end
add_index :versions, [:item_type, :item_id]

# Concern:
module Auditable
  extend ActiveSupport::Concern

  included do
    after_create  :log_create
    after_update  :log_update
    after_destroy :log_destroy
  end

  private

  def log_create
    create_version_record("create", saved_changes)
  end

  def log_update
    create_version_record("update", saved_changes) if saved_changes.any?
  end

  def log_destroy
    create_version_record("destroy", attributes)
  end

  def create_version_record(event, changes)
    Version.create!(
      item_type: self.class.name,
      item_id: id,
      event: event,
      object_changes: changes,
      whodunnit: Current.user&.id
    )
  end
end

# Usage:
class Product < ApplicationRecord
  include Auditable
end

# Retrieve audit trail:
Version.where(item_type: "Product", item_id: 1).order(created_at: :asc)
```

---

**S19. You need to seed 10,000 users with realistic data for load testing. How do you write the seed file efficiently?**

```ruby
# db/seeds.rb
require 'faker'

puts "Seeding users..."

users_data = Array.new(10_000) do |i|
  {
    name: Faker::Name.name,
    email: "user#{i}@example.com",  # Faker::Internet.email can have collisions
    password_digest: BCrypt::Password.create("password123"),
    created_at: rand(2.years.ago..Time.current),
    updated_at: Time.current
  }
end

# Insert in batches to avoid memory issues
users_data.each_slice(500) do |batch|
  User.insert_all(batch)
end

puts "Creating posts..."
user_ids = User.pluck(:id)

posts_data = Array.new(50_000) do |i|
  {
    title: Faker::Lorem.sentence(word_count: 6),
    body: Faker::Lorem.paragraphs(number: 4).join("\n\n"),
    user_id: user_ids.sample,
    published: [true, false].sample,
    created_at: rand(1.year.ago..Time.current),
    updated_at: Time.current
  }
end

posts_data.each_slice(1000) do |batch|
  Post.insert_all(batch)
end

puts "Done! Users: #{User.count}, Posts: #{Post.count}"
```

---

**S20. Your team debates using `after_create` vs `after_commit on: :create` for sending a welcome email. Settle the debate with a concrete example.**

```ruby
# SCENARIO: payment processing that can fail

class User < ApplicationRecord
  # DANGEROUS: after_create fires inside the transaction
  # If a subsequent operation in the same transaction fails and rolls back,
  # the email was already sent but the user record doesn't exist!

  after_create :send_welcome_email_DANGEROUS

  def send_welcome_email_DANGEROUS
    UserMailer.welcome(self).deliver_now
    # Email sent. But if the caller's transaction rolls back...
    # User is gone from DB, but email already sent. Confusion ensues.
  end
end

# Example of the bug:
ActiveRecord::Base.transaction do
  user = User.create!(email: "alice@example.com")  # after_create fires: email sent!
  payment_profile = PaymentProfile.create!(user: user, card_token: "invalid")
  # ^ This raises ActiveRecord::RecordInvalid
  # Transaction rolls back: user is deleted from DB
  # But welcome email was already sent to alice@example.com!
end

class User < ApplicationRecord
  # SAFE: after_commit on: :create fires ONLY after DB commit
  after_commit :send_welcome_email, on: :create

  private

  def send_welcome_email
    UserMailer.welcome(self).deliver_later  # Use deliver_later not deliver_now
    # If email job is enqueued after commit, user is guaranteed to exist in DB
  end
end

# Verdict: ALWAYS use after_commit for external side effects.
# Use after_save only for intra-transaction consistency (e.g., updating another column).
```

