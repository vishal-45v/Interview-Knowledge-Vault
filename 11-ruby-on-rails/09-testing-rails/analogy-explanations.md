# Chapter 09 — Testing Rails: Analogy Explanations

---

## Analogy 1: Testing Pyramid — Building Inspections

A skyscraper goes through three levels of inspection:

**Unit tests** (bottom of pyramid) = individual material tests:
- Test each beam's load capacity in isolation (before it's installed)
- Test each window's insulation factor in a lab
- Fast, cheap, run thousands per build
- Catches 80% of material defects

**Integration tests** (middle) = floor-level inspections:
- Test that the plumbing connects correctly to the water supply
- Test that electrical circuits work with the actual building wiring
- Slower, more complex, run hundreds per build
- Catches defects that only appear when components interact

**System tests** (top of pyramid) = final building walkthrough:
- Walk through the finished building like a resident would
- Test that the elevator goes to every floor, the HVAC heats the lobby
- Slowest, most expensive, run dozens per build
- Catches defects that only appear in the full integrated system

The pyramid shape: many unit, fewer integration, fewest system. Inverting it (few unit, many system) means: slow feedback, expensive to run, hard to pinpoint failures.

---

## Analogy 2: `let` vs `let!` — The Library Book Reservation

`let` is a **lazy reservation**:
- You call the library: "Reserve this book for me"
- The library says "OK, reserved"
- But the book stays on the shelf until you actually COME IN AND PICK IT UP
- If you never come to pick it up, the book was never touched

`let!` is an **eager checkout**:
- You call the library: "Please pull the book from the shelf right now"
- The library immediately retrieves it, puts it on the hold shelf
- Even if you never come to pick it up, the book was retrieved (DB record created)

In test terms:
- `let(:user) { create(:user) }` — user is created only when `user` is first referenced in the test
- `let!(:user) { create(:user) }` — user is created before the example runs, whether referenced or not

Rule: use `let` (lazy) unless you need the side effect to happen before the action under test. Use `let!` when the data must exist before the scenario starts (e.g., you're testing "what happens when a user already exists in the DB").

---

## Analogy 3: FactoryBot Traits — The Burger Builder

A burger factory has a base recipe: bun, patty, lettuce.

**Traits are add-ons:**
- `:cheese` → add cheese slice
- `:bacon` → add bacon strip
- `:double_patty` → double the patty

**Combining traits:**
- `build(:burger, :cheese, :bacon)` → base burger + cheese + bacon
- `build(:burger, :double_patty, :cheese)` → double patty + cheese

Without traits: you'd have a separate factory for every combination:
- `cheeseburger_factory`
- `bacon_cheeseburger_factory`
- `double_bacon_cheeseburger_factory`
- ... 2^n combinations for n add-ons

With traits: one base factory + n traits = flexible composition. Your test reads: `create(:user, :admin, :with_2fa)` and you immediately know: an admin user who has 2FA enabled.

---

## Analogy 4: `build_stubbed` — The Movie Prop Department

**`create(:user)`** is hiring a real actor: gets paid, has a real Social Security number, appears on the union payroll.

**`build_stubbed(:user)`** is a prop dummy: looks exactly like a person from camera distance (has an ID, says `persisted? → true`), but has no real actor behind it, no payroll, no union card.

For scenes where you need a crowd in the background: props are perfect — fast and cheap.

For scenes where the character needs to speak, react, open a real door: you need a real actor.

`build_stubbed` is for tests where you're checking "how does my code BEHAVE with this object?" (unit tests). Not for tests where the code needs to query the database: "show me all comments for this user" — the prop user isn't in the database, so there are no comments to find.

---

## Analogy 5: Stubs vs Mocks vs Spies — Security Camera Configurations

**Stub** = painting a picture of what you want to see:
- "When the security camera looks at door #3, ALWAYS show me this pre-recorded 'all clear' footage"
- You don't care how many times it's looked at, or when
- `allow(api_client).to receive(:fetch).and_return(mock_response)`

**Mock** = a security camera with a trip wire:
- "I expect door #3 to be checked exactly TWICE during this shift"
- If it's checked once or three times → alarm sounds (test fails)
- `expect(api_client).to receive(:fetch).exactly(2).times`
- Must be set up BEFORE the action (tells camera what to expect)

**Spy** = a security camera you review AFTER the shift:
- "I don't know what will happen, but record everything"
- After the shift, review: "was door #3 checked?"
- `spy = instance_spy(ApiClient)` → then `expect(spy).to have_received(:fetch)`
- Set up after the action

Rule: prefer stubs for dependency isolation, use mocks sparingly (they couple tests to implementation), use spies when you need to assert after the fact without prescribing exact call order.

---

## Analogy 6: VCR Cassettes — The Video Store Record

VCR (the gem, named after the video cassette recorder) records your HTTP interactions:

**First run** (no cassette exists):
1. Your test makes a real HTTP request to the API
2. VCR intercepts and records the full conversation (request + response) to a YAML file ("the cassette")
3. Test uses the real response

**Subsequent runs** (cassette exists):
1. Your test tries to make an HTTP request
2. VCR intercepts: "I have a recording of this!" → plays back the recorded response
3. No real network request made → tests are fast, deterministic, offline-capable

**Cassette expiry problem**: real API changes its response format → your cassette has the old format → tests still pass → production breaks. Solutions: regularly delete and re-record cassettes, or use `record: :new_episodes` to update on mismatch.

WebMock is different — it's a bouncer who blocks ALL network calls and returns whatever you told it to return (static fake, not recorded). More explicit but requires manual maintenance.

---

## Analogy 7: DatabaseCleaner Strategies — Restaurant Table Reset Methods

After each diner finishes (after each test), the restaurant resets the table:

**`:transaction` strategy** (fastest) = writing in pencil:
- Every diner's changes are made in pencil (wrapped in a DB transaction)
- After they leave: erase everything (rollback)
- Takes milliseconds
- Problem: doesn't work if the kitchen (background thread) needs to see the pencil marks — it only sees committed data

**`:truncation` strategy** (slower) = deep cleaning:
- After each diner: throw away ALL dishes, scrub the table completely, reset everything to factory state
- Every single row in every table is deleted via `TRUNCATE TABLE`
- Takes 1-5 seconds for large schemas
- Required for system tests where a real browser (separate thread) needs to see committed data

**`:deletion` strategy** = targeted cleaning:
- `DELETE FROM posts; DELETE FROM users;` — selective, row by row
- Slower than truncation for full cleanup, but can be scoped to specific tables
- Useful when you don't want to reset serial ID sequences

Rule: use `:transaction` for everything except system tests (which need real commits the browser can see).

---

## Analogy 8: Test Isolation — Separate Science Experiment Tables

Good science: each experiment gets its own clean table, its own reagents, its own results sheet. Experiment A's results don't contaminate Experiment B.

Bad testing (no isolation):
- Test A creates a user and leaves it in the database
- Test B counts users and expects 0 → gets 1 → fails

Good testing (isolation):
- Each test starts with a clean database (DatabaseCleaner)
- Each test creates exactly what it needs
- Tests can run in any order
- Tests can run in parallel

Test pollution sources:
- Shared global state (class variables, global variables)
- Database records not cleaned up
- File system files not deleted
- Redis keys not cleared
- Stub/mock not properly restored
- Time.now not unfrozen after `travel_to`

Each test should be: given this specific setup → when this happens → then expect this result. Fully self-contained, order-independent.

---
