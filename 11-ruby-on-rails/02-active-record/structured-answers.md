# Chapter 02 — Active Record: Structured Answers

---

## Answer 1: Comprehensive N+1 Fix Strategy

**Question:** You're given a view that iterates over `@posts` and calls `post.author.name`, `post.category.name`, and `post.comments.count`. Optimize all three.

```ruby
# Original (3 N+1 problems):
@posts = Post.limit(20)
# View accesses: post.author.name, post.category.name, post.comments.count

# Step 1: Fix author and category with includes
@posts = Post.includes(:author, :category).limit(20)
# Now: author and category loaded in 2 extra queries (IN clause)

# Step 2: Fix comments.count with counter_cache
# Migration:
class AddCommentsCountToPosts < ActiveRecord::Migration[7.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
    # Backfill existing data:
    Post.find_each { |post| Post.reset_counters(post.id, :comments) }
  end
end

class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Now post.comments.size uses comments_count column (no SQL)
# Final query: 3 SQL statements total for any number of posts

# Step 3: Only load needed columns
@posts = Post
  .select("posts.id, posts.title, posts.comments_count, posts.created_at")
  .includes(:author, :category)
  .limit(20)

# Complete solution:
@posts = Post
  .includes(:author, :category)
  .limit(20)
  .order(created_at: :desc)

# In the view:
# post.comments.size uses counter_cache (0 SQL)
# post.author.name uses included association (0 SQL)
# post.category.name uses included association (0 SQL)
# Total: 3 SQL queries total (posts + authors + categories)
```

---

## Answer 2: Complex Association Design

**Question:** Design the ActiveRecord models for a hospital where Doctors have many Patients through Appointments. Each Appointment has a status (scheduled, completed, cancelled) and notes. Patients have many medical Records.

```ruby
# Migrations:
create_table :doctors do |t|
  t.string :name, null: false
  t.string :specialty, null: false
  t.string :license_number, null: false
  t.timestamps
end

create_table :patients do |t|
  t.string :name, null: false
  t.date :date_of_birth, null: false
  t.string :blood_type
  t.timestamps
end

create_table :appointments do |t|
  t.references :doctor, null: false, foreign_key: true
  t.references :patient, null: false, foreign_key: true
  t.datetime :scheduled_at, null: false
  t.integer :status, default: 0, null: false
  t.text :notes
  t.timestamps
end

create_table :medical_records do |t|
  t.references :patient, null: false, foreign_key: true
  t.references :doctor, null: false, foreign_key: true  # who created the record
  t.string :record_type, null: false  # e.g., "lab_result", "prescription", "diagnosis"
  t.jsonb :data, null: false, default: {}
  t.date :recorded_on, null: false
  t.timestamps
end

add_index :appointments, [:doctor_id, :scheduled_at]
add_index :appointments, [:patient_id, :scheduled_at]
add_index :medical_records, [:patient_id, :recorded_on]

# Models:
class Doctor < ApplicationRecord
  has_many :appointments, dependent: :restrict_with_exception
  has_many :patients, through: :appointments, source: :patient
  has_many :medical_records_created, class_name: "MedicalRecord", foreign_key: :doctor_id

  scope :by_specialty, ->(specialty) { where(specialty: specialty) }

  validates :name, :specialty, :license_number, presence: true
  validates :license_number, uniqueness: true
end

class Patient < ApplicationRecord
  has_many :appointments, dependent: :destroy
  has_many :doctors, through: :appointments, source: :doctor
  has_many :medical_records, dependent: :destroy

  scope :with_upcoming_appointments, -> {
    joins(:appointments)
      .where(appointments: { scheduled_at: Time.current.., status: :scheduled })
      .distinct
  }

  def age
    ((Date.today - date_of_birth).to_i / 365.25).floor
  end
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient

  enum :status, { scheduled: 0, completed: 1, cancelled: 2, no_show: 3 }

  scope :upcoming, -> { where(scheduled_at: Time.current..).scheduled.order(:scheduled_at) }
  scope :for_date, ->(date) { where(scheduled_at: date.all_day) }

  validates :scheduled_at, presence: true
  validate :no_scheduling_conflicts, on: :create

  after_update :notify_patient_on_cancellation, if: :saved_change_to_status?

  private

  def no_scheduling_conflicts
    conflict = Appointment
      .where(doctor: doctor, scheduled_at: scheduled_at..(scheduled_at + 30.minutes))
      .where.not(id: id)
      .scheduled
      .exists?
    errors.add(:scheduled_at, "conflicts with another appointment") if conflict
  end

  def notify_patient_on_cancellation
    AppointmentMailer.cancellation(self).deliver_later if cancelled?
  end
end

class MedicalRecord < ApplicationRecord
  belongs_to :patient
  belongs_to :doctor

  validates :record_type, :recorded_on, :data, presence: true

  scope :prescriptions, -> { where(record_type: "prescription") }
  scope :lab_results,   -> { where(record_type: "lab_result") }
  scope :recent,        -> { order(recorded_on: :desc) }
end
```

