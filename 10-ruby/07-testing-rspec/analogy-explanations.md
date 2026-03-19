# Chapter 07: Testing with RSpec — Analogy Explanations

## 1. `describe` / `context` / `it` — A Filing Cabinet System

Think of your test suite as a filing cabinet for quality assurance reports.

`describe` is a drawer labeled with the subject ("PaymentService" or "#charge method"). `context` is a folder within that drawer representing a condition ("when the card is valid", "when the network is down"). `it` is a single report inside the folder — one specific assertion about one specific scenario.

This hierarchy means: "In the PaymentService drawer, in the 'card is valid' folder, this one report says: charges successfully and returns a confirmation ID."

```ruby
RSpec.describe PaymentService do        # drawer
  describe "#charge" do                 # sub-drawer
    context "when card is valid" do     # folder
      it "returns confirmation" do      # single report
        # one fact, one assertion
      end
    end
  end
end
```

---

## 2. `let` vs `let!` — A Lazy Chef vs a Prep Cook

`let` is a lazy chef. You describe a recipe (the block), but the chef doesn't cook anything until someone actually orders that dish in a specific test. If no one orders it, no cooking happens — no mess, no waste.

`let!` is a prep cook. The mise en place is done before service starts, regardless of whether that ingredient gets used tonight. Things are ready and waiting on the counter before the first order arrives.

```ruby
let(:user)     { User.create! }   # lazy chef: only creates when the test orders 'user'
let!(:seeded_data) { seed_db() }  # prep cook: runs before every single test
```

The lazy chef saves time and database writes. The prep cook ensures data exists for tests that don't explicitly reference it but rely on it being there (e.g., a `User.count` test that needs users in the DB).

---

## 3. `double` vs `instance_double` — A Prop Weapon vs a Real Training Weapon

In a movie fight scene, prop weapons (plain `double`) look realistic but aren't checked against real weapon specifications. An actor can hold a "sword" with the wrong grip, wrong weight, wrong blade length — the prop department doesn't enforce real weapon laws.

`instance_double` is a training weapon that exactly matches the real weapon's interface. The grip is correct, the weight distribution is accurate, the blade dimensions are checked. If the real sword has a 90cm blade (specific method with specific arity), the training weapon must match — you can't train with a 200cm blade and claim it's the same weapon.

The payoff: when the real weapon changes (a method is renamed or its signature changes), the training weapon breaks too, alerting you immediately.

```ruby
# Prop weapon — accepts anything, no verification
user = double("User", nme: "Alice")   # typo "nme" — prop doesn't care

# Training weapon — verified against real class
user = instance_double(User, nme: "Alice")  # raises: User has no #nme method
```

---

## 4. `allow` vs `expect` — Permitting vs Requiring

`allow` is a permission slip. "You ARE ALLOWED to use the restroom during class." If the student never goes, that's fine — no one checks or cares.

`expect` is a mandatory attendance sheet. "You ARE REQUIRED to attend the meeting." If the meeting ends without a check-in, someone follows up and it's considered a failure.

```ruby
allow(mailer).to receive(:send)   # permission: it CAN be called (test won't fail if it isn't)
expect(mailer).to receive(:send)  # requirement: it MUST be called (test fails if it isn't)
```

In tests: use `allow` when you need to control return values for collaborators. Use `expect` when you're asserting that a specific collaboration happened.

---

## 5. `spy` — A Security Camera System

A spy object is like a security camera that records everything without interfering. When someone passes through, the camera records it silently. Afterward, you can review the footage to see what happened.

This is different from a guard (`expect`) who would stop you and demand to see your pass immediately. The camera lets everything through and you check the recording afterward.

```ruby
audit_spy = spy("AuditService")

# Everything passes through (no failures during the call)
service.process(data)   # audit_spy.log(:processed) is recorded silently

# Review the footage afterward
expect(audit_spy).to have_received(:log).with(:processed)
```

Spies are ideal when the order of operations matters more than preventing calls, or when you want to write the assertion after the action rather than before it.

