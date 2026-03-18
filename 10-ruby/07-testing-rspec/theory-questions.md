# Chapter 07: Testing with RSpec — Theory Questions

## RSpec Structure

**Q1. Describe the full structure of an RSpec test file. What are `describe`, `context`, `it`, and how do they relate?**

`describe` declares a test group, typically named after a class or method. `context` is an alias for `describe` but by convention expresses a condition ("when...", "with...", "given..."). `it` declares an example (a single test case).

```ruby
RSpec.describe ShoppingCart do
  describe "#add_item" do
    context "when the cart is empty" do
      it "adds the item successfully" do
        cart = ShoppingCart.new
        cart.add_item(Item.new("Widget", 9.99))
        expect(cart.items.count).to eq(1)
      end

      it "sets the item quantity to 1" do
        cart = ShoppingCart.new
        cart.add_item(Item.new("Widget", 9.99))
        expect(cart.items.first.quantity).to eq(1)
      end
    end

    context "when the item already exists in the cart" do
      it "increments the quantity" do
        cart = ShoppingCart.new
        item = Item.new("Widget", 9.99)
        cart.add_item(item)
        cart.add_item(item)
        expect(cart.items.first.quantity).to eq(2)
      end
    end
  end
end
```

---

**Q2. What is `subject` in RSpec and how is it auto-defined?**

`subject` is the object under test. When `describe` receives a class, `subject` is automatically set to `described_class.new` (calling the constructor with no arguments).

```ruby
RSpec.describe Array do
  # subject is implicitly Array.new (an empty array)
  it { is_expected.to be_empty }
  it { is_expected.to respond_to(:push) }

  # Explicit subject
  subject(:populated_array) { [1, 2, 3] }
  it "has three elements" do
    expect(populated_array.length).to eq(3)
  end

  # subject is also accessible as the bare `subject` method
  it "returns the class" do
    expect(subject.class).to eq(Array)
  end
end
```

---

**Q3. Explain `let` vs `let!` and when to use each.**

`let` is lazy — the value is computed on first access within a test. `let!` is eager — the block runs before each example, even if not referenced.

```ruby
RSpec.describe Order do
  let(:user)  { User.create!(name: "Alice") }  # lazy: only created if used
  let!(:order) { Order.create!(user: user) }    # eager: always created before each test

  it "belongs to a user" do
    # user is created here (lazily, because order references it via let!)
    # order was already created by let!
    expect(order.user).to eq(user)
  end

  it "can be cancelled" do
    # order was already created by let!, even though we didn't access it above
    expect { order.cancel! }.to change { order.reload.status }.to("cancelled")
  end
end
```