---

## Answer 3: Migrations Best Practices

**Question:** Show migrations for adding a `products` table with proper constraints, an associated `inventory` table, and a partial unique index.

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string  :name, null: false
      t.string  :sku, null: false
      t.text    :description
      t.decimal :price, precision: 10, scale: 2, null: false
      t.integer :status, default: 0, null: false
      t.string  :category, null: false
      t.jsonb   :metadata, default: {}, null: false
      t.boolean :featured, default: false, null: false

      t.timestamps
    end

    # Unique index on SKU
    add_index :products, :sku, unique: true
    # Index for common filter
    add_index :products, [:status, :category]
    # Partial index: unique name only among active products
    add_index :products, :name, unique: true,
              where: "status = 0",  # 0 = active
              name: "index_products_on_name_active_unique"
    # GIN index on JSONB column for searching metadata
    add_index :products, :metadata, using: :gin
  end
end

class CreateInventories < ActiveRecord::Migration[7.1]
  def change
    create_table :inventories do |t|
      t.references :product, null: false, foreign_key: true, index: { unique: true }
      # index: { unique: true } enforces one inventory record per product (has_one)

      t.integer :quantity, default: 0, null: false
      t.integer :reserved_quantity, default: 0, null: false
      t.integer :reorder_point, default: 10, null: false
      t.string  :warehouse_location

      t.timestamps
    end

    # Check constraint: reserved can't exceed total
    execute <<~SQL
      ALTER TABLE inventories
      ADD CONSTRAINT reserved_le_quantity
      CHECK (reserved_quantity <= quantity);
    SQL
  end
end

# Models:
class Product < ApplicationRecord
  has_one :inventory, dependent: :destroy
  after_create :create_inventory!  # Always create inventory on product creation

  enum :status, { active: 0, discontinued: 1, draft: 2 }

  validates :name, :sku, :price, :category, presence: true
  validates :price, numericality: { greater_than: 0 }
  validates :sku, uniqueness: true, format: { with: /\A[A-Z0-9\-]+\z/, message: "must be uppercase alphanumeric" }
end

class Inventory < ApplicationRecord
  belongs_to :product

  validates :quantity, :reserved_quantity, numericality: { greater_than_or_equal_to: 0 }
  validate :reserved_does_not_exceed_quantity

  def available_quantity
    quantity - reserved_quantity
  end

  def reserve!(amount)
    transaction do
      lock!  # pessimistic lock for concurrent reservation
      raise InsufficientStockError if available_quantity < amount
      update!(reserved_quantity: reserved_quantity + amount)
    end
  end

  private

  def reserved_does_not_exceed_quantity
    if reserved_quantity > quantity
      errors.add(:reserved_quantity, "cannot exceed total quantity")
    end
  end
end
```

---

## Answer 4: Polymorphic Associations Done Right

**Question:** Implement a `Comment` model that can belong to `Post`, `Photo`, or `Video`. Include proper indexes and the query to get all comments for a specific commentable type.

```ruby
# Migration:
class CreateComments < ActiveRecord::Migration[7.1]
  def change
    create_table :comments do |t|
      t.references :commentable, polymorphic: true, null: false
      # Generates: commentable_type string, commentable_id bigint
      # AND: index on [commentable_type, commentable_id]

      t.references :user, null: false, foreign_key: true
      t.text :body, null: false
      t.integer :status, default: 0
      t.timestamps
    end

    add_index :comments, [:commentable_type, :commentable_id, :created_at],
              name: "index_comments_polymorphic_created"
  end
end

# Models:
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  belongs_to :user

  enum :status, { pending: 0, approved: 1, spam: 2 }

  validates :body, presence: true, length: { maximum: 5000 }

  scope :approved, -> { where(status: :approved) }
  scope :for_type, ->(type) { where(commentable_type: type.to_s.camelize) }
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
  # Optional: delegate count for display
  delegate :count, to: :comments, prefix: true  # post.comments_count via query
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

