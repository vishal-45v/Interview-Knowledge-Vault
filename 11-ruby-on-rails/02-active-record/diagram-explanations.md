# Chapter 02 — Active Record: Diagram Explanations

---

## Diagram 1: N+1 Query Problem vs Solutions

```
N+1 PROBLEM:
posts = Post.limit(5)
posts.each { |p| puts p.author.name }

SQL Execution:
  Query 1: SELECT * FROM posts LIMIT 5
           ↓ Returns 5 posts: IDs [1, 2, 3, 4, 5]

  Query 2: SELECT * FROM users WHERE id = 1   ← post 1's author
  Query 3: SELECT * FROM users WHERE id = 2   ← post 2's author
  Query 4: SELECT * FROM users WHERE id = 1   ← post 3's author (duplicate!)
  Query 5: SELECT * FROM users WHERE id = 3   ← post 4's author
  Query 6: SELECT * FROM users WHERE id = 2   ← post 5's author (duplicate!)

  Total: 6 queries (1 + N)
  For 1000 posts: 1001 queries!

─────────────────────────────────────────────────────────────────

SOLUTION A: includes (uses preload strategy by default)
posts = Post.includes(:author).limit(5)

  Query 1: SELECT * FROM posts LIMIT 5
  Query 2: SELECT * FROM users WHERE id IN (1, 2, 3)

  Total: 2 queries regardless of number of posts

─────────────────────────────────────────────────────────────────

SOLUTION B: eager_load (uses LEFT OUTER JOIN)
posts = Post.eager_load(:author).limit(5)

  Query 1: SELECT posts.id, posts.title, ..., users.id, users.name, ...
           FROM posts
           LEFT OUTER JOIN users ON users.id = posts.user_id
           LIMIT 5

  Total: 1 query (with larger result set due to JOIN)

─────────────────────────────────────────────────────────────────

WHEN TO USE WHICH:
  ┌────────────────┬──────────────────────────────────────────┐
  │ includes       │ Default choice — Rails decides strategy  │
  ├────────────────┼──────────────────────────────────────────┤
  │ preload        │ Force separate queries; avoid JOIN bloat  │
  │                │ when each assoc has many records          │
  ├────────────────┼──────────────────────────────────────────┤
  │ eager_load     │ When filtering/sorting by assoc column    │
  │                │ Post.eager_load(:author)                  │
  │                │      .where(users: { active: true })      │
  └────────────────┴──────────────────────────────────────────┘
```

---

## Diagram 2: ActiveRecord Callback Chain

```
SAVE CHAIN (new record — INSERT):

  record.save
       │
       ▼
  ┌────────────────────────────────────┐
  │ 1. before_validation               │
  │    (normalize data, set defaults)  │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 2. VALIDATIONS RUN                 │
  │    validates :name, presence: true │
  │    validates :email, uniqueness:.. │
  │    custom validate methods         │
  └────────────────┬───────────────────┘
                   │ ← if invalid: return false, add errors
                   ▼
  ┌────────────────────────────────────┐
  │ 3. after_validation                │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 4. before_save                     │
  │    (last chance to modify values)  │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 5. before_create                   │
  │    (only for new records)          │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 6. SQL INSERT runs                 │
  │    INSERT INTO users (name, email) │
  │    VALUES (?, ?)                   │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 7. after_create                    │
  │    (record now has an ID)          │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 8. after_save                      │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 9. Transaction commits             │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 10. after_commit (on: :create)     │  ← SAFE for external calls
  │     ↑ ONLY fires if DB committed  │     emails, jobs, cache invalidation
  └────────────────────────────────────┘

ROLLBACK PATH:
  If transaction rolled back → after_rollback fires instead of after_commit

─────────────────────────────────────────────────────────────────

THROW(:abort) HALTS THE CHAIN:
  before_save callback calls throw(:abort)
       │
       ▼
  Chain stops → record NOT saved → save returns false
```

---

## Diagram 3: Association Loading Strategies

```
Schema:
  users ──── posts ──── comments
     id         id          id
     name       title       body
                user_id     post_id

STRATEGY 1: PRELOAD (separate IN queries)
User.preload(:posts)

  SQL 1: SELECT * FROM users
         → users: [id:1 Alice, id:2 Bob]

  SQL 2: SELECT * FROM posts WHERE user_id IN (1, 2)
         → posts loaded and grouped by user_id in Ruby

  Memory: users with posts array populated
  Use when: has_many with many records per parent

─────────────────────────────────────────────────────────────────

STRATEGY 2: EAGER_LOAD (LEFT OUTER JOIN)
User.eager_load(:posts)

  SQL 1: SELECT users.*, posts.* FROM users
         LEFT OUTER JOIN posts ON posts.user_id = users.id

         Results: (note duplication)
         ┌────────┬───────┬───────────────────┐
         │user.id │user.nm│post.title          │
         ├────────┼───────┼───────────────────┤
         │1       │Alice  │"Ruby Tips"         │
         │1       │Alice  │"Rails Guide"       │
         │2       │Bob    │"JavaScript Basics" │
         └────────┴───────┴───────────────────┘
         ActiveRecord deduplicates users in Ruby

  Memory: users with posts array populated (same result as preload)
  Use when: filtering by posts columns in WHERE clause

─────────────────────────────────────────────────────────────────

STRATEGY 3: JOINS (for filtering only — no loading)
User.joins(:posts).where(posts: { published: true })

  SQL 1: SELECT users.* FROM users
         INNER JOIN posts ON posts.user_id = users.id
         WHERE posts.published = true

  Memory: users loaded, posts NOT in memory
  Accessing user.posts WILL trigger N+1!
  Use when: filtering by association, not displaying association data

─────────────────────────────────────────────────────────────────

COMBINED (filter + load):
User.joins(:posts)
    .where(posts: { published: true })
    .eager_load(:posts)

  Or simpler:
  User.eager_load(:posts).where(posts: { published: true })
```

