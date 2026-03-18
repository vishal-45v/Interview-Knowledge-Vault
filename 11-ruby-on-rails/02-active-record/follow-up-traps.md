# Chapter 02 — Active Record: Follow-Up Traps

---

**Trap 1: "You said includes fixes N+1. Does includes always work? Can it cause worse performance?"**

`includes` can cause performance problems in specific cases:

```ruby
# CASE 1: Including a large association when you only need a count
Post.includes(:comments).map { |p| p.comments.size }
# Loads ALL comment records into memory just to count them
# Better: counter_cache or Post.select("posts.*, COUNT(comments.id) AS comments_count").joins(:comments).group("posts.id")

# CASE 2: includes becomes a cartesian JOIN product
# Joining two has_many associations:
User.includes(:posts, :comments)
# Rails may generate:
# SELECT * FROM users LEFT JOIN posts ... LEFT JOIN comments ...
# If user has 100 posts and 100 comments: 10,000 rows returned from SQL!
# Better: use preload to force separate queries

# CASE 3: includes with conditions forces eager_load (JOIN strategy)
# and that JOIN may be slower than the N+1 was
Post.includes(:comments).where("comments.approved = true")
# This forces a LEFT OUTER JOIN — if posts table is huge, this scans everything
```

The real answer: includes/preload/eager_load aren't always better. Profile first, measure second, optimize third.

---

**Trap 2: "You said `after_commit` is safe for side effects. But what if the job enqueued in `after_commit` fails to enqueue (Redis is down)?**

The trap: `after_commit` guarantees the DB transaction committed, but NOT that the job was successfully enqueued.

```ruby
after_commit :enqueue_welcome_email, on: :create

def enqueue_welcome_email
  WelcomeEmailJob.perform_later(id)
  # If Redis is down: raises error
  # The user IS created in the DB
  # But the email is NEVER sent (no retry, no fallback)
end

# Solutions:
# 1. Rescue and log:
def enqueue_welcome_email
  WelcomeEmailJob.perform_later(id)
rescue Redis::CannotConnectError => e
  Rails.logger.error "Failed to enqueue welcome email for user #{id}: #{e.message}"
  # User exists but email not sent — acceptable for non-critical paths
end

# 2. Use solid_queue (DB-backed) instead of Sidekiq:
# If DB is up and the user was created, the job can be enqueued to the same DB
# atomic!

# 3. Use outbox pattern:
after_commit :create_outbox_record, on: :create
# A separate process polls outbox and sends emails — fully transactional
```

---

**Trap 3: "You said `update_attribute` skips validations. Does it also skip `before_save` callbacks?"**

No — `update_attribute` skips **validations** but still runs **callbacks** (including `before_save`, `after_save`, `after_commit`). This surprises many candidates.

```ruby
class User < ApplicationRecord
  before_save :expensive_recalculation
  validates :email, presence: true
end

user.update_attribute(:name, "Alice")
# 1. Skips validates :email validation ✓
# 2. STILL runs before_save :expensive_recalculation ✗ (if you wanted to skip this)

# To skip BOTH validations AND callbacks:
user.update_column(:name, "Alice")   # skips all validations and callbacks
user.update_columns(name: "Alice")   # same, supports multiple columns

# Summary:
# update:           validations ✓ | callbacks ✓ | returns t/f
# update!:          validations ✓ | callbacks ✓ | raises
# update_attribute: validations ✗ | callbacks ✓ | saves immediately
# update_column:    validations ✗ | callbacks ✗ | direct SQL
# update_columns:   validations ✗ | callbacks ✗ | direct SQL (multiple)
```

---

**Trap 4: "You mentioned `counter_cache`. What happens to the counter when you do `Post.delete_all`?"**

Nothing happens — the counter is NOT updated. `delete_all` runs a raw SQL DELETE and bypasses all ActiveRecord callbacks, including the `counter_cache` decrement callbacks.

```ruby
Post.where(user_id: 1).delete_all
# SQL: DELETE FROM posts WHERE user_id = 1
# User.posts_count is NOT decremented — now stale/wrong!

# Repair stale counters:
User.find_each do |user|
  User.reset_counters(user.id, :posts)
end

# Or use a Rake task:
namespace :maintenance do
  task fix_counter_caches: :environment do
    User.find_each do |user|
      User.reset_counters(user.id, :posts, :comments)
    end
  end
end
```

