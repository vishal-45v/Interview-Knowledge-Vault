# Chapter 09: Gems & Bundler — Follow-Up Traps

## Trap 1: `bundle exec` Is Required When Gemfile.lock Has Different Version Than System

**The trap:** Running executables directly (`rspec`, `rails`, `rake`) uses the system-installed gem version, which may differ from the Gemfile.lock version. This causes subtle "works on my machine" failures.

```bash
# System has: rspec 3.12 (installed globally via gem install rspec)
# Gemfile.lock has: rspec 3.9.2

# This runs rspec 3.12 (system):
rspec spec/
# Might pass because 3.12 has different behavior than 3.9.2

# This runs rspec 3.9.2 (locked):
bundle exec rspec spec/
# This is what your team and CI should use

# Real-world consequence:
# Teammate commits test that passes with rspec 3.12 but fails with 3.9.2
# CI uses bundle exec (correct) → CI fails
# Teammate can't reproduce → "But it passes on my machine!"
# Root cause: missing bundle exec

# Permanent fix: use binstubs
bundle binstubs rspec-core
# Creates bin/rspec — always uses bundled version
bin/rspec spec/

# Or add to PATH: always use bundle exec
# .bashrc/.zshrc:
# alias rspec='bundle exec rspec'
# alias rake='bundle exec rake'
# (careful: this might break non-bundled uses)
```

---

## Trap 2: Gemfile.lock Should Be Committed for Apps, NOT for Libraries

**The trap:** Developers incorrectly treat both applications and libraries the same way for Gemfile.lock — either always committing it or always ignoring it.

```bash
# APPLICATION (Rails app, Ruby script, CLI tool):
# → COMMIT Gemfile.lock
# Reason: you want EXACT same versions in dev/staging/prod/CI
git add Gemfile.lock
git commit -m "Lock dependency versions"

# LIBRARY / GEM:
# → DO NOT commit Gemfile.lock (or add to .gitignore)
# Reason: your consumers need to resolve their OWN deps
# If you commit Gemfile.lock, you prevent users from installing
# your gem alongside their other gems

echo "Gemfile.lock" >> .gitignore  # for gems/libraries only

# Why it matters for libraries:
# Your gem's Gemfile.lock says: "activesupport 7.1.2"
# Consumer's app has: "activesupport 7.0.8"
# If you committed Gemfile.lock, bundler might try to honor it
# and create a conflict with the consumer's own dependencies

# The gemspec controls what versions your library WORKS WITH:
spec.add_dependency "activesupport", ">= 6.1"  # flexible range
# Consumer's Gemfile.lock resolves the actual version

# Verification: check your gem's .gitignore
cat .gitignore | grep Gemfile.lock
# For a library, it should appear there
```

---

## Trap 3: `~> 2.3` Means `>= 2.3, < 3.0` — NOT `>= 2.3, < 2.4`

**The trap:** The pessimistic constraint `~>` is commonly misunderstood. Its behavior depends on how many version components you specify.

```ruby
# ~> with TWO components (major.minor):
gem "rails", "~> 7.1"
# → >= 7.1, < 8.0
# Allows: 7.1.0, 7.1.1, 7.2.0, 7.9.9
# Blocks:  8.0.0

# ~> with THREE components (major.minor.patch):
gem "rails", "~> 7.1.2"
# → >= 7.1.2, < 7.2
# Allows: 7.1.2, 7.1.3, 7.1.99
# Blocks:  7.2.0, 8.0.0

# The "pessimistic" part: assumes that:
# - Patch version changes (7.1.2 → 7.1.3) are SAFE (bug fixes)
# - Minor version changes (7.1 → 7.2) MIGHT have breaking changes
# - Major version changes (7 → 8) ARE breaking changes

# ~> with ONE component (major):
gem "rails", "~> 7"
# → >= 7, < 8  (equivalent to ~> 7.0 in practice)

# TRAP: thinking ~> 2.3 only allows 2.3.x
# It actually allows 2.3, 2.4, 2.5... up to (but not including) 3.0

# Common mistake:
gem "some_gem", "~> 2.3"  # developer thought: "only 2.3 versions"
# But some_gem 2.9 is released with breaking API changes
# And your code breaks because ~> 2.3 allowed 2.9

# Better: be more specific
gem "some_gem", "~> 2.3.0"  # only 2.3.x versions
# OR specify exact range:
gem "some_gem", ">= 2.3", "< 2.4"  # same as ~> 2.3.0
```

