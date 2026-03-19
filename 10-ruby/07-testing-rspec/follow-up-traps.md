# Chapter 07: Testing with RSpec — Follow-Up Traps

## Trap 1: `let` Is Lazy — It Won't Run Until Referenced

**The trap:** `let` is only evaluated when the variable is first accessed inside an example. If you `let` a side-effecting operation (like `create(:record)`) and then write a test that never references the variable, the record is never created.

```ruby
RSpec.describe Report do
  let(:user)    { User.create!(name: "Alice") }
  let(:orders)  { create_list(:order, 5, user: user) }  # lazy!

  it "counts user orders" do
    # BUG: orders is never referenced, so no orders are created
    # This test will likely fail or behave unexpectedly
    expect(user.orders.count).to eq(5)  # => 0 — orders were never created!
  end

  # FIX 1: reference the variable
  it "counts user orders" do
    orders  # force evaluation
    expect(user.orders.count).to eq(5)
  end

  # FIX 2: use let! for eager evaluation
  let!(:orders) { create_list(:order, 5, user: user) }
  it "counts user orders" do
    expect(user.orders.count).to eq(5)  # now works
  end
end
```

**Interview follow-up:** "When does `let` get memoized?"
Answer: Once per example. Each new `it` block gets a fresh evaluation. `let` is NOT shared across examples — only across multiple calls to the same `let` name within one `it` block.

---

## Trap 2: `let!` vs `let` — Eager Evaluation Means DB Hits Even for Passing Tests

**The trap:** Using `let!` everywhere for safety causes unnecessary database writes in every test, even when the data isn't needed.

```ruby
RSpec.describe UserPolicy do
  let!(:admin)    { create(:user, :admin) }    # always created (even in tests that don't need it)
  let!(:articles) { create_list(:article, 10) } # always created

  it "allows admin to read any article" do
    # Only needs admin — but articles were also created
    policy = UserPolicy.new(admin)
    expect(policy.read?).to be true
  end

  it "returns correct count" do
    # Needs articles
    expect(Article.count).to eq(10)
  end

  # Better: use let for admin (only needed in some tests)
  # Use let! only for articles (needed for the count test to work)
end
```

**Rule of thumb:** Use `let` by default. Use `let!` only when the side effect (DB record creation) must exist even if the variable name is never referenced in the test body.

---

## Trap 3: `before(:all)` Shares State and Causes Test Pollution

**The trap:** `before(:all)` creates shared state across all examples. If one test modifies the state (updating a record, clearing a collection), subsequent tests see the modified state.

```ruby
RSpec.describe AdminDashboard do
  before(:all) do
    @admin = User.create!(role: :admin, name: "Admin")  # shared!
    @users = User.create!([{ name: "A" }, { name: "B" }])
  end

  it "can promote users" do
    @users.first.update!(role: :editor)  # MUTATES SHARED STATE
    # ...test passes
  end

  it "sees all users as viewers" do
    # BROKEN: @users.first is now an editor because the previous test mutated it
    expect(User.viewer.count).to eq(2)  # => fails: only 1 viewer now
  end

  # FIX: use before(:each) for fresh state, or use DB transaction rollback
  before(:each) do  # each test gets its own records
    @admin = User.create!(role: :admin, name: "Admin")
    @users = User.create!([{ name: "A" }, { name: "B" }])
  end
end
```

**When is `before(:all)` safe?** For read-only, immutable setup (loading a fixture file, connecting to an external service once). Never use it when any test might write to the shared state.

---

## Trap 4: `allow` vs `expect` — `allow` Never Fails If Not Called

**The trap:** Using `allow` when you mean `expect` makes the test pass even if the method is never called at all. `allow` means "it's OK if this is called" — not "assert that this was called."