class Video < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# Query examples:
# Get all post comments:
Comment.where(commentable_type: "Post").approved
# Better with type safety:
Comment.for_type("Post").approved.includes(:user, :commentable)

# Get comments for a specific record:
post.comments.approved.includes(:user).order(created_at: :desc)

# Load posts with their approved comments (avoid N+1):
Post.includes(:comments).where(comments: { status: 1 })
# Note: this includes all comments filtered by approved, not just the first N

# Eager load polymorphic: Rails runs one query per type found in results
Comment.includes(:commentable).where(status: :approved).limit(50)
# Rails: SELECT * FROM comments WHERE status=1 LIMIT 50
#        SELECT * FROM posts WHERE id IN (...)  [for Post type]
#        SELECT * FROM videos WHERE id IN (...) [for Video type]
```

---

## Answer 5: Transactions and Data Integrity

**Question:** Implement a money transfer between two accounts that handles concurrent requests safely.

```ruby
# Migration:
class CreateAccounts < ActiveRecord::Migration[7.1]
  def change
    create_table :accounts do |t|
      t.references :user, null: false, foreign_key: true
      t.string  :currency, null: false, default: "USD"
      t.decimal :balance, precision: 15, scale: 2, null: false, default: 0
      t.integer :lock_version, default: 0  # for optimistic locking fallback
      t.timestamps
    end

    execute "ALTER TABLE accounts ADD CONSTRAINT balance_non_negative CHECK (balance >= 0)"
  end
end

class CreateTransfers < ActiveRecord::Migration[7.1]
  def change
    create_table :transfers do |t|
      t.references :from_account, null: false, foreign_key: { to_table: :accounts }
      t.references :to_account, null: false, foreign_key: { to_table: :accounts }
      t.decimal :amount, precision: 15, scale: 2, null: false
      t.string  :status, default: "pending", null: false
      t.string  :reference, null: false
      t.timestamps
    end

    add_index :transfers, :reference, unique: true
  end
end

# Service:
class MoneyTransferService
  class InsufficientFunds < StandardError; end
  class SameAccountError < StandardError; end

  def initialize(from_account_id:, to_account_id:, amount:)
    @from_account_id = from_account_id
    @to_account_id   = to_account_id
    @amount          = amount.to_d
  end

  def call
    validate_inputs!

    transfer = nil

    Account.transaction do
      # Lock both accounts — order by ID to prevent deadlocks
      # ALWAYS acquire locks in consistent order!
      accounts = Account.lock
                        .where(id: [@from_account_id, @to_account_id])
                        .order(:id)
                        .to_a

      from_account = accounts.find { |a| a.id == @from_account_id }
      to_account   = accounts.find { |a| a.id == @to_account_id }

      raise InsufficientFunds if from_account.balance < @amount

      from_account.update!(balance: from_account.balance - @amount)
      to_account.update!(balance: to_account.balance + @amount)

      transfer = Transfer.create!(
        from_account: from_account,
        to_account: to_account,
        amount: @amount,
        status: "completed",
        reference: generate_reference
      )
    end

    transfer
  rescue ActiveRecord::RecordNotFound
    raise ArgumentError, "Account not found"
  end

  private

  def validate_inputs!
    raise ArgumentError, "Amount must be positive" unless @amount.positive?
    raise SameAccountError if @from_account_id == @to_account_id
    raise ArgumentError, "Amount too large" if @amount > 1_000_000
  end

  def generate_reference
    "TXN-#{Time.current.strftime('%Y%m%d')}-#{SecureRandom.hex(8).upcase}"
  end
end

# Controller:
class TransfersController < ApplicationController
  def create
    service = MoneyTransferService.new(
      from_account_id: current_user.account.id,
      to_account_id:   params[:to_account_id],
      amount:          params[:amount]
    )
    transfer = service.call
    render json: { reference: transfer.reference, status: "completed" }
  rescue MoneyTransferService::InsufficientFunds
    render json: { error: "Insufficient funds" }, status: :unprocessable_entity
  rescue ArgumentError => e
    render json: { error: e.message }, status: :bad_request
  end