---

## Trap 4: `gem install` vs `bundle install` Have Different Scopes

**The trap:** Confusing `gem install` (system-wide) with `bundle install` (project-specific), leading to gems being available in one context but not the other.

```bash
# gem install: installs to GEM_HOME (system/rbenv-managed)
# Available to ALL projects using this Ruby version
gem install httparty
# Now httparty is available anywhere for this Ruby version

# bundle install: installs based on Gemfile, records in Gemfile.lock
# Only available when running through Bundler for THIS project
bundle install
# Installs exactly what Gemfile.lock specifies

# THE TRAP:
# You: gem install some_gem  (installs system-wide)
# You: ruby script.rb        (requires some_gem — WORKS)
# Colleague: bundle exec ruby script.rb  (uses bundle — FAILS: gem not in Gemfile)

# Other direction:
# You: bundle install  (adds to Gemfile.lock)
# You: bundle exec rspec  (WORKS)
# You: rspec spec/    (FAILS: system rspec may not have the bundled version)

# Debugging which gems are available:
ruby -e "require 'some_gem'; puts 'loaded'" 2>&1      # system context
bundle exec ruby -e "require 'some_gem'; puts 'loaded'" 2>&1  # bundled context

# When gem install IS appropriate:
# - Installing CLI tools you use globally (bundler, rake, pry, gem-release)
# - Installing gems for one-off scripts not in a Bundler project
# - Installing the gem to your Ruby version's base (not project-specific)
```

---

## Trap 5: Bundler Resolution Can Silently Downgrade Gems

**The trap:** When you add a new gem to the Gemfile that has strict transitive dependencies, Bundler may resolve the conflict by downgrading OTHER gems you already had. This can break things subtly.

```ruby
# Before: Gemfile.lock has rails 7.1.2
# You add: gem "some_old_gem" (which requires rails < 7.0)

# bundle install output (quietly):
# Using rails 6.1.7 (downgraded from 7.1.2)
# Bundle complete! 25 Gemfile dependencies, 78 gems now installed.

# Your app might fail at runtime with:
# NoMethodError: undefined method `new_rails_71_feature' for ...
# → Because you're now on rails 6.1.7

# Detecting silent downgrades:
git diff Gemfile.lock  # ALWAYS check this after bundle install

# bundle install output verbosity:
bundle install --verbose  # shows which gems are being installed/changed

# Safer workflow:
bundle update --conservative rails  # update only rails, don't touch others
# vs:
bundle update rails  # updates rails AND all its transitive deps

# bundle check — verifies bundle is satisfied without installing:
bundle check  # exits 1 if any gem would be installed/changed
```

---

## Trap 6: Git Gems Don't Respect Semantic Versioning

**The trap:** When you use `gem "xyz", github: "user/xyz"` without a `tag:`, you get whatever the default branch HEAD points to. This means your gem can change any time the author pushes, breaking your build.

```ruby
# RISKY: gets current HEAD of main/master
gem "my_lib", github: "user/my_lib"

# Every developer who runs `bundle install` might get a different version!
# bundle update my_lib fetches new commits

# SAFE: pin to a specific tag (maps to a release)
gem "my_lib", github: "user/my_lib", tag: "v1.2.3"

# SAFER BUT NOT IDEAL: pin to a specific commit SHA
gem "my_lib", github: "user/my_lib", ref: "deadbeef1234"
# Immutable but harder to audit/update

# The Gemfile.lock records the resolved commit SHA:
# GEM
#   remote: https://github.com/user/my_lib.git
#   revision: deadbeef...
#   specs:
#     my_lib (1.2.3)  ← version from gemspec at that commit

# Updating a git gem:
bundle update my_lib  # fetches latest, updates Gemfile.lock
# Always review the diff to see what changed
```

---

## Trap 7: `require: false` Doesn't Remove the Gem from the Bundle

**The trap:** Thinking `require: false` means the gem won't be installed or available. It just means Bundler won't auto-require it; the gem is still in the bundle.

```ruby
# Gemfile:
gem "nokogiri", require: false  # installed but NOT auto-required

# What this means:
Bundler.require  # does NOT call `require 'nokogiri'`

# But you can still manually require it:
require 'nokogiri'  # WORKS — gem is in the bundle, just not auto-required

# gem "nokogiri", require: false is typically used for:
# 1. Heavy gems that are only needed in specific code paths
gem "pdfkit", require: false
# In the controller that generates PDFs:
require 'pdfkit'  # only loaded when needed

