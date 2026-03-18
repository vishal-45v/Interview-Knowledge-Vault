# Chapter 09: Gems & Bundler — Scenario Questions

**S1. A new developer clones your Rails app and runs `rails server` — it fails. Walk them through the correct setup.**

```bash
# Wrong: directly running rails
$ rails server
# => Could not find gem 'rails (~> 7.1)' in locally installed gems

# Correct setup sequence:
# 1. Install the right Ruby version
rbenv install 3.2.0  # or use .ruby-version auto-detection
rbenv local 3.2.0

# 2. Install Bundler (if not present for this Ruby version)
gem install bundler

# 3. Install all gems from Gemfile.lock
bundle install
# Downloads and installs all gems pinned in Gemfile.lock

# 4. Now run via bundle exec (or use binstubs)
bundle exec rails server
# OR (using the project binstub):
bin/rails server

# Why bin/rails works without bundle exec:
# cat bin/rails:
#!/usr/bin/env ruby
load File.expand_path('../spring', __FILE__)
# It sets up Bundler automatically via spring or explicit require 'bundler/setup'

# Quick verification all gems are correct:
bundle check  # exits 0 if all gems satisfied, 1 otherwise
```

---

**S2. You inherit a Rails app where `bundle install` fails with a version conflict. Diagnose and fix it.**

```bash
# Error output:
# Bundler could not find compatible versions for gem "activesupport":
#   In Gemfile:
#     rails (~> 7.1) was resolved to 7.1.2, which depends on
#       activesupport (= 7.1.2)
#
#     aws-sdk-rails (~> 3.9) was resolved to 3.9.0, which depends on
#       activesupport (>= 6.1, < 8.0)
#
# ← This means aws-sdk-rails 3.9.0 only supports up to activesupport 7.x
# but rails 7.1.2 needs activesupport 7.1.2 exactly

# Diagnosis steps:
bundle exec gem dependency activesupport  # see what requires it
bundle outdated                           # what's outdated
```

```ruby
# Fix options:

# Option 1: Try updating both conflicting gems together
# bundle update rails aws-sdk-rails

# Option 2: Pin one of them to a compatible version
# Gemfile:
gem "aws-sdk-rails", "~> 3.8"  # older version compatible with Rails 7.1

# Option 3: Use Bundler's verbose output to trace the conflict
# bundle install --verbose

# Option 4: Use bundle viz to see the dependency graph visually
# bundle viz && open bundler_dependencies.png  # requires ruby-graphviz

# Prevention: use bundle check in CI
# .github/workflows/ci.yml:
# - run: bundle check || bundle install --frozen
```

---

**S3. You need to use a gem that has a security vulnerability but no fix is available yet. What do you do?**

```ruby
# Step 1: Understand the vulnerability
bundle audit check --update
# Output:
# Name:        nokogiri
# Version:     1.14.0
# CVE:         CVE-2023-24815
# Criticality: Medium
# URL:         ...
# Title:       Buffer over-read in Nokogiri::XML::Node#parse

# Step 2: Assess impact to YOUR app
# - Does your app use the vulnerable code path?
# - Is the input untrusted (from users) or trusted (internal data)?

# Step 3: Options:
# a) Update to patched version
gem "nokogiri", "~> 1.15"  # update in Gemfile, then bundle update nokogiri

# b) If no fix exists, add to ignore list with justification
# .bundler-audit.yml:
ignore:
  - CVE-2023-24815  # Only affects XML parsing from untrusted sources;
                    # we only parse internal data. Reviewed 2024-01-15.

# c) Implement compensating controls
# Sanitize input before passing to nokogiri
# Add WAF rules to block malicious payloads

# d) Fork the gem and patch it
gem "nokogiri", git: "https://github.com/ourorg/nokogiri.git",
                branch: "fix/CVE-2023-24815"

# Step 4: Document the decision in your security log
# SECURITY.md or Jira ticket with:
# - CVE number
# - Affected versions
# - Why you can't update (if applicable)
# - Compensating controls
# - Timeline for fix

# Step 5: Set a calendar reminder to re-check for a patch
```

---

