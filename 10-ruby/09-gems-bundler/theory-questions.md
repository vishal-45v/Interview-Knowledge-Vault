# Chapter 09: Gems & Bundler — Theory Questions

## RubyGems Fundamentals

**Q1. What is RubyGems and what does `gem install` do under the hood?**

RubyGems is Ruby's package manager, included in the standard library since Ruby 1.9. It manages the download, installation, and loading of gem packages.

```ruby
# gem install downloads, unpacks, and registers the gem
# gem install rails --version "~> 7.1"

# What it does:
# 1. Resolves the gem name against a gem source (rubygems.org by default)
# 2. Downloads the .gem file (a TAR archive containing metadata + code)
# 3. Unpacks into GEM_HOME (e.g., ~/.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/)
# 4. Generates .gemspec index entry
# 5. Runs any post-install hooks (extconf.rb for C extensions → compile)

# Finding where gems are installed:
Gem.paths.home   # => "/usr/local/lib/ruby/gems/3.2.0"
Gem.paths.path   # => ["/usr/local/lib/ruby/gems/3.2.0", ...]

# List installed gems:
# gem list                  (all)
# gem list rails            (specific gem)
# gem list --local          (no network lookup)

# Gem directory structure:
# gems/rails-7.1.0/        ← gem code
#   lib/                   ← require paths
#   bin/                   ← executables
#   ext/                   ← C extensions (compiled to .so/.bundle)
# specifications/rails-7.1.0.gemspec  ← metadata
# doc/                     ← rdoc (if generated)
```

---

**Q2. What is `gem list` and `gem server`?**

```ruby
# gem list — show installed gems
# gem list                 — all gems
# gem list ^rails          — gems starting with "rails"
# gem list rails --remote  — check rubygems.org for versions

# Programmatic access:
Gem::Specification.all.map(&:name).sort

# gem server — starts a local gem server for browsing docs
# gem server --port=8808
# Serves RDoc at http://localhost:8808

# Check gem version in code:
Gem.loaded_specs["rails"].version.to_s  # => "7.1.2"
```

---

## Bundler

**Q3. What is Bundler and why is it necessary?**

Bundler ensures that a project's dependencies are the exact same versions across all environments (development, staging, production, CI). Without Bundler, developers might have different gem versions installed system-wide, leading to "works on my machine" bugs.

```ruby
# Without Bundler: system gem version used (may differ per machine)
require 'nokogiri'  # loads whatever version is installed globally

# With Bundler: pinned to Gemfile.lock version
# bundle exec ruby script.rb
# All requires load from the locked gem set

# Gemfile.lock pins exact versions:
# GEM
#   remote: https://rubygems.org/
#   specs:
#     nokogiri (1.15.4)
#       racc (~> 1.4)
#     racc (1.7.3)
#
# Bundler's guarantee: any machine running `bundle install` gets
# these EXACT versions (not ^1.15.4 — exactly 1.15.4)
```

---

**Q4. Explain the Gemfile DSL: `gem`, `source`, `group`, `git`, `path`.**

```ruby
# source: where to find gems
source "https://rubygems.org"
source "https://gems.mycompany.com" do
  gem "private_gem"  # scoped to this source only
end

# gem: declare a dependency
gem "rails", "~> 7.1"           # any 7.1.x
gem "pg", ">= 1.0", "< 2.0"    # range constraint
gem "puma"                       # any version
gem "redis", require: "redis/connection/hiredis"  # custom require path
gem "aws-sdk-s3", require: false  # don't auto-require on bundle exec

# group: load only in specific environments
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end

group :development do
  gem "pry-rails"
  gem "letter_opener"
end

group :production do
  gem "rack-attack"
end

# git: use a gem from a GitHub repo
gem "rails", git: "https://github.com/rails/rails.git", branch: "main"
gem "my_gem",  git: "git@github.com:user/my_gem.git", tag: "v1.2.3"
gem "fix_gem", git: "https://github.com/user/gem.git", ref: "abc123f"  # specific commit

# path: use a gem from a local directory
gem "my_lib", path: "../my_lib"            # relative path
gem "shared", path: "/absolute/path/lib"   # absolute path

# platforms: OS or Ruby engine specific
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw]  # Windows only
```

