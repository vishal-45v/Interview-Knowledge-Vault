# Chapter 09 — Testing Rails: Follow-Up Traps

---

**Trap 1: "You use `let(:user) { create(:user) }` in a shared context used by 50 specs. What's the performance impact?"**

```ruby
# let is lazy — only evaluated when referenced
# BUT each example that references `user` creates a new user record in the DB
# 50 examples × 1 DB write = 50 DB writes (if all reference user)

# This is by design — test isolation. Each test gets a fresh user.
# The "problem" only appears when:

# 1. Over-creating: using create when build is sufficient
describe "name validation" do
  let(:user) { create(:user) }  # UNNECESSARY DB write
  it { expect(user.name).to be_present }
end
# Fix: let(:user) { build(:user) } — no DB write needed for simple attribute tests

# 2. Creating in outer context, not using in all inner tests
describe User do
  let(:user) { create(:user) }  # Created for EVERY example in this describe block

  it "validates presence of email" do
    # user referenced → DB write
  end

  it "returns correct full name" do
    u = User.new(first_name: "Alice", last_name: "Smith")
    expect(u.full_name).to eq("Alice Smith")  # Doesn't use outer `user` — but let is lazy
    # BUT if another it in the group references user, let evaluates it
    # If THIS it doesn't use user → no DB write for this example!
  end
end

# 3. let! forces evaluation even if not referenced
describe User do
  let!(:user) { create(:user) }  # Created for EVERY example regardless!
  it "tests something unrelated" do
    # user not referenced, but already created by let!
    # Unnecessary DB write
  end
end

# Rule: use let (lazy) not let! unless you need the side effect (e.g., record must exist in DB before action)
```

---

**Trap 2: "Your test uses `stub_const('STRIPE_KEY', 'test_key')`. After the test, a different test fails because STRIPE_KEY is wrong. Why?"**

```ruby
# stub_const in RSpec does restore the original value after the example
# BUT: only if you use it inside an example (it block) or before block that's scoped correctly

# The trap: using stub_const in before(:all) or after(:context)
# These run ONCE for the entire context, not per-example
# RSpec's cleanup hooks may not restore properly

describe PaymentService do
  before(:all) do
    stub_const('STRIPE_KEY', 'test_key')  # Set once
  end
  # after(:all) does NOT automatically restore stub_const
  # → STRIPE_KEY stays as 'test_key' after this context finishes!
end

# Fix: use before(:each) / before (default)
describe PaymentService do
  before do
    stub_const('STRIPE_KEY', 'test_key')  # Restored after each example
  end
end

# Or use RSpec's allow_any_instance_of:
before do
  allow(ENV).to receive(:[]).with('STRIPE_KEY').and_return('test_key')
end

# Real gotcha: constants loaded at boot time
# If STRIPE_KEY = ENV['STRIPE_KEY'] runs at class load, stub_const on the constant
# doesn't change where the value was already consumed
```

---

**Trap 3: "You mock an external API call with `allow(ExternalApi).to receive(:fetch).and_return(data)`. The test passes. But in production, the real API returns different keys. What did you test?"**

```ruby
# You tested that YOUR CODE handles `data` correctly
# You did NOT test that the API actually returns that structure

# Danger: API response contract drift
# Test mock: { "user_id" => 1, "email" => "alice@example.com" }
# Real API response: { "userId" => 1, "emailAddress" => "alice@example.com" }
# Test passes → production breaks

# Solution 1: Contract tests (Pact)
# Define the expected contract and test both sides

# Solution 2: VCR cassettes (record real responses)
# Record actual API responses once, replay them in tests
require 'vcr'
VCR.use_cassette("external_api/user_42") do
  result = ExternalApi.fetch(user_id: 42)
  expect(result.email).to eq("alice@example.com")
end
# VCR makes REAL request first time, records to cassette file
# Subsequent test runs replay the recording — no real HTTP
# Problem: recordings go stale when API changes

# Solution 3: Integration tests against sandbox/staging API
# A separate test suite that runs against real (sandbox) API periodically
# Catches real API changes

# Solution 4: Typed response parsing
class ExternalApi
  Response = Struct.new(:user_id, :email, keyword_init: true)

  def self.parse_response(raw)
    Response.new(
      user_id: raw.fetch("userId"),  # fetch raises KeyError if key missing
      email:   raw.fetch("emailAddress")
    )
  end
end
# Test the parser with real (or VCR) response shapes
```

---

**Trap 4: "You're testing a Sidekiq job. You call `MyJob.perform_later(user.id)` in the test. The job runs immediately. Why?"**