# 2. Gems with non-standard require paths
gem "aws-sdk-s3", require: false
require "aws-sdk-s3"  # explicit require is fine

# 3. Optional features
gem "redis", require: false
# Only load Redis if configured:
require 'redis' if ENV['REDIS_URL'].present?

# TRAP: thinking the gem is not in the bundle
# Even with require: false:
# - The gem IS downloaded and installed
# - It IS present in vendor/bundle (if vendoring)
# - It IS listed in Gemfile.lock
# - It IS included in Bundler's load path
```

---

## Trap 8: `bundle update` Without Arguments Updates EVERYTHING

**The trap:** Running `bundle update` with no arguments updates all gems to the latest version satisfying Gemfile constraints. This can cause widespread failures in a single command.

```bash
# DANGEROUS: updates everything at once
bundle update

# After this: many gems may be at new major/minor versions
# Your test suite might have 50 failures from various gems changing behavior

# SAFE: update one gem at a time
bundle update rails           # only updates rails (and its hard dependencies)
bundle update nokogiri        # only updates nokogiri
bundle update pg              # only updates pg

# SAFER: update with --conservative flag
bundle update rails --conservative
# Conservative mode: tries to minimize the number of gems changed
# → updates rails but avoids updating other gems that don't NEED to change

# REVIEW the diff after any bundle update:
git diff Gemfile.lock | head -100

# Track which gems updated (useful for CHANGELOG):
bundle update 2>&1 | grep "Fetching" | awk '{print $2}'
# Or use bundler-diff gem for nicer output:
# gem install bundler-diff
# bundle diff  (shows before/after versions)

# Safe update workflow for a production app:
# 1. bundle outdated  (see what's stale)
# 2. Pick ONE gem to update
# 3. bundle update gem_name
# 4. git diff Gemfile.lock  (review what changed)
# 5. bundle exec rspec  (run tests)
# 6. git commit -m "Update gem_name to X.Y.Z"
# Repeat for each gem
```

---

## Trap 9: Gemfile Groups Not Excluded in Production Can Cause Issues

**The trap:** By default, `bundle install` installs ALL groups including development and test. In production, you should exclude them to keep the footprint small.

```bash
# DEVELOPMENT/CI: install everything
bundle install

# PRODUCTION: exclude development and test groups
bundle install --without development test
# OR (modern bundler syntax):
bundle config set --local without 'development test'
bundle install

# Heroku (auto-configured):
# Heroku automatically sets BUNDLE_WITHOUT=development:test

# Docker production image:
# Dockerfile:
RUN bundle install --without development test --jobs=4 --frozen

# Why this matters in production:
# - byebug, pry-rails, etc. have debugging overhead
# - rspec-rails loads test helpers that shouldn't be in production
# - factory_bot_rails registers factories, adding startup time
# - Some dev gems use more memory or have security implications

# Checking which groups are excluded:
bundle config list
# BUNDLE_WITHOUT: "development:test"

# Trap: if you deploy with --without production by accident
bundle config set --local without 'production'
bundle install
# → production gems not installed → app fails to start
```

---

## Trap 10: Gem Version Pinning Can Block Security Updates

**The trap:** Over-constraining gem versions (using `=` or very tight ranges) prevents getting security patches.

```ruby
# TOO TIGHT: exact pin prevents security patches
gem "activerecord", "= 7.1.2"
# When CVE is fixed in 7.1.3, you CANNOT get it without manually changing the Gemfile

# BETTER: allow patch updates (security patches are usually patches)
gem "rails", "~> 7.1.2"  # → >= 7.1.2, < 7.2 (gets patch security fixes)

# EVEN BETTER for most gems: allow minor updates
gem "rails", "~> 7.1"    # → >= 7.1, < 8.0

# BEST PRACTICE: only pin when you have a specific reason
gem "some_gem", "= 2.1.5"  # only if 2.1.6+ has a breaking change for YOU

# Setting up automated dependency updates:
# dependabot.yml in your GitHub repo:
version: 2
updates:
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "weekly"
    # Creates weekly PRs to update gems, runs CI to verify

# Or use Renovate bot for more configuration options

# For critical security updates: subscribe to ruby security mailing list
# https://www.ruby-lang.org/en/security/
# And RubyGems security alerts: https://groups.google.com/forum/#!forum/rubygems-security
```