---

## 6. `before(:each)` vs `before(:all)` — Fresh Start vs Inherited Mess

`before(:each)` is like renting a hotel room that's cleaned between every guest. Each new guest (test) gets fresh towels, a made bed, no trace of the previous occupant.

`before(:all)` is like a co-working space. The setup (table arrangement, whiteboards) is done once in the morning and shared by everyone throughout the day. If one person wipes the whiteboard, everyone after them sees blank boards — even if they were relying on what was written there.

```ruby
before(:each) { @cart = ShoppingCart.new }  # fresh cart per test — no surprises
before(:all)  { @cart = ShoppingCart.new }  # shared cart — one test's items remain for the next
```

---

## 7. `shared_examples` — Standard Form Templates

`shared_examples` are like form templates at a government office. Whether you're registering a car, boat, or motorcycle, certain fields appear on every form: owner name, registration date, identification number.

Instead of rewriting those common fields on every specialized form, you stamp them from the template. Every vehicle type shares the base form but adds its own vehicle-specific sections.

```ruby
RSpec.shared_examples "a registerable entity" do
  it { is_expected.to respond_to(:register!) }
  it { is_expected.to respond_to(:registration_number) }
  it { is_expected.to have_db_column(:registered_at) }
end

RSpec.describe Car,        type: :model do
  it_behaves_like "a registerable entity"
  # Plus Car-specific tests
end

RSpec.describe Motorcycle, type: :model do
  it_behaves_like "a registerable entity"
  # Plus Motorcycle-specific tests
end
```

---

## 8. `VCR` — A Music Recording Studio

VCR works like a recording studio's session. The first time you record, the musicians actually play live (real HTTP calls are made). That session is captured and burned to tape (cassette file on disk).

For every future performance, instead of flying the musicians back in (making real API calls), the studio plays the tape. The audience (your tests) can't tell the difference — they hear the same exact performance each time.

```ruby
VCR.use_cassette("stripe/charge_success") do
  # First run: real Stripe API call recorded to cassette file
  # Subsequent runs: cassette plays back the recorded response
  result = stripe.charge(amount: 100, token: "tok_visa")
end
```

The tape (cassette) can be committed to git, so your team has consistent, realistic test responses without everyone needing API keys or network access.

---

## 9. Factory Bot — A Blueprint + Assembly Line

Factory Bot is like a car manufacturer's system. A `factory` is the blueprint — it defines the default specifications for a model (engine size, color, number of doors).

`build` is the test-drive car that comes off the line but never gets registered (not persisted to the database).

`create` is the fully registered car — licensed plates assigned, in the DMV database, ready for the road.

`build_stubbed` is a showroom dummy — looks exactly like a registered car (has all the right fields including a fake ID number), but it's not in any database and can't actually be driven anywhere. Perfect when you need the appearance of a real record but not the database overhead.

`traits` are option packages — "Sport Package" adds sport wheels and performance engine. Mix and match: `create(:car, :sport, :red, :convertible)`.

---

## 10. `SimpleCov` — A Thermal Imaging Camera for Code

Line coverage is like a thermal camera scanning your codebase after tests run. Code that got executed glows warm (green). Code that never ran is cold and dark (red).

A thermal scan can tell you which rooms were entered, but it can't tell you whether the rooms were used *correctly*. Someone could run through every room without doing anything useful — 100% line coverage doesn't mean 100% correctness.

Branch coverage adds another dimension: it's like checking whether someone walked through BOTH the left door AND the right door in each fork in the corridor. A line with `if user.admin?` requires tests where `user.admin?` is both `true` and `false` to reach 100% branch coverage.

```ruby
# Line covered (both branches not tested):
result = user.admin? ? :full_access : :restricted  # line hit ✓

# Branch coverage requires both:
#   Test 1: user.admin? == true  (full_access path)
#   Test 2: user.admin? == false (restricted path)
```

The real value: coverage shows you where you haven't been, not whether where you went was correct.