end
```

---

## Answer 6: Enum with State Machine Behaviour

**Question:** Implement an Order model with status transitions that are validated (e.g., can't go from shipped back to pending).

```ruby
class Order < ApplicationRecord
  VALID_TRANSITIONS = {
    "pending"    => %w[processing cancelled],
    "processing" => %w[shipped cancelled],
    "shipped"    => %w[delivered],
    "delivered"  => [],
    "cancelled"  => []
  }.freeze

  enum :status, {
    pending:    "pending",
    processing: "processing",
    shipped:    "shipped",
    delivered:  "delivered",
    cancelled:  "cancelled"
  }

  validate :status_transition_valid, if: :status_changed?

  # Transition helpers:
  def transition_to!(new_status)
    new_status = new_status.to_s
    unless VALID_TRANSITIONS[status]&.include?(new_status)
      raise InvalidTransition, "Cannot transition from #{status} to #{new_status}"
    end
    update!(status: new_status)
  end

  def can_transition_to?(new_status)
    VALID_TRANSITIONS[status]&.include?(new_status.to_s) || false
  end

  private

  def status_transition_valid
    return unless status_was  # skip for new records
    old_status = status_was.to_s
    new_status = status.to_s
    unless VALID_TRANSITIONS[old_status]&.include?(new_status)
      errors.add(:status, "cannot transition from #{old_status} to #{new_status}")
    end
  end

  class InvalidTransition < StandardError; end
end

# Usage:
order = Order.create!(status: :pending)
order.transition_to!(:processing)   # OK
order.transition_to!(:pending)      # raises InvalidTransition
order.update!(status: :pending)     # raises validation error

# Callbacks per state:
after_update :send_shipping_notification, if: -> { saved_change_to_status? && shipped? }
after_update :process_refund,             if: -> { saved_change_to_status? && cancelled? }
```

---

## Answer 7: Scopes, Joins, and Aggregation

**Question:** Write a query to find the top 10 authors by number of published posts in the last 30 days, including their post count.

```ruby
class Author < ApplicationRecord
  has_many :posts

  def self.top_by_published_posts(limit: 10, since: 30.days.ago)
    joins(:posts)
      .where(posts: { published: true, created_at: since.. })
      .select("authors.*, COUNT(posts.id) AS posts_count")
      .group("authors.id")
      .order("posts_count DESC")
      .limit(limit)
  end
end

# Usage:
top_authors = Author.top_by_published_posts(limit: 10, since: 30.days.ago)
top_authors.each do |author|
  puts "#{author.name}: #{author.posts_count} posts"
  # posts_count is a virtual attribute from the SELECT
end

# Including post details (avoid N+1 after the aggregate query):
top_author_ids = top_authors.map(&:id)
authors_with_posts = Author
  .includes(posts: :category)
  .where(id: top_author_ids)
  .map { |a| [a.id, a] }
  .to_h

# Alternative with window functions (PostgreSQL):
Author.from(
  Post
    .select("user_id AS author_id, COUNT(*) AS posts_count")
    .where(published: true, created_at: 30.days.ago..)
    .group(:user_id)
    .order("posts_count DESC")
    .limit(10),
  :top_posts
).joins("JOIN authors ON authors.id = top_posts.author_id")
 .select("authors.*, top_posts.posts_count")
```

---

## Answer 8: ActiveRecord Callbacks — Correct Patterns

**Question:** What are the best practices for using callbacks? Illustrate with before_save, after_commit, and around_destroy.

```ruby
class Order < ApplicationRecord
  # GOOD: before_save for data normalization (pure Ruby, no external calls)
  before_save :normalize_address
  before_save :calculate_total  # computed from line_items

  # GOOD: after_commit for external side effects
  after_commit :enqueue_fulfillment_job, on: :create
  after_commit :notify_status_change, on: :update, if: :saved_change_to_status?

  # GOOD: around_destroy for cleanup with proper error handling
  around_destroy :cleanup_external_resources

  private

  def normalize_address
    self.shipping_city = shipping_city&.strip&.titleize
    self.shipping_zip  = shipping_zip&.gsub(/[^0-9]/, "")
  end

  def calculate_total
    self.subtotal  = line_items.sum { |item| item.quantity * item.unit_price }
    self.tax       = subtotal * TaxCalculator.rate_for(shipping_state)
    self.total     = subtotal + tax + shipping_cost
  end
  # NOTE: this recalculates every save — if performance is an issue,
  # only recalculate if line items changed:
  # def calculate_total
  #   return unless line_items_changed?  # need to track this separately
  # end

  def enqueue_fulfillment_job
    OrderFulfillmentJob.perform_later(id)
    # after_commit: safe because order EXISTS in DB when job runs
  end

  def notify_status_change
    OrderStatusMailer.status_changed(id, status_before_last_save, status).deliver_later
  end

  def cleanup_external_resources
    # around_destroy: code before yield runs before destroy, after yield runs after
    file_keys = documents.pluck(:s3_key)  # collect before destroy

    yield  # performs the destruction

    # After successful destroy, clean up external resources
    S3CleanupJob.perform_later(file_keys)
  rescue ActiveRecord::Rollback
    # yield raised Rollback — resources NOT deleted, don't clean up
    raise
  end