```ruby
# In test environment, Rails uses :test adapter for ActiveJob by default
# :test adapter DOES NOT execute jobs immediately
# It queues them in memory but doesn't run them

# If your job IS running immediately, one of these is true:
# 1. You're using Sidekiq::Testing.inline! mode
require 'sidekiq/testing'
Sidekiq::Testing.inline!  # All Sidekiq jobs execute synchronously inline

# 2. You're calling .perform_now not .perform_later
MyJob.perform_now(user.id)  # Runs immediately regardless of adapter

# 3. You have this in spec_helper.rb:
config.before(:each, type: :job) do
  ActiveJob::Base.queue_adapter = :test
  perform_enqueued_jobs  # This DOES run jobs immediately within the block
end

# Correct testing pattern:
require 'rails_helper'

RSpec.describe MyJob, type: :job do
  it "enqueues the job" do
    expect { MyJob.perform_later(user.id) }.to have_enqueued_job(MyJob)
  end

  it "performs the job" do
    perform_enqueued_jobs do
      MyJob.perform_later(user.id)
    end
    # Now assert side effects
  end

  it "tests job logic directly" do
    MyJob.perform_now(user.id)  # Execute directly, bypass queue
    # Assert side effects
  end
end
```

---

**Trap 5: "`build_stubbed` is faster than `create`. But your test fails when you use `build_stubbed`. What might be the cause?"**

```ruby
# build_stubbed creates a fake-persisted object without hitting the DB
# It stubs: id (random), persisted? → true, new_record? → false
# It does NOT touch the database at all

# Fails when:
# 1. Code queries the database for associations
class Post
  def recent_comments
    comments.order(created_at: :desc).limit(5)  # DB query on association
  end
end

user = build_stubbed(:user)
post = build_stubbed(:post, user: user)
# post.recent_comments → tries to query DB with fake ID → no real records exist!

# 2. Validations that check uniqueness
class User
  validates :email, uniqueness: true  # DB query
end
user = build_stubbed(:user)
user.valid?  # Runs uniqueness check → queries DB → no record found → might not behave as expected

# 3. Callbacks that call external services with real DB calls
class Order
  after_create :charge_stripe!  # Won't run with build_stubbed (not persisted)
end

# Rule: build_stubbed is best for pure unit tests of:
# - Simple method logic
# - Validations (non-uniqueness)
# - Serializers
# - Policies (Pundit)

# Use create when you need:
# - Database queries in the code under test
# - Association traversal
# - after_create/after_commit callbacks
# - System/integration tests
```

---

**Trap 6: "You use `DatabaseCleaner.strategy = :transaction` but your system tests are leaving data behind. Why?"**

```ruby
# DatabaseCleaner :transaction wraps each test in a transaction that's rolled back
# Works for: unit tests, request specs, controller specs that run in same DB connection

# DOES NOT work for: system tests that use a real browser (Selenium/Cuprite)
# Because:
# - Your Rails app runs in one thread (with one DB connection/transaction)
# - Your browser (Selenium) makes HTTP requests that are processed in DIFFERENT threads
# - Different threads = different DB connections = can't share a transaction!
# - Transaction rollback happens on Rails test thread, not browser thread

# Fix: use :truncation for system tests
RSpec.configure do |config|
  config.before(:each, type: :system) do
    DatabaseCleaner.strategy = :truncation  # Deletes all rows after each system test
  end

  config.before(:each, type: :request) do
    DatabaseCleaner.strategy = :transaction  # Faster — rollback for request specs
  end
end

# Performance note: truncation is much slower than transaction
# Minimize system tests; use Capybara's JavaScript-less mode for non-JS tests

# Alternative: use transactional system tests (Rails 5.1+)
# By forcing Capybara to share the same connection
# config/support/capybara.rb:
class ActiveRecord::Base
  mattr_accessor :shared_connection
  @@shared_connection = nil

  def self.connection
    @@shared_connection || retrieve_connection
  end
end
ActiveRecord::Base.shared_connection = ActiveRecord::Base.connection
```

---

**Trap 7: "Your test asserts `expect(response.body).to include('Success')`. It passes locally but fails in CI. The response body is JSON in CI. What happened?"**