**S4. You're releasing a gem. Walk through the publish workflow.**

```bash
# 1. Ensure tests pass
bundle exec rspec

# 2. Update version
# lib/my_gem/version.rb:
# VERSION = "1.2.0"

# 3. Update CHANGELOG.md
# Add release notes for 1.2.0

# 4. Build the gem locally and verify
gem build my_gem.gemspec
# Creates: my_gem-1.2.0.gem

# 5. Test the built gem in isolation
gem install my_gem-1.2.0.gem
ruby -e "require 'my_gem'; puts MyGem::VERSION"

# 6. Tag and push
git add .
git commit -m "Release v1.2.0"
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin main --tags

# 7. Push to RubyGems.org
gem push my_gem-1.2.0.gem
# Prompts for rubygems.org credentials first time

# Alternative: use gem-release gem for automation
bundle exec gem bump minor  # bumps version
bundle exec gem release      # builds, tags, pushes

# For private gem server (Gemfury):
gem push my_gem-1.2.0.gem --host https://gem.fury.io/mycompany/

# Using Rake tasks (already in skeleton):
rake release  # equivalent to: bump + build + tag + push
```

---

**S5. Your CI is failing with "Your Gemfile.lock is out of date." Fix it.**

```bash
# Error in CI:
# ! The `bundle install --frozen` flag is set and the Gemfile.lock
# is not up to date. Please run `bundle install` locally and commit
# the updated Gemfile.lock.

# Why this happens:
# - Someone added a gem to Gemfile but didn't commit the updated Gemfile.lock
# - Gemfile.lock was generated on a different platform (missing PLATFORM entry)
# - Bundler version mismatch between local and CI

# Fix 1: Regenerate locally and commit
bundle install  # regenerates Gemfile.lock
git add Gemfile.lock
git commit -m "Update Gemfile.lock"
git push

# Fix 2: Add missing platform (common with arm64 dev + x86_64 CI)
bundle lock --add-platform x86_64-linux
git add Gemfile.lock && git commit -m "Add Linux platform to Gemfile.lock"

# Fix 3: If Bundler version differs
bundle update --bundler  # update the BUNDLED WITH version in Gemfile.lock

# Prevention: add to CI check
# .github/workflows/ci.yml:
- name: Check Gemfile.lock is up to date
  run: bundle check
  # OR:
- name: Install gems (frozen)
  run: bundle install --frozen --jobs=4 --retry=3
```

---

**S6. You need to override a gem's behavior without forking it. Show the patterns.**

```ruby
# Pattern 1: Monkey patch (risky, but sometimes necessary for quick fixes)
# config/initializers/nokogiri_patch.rb
require 'nokogiri'

module Nokogiri
  module XML
    class Document
      def safe_parse(html)
        parse(html.encode('UTF-8', invalid: :replace, undef: :replace))
      end
    end
  end
end

# Pattern 2: Override in a gem fork (proper way)
# 1. Fork the gem on GitHub
# 2. Apply your fix
# 3. Reference your fork in Gemfile:
gem "nokogiri",
    github: "yourorg/nokogiri",
    branch: "fix/encoding-issue"

# Pattern 3: Use refinements (safe monkey patch)
module SaferNokogiri
  refine Nokogiri::XML::Document do
    def safe_parse(html)
      parse(html.encode('UTF-8', invalid: :replace))
    end
  end
end

# Pattern 4: Decorator/wrapper
class SafeXMLParser
  def initialize(parser = Nokogiri::XML)
    @parser = parser
  end

  def parse(html)
    @parser.parse(html.encode('UTF-8', invalid: :replace))
  rescue Nokogiri::XML::SyntaxError => e
    NullDocument.new
  end
end

# Which to use:
# - Refinements: safest, most isolated
# - Decorator: cleanest OO, no monkey patch
# - Fork: when the fix is substantial and you want to upstream it
# - Monkey patch: only for tiny fixes in initializers, with a comment
```

---

**S7. Your deployment takes 10 minutes just to install gems. How do you speed it up?**