---

**Trap 5: "What's the difference between `size`, `count`, and `length` on an ActiveRecord relation?"**

```ruby
posts = Post.where(published: true)

posts.size
# Smart: if association is loaded in memory → length (no SQL)
#        if not loaded → COUNT(*) SQL query
# Also: if counter_cache exists → reads from counter column

posts.count
# ALWAYS runs COUNT(*) SQL query regardless of loading state
# Use when you want a fresh DB count

posts.length
# ALWAYS loads all records into memory first, then returns array length
# Worst for performance — avoid on large collections

# Gotcha:
user = User.find(1)
user.posts.size    # Uses counter_cache if available, otherwise COUNT SQL
user.posts.load.size  # After .load, all posts in memory → no SQL
user.posts.count   # ALWAYS SQL COUNT even after load
```

---

**Trap 6: "You mentioned optimistic locking. Does it work across multiple processes vs multiple threads?"**

Yes — optimistic locking works across both threads and processes because the check happens at the database level, not in-memory:

```ruby
# Thread 1 (process A):                Thread 2 (process B):
# record = Record.find(1)              record = Record.find(1)
# record.lock_version # => 5          record.lock_version # => 5
# record.update!(name: "A")           record.update!(name: "B")
# # SQL: UPDATE records SET name='A', lock_version=6
# #       WHERE id=1 AND lock_version=5
# # → SUCCEEDS (lock_version was 5, now 6)
#                                      # SQL: UPDATE records SET name='B', lock_version=6
#                                      #       WHERE id=1 AND lock_version=5
#                                      # → 0 rows updated (lock_version is now 6, not 5)
#                                      # Rails detects 0 rows → raises StaleObjectError

# Key insight: the WHERE id=? AND lock_version=? clause ensures atomicity
# If 0 rows updated → conflict detected
```

---

**Trap 7: "Explain the default_scope gotcha with associations:"**

```ruby
class Post < ApplicationRecord
  default_scope { where(deleted_at: nil) }
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post  # This join uses Post's default_scope!
end

# This query includes Post's default_scope:
Comment.joins(:post).to_sql
# => SELECT comments.* FROM comments
#    INNER JOIN posts ON posts.id = comments.post_id
#    WHERE posts.deleted_at IS NULL   ← from default_scope!

# This is usually CORRECT behaviour, but can cause bugs:
# If you do Post.unscoped.joins(:comments) — the unscoped only applies to Post, not comment's use of Post
# And if Post has default_scope, counting Post.all in an admin panel hides deleted posts

# The real danger: developers don't realize association queries are filtered by default_scope
# A "has_many :posts" on User will NEVER return deleted posts without unscoped
user.posts            # WHERE deleted_at IS NULL AND user_id = ?
user.posts.unscoped   # Removes all scopes — WRONG: removes user_id filter too!
Post.unscoped.where(user_id: user.id)  # Correct way
```

---

**Trap 8: "includes vs preload vs eager_load — when does includes choose JOIN vs separate queries?"**

```ruby
# includes chooses the strategy based on whether you reference the association table in WHERE/ORDER/HAVING

# Strategy: PRELOAD (separate queries) — default when no reference to association
Post.includes(:author)
# SQL 1: SELECT * FROM posts
# SQL 2: SELECT * FROM users WHERE id IN (1, 2, 3, ...)

# Strategy: EAGER_LOAD (JOIN) — forced when referencing association table
Post.includes(:author).where("users.name = ?", "Alice")
# SQL: SELECT posts.*, users.* FROM posts
#      LEFT OUTER JOIN users ON users.id = posts.user_id
#      WHERE (users.name = 'Alice')

# Forced eager_load:
Post.includes(:author).references(:users)  # explicit force
Post.includes(:author).where(users: { active: true })  # implicit force

# Why it matters:
# JOIN strategy → one large result set (cartesian product if multiple has_many)
# Separate queries → multiple small result sets (usually faster for has_many)
```

---