```ruby
class NotificationService
  def notify(user)
    UserMailer.welcome(user).deliver_later
  end
end

RSpec.describe NotificationService do
  let(:user)    { instance_double(User, email: "alice@example.com") }
  let(:service) { NotificationService.new }

  # BUG: This test always passes even if deliver_later is never called
  it "sends welcome email" do
    mailer_double = instance_double(ActionMailer::MessageDelivery)
    allow(UserMailer).to receive(:welcome).and_return(mailer_double)
    allow(mailer_double).to receive(:deliver_later)  # allow — doesn't assert!

    service.notify(user)
    # No assertion — test passes regardless of whether email was sent
  end

  # CORRECT: use expect to assert the call happened
  it "sends welcome email" do
    mailer_double = instance_double(ActionMailer::MessageDelivery)
    allow(UserMailer).to receive(:welcome).and_return(mailer_double)
    expect(mailer_double).to receive(:deliver_later)  # ASSERTS it's called

    service.notify(user)
  end

  # OR: use spy + have_received
  it "sends welcome email" do
    mailer_spy = spy("mailer")
    allow(UserMailer).to receive(:welcome).and_return(mailer_spy)

    service.notify(user)

    expect(mailer_spy).to have_received(:deliver_later)  # assert after the fact
  end
end
```

---

## Trap 5: `instance_double` Requires the Class to Be Loaded

**The trap:** `instance_double(ClassName)` looks up the real class to verify the interface. If the class isn't loaded at spec evaluation time, you get a `NameError` or the double won't verify.

```ruby
# In spec_helper.rb / rails_helper.rb
# Rails autoloading usually handles this, but in isolated unit tests it's a trap

# TRAP: class not loaded
instance_double("LegacyPaymentProcessor")  # Ruby string form — defers lookup
instance_double(LegacyPaymentProcessor)    # constant form — fails if not loaded

# FIX: require the file explicitly in unit tests
require_relative '../../app/services/legacy_payment_processor'

# OR use string form which defers verification:
instance_double("LegacyPaymentProcessor", process: true)
# Verification still happens but only if the class is loaded when the test runs

# In Rails specs this usually isn't a problem because rails_helper loads the app
# It's a real trap in lightweight/unit test suites that don't load the full app
```

---

## Trap 6: Subject Is Auto-Instantiated — Constructor Arguments Matter

**The trap:** `subject` is automatically `described_class.new`. If the class requires constructor arguments, the auto-subject raises `ArgumentError`.

```ruby
class PaymentProcessor
  def initialize(gateway:, logger:)  # keyword arguments required
    @gateway = gateway
    @logger  = logger
  end
end

RSpec.describe PaymentProcessor do
  # subject is implicitly PaymentProcessor.new — but constructor requires args!
  it "exists" do
    expect(subject).to be_a(PaymentProcessor)  # => ArgumentError!
  end

  # FIX: define subject explicitly
  subject(:processor) do
    PaymentProcessor.new(
      gateway: instance_double(StripeGateway),
      logger:  instance_double(Logger)
    )
  end

  it "processes payments" do
    expect(processor).to be_a(PaymentProcessor)
  end
end

# Also watch out: described_class.new and Subject.new SHARE the auto-subject
# If you change the constructor and forget to update subject definitions in specs,
# every spec in that describe block breaks at once.
```

---

## Trap 7: `have_received` Must Come After a `spy` or an `allow`

**The trap:** You can't use `have_received` on a regular `double` unless you first set up `allow`. If you just call a method on a double with no prior stub, it raises `MockExpectationError`.

```ruby
user = double("User")

# WRONG: calling a method on double with no allow/expect setup
user.save   # => RSpec::Mocks::MockExpectationError: unexpected message :save
expect(user).to have_received(:save)  # never gets here

# CORRECT approach 1: spy (accepts everything)
user = spy("User")
user.save
expect(user).to have_received(:save)  # works

# CORRECT approach 2: allow first, then have_received
user = double("User")
allow(user).to receive(:save)
user.save
expect(user).to have_received(:save)  # works

# CORRECT approach 3: expect + receive (set up BEFORE the call)
user = double("User")
expect(user).to receive(:save)  # must be BEFORE the call
user.save  # satisfies the expectation
```

---

## Trap 8: `and_return` with Multiple Values — Exhaustion Returns Last Value

**The trap:** When you stub with multiple return values, after the list is exhausted, subsequent calls return the *last* value — not `nil` and not an error.

```ruby
iterator = double("iterator")
allow(iterator).to receive(:next).and_return(1, 2, 3)

iterator.next  # => 1
iterator.next  # => 2
iterator.next  # => 3
iterator.next  # => 3  ← returns last value again, not nil or error!
iterator.next  # => 3  ← same

# This can mask bugs where code calls a method more times than expected

# If you want it to raise after exhaustion:
allow(iterator).to receive(:next)
  .and_return(1, 2, 3)
# At call 4+: still returns 3, not StopIteration

# Better for testing exhaustion explicitly:
allow(iterator).to receive(:next).and_return(1, 2)
allow(iterator).to receive(:next).and_raise(StopIteration)
# But and_return and and_raise can't be directly chained this way

# Correct way to simulate exhaustion:
call_count = 0
allow(iterator).to receive(:next) do
  call_count += 1
  raise StopIteration if call_count > 3
  call_count
end
```

