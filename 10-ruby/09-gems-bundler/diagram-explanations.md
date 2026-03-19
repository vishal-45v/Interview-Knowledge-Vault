# Chapter 09: Gems & Bundler — Diagram Explanations

## Diagram 1: Bundler Dependency Resolution Flow

```
bundle install — FULL RESOLUTION FLOW

  ┌─────────────────┐
  │    Gemfile       │
  │  gem "rails"     │
  │  gem "sidekiq"   │
  │  gem "pg"        │
  └────────┬────────┘
           │
           ▼
  ┌──────────────────────────────────────────────────────┐
  │          DEPENDENCY GRAPH CONSTRUCTION               │
  │                                                      │
  │  rails 7.1.2 ───→ activesupport = 7.1.2             │
  │                ──→ activerecord = 7.1.2              │
  │                ──→ actionpack = 7.1.2                │
  │                                                      │
  │  sidekiq 7.0.x ─→ connection_pool >= 2.3            │
  │                ──→ redis >= 4.5                      │
  │                                                      │
  │  pg 1.5.x ─────→ (no gem dependencies, C extension) │
  └──────────────────────────┬───────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────┐
  │              MOLINILLO SAT SOLVER                    │
  │                                                      │
  │  Find versions satisfying ALL constraints:           │
  │  activesupport: must be = 7.1.2 (from rails)        │
  │  redis:         must be >= 4.5  (from sidekiq)      │
  │                 → select redis 5.0.8 ✓               │
  │  connection_pool: >= 2.3                             │
  │                 → select connection_pool 2.4.1 ✓    │
  └──────────────────────────┬───────────────────────────┘
                             │
                   CONFLICT? ─── YES → Bundler::VersionConflict error
                             │
                            NO
                             │
                             ▼
  ┌──────────────────────────────────────────────────────┐
  │               WRITE Gemfile.lock                     │
  │                                                      │
  │  GEM                                                 │
  │    remote: https://rubygems.org/                    │
  │    specs:                                           │
  │      rails (7.1.2)                                  │
  │        activesupport (= 7.1.2)                      │
  │      activesupport (7.1.2)                          │
  │      sidekiq (7.0.9)                                │
  │        redis (>= 4.5)                               │
  │      redis (5.0.8)                                  │
  │      pg (1.5.4)                                     │
  └──────────────────────────────────────────────────────┘
```

---

## Diagram 2: Gem Load Path and `require`

```
How Ruby finds a gem when you call `require 'rails'`

WITHOUT BUNDLER:
  $LOAD_PATH = [
    "/usr/local/lib/ruby/gems/3.2.0/gems/rails-7.0.4/lib",  ← system gem
    "/usr/local/lib/ruby/gems/3.2.0/gems/rails-7.1.2/lib",  ← newer system gem
    "/usr/local/lib/ruby/3.2.0",
    "."
  ]

  require 'rails'  →  searches in order  →  finds first rails.rb
                   →  /usr/local/.../rails-7.0.4/lib/rails.rb
                   →  loads THAT version (might not be what you want!)

WITH BUNDLER (bundle exec or require 'bundler/setup'):
  Bundler reads Gemfile.lock → rails = 7.1.2

  Bundler modifies $LOAD_PATH:
  $LOAD_PATH = [
    "/usr/local/lib/ruby/gems/3.2.0/gems/rails-7.1.2/lib",  ← only locked version
    ← all other rails versions REMOVED from path
    "/usr/local/lib/ruby/3.2.0",
    "."
  ]

  require 'rails'  →  searches path  →  finds rails-7.1.2/lib/rails.rb ✓

GEM LOAD PATH INSPECTION:

  # Which rails is actually loaded?
  require 'rails'
  puts $LOADED_FEATURES.grep(/rails/).first
  # → /usr/local/lib/ruby/gems/3.2.0/gems/rails-7.1.2/lib/rails.rb

  puts Gem.loaded_specs['rails'].version
  # → 7.1.2

  # Full load path:
  $LOAD_PATH.each { |p| puts p }
```

---

## Diagram 3: Gemfile.lock Structure