---

**Q5. What is semantic versioning (`~>`, `>=`, `=`) in Gemfile constraints?**

```ruby
# EXACT version
gem "rails", "= 7.1.2"      # Only 7.1.2, nothing else

# Optimistic constraint ~> (pessimistic version constraint)
gem "rails", "~> 7.1"       # >= 7.1, < 8.0  (patch+minor updates allowed)
gem "rails", "~> 7.1.2"     # >= 7.1.2, < 7.2  (patch updates only)
gem "rails", "~> 7"         # >= 7, < 8  (same as ~> 7.0 essentially)

# Standard comparison operators
gem "pg", ">= 1.0"          # any version >= 1.0
gem "pg", ">= 1.0", "< 2.0" # range: 1.0 to just below 2.0
gem "puma", "< 6"           # below 6 (usually means "not yet tested with 6")

# Semantic versioning convention:
# Major.Minor.Patch
# 7     .1    .2
# Major: breaking changes
# Minor: backward-compatible new features
# Patch: backward-compatible bug fixes

# Why ~> is preferred:
# gem "rack", "~> 2.2" — gets security patches (2.2.1, 2.2.2)
#                        but NOT 3.0 which might have breaking changes

# Bad practice: no constraint
gem "nokogiri"  # gets whatever is latest — future major versions might break your app
```

---

**Q6. What does `bundle install` do, and when should you run it?**

```ruby
# bundle install:
# 1. Reads Gemfile to find required gems and constraints
# 2. Resolves the full dependency graph (gems + their transitive deps)
# 3. Downloads and installs any missing gems
# 4. Writes/updates Gemfile.lock with exact resolved versions

# When to run bundle install:
# - After cloning a project
# - After adding/changing gems in the Gemfile
# - After checking out a branch that has different Gemfile.lock
# - After a teammate changes the Gemfile

# Options:
# bundle install --without production  (skip production group)
# bundle install --path vendor/bundle  (install gems locally, not system-wide)
# bundle install --jobs 4              (parallel downloads)
# bundle install --frozen              (fail if Gemfile.lock would change — for CI)

# In CI/CD, use --frozen to catch uncommitted Gemfile.lock changes:
# bundle install --frozen --retry=3

# The resolution algorithm:
# Bundler uses a SAT solver to find versions satisfying ALL constraints simultaneously
# If no solution exists → Bundler::VersionConflict error
# Example conflict:
# gem "A" requires "C" >= 2.0
# gem "B" requires "C" < 2.0
# → No version of C satisfies both → conflict
```

---

**Q7. What is `bundle exec` and why is it required?**

`bundle exec` runs a command in the context of the current bundle (Gemfile.lock). It prepends the locked gem paths to `$LOAD_PATH`, ensuring the bundled versions are used.

```ruby
# Without bundle exec: uses system gem version
rspec spec/  # might use rspec 3.8 if that's installed system-wide

# With bundle exec: uses Gemfile.lock version
bundle exec rspec spec/  # uses whatever version is in Gemfile.lock

# Why this matters:
# System: rspec 3.12 installed
# Gemfile.lock: rspec 3.10 locked
#
# rspec spec/        → uses 3.12 (system) — different behavior!
# bundle exec rspec  → uses 3.10 (locked) — consistent

# binstubs — shortcuts to avoid typing bundle exec every time:
bundle binstubs rspec-core  # creates bin/rspec
bin/rspec spec/              # equivalent to bundle exec rspec

# Gemfile.lock difference with system:
# If you run `rspec` directly and get a different version than expected,
# it's because bundle exec is missing

# In Rails, bin/ directory contains binstubs:
bin/rails server    # = bundle exec rails server
bin/rake            # = bundle exec rake
bin/rspec           # = bundle exec rspec
```