---

## Trap 9: Test Order Dependency — `before(:all)` Instance Variables Persist

**The trap:** Instance variables set in `before(:all)` persist across all tests. But `let` variables defined in a `before(:all)` context are re-evaluated per test, causing confusing mixtures.

```ruby
RSpec.describe OrderProcessor do
  before(:all) do
    @shared_queue = []  # shared array — mutations persist
  end

  it "adds items to queue" do
    @shared_queue << "item1"
    expect(@shared_queue).to include("item1")
  end

  it "starts with an empty queue" do
    # FAILS: @shared_queue still contains "item1" from previous test
    expect(@shared_queue).to be_empty
  end

  # Fix: use before(:each) for mutable state
  before(:each) do
    @fresh_queue = []  # fresh each time
  end

  it "starts fresh" do
    expect(@fresh_queue).to be_empty  # always passes
  end
end
```

---

## Trap 10: Verifying Doubles with `nil` Responses Hide Missing Method Errors

**The trap:** When you stub a method with `and_return(nil)` on an `instance_double`, the spec passes even if your code never uses the return value. But if the real method actually returns something critical, the integration will fail.

```ruby
# Overly permissive stub — hides that the code ignores an important return value
user = instance_double(User)
allow(user).to receive(:validate!).and_return(nil)

service.process(user)  # validate! is called, nil returned and ignored
# Test passes — but the real User#validate! returns true/false!
# If code should check the return value, this test doesn't catch the bug

# Better: stub with realistic return values and test behavior based on them
allow(user).to receive(:validate!).and_return(false)
expect { service.process(user) }.to raise_error(ValidationError)

allow(user).to receive(:validate!).and_return(true)
expect { service.process(user) }.not_to raise_error
```

---

## Trap 11: Factory Associations Can Cause N+1 Object Creation

**The trap:** Factory Bot associations eagerly create the full object graph. `create(:order)` may create a `:user`, which creates an `:address`, which creates a `:country`. For tests checking counts, this inflates the numbers.

```ruby
FactoryBot.define do
  factory :order do
    association :user  # creates a User
    association :product  # creates a Product (which creates a Category, etc.)
  end
end

RSpec.describe OrderQuery do
  let!(:orders) { create_list(:order, 5) }  # creates 5 orders + 5 users + 5 products + etc.

  it "returns all orders" do
    expect(OrderQuery.new.all.count).to eq(5)  # works
  end

  it "user count is correct" do
    # TRAP: there are 5 extra users (one per order factory)
    expect(User.count).to eq(5)  # might fail if other tests created users
  end

  # Fix: use build_stubbed for unit tests, explicit factory control for integration
  # Use traits to avoid unnecessary associations:
  factory :order do
    trait :without_associations do
      user { nil }
      product { nil }
    end
  end
end
```

---

## Trap 12: Matching Arguments with `anything` vs `hash_including` vs Exact Match

**The trap:** Using `anything` for hash arguments lets through incorrect keys/values. Using exact match is too brittle when the hash has extra keys. `hash_including` is the right tool.

```ruby
payment = instance_double(PaymentGateway)

# OVERLY PERMISSIVE: anything accepts wrong hash too
allow(payment).to receive(:charge).with(anything)
payment.charge(wrong_key: "wrong_value")  # passes — any arg accepted

# OVERLY BRITTLE: exact match fails if hash has extra keys
allow(payment).to receive(:charge).with(
  amount: 100, currency: "usd"
)
payment.charge(amount: 100, currency: "usd", idempotency_key: "abc123")
# => fails because hash has extra key

# CORRECT: hash_including checks required keys, ignores extras
allow(payment).to receive(:charge).with(
  hash_including(amount: 100, currency: "usd")
)
payment.charge(amount: 100, currency: "usd", idempotency_key: "abc123")  # passes!

# For arrays:
allow(list).to receive(:add).with(array_including(1, 2))

# For nested structures:
allow(api).to receive(:post).with(
  hash_including(
    body: hash_including(user_id: anything)
  )
)
```
