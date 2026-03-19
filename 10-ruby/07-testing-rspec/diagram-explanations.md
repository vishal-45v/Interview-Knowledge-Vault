# Chapter 07: Testing with RSpec — Diagram Explanations

## Diagram 1: RSpec Execution Order

```
Full test run execution order:

before(:suite)
│   (runs once before all spec files)
│   └── Database setup, SimpleCov start, seeding
│
├── SPEC FILE 1 (e.g., user_spec.rb)
│   │
│   ├── before(:all) for describe "User"
│   │   └── runs once per describe group
│   │
│   ├── EXAMPLE 1: "has a name"
│   │   ├── before(:each)            ← runs before this example
│   │   ├── let values               ← evaluated lazily when first accessed
│   │   ├── it block executes
│   │   └── after(:each)             ← runs after this example
│   │
│   ├── EXAMPLE 2: "validates email"
│   │   ├── before(:each)            ← fresh run (not shared with Example 1)
│   │   ├── let values               ← re-evaluated (not memoized between examples)
│   │   ├── it block executes
│   │   └── after(:each)
│   │
│   └── after(:all) for describe "User"
│       └── runs once after all examples in the group
│
├── SPEC FILE 2 (e.g., order_spec.rb)
│   └── (same pattern)
│
└── after(:suite)
    └── DatabaseCleaner final clean, coverage report generation

┌────────────────────────────────────────────────────────────────┐
│  SCOPE OF MEMOIZATION:                                         │
│                                                                │
│  let(:user) { User.new }                                       │
│                                                                │
│  Example 1:                                                    │
│    user.name      ← evaluates block, memoized as user1        │
│    user.email     ← returns same user1                        │
│  Example 2:                                                    │
│    user.name      ← NEW evaluation, creates user2             │
│    user.email     ← returns same user2 (memoized in Example2) │
│                                                                │
│  let values are memoized PER EXAMPLE, not across examples.    │
└────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Double / Spy / Verifying Double Comparison

```
TYPE OF TEST DOUBLE — DECISION TREE

Start: What kind of stand-in do I need?
         │
         ▼
   ┌─────────────────────────────────────────┐
   │  Does the real class EXIST in my code?   │
   └─────────────────────────────────────────┘
         │                    │
        YES                   NO
         │                    │
         ▼                    ▼
   ┌──────────────┐     ┌──────────────┐
   │ instance_    │     │   double     │
   │ double(Real  │     │  (generic,   │
   │ Class)       │     │  no verify)  │
   │              │     └──────────────┘
   │ VERIFIES:    │
   │ - method     │
   │   exists     │
   │ - arity      │
   │   matches    │
   └──────┬───────┘
          │
   Do I want to:
   ASSERT BEFORE call  OR  ASSERT AFTER call?
          │                         │
      BEFORE                      AFTER
          │                         │
          ▼                         ▼
   expect(dbl).to             spy or instance_spy
   receive(:method)           + have_received
   (set up BEFORE action)     (assert AFTER action)


COMPARISON TABLE:

  ┌──────────────────┬─────────────┬──────────┬─────────────────┐
  │ Type             │ Verifies    │ Fail if  │ Use When        │
  │                  │ Interface?  │ Not Used?│                 │
  ├──────────────────┼─────────────┼──────────┼─────────────────┤
  │ double           │ No          │ Depends  │ No real class   │
  │ instance_double  │ Yes         │ Depends  │ Class exists    │
  │ class_double     │ Yes (class) │ Depends  │ Class methods   │
  │ spy              │ No          │ No       │ Assert after    │
  │ instance_spy     │ Yes         │ No       │ Verified+after  │
  └──────────────────┴─────────────┴──────────┴─────────────────┘

  "Fail if Not Used?" = YES if you used expect().to receive()
                        NO  if you used allow().to receive()
