# Chapter 09 — Testing Rails: Diagram Explanations

---

## Diagram 1: Rails Testing Pyramid

```
                        /\
                       /  \
                      / SY \      System Tests (Capybara + Browser)
                     / STEM \     - Full browser interaction
                    /________\    - JavaScript execution
                   /          \   - Slowest (seconds per test)
                  /  REQUEST   \  Request Specs / Integration Tests
                 /   SPECS      \ - Real HTTP, full middleware stack
                /________________\ - Moderate speed
               /                  \
              /   UNIT TESTS        \
             /  (Model / Service /   \
            /   Policy / Job tests)   \
           /____________________________\
           - No HTTP, no browser
           - Direct method calls
           - Fastest (milliseconds)
           - Most numerous

Recommended distribution:
  Unit tests:    70%  (model, service, policy, job, serializer)
  Request specs: 20%  (controller behavior, auth, format)
  System tests:  10%  (critical user flows, JS interactions)
```

---

## Diagram 2: RSpec Test Anatomy

```ruby
RSpec.describe PostsController, type: :request do
#              ─────────────────            ──────────────
#              subject under test           metadata (enables helpers)

  subject { described_class }  # → PostsController

  let(:user)  { create(:user) }          # lazy: created when first referenced
  let!(:post) { create(:post, user: user) }  # eager: created before each example

  before(:each) do                       # runs before every `it` in this group
    sign_in user
  end

  describe "#index" do                   # organizing group (describes a method)
    context "as authenticated user" do   # organizing group (describes a scenario)

      it "returns 200" do                # one example = one behavior
        get posts_path
        expect(response).to have_http_status(:ok)  # assertion
      end

      it "returns posts in order" do
        get posts_path
        json = JSON.parse(response.body)
        expect(json['data'].map { |p| p['id'] }).to eq(Post.order(created_at: :desc).pluck(:id))
      end
    end

    context "as unauthenticated user" do
      before { sign_out user }

      it "redirects to sign in" do
        get posts_path
        expect(response).to redirect_to(sign_in_path)
      end
    end
  end
end
```

---

## Diagram 3: FactoryBot Object Creation Comparison

```
Method              DB Write?   Callbacks?   Associations?   Use Case
──────────────      ─────────   ──────────   ─────────────   ─────────────────
build(:user)        No          before_*     Not persisted   Quick object creation
                                             only            Unit tests, form tests

build_stubbed(:user) No (fake   None         Stubbed         Pure unit tests,
                    persisted)               associations    policy tests, serializers

create(:user)       Yes         All          Persisted       Integration/request
                                callbacks    in DB           specs, system tests

create_list(:user,  Yes (×n)    All          Persisted       Multiple records needed
  3)                            callbacks

attributes_for(:user) No        None         None (hash)     Testing strong params,
                                                             form attribute filling

─────────────────────────────────────────────────────────────────────

PERFORMANCE IMPACT (relative times):
  build_stubbed: ~0.1ms    (no DB)
  build:         ~0.2ms    (no DB, callbacks)
  create:        ~5-20ms   (DB write, callbacks)
  create_list(5): ~25-100ms (5 × create cost)

Rule: default to build_stubbed, upgrade to create only when DB is needed
```

---

## Diagram 4: Database Cleaning Strategies

```
:transaction (default, fastest):

  Test A begins
  ┌──────────────────────────────────────────────┐
  │ BEGIN TRANSACTION (test wrapper)             │
  │   create(:user) → INSERT users (uncommitted) │
  │   expect(User.count).to eq(1)  ← SEES 1     │
  │                                              │
  │   Browser (Capybara system test):            │
  │   GET /users → SELECT FROM users            │
  │   → Returns 0 rows! (uncommitted data!)      │
  │   → Test fails!                              │
  │ ROLLBACK → user removed                      │
  └──────────────────────────────────────────────┘

  ⚠ Use :transaction for request/unit specs
  ✗ Doesn't work for system tests (multi-thread)

─────────────────────────────────────────────────────────────────────

:truncation (system tests):

  Test A begins
    create(:user) → INSERT users (COMMITTED to DB)
    browser visits page → sees the user ✓
  Test A ends
    TRUNCATE users, posts, comments, ... (DELETE all rows)
  Test B begins — clean slate ✓

  ⚠ Slower (TRUNCATE is expensive for large schemas)
  ✓ Required for system tests

─────────────────────────────────────────────────────────────────────

DatabaseCleaner configuration:
  RSpec.configure do |config|
    config.before(:suite)  { DatabaseCleaner.clean_with(:truncation) }
    config.before(:each)   { DatabaseCleaner.strategy = :transaction  }
    config.before(:each, type: :system) do
      DatabaseCleaner.strategy = :truncation
    end
    config.before(:each)   { DatabaseCleaner.start }
    config.after(:each)    { DatabaseCleaner.clean }
  end
```

---

## Diagram 5: Test Double Types

```
STUB — returns canned values:
  allow(stripe_client).to receive(:charge).and_return(mock_response)
  # No assertion: whether it was called or not doesn't affect test outcome
  # Purpose: isolate code from external dependency

MOCK — verifies expected interactions:
  expect(mailer).to receive(:welcome_email).with(user).once
  # Must be called exactly as specified
  # Fails if: not called, called wrong number of times, wrong arguments
  # Purpose: verify collaboration between objects

SPY — records calls for later assertion:
  spy = instance_spy(StripeClient)
  PaymentService.new(spy).charge(100)
  expect(spy).to have_received(:charge).with(100)
  # Assert after the fact
  # Purpose: flexible verification without pre-setup ordering

PARTIAL DOUBLE — stub only specific methods:
  allow(user).to receive(:email).and_return("test@example.com")
  # Other methods still call the real implementation
  # Purpose: override one method without mocking the whole object

─────────────────────────────────────────────────────────────────────

WHEN TO USE EACH:
  Stub:  Always when isolating from external services/slow operations
  Mock:  When verifying side effects that can't be asserted via state
  Spy:   When you need to verify calls but want to let action run first
  Real:  Unit tests for the actual class, not its collaborators

WARNING SIGNS OF OVER-MOCKING:
  - Test passes but production breaks (mock doesn't match real interface)
  - Test reads like "verify implementation, not behavior"
  - Mocking methods on the class UNDER TEST (you're testing mocks, not code)
```

---