```bash
# Problem: bundle install re-downloads gems on every deploy

# Solution 1: Cache the bundle between deployments
# Capistrano: shared_path/bundle
# .capistrano/deploy.rb:
set :bundle_path, -> { shared_path.join('bundle') }
set :bundle_flags, '--frozen --quiet'
set :bundle_jobs, 4  # parallel download

# Solution 2: Docker layer caching
# Dockerfile:
COPY Gemfile Gemfile.lock ./
RUN bundle install --frozen --jobs=4 --retry=3
# Copy the rest AFTER bundle install
# → gems layer is cached as long as Gemfile.lock doesn't change
COPY . .

# Solution 3: Use pre-compiled gem variants
# nokogiri, grpc, and other gems with C extensions have binary packages
# Modern bundler auto-uses them; verify with:
bundle install --verbose | grep "Using nokogiri"
# "Using nokogiri 1.15.4 (arm64-darwin)" ← binary gem
# "Building native extensions (gcc...)"  ← compiling from source (slow!)

# Force binary gems:
bundle config set --local force_ruby_platform false  # use platform gems (default)
bundle config set --local force_ruby_platform true   # compile everything (slow!)

# Solution 4: GitHub Actions cache
# .github/workflows/ci.yml:
- name: Cache gems
  uses: actions/cache@v3
  with:
    path: vendor/bundle
    key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
    restore-keys: |
      ${{ runner.os }}-gems-

- name: Install gems
  run: |
    bundle config path vendor/bundle
    bundle install --jobs=4 --retry=3
```

---

**S8. You need to test a gem you're developing against multiple Rails versions. How do you set this up?**

```ruby
# gemspec: keep runtime dependencies broad
spec.add_dependency "activesupport", ">= 6.1"
# NOT: ">= 7.1" — this would exclude Rails 6.x users

# Create multiple Gemfiles for testing different versions:
# gemfiles/rails_61.gemfile:
source "https://rubygems.org"
gem "rails", "~> 6.1"
gemspec

# gemfiles/rails_70.gemfile:
source "https://rubygems.org"
gem "rails", "~> 7.0"
gemspec

# gemfiles/rails_71.gemfile:
source "https://rubygems.org"
gem "rails", "~> 7.1"
gemspec

# Run tests against specific Gemfile:
BUNDLE_GEMFILE=gemfiles/rails_70.gemfile bundle install
BUNDLE_GEMFILE=gemfiles/rails_70.gemfile bundle exec rspec

# Automate with appraisal gem:
# Appraisals file:
appraise "rails-6.1" do
  gem "rails", "~> 6.1"
end

appraise "rails-7.0" do
  gem "rails", "~> 7.0"
end

appraise "rails-7.1" do
  gem "rails", "~> 7.1"
end

# bundle exec appraisal install   # install all
# bundle exec appraisal rspec     # run against all versions

# GitHub Actions matrix:
# .github/workflows/test.yml:
strategy:
  matrix:
    rails: ["6.1", "7.0", "7.1"]
steps:
  - run: BUNDLE_GEMFILE=gemfiles/rails_${{ matrix.rails }}.gemfile bundle exec rspec
```

---

**S9. You need to audit all your gems for licensing compliance (e.g., no GPL gems in commercial app).**

```ruby
# Use license_finder gem
# gem install license_finder
# license_finder

# It reads Gemfile.lock and reports each gem's license

# Example output:
# Dependencies approved:
#   rails, 7.1.2, MIT
#   pg, 1.5.3, BSD-2-Clause
#
# Dependencies lacking approval:
#   some_gem, 1.0.0, GPL-3.0  ← PROBLEM for commercial use

# Common licenses and their implications:
# MIT, BSD-2, BSD-3, Apache-2.0 → permissive, safe for commercial use
# LGPL → usually ok for dynamic linking (gems), review carefully
# GPL-3.0 → copyleft — using in commercial closed-source may be prohibited
# AGPL-3.0 → must open source even for network use — avoid in SaaS

# Programmatically check licenses:
require 'bundler'
Bundler.load.specs.each do |spec|
  license = spec.license || spec.licenses.first
  puts "#{spec.name}: #{license}"
end

# Approve known licenses in license_finder:
license_finder approvals add "MIT"
license_finder approvals add "Apache 2.0"
license_finder approvals add "BSD"

# Approve a specific gem with unapproved license (with justification):
license_finder approvals add some_gem --who "Alice" --why "Verified GPL usage is acceptable here"

# CI integration:
# - name: Check licenses
#   run: bundle exec license_finder
```