---

**Q8. Explain `Gemfile.lock` — what it contains and what it means for reproducibility.**

```ruby
# Gemfile.lock structure:
# GEM              ← gem sources section
#   remote: https://rubygems.org/
#   specs:
#     activerecord (7.1.2)     ← exact version, resolved
#       activemodel (= 7.1.2)  ← exact transitive dependency
#       activesupport (= 7.1.2)
#     rails (7.1.2)
#       activerecord (= 7.1.2)
#       ...
#
# PLATFORMS        ← which platforms this lock covers
#   arm64-darwin-22
#   x86_64-linux
#
# DEPENDENCIES     ← top-level gems from your Gemfile
#   rails (~> 7.1)
#   pg (>= 1.0)
#
# BUNDLED WITH     ← bundler version that created this lock
#   2.4.10

# Reading Gemfile.lock:
# "activerecord (7.1.2)" means EVERY developer, CI server, production box
# will get activerecord 7.1.2 — not 7.1.1, not 7.1.3

# Gemfile.lock should be committed for APPLICATIONS:
# - Guarantees same versions everywhere
# - Easy to bisect what changed between deployments
# - Required for reproducible builds

# Gemfile.lock should NOT be committed for LIBRARIES:
# - Your gem should work with a range of dep versions
# - Committing a lock would prevent consumers from resolving their own deps
# - Add Gemfile.lock to .gitignore for gems/libraries

# Updating the lock file:
# bundle update rails        — update only rails (+ its deps)
# bundle update              — update ALL gems to latest allowed by Gemfile
# bundle lock --update rails — update lock without installing
```

---

**Q9. What is the difference between `gem update` and `bundle update`?**

```ruby
# gem update: RubyGems command — installs latest versions system-wide
gem update rails       # installs latest rails version globally (ignores Gemfile!)
gem update             # updates ALL system gems — dangerous!

# bundle update: Bundler command — updates within Gemfile constraints
bundle update rails    # resolves latest rails version satisfying Gemfile constraints
                       # updates Gemfile.lock, installs if needed

bundle update          # updates ALL gems in the Gemfile to latest allowed versions
                       # rewrites Gemfile.lock — review the diff carefully!

# Safe update workflow:
# 1. Check what's outdated
bundle outdated        # shows which gems have newer available versions

# 2. Update one gem at a time
bundle update pg       # update just pg

# 3. Review the lock file diff
git diff Gemfile.lock  # see what changed

# 4. Run tests
bundle exec rspec

# 5. Commit if green
git add Gemfile.lock && git commit -m "Update pg to 1.5.4"

# Never do:
bundle update          # updates everything at once — test failures come from unknown gem changes
```

---

**Q10. How do you create a gem with `bundle gem`? What is the gemspec?**

```ruby
# Create gem skeleton:
bundle gem my_awesome_gem --coc --mit --test=rspec

# Generated structure:
# my_awesome_gem/
# ├── Gemfile                    ← for development dependencies
# ├── my_awesome_gem.gemspec     ← gem metadata
# ├── lib/
# │   ├── my_awesome_gem.rb      ← main require file
# │   └── my_awesome_gem/
# │       └── version.rb         ← version constant
# ├── spec/
# │   ├── spec_helper.rb
# │   └── my_awesome_gem_spec.rb
# ├── Rakefile
# ├── README.md
# └── CHANGELOG.md

# The gemspec:
Gem::Specification.new do |spec|
  spec.name          = "my_awesome_gem"
  spec.version       = MyAwesomeGem::VERSION
  spec.authors       = ["Your Name"]
  spec.email         = ["you@example.com"]

  spec.summary       = "Brief summary"
  spec.description   = "Longer description"
  spec.homepage      = "https://github.com/you/my_awesome_gem"
  spec.license       = "MIT"

  spec.required_ruby_version = ">= 3.0.0"

  spec.files         = Dir["lib/**/*.rb", "README.md", "LICENSE.txt"]
  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  # Runtime dependencies (installed when gem is installed)
  spec.add_dependency "activesupport", ">= 6.0"

  # Development dependencies (only for gem development, NOT for consumers)
  spec.add_development_dependency "rspec", "~> 3.12"
  spec.add_development_dependency "rubocop", "~> 1.57"
end
```