**Trap 9: "What happens to before_destroy callbacks when dependent: :destroy is set on a parent?"**

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
  before_destroy :check_no_featured_comments

  def check_no_featured_comments
    throw(:abort) if comments.featured.any?
  end
end

# The order matters:
# 1. before_destroy on Post runs: checks for featured comments
# 2. If NOT aborted: Rails destroys comments one by one (dependent: :destroy)
#    Each comment.destroy fires Comment's callbacks
# 3. Then Post is destroyed

# GOTCHA: the before_destroy check reads comments BEFORE they're destroyed
# But then Rails ALSO loads those comments for dependent: :destroy
# This causes N+1 queries: once for the check, once for destroying

# Also: if a Comment's before_destroy throws(:abort), the comment isn't destroyed
# But Post destruction continues! Unless Post checks the return value.
# Actually: if comment.destroy returns false, Rails raises ActiveRecord::RecordNotDestroyed
# which propagates up and rolls back the transaction.

# Safer: use dependent: :restrict_with_exception and handle in service object
```

---

**Trap 10: "You have a uniqueness validation on email. Can two users with the same email be created simultaneously?"**

Yes! This is a classic race condition:

```ruby
class User < ApplicationRecord
  validates :email, uniqueness: true
end

# Thread 1:                      Thread 2:
# User.find_by(email:) → nil    User.find_by(email:) → nil
# User.create!(email: "a@b.c") User.create!(email: "a@b.c")
# → SUCCEEDS                    → SUCCEEDS (validation passed for both!)
# Now DB has two users with same email

# RAILS VALIDATES IN RUBY — it's a SELECT then INSERT, not atomic
# Fix: database-level unique index
# add_index :users, :email, unique: true
# The DB will reject the second INSERT with ActiveRecord::RecordNotUnique

# Complete solution:
class User < ApplicationRecord
  validates :email, uniqueness: { case_sensitive: false }  # Ruby-level check
  # Plus migration:
  # add_index :users, "lower(email)", unique: true  # PostgreSQL case-insensitive

  # Handle race condition gracefully:
  def self.find_or_create_with_email(email)
    find_or_create_by(email: email.downcase)
  rescue ActiveRecord::RecordNotUnique
    retry  # Another process created it first, find it now
  end
end
```

---

**Trap 11: "What is the behaviour difference between `ActiveRecord::Rollback` and other exceptions in transactions?"**

```ruby
ActiveRecord::Base.transaction do
  user.save!
  raise ActiveRecord::Rollback  # Transaction rolls back silently
  # ActiveRecord::Rollback is caught by the transaction block
  # It does NOT propagate outside the block
  # Returns nil from the transaction block
end
# Code continues here, no exception raised

ActiveRecord::Base.transaction do
  user.save!
  raise StandardError, "something wrong"  # Transaction rolls back
  # StandardError is NOT caught by the transaction block
  # It propagates outside
end
# StandardError raised here — must rescue externally

# Nested transaction gotcha:
ActiveRecord::Base.transaction do
  ActiveRecord::Base.transaction do  # No requires_new: true
    raise ActiveRecord::Rollback  # This is SILENTLY IGNORED
    # Without requires_new, nested transaction is a no-op savepoint
    # ActiveRecord::Rollback in nested tx just exits the inner block
    # Outer transaction continues as if nothing happened
  end
  user.save!  # This runs and outer tx commits!
end
```

---

**Trap 12: "What does `count` vs `size` return when you scope an association with `where`?"**

```ruby
user = User.find(1)

# These are different!
user.posts.count
# Always SQL: SELECT COUNT(*) FROM posts WHERE user_id = 1

user.posts.size
# If posts not loaded: SQL COUNT
# If counter_cache: reads from users.posts_count (doesn't account for where!)

# The gotcha:
user.posts.published.count  # SQL: SELECT COUNT(*) ... WHERE published = true
user.posts.size              # Uses counter_cache for ALL posts, ignores .published scope!

# Scoped counter_cache (Rails 7+):
class Post < ApplicationRecord
  belongs_to :user, counter_cache: :total_posts_count
  # counter_cache only tracks ALL posts
  # There is NO built-in scoped counter_cache
  # For scoped counts, use a separate counter or a SQL count
end
```