---

**S10. You want to understand why a specific gem is included in your bundle (which gem depends on it).**

```bash
# Use bundle info to see a gem's details:
bundle info nokogiri
# * nokogiri (1.15.4)
#   Summary: An HTML, XML, SAX, and Reader parser with XPath and CSS selector support
#   Homepage: https://nokogiri.org
#   Path: /path/to/gems/nokogiri-1.15.4

# Use bundle list to see all gems:
bundle list

# Find which gem requires nokogiri:
bundle exec gem dependency nokogiri --reverse-dependencies
# → rails-html-sanitizer 1.6.0 depends on nokogiri
# → kaminari 1.2.2 depends on nokogiri

# Or use grep on Gemfile.lock:
grep -B5 "nokogiri" Gemfile.lock

# For deep dependency chains:
# bundler-stats gem provides detailed dependency tree output:
bundle exec bundler-stats nokogiri

# In Ruby code:
require 'bundler'
spec = Bundler.load.specs.find { |s| s.name == "nokogiri" }
puts spec.dependencies.map(&:name)  # what nokogiri depends on

# Find who depends on a gem (reverse lookup):
Bundler.load.specs.select do |spec|
  spec.dependencies.any? { |dep| dep.name == "nokogiri" }
end.map(&:name)
```

---

**S11. Your application has a `Gemfile.lock` that lists gems not used in code. Should you remove them?**

```ruby
# Unused gems = security surface area, slower bundle install, potential conflicts
# To find unused gems: use the bundlesize or unused gem tools

# Step 1: Identify unused requires
# Use debase or bundler's built-in deprecation
bundle exec rails stats  # gives summary but not unused gems

# Use debride gem (finds methods/gems not called):
# gem install debride
# debride --rails lib/ app/ spec/

# Step 2: For each suspected unused gem, check:
# - Is it required somewhere? grep -r "require 'gem_name'" .
# - Is it a transitive dependency? bundle exec gem dependency gem_name
# - Is it environment-specific? group :development, :test do...

# Step 3: Remove carefully
# Gemfile: remove the gem line
# Then:
bundle install  # updates Gemfile.lock

# Step 4: Verify nothing broke
bundle exec rspec
bundle exec rails server  # manual verification

# Step 5: Check if the gem is auto-required:
# Bundler.require(:default) auto-requires all default group gems
# Even if you don't explicitly `require` it in code,
# Bundler.require might be doing it for you at boot

# Gems that appear "unused" but are actually needed:
# - tzinfo-data (Windows timezone support)
# - bootsnap (startup speed)
# - listen (file system watching)
# - spring (development speed)
# - rdoc, ri (documentation)

# Decision rule: if removing a gem causes tests/app to fail → keep it
# If removing causes nothing to break → remove it
```

---

**S12. Show how to use `bundle exec` in a Makefile/script for a consistent developer experience.**

```makefile
# Makefile
.PHONY: setup test lint deploy

setup:
	gem install bundler
	bundle install
	bundle exec rails db:setup

test:
	bundle exec rspec --format progress --color

test-watch:
	bundle exec guard

lint:
	bundle exec rubocop --parallel

lint-fix:
	bundle exec rubocop --auto-correct

console:
	bundle exec rails console

server:
	bundle exec rails server -b 0.0.0.0 -p 3000

routes:
	bundle exec rails routes

migrate:
	bundle exec rails db:migrate

rollback:
	bundle exec rails db:rollback

seed:
	bundle exec rails db:seed

ci: lint test

# Shell script alternative:
#!/usr/bin/env bash
# bin/setup

set -e

echo "Installing dependencies..."
bundle install

echo "Setting up database..."
bundle exec rails db:create db:migrate

echo "Seeding database..."
bundle exec rails db:seed

echo "✓ Setup complete! Run 'bin/rails server' to start."
```