---

## Diagram 4: Schema.rb vs Structure.sql

```
schema.rb                          structure.sql
(database-agnostic Ruby DSL)       (raw SQL DDL from pg_dump)
─────────────────────────────      ─────────────────────────────────────

ActiveRecord::Schema[7.1].define   --
  do                               -- PostgreSQL database dump
  create_table "users" do |t|      --
    t.string "email"               CREATE TABLE public.users (
    t.string "name"                  id bigint NOT NULL,
    t.datetime "created_at"          email character varying NOT NULL,
    t.datetime "updated_at"          name character varying,
  end                                created_at timestamp(6) NOT NULL,
                                     updated_at timestamp(6) NOT NULL
  add_index "users", ["email"],    );
    unique: true
end                                CREATE UNIQUE INDEX index_users_on_email
                                     ON public.users USING btree (email);

                                   -- PostgreSQL-specific features:
                                   CREATE TYPE mood AS ENUM
                                     ('sad', 'ok', 'happy');

                                   CREATE INDEX CONCURRENTLY ...

                                   CREATE FUNCTION ...

                                   ALTER TABLE users
                                     ADD CONSTRAINT check_email_format
                                     CHECK (email ~* '...');

CHOOSE schema.rb when:             CHOOSE structure.sql when:
  - SQLite or MySQL                  - PostgreSQL with custom types
  - Simple schemas                   - Stored procedures / functions
  - Team uses multiple DB adapters   - Check constraints
  - Faster to read and review        - Partial indexes
                                     - Triggers
                                     - Exact schema fidelity required

Set via:
  config.active_record.schema_format = :sql  # for structure.sql
  config.active_record.schema_format = :ruby # for schema.rb (default)
```

---

## Diagram 5: Optimistic vs Pessimistic Locking Flow

```
OPTIMISTIC LOCKING (lock_version column):

  Thread A                          Thread B
  ────────                          ────────
  account = Account.find(1)         account = Account.find(1)
  # lock_version: 5                 # lock_version: 5
  # balance: 1000                   # balance: 1000
         │                                  │
         │ (both work independently)        │
         │                                  │
  account.balance = 900             account.balance = 800
  account.save!                     account.save!
         │                                  │
         ▼                                  │
  UPDATE accounts                           │
  SET balance=900,                          │
      lock_version=6                        │
  WHERE id=1                                │
  AND lock_version=5  ← SUCCEEDS            │
         │                                  ▼
         │                         UPDATE accounts
         │                         SET balance=800,
         │                             lock_version=6
         │                         WHERE id=1
         │                         AND lock_version=5  ← 0 rows!
         │                                  │
         │                                  ▼
         │                         Rails raises StaleObjectError
         │                         Thread B must reload and retry
         │
  balance in DB: 900, lock_version: 6

─────────────────────────────────────────────────────────────────

PESSIMISTIC LOCKING (SELECT ... FOR UPDATE):

  Thread A                          Thread B
  ────────                          ────────
  Account.transaction do            Account.transaction do
    account = Account              (waits for Thread A's
      .lock.find(1)                 lock to be released)
    # DB LOCK acquired ───┐
    #                      │
    # Thread B blocked ◄───┘
    #
    account.balance -= 100
    account.save!
  end  # LOCK RELEASED
                                   # Thread B unblocked
                                   account = Account.lock.find(1)
                                   # sees updated balance (900)
                                   account.balance -= 100
                                   account.save!
                                   end

  Result: balance correctly = 800 (two deductions applied)
  Risk: deadlock if Thread A and B lock records in different orders
```

---

## Diagram 6: ActiveRecord Query Building (Relation Composition)

```
Every AR query method returns an ActiveRecord::Relation (lazy)
The SQL is NOT built until the relation is "executed"

  User                          ← ActiveRecord::Base (class)
    .where(active: true)        ← ActiveRecord::Relation (WHERE active = true)
    .order(name: :asc)          ← ActiveRecord::Relation (+ ORDER BY name ASC)
    .limit(10)                  ← ActiveRecord::Relation (+ LIMIT 10)
    .includes(:posts)           ← ActiveRecord::Relation (+ loads posts separately)
    .select(:id, :name, :email) ← ActiveRecord::Relation (SELECT id, name, email)

  ← NO SQL yet! Just a query builder object ←

  Execution triggers (materializes the relation):
    .to_a          → executes SQL, returns Array of AR objects
    .each          → executes SQL, iterates
    .map           → executes SQL, maps
    .first/last    → executes SQL with LIMIT 1 (ORDER BY id ASC/DESC)
    .count         → executes SELECT COUNT(*) SQL
    .pluck(:name)  → executes SELECT name SQL, returns Array of values
    .exists?       → executes SELECT 1 ... LIMIT 1

  The final SQL:
  SELECT id, name, email FROM users
  WHERE active = true
  ORDER BY name ASC
  LIMIT 10

  (includes runs as separate query after main query)
```