---

**Q11. What is a private gem server and how do you configure it?**

```ruby
# Options for private gems:
# 1. Gemfury (hosted)
# 2. JFrog Artifactory (self-hosted)
# 3. GitHub Packages
# 4. geminabox (lightweight self-hosted)
# 5. Private S3 bucket with gem-in-a-box

# Configuring in Gemfile:
source "https://gem.fury.io/mycompany/" do
  gem "private_analytics"
  gem "company_auth"
end

# Authentication:
# Option 1: .netrc file (~/.netrc)
# machine gem.fury.io
#   login TOKEN
#   password x

# Option 2: Bundler config
bundle config set --global https://gem.fury.io/mycompany/ "TOKEN:x"
# Stored in ~/.bundle/config (don't commit this!)

# Option 3: Environment variable in CI
# BUNDLE_GEM__FURY__IO__MYCOMPANY__=TOKEN:x bundle install

# For GitHub Packages:
source "https://rubygems.pkg.github.com/myorg" do
  gem "private_gem"
end
# bundle config set --global https://rubygems.pkg.github.com/myorg GITHUB_TOKEN:x

# For geminabox (self-hosted):
source "http://gems.internal.company.com"
# Or per-gem:
gem "internal_lib", source: "http://gems.internal.company.com"
```

---

**Q12. What are Rake tasks and how do you define and run them?**

```ruby
# Rake is Ruby's build tool (like make, but in Ruby)
# Rakefile defines tasks

# Basic task:
task :greet do
  puts "Hello from Rake!"
end
# Run: rake greet

# Task with dependencies:
task build: [:clean, :compile] do
  puts "Build complete"
end

task :clean do
  rm_rf "build/"
end

task :compile do
  sh "gcc src/*.c -o build/app"
end

# Namespace for organization:
namespace :db do
  desc "Migrate the database"
  task :migrate do
    sh "bundle exec rails db:migrate"
  end

  task :rollback do
    sh "bundle exec rails db:rollback"
  end
end
# Run: rake db:migrate

# Default task:
task default: [:test]

# Using FileList and file tasks:
require 'rake'

SRC_FILES = FileList["src/**/*.rb"]

task :test => SRC_FILES do
  sh "bundle exec rspec"
end

# In Rails, Rake tasks go in lib/tasks/*.rake:
# lib/tasks/data_migration.rake
namespace :data do
  desc "Backfill legacy user records"
  task backfill_users: :environment do  # :environment loads Rails
    User.where(migrated: false).find_each do |user|
      UserMigrationJob.perform_later(user.id)
    end
    puts "Enqueued #{User.where(migrated: false).count} users"
  end
end
```

---

**Q13. What is `bundle lock --add-platform` and why does it matter in CI/CD?**

```ruby
# Gemfile.lock records which platforms it was generated for
# If a gem has native extensions, different platforms need different binaries

# Problem: lock file generated on macOS (arm64-darwin-22)
# CI runs on Linux (x86_64-linux)
# Some gems have pre-compiled binaries for specific platforms

# Solution: add the target platform to the lock file
bundle lock --add-platform x86_64-linux    # for Linux CI
bundle lock --add-platform arm64-darwin    # for Apple Silicon dev

# This adds to Gemfile.lock:
# PLATFORMS
#   arm64-darwin-22
#   x86_64-linux       ← added

# Why it matters for nokogiri:
# nokogiri 1.15.4 has pre-compiled gems for:
#   x86_64-linux → downloads binary (fast, no gcc needed)
#   arm64-darwin → downloads binary (fast)
# Without the platform in the lock: Bundler might compile from source

# Docker-based CI: add in your setup script or locally before commit
# bundle lock --add-platform x86_64-linux
# git add Gemfile.lock && git commit -m "Add Linux platform to lock"
```