end
```

---

## Answer 9: Performance with `find_each` and Batch Processing

**Question:** Write a script to recalculate the `reputation_score` for all 500,000 users and update them efficiently.

```ruby
# Naive (crashes or very slow):
# User.all.each { |u| u.update!(reputation_score: calculate_score(u)) }

# Better: find_each with individual updates
User.find_each(batch_size: 500) do |user|
  score = ReputationCalculator.new(user).calculate
  user.update_column(:reputation_score, score)
  # update_column: skips validations/callbacks — fine for maintenance scripts
end

# Even better: calculate in SQL if possible
# (avoids loading Ruby objects at all)
User.find_in_batches(batch_size: 500) do |batch|
  ids = batch.map(&:id)

  # If the calculation can be done in SQL:
  User.where(id: ids).each do |user|
    User.where(id: user.id).update_all(
      reputation_score: user.posts.published.count * 10 +
                        user.comments.count * 2
    )
  end
end

# Best approach: SQL UPDATE with subquery (single operation)
User.connection.execute(<<~SQL)
  UPDATE users
  SET reputation_score = (
    SELECT COUNT(*) * 10
    FROM posts
    WHERE posts.user_id = users.id AND posts.published = true
  ) + (
    SELECT COUNT(*) * 2
    FROM comments
    WHERE comments.user_id = users.id
  )
SQL
# Warning: bypasses all callbacks and validations
# Fine for maintenance; add proper indexes on posts(user_id) and comments(user_id)

# With progress tracking:
total = User.count
processed = 0

User.find_each(batch_size: 1000) do |user|
  score = ReputationCalculator.new(user).calculate
  user.update_column(:reputation_score, score)
  processed += 1
  puts "#{processed}/#{total}" if (processed % 10_000).zero?
end
```

---

## Answer 10: STI vs Polymorphic — Design Decision

**Question:** You need to model Notifications that can be SMS, Email, or Push notifications. Each type has different fields. Compare STI vs Polymorphic association for this use case.

```ruby
# Option A: STI
class Notification < ApplicationRecord
  # type column determines subclass
  # All fields in one table — NULLable for inapplicable fields:
  # sms_to, push_token, email_to, email_subject, body, sent_at, user_id
end

class SmsNotification < Notification
  validates :sms_to, presence: true, format: { with: /\A\+1\d{10}\z/ }

  def send!
    SmsGateway.send(to: sms_to, message: body)
    update!(sent_at: Time.current)
  end
end

class EmailNotification < Notification
  validates :email_to, :email_subject, presence: true

  def send!
    NotificationMailer.custom(to: email_to, subject: email_subject, body: body).deliver_later
    update!(sent_at: Time.current)
  end
end

# Use STI when:
# - Shared behaviour and shared columns dominate
# - Querying all notification types together is common
# - Type-specific fields are few

# Option B: Polymorphic (separate tables per type)
class Notification < ApplicationRecord
  belongs_to :notifiable, polymorphic: true
  belongs_to :user
  enum :channel, { sms: "sms", email: "email", push: "push" }
end

class SmsDelivery < ApplicationRecord
  has_one :notification, as: :notifiable
  validates :phone_number, presence: true
end

class EmailDelivery < ApplicationRecord
  has_one :notification, as: :notifiable
  validates :to_address, :subject, presence: true
end

# Use polymorphic when:
# - Type-specific fields are numerous and distinct
# - Each type has very different validation and behaviour
# - Table per type avoids NULL column sprawl

# Recommendation for notifications: STI is usually better
# Shared: user_id, body, sent_at, created_at, channel
# Type-specific: 1-2 extra fields per type → nullable columns are acceptable
# Benefit: one query for "all pending notifications" across types
```