Use `let!` when you need side effects to exist before the test runs (e.g., database records that a tested query should return, even if the test doesn't reference the variable directly).

---

**Q4. How do `before` and `after` hooks work? What is the difference between `before(:each)` and `before(:all)`?**

```ruby
RSpec.describe DatabaseService do
  before(:all) do    # runs ONCE before all examples in this group
    @connection = Database.connect  # shared across all examples — DANGEROUS
  end

  after(:all) do     # runs ONCE after all examples
    @connection.disconnect
  end

  before(:each) do   # runs before EACH example (default when no arg given)
    @table = @connection.create_temp_table
  end

  after(:each) do    # runs after EACH example
    @connection.drop_table(@table)
  end

  it "inserts records" do
    @table.insert(id: 1, name: "Test")
    expect(@table.count).to eq(1)
  end

  it "deletes records" do
    @table.insert(id: 1, name: "Test")
    @table.delete(id: 1)
    expect(@table.count).to eq(0)
  end
end
```

`before(:all)` shares state between tests — a modification in one example can affect another. This causes test pollution. Prefer `before(:each)` (or just `before`) for isolation.

---

**Q5. What is the difference between `expect` syntax and `should` syntax?**

```ruby
# expect syntax (recommended, default in RSpec 3)
expect(actual).to eq(expected)
expect(actual).not_to be_nil
expect { code }.to raise_error(ArgumentError)

# should syntax (legacy, requires monkey patching Object)
actual.should eq(expected)
actual.should_not be_nil
lambda { code }.should raise_error(ArgumentError)

# The should syntax monkey patches every object with:
# def should(matcher) ...
# This pollutes every object, and fails on BasicObject subclasses
# RSpec 3 disabled it by default; enable with:
# RSpec.configure { |c| c.expect_with(:rspec) { |e| e.syntax = :should } }
```

---

**Q6. Explain the most important RSpec matchers: `eq`, `eql`, `equal`, `be`, and `be_truthy`.**

```ruby
# eq: value equality using ==
expect(1 + 1).to eq(2)
expect("hello").to eq("hello")   # different objects, same value ✓

# eql: value equality using eql? (stricter type check)
expect(1).to eql(1)
expect(1).not_to eql(1.0)       # Integer vs Float — fails eql?

# equal: object identity using equal? (same object_id)
str = "hello"
expect(str).to equal(str)        # ✓ same object
expect("hello").not_to equal("hello")  # ✓ different objects

# be with argument: uses ===
expect(1..10).to include(5)
expect(String).to be === "hello"

# Predicate matchers (be_*)
expect([]).to be_empty           # calls .empty?
expect(nil).to be_nil            # calls .nil?
expect(42).to be_positive        # calls .positive?
expect("hello").to be_a(String)  # calls .is_a?(String)
expect(obj).to be_frozen         # calls .frozen?

# be_truthy vs be_falsy vs be_nil
expect(1).to    be_truthy    # anything not nil or false
expect(false).to be_falsy    # nil or false
expect(nil).to  be_nil       # only nil
expect(false).to be_falsy    # false is falsy but not nil
expect(false).not_to be_nil  # false is NOT nil
```

---

**Q7. Explain the `change` matcher with all its forms.**

```ruby
# Basic change — checks that value changes
expect { Post.create!(title: "Hello") }.to change { Post.count }

# By a specific amount
expect { Post.create!(title: "Hello") }.to change { Post.count }.by(1)
expect { Post.destroy_all }.to change { Post.count }.by(-Post.count)

# From/to specific values
expect { post.publish! }.to change { post.status }.from("draft").to("published")

# Change with multiple assertions (compound)
expect { post.publish! }
  .to change { post.status }.to("published")
  .and change { post.published_at }.from(nil)

# NOT change
expect { user.name }.not_to change { User.count }
```

---

**Q8. What are RSpec doubles and what is the difference between `double`, `instance_double`, `class_double`, and `spy`?**

```ruby
# double: generic test double, no verification of real interface
user_double = double("User", name: "Alice", age: 30)
user_double.name  # => "Alice"
user_double.nonexistent_method  # => raises RSpec::Mocks::MockExpectationError

# instance_double: VERIFIED double — checks that methods exist on the class
user_double = instance_double(User, name: "Alice")
# Raises error if User class doesn't have a #name instance method
# Raises error if you stub a method with wrong arity

# class_double: verified double for class-level methods
mailer_double = class_double(UserMailer, deliver_welcome: nil)
# Raises error if UserMailer doesn't have .deliver_welcome class method

# spy: like double but records all messages without requiring expectations upfront
spy_obj = spy("database")
spy_obj.insert(record)   # doesn't raise — spy accepts any message
spy_obj.query(sql)       # records it

# Verify afterward
expect(spy_obj).to have_received(:insert).with(record)
expect(spy_obj).to have_received(:query).once

# instance_spy: verified spy
user_spy = instance_spy(User)
user_spy.save
expect(user_spy).to have_received(:save)
```

---

**Q9. Explain `allow` vs `expect` for message expectations.**

```ruby
user = instance_double(User)

# allow: sets up a stub — doesn't fail if never called
allow(user).to receive(:name).and_return("Alice")
user.name  # => "Alice"
# If name is never called, test still passes

# expect: sets up a mock expectation — FAILS if not called
expect(user).to receive(:save)
# If user.save is never called before the test ends → FAILURE

# allow is for: controlling return values of collaborators
# expect is for: asserting that a collaboration happened

# Combined stub + return value:
allow(user).to receive(:name).and_return("Alice")
allow(user).to receive(:age).and_return(30)

# With arguments:
allow(service).to receive(:find).with(42).and_return(user)
allow(service).to receive(:find).with(anything).and_return(nil)

# Message count constraints:
expect(mailer).to receive(:send_email).once
expect(cache).to receive(:fetch).at_least(:twice)
expect(db).to receive(:query).exactly(3).times
```

---

**Q10. What are `shared_examples` and `shared_context`? When should you use each?**

```ruby
# shared_examples: reusable behavior specs for objects with the same interface
RSpec.shared_examples "a sortable collection" do
  it "responds to sort" do
    expect(subject).to respond_to(:sort)
  end

  it "returns elements in ascending order by default" do
    expect(subject.sort).to eq(subject.sort { |a, b| a <=> b })
  end
end

RSpec.describe Array do
  subject { [3, 1, 2] }
  it_behaves_like "a sortable collection"
end

RSpec.describe MyCustomCollection do
  subject { MyCustomCollection.new([3, 1, 2]) }
  it_behaves_like "a sortable collection"
end

# shared_context: reusable setup (let, before hooks, helpers)
RSpec.shared_context "authenticated user" do
  let(:user) { create(:user, role: :admin) }
  let(:token) { JWT.encode({ user_id: user.id }, 'secret') }
  let(:headers) { { "Authorization" => "Bearer #{token}" } }

  before { user.confirm! }
end

RSpec.describe AdminController, type: :request do
  include_context "authenticated user"

  it "allows access to admin dashboard" do
    get "/admin", headers: headers
    expect(response).to have_http_status(:ok)
  end
end
```

---

**Q11. Explain the `and_return`, `and_raise`, `and_yield`, and `and_call_original` stub responses.**

```ruby
# and_return: return a value (or multiple values for successive calls)
allow(service).to receive(:find).and_return(user)
allow(service).to receive(:next).and_return(1, 2, 3)  # first call→1, second→2, third→3

# and_raise: raise an exception
allow(api_client).to receive(:get).and_raise(Net::TimeoutError, "Request timed out")

# and_yield: invoke the passed block with given arguments
allow(file).to receive(:each_line).and_yield("line 1\n").and_yield("line 2\n")

# and_call_original: execute the real method (useful when partially stubbing)
allow(User).to receive(:find).and_call_original
allow(User).to receive(:find).with(999).and_raise(ActiveRecord::RecordNotFound)
# → find(42) calls real method, find(999) raises error

# Combining:
allow(service).to receive(:process) do |input|
  # Custom block — has access to arguments
  input.upcase
end
```

---

**Q12. What is the `have_received` matcher and when must it be used?**

```ruby
# have_received is for asserting AFTER the call happened
# Required for: spy objects and calls to allow() stubs

user = spy("User")
service = UserService.new(user)
service.register("Alice")  # internally calls user.save

# Assert the collaboration happened:
expect(user).to have_received(:save).once

# With arguments:
expect(user).to have_received(:name=).with("Alice")

# Order doesn't matter with have_received unless you use ordered
expect(user).to have_received(:validate).before(:save)

# Difference from expect(...).to receive:
# expect + receive: must be set BEFORE the call (is a mock expectation)
# have_received:    asserted AFTER the call (needs allow or spy first)

user2 = double("User")
allow(user2).to receive(:save)
user2.save

expect(user2).to have_received(:save)  # ✓
```

---

**Q13. What is Factory Bot and how do `build`, `create`, `build_stubbed` differ?**

```ruby
# Define a factory
FactoryBot.define do
  factory :user do
    name  { "Alice" }
    email { "alice@example.com" }
    role  { :viewer }

    trait :admin do
      role { :admin }
      name { "Admin User" }
    end

    trait :with_posts do
      after(:create) do |user|
        create_list(:post, 3, author: user)
      end
    end

    sequence(:email) { |n| "user#{n}@example.com" }
  end
end

# build: instantiates the object WITHOUT saving to DB
user = build(:user)           # User.new with attributes set
user.persisted?               # => false

# create: instantiates AND saves to DB (hits the database)
user = create(:user)          # User.create!
user.persisted?               # => true

# build_stubbed: creates a fake-persisted object (no DB hit)
user = build_stubbed(:user)
user.id                       # => some integer (stubbed)
user.persisted?               # => true (stubbed)
user.save                     # raises — stubs prevent DB writes

# When to use which:
# build        → unit tests, no DB needed
# create       → integration tests, queries, associations
# build_stubbed → fast unit tests with persisted-looking objects
```

---

**Q14. What is VCR and how does it work for testing HTTP interactions?**

```ruby
# VCR records real HTTP calls and replays them in subsequent test runs
require 'vcr'

VCR.configure do |config|
  config.cassette_library_dir = "spec/fixtures/vcr_cassettes"
  config.hook_into :webmock  # intercepts at WebMock level
  config.configure_rspec_metadata!
  config.filter_sensitive_data('<API_KEY>') { ENV['API_KEY'] }
end

# Usage — first run makes real request and records it
RSpec.describe WeatherService do
  it "fetches current temperature", :vcr do
    service = WeatherService.new(api_key: ENV['API_KEY'])
    result = service.temperature_for("London")
    expect(result[:celsius]).to be_a(Numeric)
  end
  # Second run plays back from cassette — no real HTTP
end

# Explicit cassette naming:
it "handles API errors" do
  VCR.use_cassette("weather/api_error") do
    result = service.temperature_for("UnknownCity")
    expect(result[:error]).to eq("city not found")
  end
end
```

---

**Q15. What is SimpleCov and how do you interpret coverage reports?**

```ruby
# In spec_helper.rb or spec/support/simplecov.rb — BEFORE any application code
require 'simplecov'
SimpleCov.start do
  add_filter '/spec/'
  add_filter '/config/'

  add_group 'Models',      'app/models'
  add_group 'Controllers', 'app/controllers'
  add_group 'Services',    'app/services'

  minimum_coverage 90          # fail if below 90%
  minimum_coverage_by_file 70  # fail if any file below 70%
end

# SimpleCov tracks WHICH LINES were executed during tests
# Line coverage: was this line run at least once?
# Branch coverage (SimpleCov 0.18+): were both if/else branches taken?

SimpleCov.start do
  enable_coverage :branch  # enable branch coverage
end

# Interpreting the report:
# Green: covered (executed during tests)
# Red: not covered (never executed)
# Yellow (branch): line executed but not all branches taken

# What 100% line coverage does NOT guarantee:
# - All edge cases tested
# - Correct behavior tested (you might hit every line but not assert results)
# - Thread safety
# - Integration between components
```

---

**Q16. What is `RSpec.configure` and what can you set in it?**

```ruby
RSpec.configure do |config|
  # Order randomization (finds order-dependent test failures)
  config.order = :random
  config.seed  = 12345  # reproducible run with same seed

  # Include helpers
  config.include FactoryBot::Syntax::Methods
  config.include Rails.application.routes.url_helpers, type: :request

  # Global before/after hooks
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end

  # Metadata-driven behavior
  config.before(:each, :js) do
    # Run JS-capable browser for examples tagged with :js
  end

  # Filter examples
  config.filter_run_when_matching :focus
  config.run_all_when_everything_filtered = true

  # Treat warnings as errors
  config.warnings = true
end
```

---

**Q17. Explain `:focus`, `:skip`, and `:pending` metadata.**

```ruby
# :focus — run only focused examples (with filter_run_when_matching :focus)
it "important test", :focus do
  expect(1 + 1).to eq(2)
end

fit "shorthand for :focus" do  # f prefix = focused
  expect(true).to be true
end

fdescribe "focused describe group" do  # all examples in group focused
  it "runs" do; end
end

# :skip — skip the example entirely (no output about pending)
it "not implemented yet", :skip do
  # code here is never run
end

xit "shorthand for skip" do  # x prefix = skipped
  expect(false).to be true  # never runs
end

# :pending — run but expect failure; PASSES if failing, FAILS if it passes
it "expected to fail", :pending do
  expect(1).to eq(2)   # test "passes" because it's pending and fails
end

pending "not yet done"  # mark with message

it "is pending explicitly" do
  pending "fix the flaky network call"
  expect(external_api_call).to succeed  # runs but expects failure
end
```

---

**Q18. What is the difference between `instance_double` and `double`? When does `instance_double` catch real bugs?**

```ruby
class UserService
  def find_user(id)
    User.find(id)
  end
end

# double: no verification — accepts any method/arity
user = double("User", nme: "Alice")  # typo: nme instead of name
user.nme  # => "Alice" — passes!
# But real User objects have no .nme method → bug undetected

# instance_double: VERIFIED against actual class interface
user = instance_double(User, nme: "Alice")
# => raises: User does not implement: nme (RSpec::Mocks::MockExpectationError)

# Also verifies argument counts:
user = instance_double(User)
allow(user).to receive(:update).with(1, 2)  # User#update takes (attrs) not (n, n)
# => raises: Wrong number of arguments

# CATCHES REAL BUGS:
# 1. Renamed methods: rename User#full_name → User#display_name
#    instance_double catches all stubs using :full_name automatically
# 2. Changed signatures: method receives different parameters
# 3. Private methods being stubbed: raises error if method is private
```

---

**Q19. What is the RSpec test execution order and how do `before(:suite)`, `before(:all)`, `before(:each)` relate?**

```
Test execution sequence:

  before(:suite)    ← once, before all spec files load
    ↓
  before(:all)      ← once per describe/context group
    ↓
  before(:each)     ← before EVERY it block
    ↓
  let/subject       ← lazy: when first accessed in the example
    ↓
  it block runs
    ↓
  after(:each)      ← after EVERY it block
    ↓
  after(:all)       ← once per describe/context group
    ↓
  after(:suite)     ← once, after everything
```

```ruby
RSpec.describe "Execution Order" do
  before(:all)  { puts "1. before all" }
  before(:each) { puts "3. before each" }
  after(:each)  { puts "5. after each" }
  after(:all)   { puts "7. after all" }

  let(:val) { puts "4. let evaluated"; 42 }

  it "first test" do
    puts "#{val} - running first test"  # triggers let on access
  end

  it "second test" do
    puts "#{val} - running second test"
  end
end

# Output:
# 1. before all
# 3. before each
# 4. let evaluated
# 42 - running first test
# 5. after each
# 3. before each
# 4. let evaluated  ← re-evaluated for EACH test (let is not shared)
# 42 - running second test
# 5. after each
# 7. after all
```

---

**Q20. How does Minitest differ from RSpec in philosophy and syntax?**

```ruby
# Minitest — part of Ruby standard library, minimal, fast
require 'minitest/autorun'

class UserTest < Minitest::Test
  def setup          # like before(:each)
    @user = User.new(name: "Alice")
  end

  def teardown       # like after(:each)
    # cleanup
  end

  def test_has_name  # methods must start with test_
    assert_equal "Alice", @user.name
  end

  def test_email_validation
    @user.email = "not-valid"
    refute @user.valid?
    assert_includes @user.errors[:email], "is invalid"
  end

  def test_raises_on_nil_name
    assert_raises(ArgumentError) { User.new(name: nil) }
  end
end

# Minitest::Spec — DSL closer to RSpec (but still Minitest under the hood)
require 'minitest/spec'

describe User do
  subject { User.new(name: "Alice") }

  it "has a name" do
    subject.name.must_equal "Alice"
  end
end

# Key differences:
# RSpec:     BDD-focused, rich DSL, extensive matcher library, slower to load
# Minitest:  xUnit-style + spec mode, stdlib, fast, less magic, easier to debug
```

---

**Q21. How do you test for changes to multiple things in one `expect` block?**

```ruby
# Using compound matchers with .and
expect { post.publish! }
  .to change { post.status }.from("draft").to("published")
  .and change { post.published_at }.from(nil)
  .and change { Post.published.count }.by(1)

# and_also for non-change assertions:
expect(result)
  .to be_a(Hash)
  .and include(:success)
  .and have_key(:data)

# Multiple assertions in one example (acceptable when strongly related):
aggregate_failures "response structure" do
  expect(response.status).to eq(200)
  expect(response.body).to include("success")
  expect(response.headers["Content-Type"]).to include("application/json")
end
# aggregate_failures runs ALL assertions and reports all failures at once
# (vs stopping at first failure)
```

---

**Q22. What are verifying doubles and why should you prefer them over plain doubles?**

```ruby
# Plain double: no connection to real class — stubs survive renames
payment = double("PaymentProcessor",
  process_card: true,       # method name might not exist
  charge_ammount: 100       # typo: should be charge_amount
)

# Test passes even though:
# - process_card might have been renamed to process_payment
# - charge_ammount is a typo

# Verifying doubles: sync with actual class interface
payment = instance_double(PaymentProcessor,
  process_payment: true     # raises if PaymentProcessor#process_payment doesn't exist
)

# When the real method is renamed, ALL instance_doubles referencing the old name fail
# → forces you to update both the code and the tests

# instance_double  → verifies instance method interface
# class_double     → verifies class method interface
# object_double    → verifies a specific object's interface (for singletons)

config = object_double(Rails.application.config,
  database_url: "postgres://..."
)
```

---

**Q23. Explain the RSpec `:aggregate_failures` feature.**

```ruby
# Without aggregate_failures: stops at first failure
it "validates user" do
  expect(user.name).to eq("Alice")    # fails here...
  expect(user.email).to be_present    # ...never reaches this
  expect(user.age).to be >= 18        # ...or this
end
# You see only the first failure; have to fix and re-run for others

# With aggregate_failures: all assertions run, all failures reported
it "validates user", :aggregate_failures do
  expect(user.name).to eq("Alice")
  expect(user.email).to be_present
  expect(user.age).to be >= 18
end
# Reports ALL failures at once:
# 1) expected "Bob" to eq "Alice"
# 2) expected nil to be present

# Or as a block for specific assertions:
it "returns a valid response" do
  response = api.call

  aggregate_failures "response shape" do
    expect(response[:status]).to eq(200)
    expect(response[:body]).to be_a(Hash)
    expect(response[:headers]).to include("Content-Type")
  end
end
```

---

**Q24. How do you test time-sensitive code in RSpec?**

```ruby
# Using Timecop gem
require 'timecop'

it "expires tokens after 24 hours" do
  token = Token.generate

  Timecop.travel(25.hours.from_now) do
    expect(token.expired?).to be true
  end
end

it "is not expired immediately after creation" do
  Timecop.freeze do  # freezes time
    token = Token.generate
    expect(token.expired?).to be false
  end
end

# Using ActiveSupport::Testing::TimeHelpers (Rails)
it "expires after one day" do
  token = Token.generate

  travel_to 25.hours.from_now do
    expect(token.expired?).to be true
  end
end

# Using RSpec's own time helpers (rspec-rails)
# freeze_time, travel_to

# Manually: inject time as a dependency (cleanest, no gems needed)
class Session
  def initialize(created_at: Time.now)
    @created_at = created_at
  end

  def expired?
    Time.now > @created_at + 3600
  end
end

it "expires after one hour" do
  session = Session.new(created_at: 2.hours.ago)
  expect(session.expired?).to be true
end
```

---

**Q25. How do you use `have_attributes` and `respond_to` matchers?**

```ruby
user = User.new(name: "Alice", age: 30, email: "alice@example.com")

# have_attributes: checks multiple attribute values at once
expect(user).to have_attributes(
  name:  "Alice",
  age:   30,
  email: "alice@example.com"
)

# Equivalent to:
expect(user.name).to  eq("Alice")
expect(user.age).to   eq(30)
expect(user.email).to eq("alice@example.com")

# Useful with Factory Bot:
expect(create(:user, :admin)).to have_attributes(
  role:  :admin,
  admin?: true
)

# respond_to: checks interface
expect(user).to respond_to(:name, :email, :save, :valid?)
expect(user).to respond_to(:name).with(0).arguments
expect(user).to respond_to(:update).with_unlimited_arguments
```