---

**Q14. How does `require` interact with Bundler's gem loading?**

```ruby
# Without Bundler — loads from $LOAD_PATH (system gems):
require 'rails'  # finds first rails.rb in $LOAD_PATH

# With Bundler (bundle exec or Bundler.setup):
# Bundler modifies $LOAD_PATH to point to the locked gem versions

# In a Rails app (config/boot.rb):
require 'bundler/setup'   # sets up $LOAD_PATH from Gemfile.lock
Bundler.require(:default) # requires all gems not in groups

# In a plain Ruby script:
require 'bundler/autoload'  # or:
Bundler.require             # auto-requires all default group gems

# require: false in Gemfile — don't auto-require:
gem "nokogiri", require: false
# The gem is in the bundle (can be required manually), but
# Bundler.require won't auto-require it on startup

# Manual require later:
require "nokogiri"  # safe because bundler ensures the right version is available

# Custom require path:
gem "redis-rb", require: "redis"  # require "redis" instead of "redis-rb"

# Understanding $LOAD_PATH:
puts $LOAD_PATH.grep(/nokogiri/)
# => ["/path/to/gems/nokogiri-1.15.4/lib"]
```

---

**Q15. What is `bundler-audit` and what security problems does it find?**

```ruby
# bundler-audit scans your Gemfile.lock against the Ruby Advisory Database
# It finds gems with known CVEs (security vulnerabilities)

# Install and run:
# gem install bundler-audit
# bundle audit check --update

# Output example:
# Name: activerecord
# Version: 7.0.4
# CVE: CVE-2023-22794
# Criticality: High
# URL: https://groups.google.com/forum/#!topic/rubyonrails-security/...
# Title: SQL Injection Vulnerability via ActiveRecord::Base#find_by

# bundle audit update — updates the advisory database
# bundle audit check  — check without updating

# Integrate in CI:
# .github/workflows/security.yml
# - name: Audit gems
#   run: bundle exec bundle-audit check --update

# Ignore known false positives:
# .bundler-audit.yml
ignore:
  - CVE-2023-XXXX  # false positive — not applicable to our use case

# Related: ruby_advisory_db
# https://github.com/rubysec/ruby-advisory-db
# Community-maintained database of vulnerable gem versions
```

---

**Q16. How does Bundler resolve conflicting dependencies?**

```ruby
# Scenario: Two gems require incompatible versions of a shared dependency
# Gemfile:
gem "gem_a"  # requires json >= 2.0, < 3.0
gem "gem_b"  # requires json ~> 1.8 (i.e., >= 1.8, < 2.0)

# bundle install output:
# Bundler could not find compatible versions for gem "json":
#   In Gemfile:
#     gem_a was resolved to 1.0.0, which depends on
#       json (>= 2.0, < 3.0)
#     gem_b was resolved to 2.0.0, which depends on
#       json (~> 1.8)
#
# Could not find gem 'json >= 2.0 AND ~> 1.8' in rubygems repository

# Solutions:
# 1. Update the conflicting gems to versions that agree
bundle update gem_a gem_b

# 2. Pin to versions you know are compatible
gem "gem_a", "1.2.0"  # version that requires json ~> 1.8
gem "gem_b", "2.0.0"

# 3. Fork and patch one of the gems to relax its constraint (last resort)

# Diagnosing conflicts:
bundle exec gem dependency json  # show what requires json and what versions
bundle viz                        # generate a visual dependency graph (requires ruby-graphviz)
```

---

**Q17. What is `gem 'rails', github: ...` vs `gem 'rails', git: ...`?**