```
GEMFILE.LOCK ANATOMY

  ┌──────────────────────────────────────────────────────────┐
  │  GEM                                                     │
  │    remote: https://rubygems.org/                        │  ← where gems come from
  │    specs:                                               │
  │      rails (7.1.2)                                      │  ← exact version
  │        actionpack (= 7.1.2)                             │  ← transitive dep
  │        actionview (= 7.1.2)                             │
  │        activejob (= 7.1.2)                              │
  │        activemodel (= 7.1.2)                            │
  │        activerecord (= 7.1.2)                           │
  │        activestorage (= 7.1.2)                          │
  │        activesupport (= 7.1.2)                          │
  │                                                         │
  │      activerecord (7.1.2)                               │
  │        activemodel (= 7.1.2)                            │
  │        activesupport (= 7.1.2)                          │
  │                                                         │
  │  GIT                                                    │  ← git gems
  │    remote: https://github.com/user/my_gem.git          │
  │    revision: deadbeef1234...                            │  ← pinned commit
  │    branch: main                                         │
  │    specs:                                               │
  │      my_gem (1.0.0)                                     │
  │                                                         │
  │  PATH                                                   │  ← local path gems
  │    remote: ../shared_lib                               │
  │    specs:                                               │
  │      shared_lib (2.1.0)                                 │
  │                                                         │
  │  PLATFORMS                                              │  ← OS/arch targets
  │    arm64-darwin-22                                      │
  │    x86_64-linux                                         │
  │                                                         │
  │  DEPENDENCIES                                           │  ← top-level Gemfile deps
  │    rails (~> 7.1)                                       │
  │    pg (>= 1.0)                                          │
  │    sidekiq (~> 7.0)                                     │
  │                                                         │
  │  BUNDLED WITH                                           │  ← bundler version
  │    2.4.10                                               │
  └──────────────────────────────────────────────────────────┘

  LOCK FILE CONTRACT:
  ┌────────────────────────────────────────────────────┐
  │  Gemfile says:   rails "~> 7.1"  (constraint)      │
  │  Lock says:      rails 7.1.2     (exact resolution) │
  │                                                     │
  │  Any machine running `bundle install`:              │
  │  → Gets rails 7.1.2 (not 7.1.1, not 7.1.3)        │
  │  → Reproducible across dev/CI/staging/prod         │
  └────────────────────────────────────────────────────┘
```

---

## Diagram 4: `bundle exec` vs Direct Execution

```
WITHOUT bundle exec:
  ┌──────────────────────────────────────────────────────┐
  │  Terminal: $ rspec spec/                             │
  │                                                     │
  │  OS looks up 'rspec' in PATH:                       │
  │  → /usr/local/bin/rspec (symlink to system gem)     │
  │  → Loaded from: ~/.rbenv/.../gems/rspec-3.12.0/bin/ │
  │                                                     │
  │  $LOAD_PATH includes: ALL system gem versions       │
  │  → might load wrong gem versions!                   │
  └──────────────────────────────────────────────────────┘

WITH bundle exec:
  ┌──────────────────────────────────────────────────────┐
  │  Terminal: $ bundle exec rspec spec/                 │
  │                                                     │
  │  bundle exec:                                       │
  │  1. Reads Gemfile.lock                              │
  │  2. Sets BUNDLE_GEMFILE environment var             │
  │  3. Modifies $LOAD_PATH to locked versions only     │
  │  4. Finds rspec from locked gems                    │
  │  → /path/to/gems/rspec-3.9.2/bin/rspec             │
  │                                                     │
  │  $LOAD_PATH includes: ONLY locked gem versions      │
  │  → Consistent, correct behavior!                    │
  └──────────────────────────────────────────────────────┘

BINSTUB SHORTCUT:
  bundle binstubs rspec-core
  Creates: bin/rspec

  bin/rspec content:
  #!/usr/bin/env ruby
  require 'rubygems'
  require 'bundler/setup'   ← sets up load path from Gemfile.lock
  load Gem.bin_path('rspec-core', 'rspec')

  Usage: bin/rspec spec/
  → Equivalent to: bundle exec rspec spec/
  → No "bundle exec" typing required
```

---

## Diagram 5: Semantic Versioning Constraint Ranges

```
VERSION NUMBER: MAJOR.MINOR.PATCH
                  3   . 2   . 1

CONSTRAINT INTERPRETATION:

  Constraint      Meaning             Allows                Blocks
  ───────────────────────────────────────────────────────────────────
  = 3.2.1        exact only          3.2.1                 3.2.2, 3.3.0
  >= 3.2         at least            3.2.0, 4.0.0, 5.x    3.1.x
  > 3.2          strictly greater    3.2.1, 4.0.0          3.2.0, 3.1.x
  < 4.0          less than           3.9.9, 3.2.1          4.0.0
  ~> 3.2         pessimistic 2-part  3.2.0...3.9.x         4.0.0
  ~> 3.2.1       pessimistic 3-part  3.2.1...3.2.x         3.3.0
  ~> 3           pessimistic 1-part  3.0.0...3.x.x         4.0.0

VISUAL RANGE FOR ~>:

  ~> 3.2 allowed range:
  ──────────────────────────────────────────────────────────────────
  ... 3.1  [3.2.0 ─────────────────────────── 3.9.x]  4.0.0 ...
                 ↑ allowed                          ↑ blocked here

  ~> 3.2.1 allowed range:
  ──────────────────────────────────────────────────────────────────
  3.2.0  [3.2.1 ──── 3.2.9...]  3.3.0 ...
               ↑ allowed    ↑ blocked here

COMMON PATTERNS:

  gem "rails",     "~> 7.1"     # allow 7.1.x, 7.2, 7.3 but not 8.x
  gem "pg",        "~> 1.5"     # allow 1.5.x, 1.6 but not 2.x
  gem "devise",    "~> 4.9"     # allow 4.9.x but not 5.x (breaking)
  gem "redis",     "~> 5.0"     # allow 5.0.x but not 6.x
  gem "puma",      ">= 5.0", "< 7"  # explicit range when ~> isn't precise enough
```