```

---

## Diagram 3: `allow` vs `expect` Timing Diagram

```
TEST TIMELINE:

  ── allow ──────────────────────────────────────────────────────

  allow(obj).to receive(:method)
          │
          │   set up BEFORE
          │   (no assertion made)
          ▼
  ┌───────────────────────────────┐
  │   execute subject code        │
  │   obj.method is called        │ → returns stubbed value
  └───────────────────────────────┘
          │
          ▼
  expect(obj).to have_received(:method)   ← assert AFTER
          │
  (test passes if method was called,
   but test would ALSO PASS if method wasn't called without have_received)


  ── expect ─────────────────────────────────────────────────────

  expect(obj).to receive(:method)
          │
          │   sets up EXPECTATION before
          │   (test WILL FAIL if method is never called)
          ▼
  ┌───────────────────────────────┐
  │   execute subject code        │
  │   obj.method is called        │ → satisfies expectation
  └───────────────────────────────┘
          │
          ▼
  RSpec verifies at end of example:
    Was expect().to receive() satisfied? YES → pass, NO → fail


  SIDE BY SIDE:

  allow (permissive):           expect (assertive):
  ┌────────────────────┐        ┌─────────────────────┐
  │ SETUP (allow)      │        │ SETUP (expect)       │
  │ ↓                  │        │ ↓                    │
  │ RUN CODE           │        │ RUN CODE             │
  │ ↓                  │        │ ↓                    │
  │ ASSERT (optional)  │        │ AUTO-ASSERT at end   │
  └────────────────────┘        └─────────────────────┘
```

---

## Diagram 4: Factory Bot Object States

```
FACTORY BOT — OBJECT PERSISTENCE STATES

  FactoryBot.define do
    factory :user do
      name  { "Alice" }
      email { "alice@example.com" }
    end
  end

  ┌──────────────────────────────────────────────────────────────┐
  │   build(:user)                                               │
  │   ┌─────────────────┐                                        │
  │   │  User object     │   ← attributes set                   │
  │   │  id: nil         │   ← no ID assigned                   │
  │   │  persisted?: NO  │   ← not in database                  │
  │   └─────────────────┘                                        │
  │   Speed: ⚡⚡⚡ (no DB hit)                                   │
  │   Use: unit tests, no associations needed                    │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │   create(:user)                                              │
  │   ┌─────────────────┐      ┌──────────────────┐             │
  │   │  User object     │ ───→ │   DATABASE        │            │
  │   │  id: 42          │      │   users table     │            │
  │   │  persisted?: YES │      │   id=42 stored    │            │
  │   └─────────────────┘      └──────────────────┘             │
  │   Speed: ⚡ (DB write)                                        │
  │   Use: integration tests, queries, associations              │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │   build_stubbed(:user)                                       │
  │   ┌─────────────────┐                                        │
  │   │  User object     │   ← attributes set                   │
  │   │  id: 1001        │   ← FAKE ID (no real DB sequence)    │
  │   │  persisted?: YES │   ← STUBBED to return true           │
  │   │  save raises!    │   ← DB writes are blocked            │
  │   └─────────────────┘                                        │
  │   Speed: ⚡⚡⚡ (no DB hit, but looks persisted)              │
  │   Use: unit tests needing persisted-looking objects          │
  └──────────────────────────────────────────────────────────────┘

  TRAITS — modifier packages:

  factory :user do
    name { "Default User" }

    trait :admin do
      name { "Admin" }
      role { :admin }
    end

    trait :with_posts do
      after(:create) { |u| create_list(:post, 3, user: u) }
    end
  end

  create(:user, :admin, :with_posts)
  #   → admin user + 3 posts created
```

---

## Diagram 5: RSpec Matcher Decision Map

```
What are you testing?

              ┌──────────────────────────────────────────┐
              │         WHAT TO ASSERT?                  │
              └──────────────────────────────────────────┘
                              │
     ┌────────────────────────┼────────────────────────┐
     ▼                        ▼                        ▼
  VALUES                  BEHAVIOR                STRUCTURE
(equality)             (side effects)          (interface/shape)
     │                        │                        │
     ▼                        ▼                        ▼
  eq(val)              change{expr}            respond_to(:meth)
  eql(val)             raise_error(E)          have_attributes(k:v)
  equal(obj)           output(str)             be_a(ClassName)
  be_nil               throw_symbol             include(key)
  be_truthy/falsy      have_enqueued_job        match_array(arr)
  be > x               have_received            contain_exactly
  be_between           have_enqueued_mail       be_kind_of

  EQUALITY PRECISION:

  eq    → uses ==      (value equality, loose typing)
  eql   → uses eql?   (value equality, strict typing)
  equal → uses equal? (object identity / same object_id)

  "hello" == "hello"    → true   (eq passes)
  "hello".eql?("hello") → true   (eql passes)
  "hello".equal?("hello") → false (equal fails — different objects)

  1 == 1.0              → true   (eq passes)
  1.eql?(1.0)           → false  (eql fails — different types)

  OBJECT MATCHING:

  expect(user).to have_attributes(name: "Alice", age: 30)
     ↓ equivalent to:
  expect(user.name).to eq("Alice")
  expect(user.age).to  eq(30)

  expect(response).to match(hash_including("status" => 200))
     ↓ checks hash has at least these keys with these values
```

---

## Diagram 6: VCR Cassette Lifecycle

```
VCR CASSETTE WORKFLOW

  FIRST RUN (cassette doesn't exist):

  Test code                 VCR                  Real API
  ─────────────────────────────────────────────────────────
  api.get("/users")  ──→  VCR: no cassette
                          VCR: allow real call
                    ──────────────────────────→  GET /users
                    ←──────────────────────────  200 {"users": [...]}
                          VCR: record to cassette
                          ← cassette written to disk →
  result = {...}    ←──  return response

  SUBSEQUENT RUNS (cassette exists):

  Test code                 VCR               Cassette File
  ─────────────────────────────────────────────────────────
  api.get("/users")  ──→  VCR: cassette found!
                          VCR: intercept HTTP call
                    ──→  read cassette file
                    ←──  replay recorded response
  result = {...}    ←──  return replayed response

  (Real API is NEVER called — network not needed)

  CASSETTE FILE STRUCTURE (YAML):

  spec/fixtures/vcr_cassettes/api/get_users.yml
  ┌───────────────────────────────────────────────┐
  │  http_interactions:                           │
  │  - request:                                   │
  │      method: get                              │
  │      uri: https://api.example.com/users       │
  │      headers:                                 │
  │        Authorization: Bearer <API_KEY>        │  ← filtered
  │    response:                                  │
  │      status: { code: 200 }                   │
  │      body: { string: '{"users":[...]}' }      │
  │  recorded_at: "2026-03-17T00:00:00Z"          │
  └───────────────────────────────────────────────┘

  CONFIGURATION:

  VCR.configure do |config|
    config.filter_sensitive_data('<API_KEY>') { ENV['API_KEY'] }
    config.record = :new_episodes  ← record new, replay old
    # record modes:
    # :none       → only playback, fail if no cassette
    # :new_episodes → record new requests, replay existing
    # :all        → re-record everything on each run
    # :once       → record once, replay forever
  end
```