```ruby
# Both use a Git repository as the gem source

# github: shorthand — adds https://github.com/ prefix automatically
gem "rails", github: "rails/rails"
# Equivalent to:
gem "rails", git: "https://github.com/rails/rails.git"

# Additional git options:
gem "my_gem", github: "user/my_gem", branch: "feature/new-api"
gem "my_gem", github: "user/my_gem", tag: "v2.0.0"
gem "my_gem", github: "user/my_gem", ref: "deadbeef1234"  # specific commit SHA

# When to use:
# - Testing a pull request fix before it's released
# - Using a forked gem with custom patches
# - Living on the bleeding edge of a gem's development

# Bundler caches the git repo:
# ~/.bundle/cache/git/rails-abc123/

# bundle update rails  — updates the git reference (re-clones or fetches)
```

---

**Q18. What is the difference between `add_dependency` and `add_development_dependency` in a gemspec?**

```ruby
Gem::Specification.new do |spec|
  # add_dependency (or add_runtime_dependency):
  # Installed automatically when a CONSUMER installs your gem
  spec.add_dependency "activesupport", ">= 6.0"
  spec.add_dependency "nokogiri", "~> 1.15"

  # add_development_dependency:
  # Only installed when developing the gem itself
  # NOT installed when a consumer does: gem install your_gem
  spec.add_development_dependency "rspec", "~> 3.12"
  spec.add_development_dependency "rubocop"
  spec.add_development_dependency "byebug"
end

# For consumers: gem install your_gem
# → activesupport + nokogiri installed ✓
# → rspec NOT installed (development only) ✓

# For development: bundle install (reads Gemfile which references gemspec)
# In Gemfile:
gemspec  # reads dependencies from .gemspec
# → all dependencies including development are installed ✓
```

---

**Q19. How do you vendor gems? When should you?**

```ruby
# Vendoring: copying gem code into your repository
bundle install --path vendor/bundle

# After this, gems are in:
# vendor/bundle/ruby/3.2.0/gems/

# Tell bundler to always use the vendor path:
bundle config set --local path 'vendor/bundle'
# Creates .bundle/config with:
# BUNDLE_PATH: "vendor/bundle"

# Pros of vendoring:
# - Deploy without internet access
# - Immutable gem code committed to repo
# - No dependency on rubygems.org availability
# - Docker builds are faster (gems already in the image layer)

# Cons of vendoring:
# - Repository size grows significantly (Rails app: hundreds of MBs)
# - All developers need to pull the vendor directory
# - Security updates require committing new gem files

# Heroku approach (bundle cache):
# Heroku caches the bundle between deploys (equivalent benefit, no repo bloat)

# When vendoring makes sense:
# - Air-gapped deployments (no internet)
# - Strictly controlled environments
# - When gems must be audited before use

# .gitignore for non-vendored approach:
echo "/vendor/bundle" >> .gitignore
```

---

**Q20. What happens if you run `ruby` directly vs `bundle exec ruby`?**

```ruby
# Direct ruby execution uses SYSTEM $LOAD_PATH:
ruby -e "require 'rails'; puts Rails::VERSION::STRING"
# → uses system-installed Rails version

# bundle exec ruby uses BUNDLED $LOAD_PATH:
bundle exec ruby -e "require 'rails'; puts Rails::VERSION::STRING"
# → uses Gemfile.lock Rails version

# Real-world issue:
# Gemfile.lock: rails 7.1.2
# System gem:   rails 7.0.4 (or 8.0.0 if you updated globally)
#
# ruby script.rb     → might fail if script uses Rails 7.1 features
# bundle exec ruby   → uses Rails 7.1.2 as expected

# Checking what's loaded:
bundle exec ruby -e "puts $LOAD_PATH.grep(/rails/).first"
# vs:
ruby -e "puts $LOAD_PATH.grep(/rails/).first"

# For executables in the PATH (rspec, rails, rake):
which rspec                # might find system rspec
bundle exec which rspec    # finds bundled rspec executable
```