```ruby
# Local: request accepts HTML → renders success flash in HTML response
# CI: different Accept header or format → renders JSON → no flash in JSON

# Root cause: test doesn't specify format, inherits from environment defaults
# Your CI might have different Accept header defaults

# Always specify format in request specs:
get '/posts/1', headers: { 'Accept' => 'application/json' }
# or
get '/posts/1.json'

# Better: test what you actually care about
# For JSON APIs:
expect(response).to have_http_status(:ok)
body = JSON.parse(response.body)
expect(body['data']['title']).to eq('My Post')

# For HTML responses:
expect(response).to have_http_status(:ok)
expect(response.body).to include('My Post')
# OR use Capybara matchers:
expect(page).to have_content('My Post')

# Pro tip: set default format in spec helper for API tests:
RSpec.describe "Posts API", type: :request do
  let(:headers) { { 'Content-Type' => 'application/json', 'Accept' => 'application/json' } }

  it "returns a post" do
    get "/api/v1/posts/#{post.id}", headers: headers
    expect(response.content_type).to include('application/json')
  end
end
```

---

**Trap 8: "A developer says 'I have 95% code coverage so our code must be well tested.' What's wrong with this claim?"**

```ruby
# Coverage measures which lines were EXECUTED, not which behaviors were TESTED

# High coverage + no assertions:
def calculate_discount(price, tier)
  if tier == :premium
    price * 0.8   # 20% off
  else
    price
  end
end

it "calculates discount" do
  calculate_discount(100, :premium)  # Line executed!
  calculate_discount(100, :standard) # Line executed!
  # No expect(...) → 100% coverage, zero assertions!
end

# Mutation testing reveals the gap:
# Stryker/Mutant changes your code: price * 0.8 → price * 0.9
# If tests still pass → you're not actually testing the 20% value!

# Coverage as a signal, not a guarantee:
# Good: "our coverage dropped from 85% to 60% → we added unexercised code"
# Bad: "95% coverage means we're confident in our code"

# Better metrics:
# - Mutation test score (what % of mutations are caught by tests)
# - Test behavior coverage (are edge cases, error paths tested?)
# - Test reliability (do tests fail when bugs are introduced?)

# SimpleCov can enforce a minimum coverage threshold:
SimpleCov.minimum_coverage 90
# But 90% coverage with no assertions is worse than 70% with meaningful assertions
```

---

**Trap 9: "Your shared example group has a test that depends on a method `create_user` defined in one spec but not another. It works in one file, fails in another. Why?"**

```ruby
# Shared examples run in the context of the including describe block
# Methods defined outside the shared example may or may not be available

# BROKEN:
# spec/support/shared_examples/authenticatable.rb
shared_examples "authenticatable" do
  it "redirects unauthenticated users" do
    get path  # `path` defined in including context?
    expect(response).to redirect_to(sign_in_path)
  end
end

# spec/requests/posts_spec.rb
describe "Posts" do
  let(:path) { '/posts' }
  it_behaves_like "authenticatable"  # Works — path is defined
end

# spec/requests/comments_spec.rb
describe "Comments" do
  # forgot to define `path`
  it_behaves_like "authenticatable"  # FAILS — path not defined
end

# Fix: require all necessary context to be defined in the shared example params
shared_examples "authenticatable" do |path:|
  it "redirects unauthenticated users" do
    get path
    expect(response).to redirect_to(sign_in_path)
  end
end

it_behaves_like "authenticatable", path: '/posts'
it_behaves_like "authenticatable", path: '/comments'

# Or use subject / parameters explicitly:
shared_examples "requires authentication" do
  before { subject }
  it { is_expected.to redirect_to(sign_in_path) }
end
```

---

**Trap 10: "You use `allow_any_instance_of(User).to receive(:send_welcome_email)`. Why is this considered a code smell?"**

```ruby
# allow_any_instance_of is an escape hatch for testing legacy or poorly-designed code
# It bypasses the normal test double mechanism

# Problems:
# 1. Unclear which instance is being stubbed
# 2. Multiple instances can be ambiguous: which user's method is stubbed?
# 3. It signals that your code is hard to test (tight coupling)
# 4. No verification: are you testing the right interaction?

# Example:
allow_any_instance_of(User).to receive(:send_welcome_email).and_return(true)
# Which user? ALL users? Does this affect other code in the test?

# BETTER: inject the dependency or test the result, not the method call
# Instead of mocking send_welcome_email, test that the email was enqueued:
expect { user.register! }.to have_enqueued_mail(UserMailer, :welcome).with(user)

# Or stub at the service level:
allow(UserMailer).to receive(:welcome).and_return(double(deliver_later: nil))

# The only valid use of allow_any_instance_of:
# - Truly legacy code you can't refactor
# - Third-party classes you don't control
# - Time-boxed spike testing

# Preferred pattern: explicit object creation and stubbing
user = create(:user)
allow(user).to receive(:send_welcome_email)
UserRegistrationService.new(user).call
expect(user).to have_received(:send_welcome_email)
```

---
