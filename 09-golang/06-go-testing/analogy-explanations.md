# Go Testing — Analogy Explanations

## Table-Driven Tests: A Checklist for a Calculator

Imagine you are a quality engineer at a calculator factory. Your job is to verify every calculator works correctly before shipping. You could write a separate test procedure for each button combination: "Test Procedure #1: Press 2, press +, press 3, expect 5." Then "Test Procedure #2: Press -1, press +, press -2, expect -3." Then one for each of the 40 other cases.

Instead, you create a **test matrix** — a single spreadsheet with columns: Input A, Input B, Operation, Expected Result. You run the same procedure down every row. Adding a new test case means adding one row to the spreadsheet.

Table-driven tests in Go are that spreadsheet:

```go
tests := []struct {
    a, b int
    op   string
    want int
}{
    {2, 3, "+", 5},
    {-1, -2, "+", -3},
    {10, 3, "-", 7},
    // Adding a new case: add one line
    {0, 0, "+", 0},
}
for _, tc := range tests {
    t.Run(tc.op, func(t *testing.T) {
        // Same procedure for every row
        got := calculate(tc.a, tc.b, tc.op)
        if got != tc.want { t.Errorf(...) }
    })
}
```

The power: reviewers see your complete test coverage at a glance. Test logic is written once. Adding a new edge case is one line. It is the software equivalent of a well-organized QA checklist.

---

## Benchmark: Timing How Fast a Chef Can Chop Vegetables

You want to compare two chefs (two implementations). You don't time them once — one chop is too fast to measure accurately and luck plays too big a role. Instead, you say: "Chop for exactly 5 minutes. We'll count how many vegetables you chop."

Then you compute: vegetables per second. That's a stable, meaningful metric.

`b.N` is the testing framework's way of saying "do it N times." The framework keeps doubling N until the total time is stable enough to be meaningful. Then it reports nanoseconds per operation.

```go
func BenchmarkChopVegetables(b *testing.B) {
    // Setup (not timed)
    knife := getKnife()
    vegetables := loadVegetables()
    b.ResetTimer() // don't count setup time

    for i := 0; i < b.N; i++ { // framework decides N
        chop(knife, vegetables[i%len(vegetables)])
    }
}
// Output: BenchmarkChopVegetables-8    500000    2500 ns/op
// = chef can chop 400,000 vegetables per second
```

`b.ResetTimer()` is the equivalent of "ready... go!" — you don't count the time the chef spent walking to the counter and picking up the knife. You start the stopwatch when they actually begin chopping.

`b.ReportAllocs()` is watching whether the chef asks for a new cutting board for every vegetable (wasteful allocations) or reuses the same board (efficient, zero allocations).

---

## httptest: A Fake Restaurant for Practice Waiters

Imagine you are training waiters. You want to test whether they can take an order, communicate it to the kitchen, and return the right food. But you don't want to use a real kitchen with real food — it's expensive, slow, and you can't control what the kitchen produces to test specific scenarios.

So you build a **practice restaurant**: a simulated kitchen that you can program. Tell it "when you receive order #5 (the salmon), return this specific plate." Now you can test every scenario: what does the waiter do when the kitchen returns the wrong dish? What when the kitchen is closed (503)? What when the kitchen takes too long?

`httptest.NewServer` is your practice restaurant. You program exactly what responses it will give, and then you train your HTTP client (waiter) against it:

```go
// The practice kitchen — programmed to respond predictably
kitchen := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path == "/order/5" {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"dish":"salmon","cooked":true}`))
    } else {
        w.WriteHeader(http.StatusNotFound)
    }
}))
defer kitchen.Close()

// The waiter — train them against the practice kitchen
waiter := NewOrderClient(kitchen.URL)
dish, err := waiter.PlaceOrder(5)
// Test that the waiter handles the response correctly
```

The advantage: no network latency, no flakiness, full control over every response, including error scenarios that would be very hard to trigger in a real system.

---

## Fuzz Testing: A Monkey Randomly Pressing Buttons to Find Bugs

You've built a music synthesizer app with hundreds of buttons, knobs, and sliders. You've written tests for all the documented combinations. But a curious monkey climbs on your synthesizer and starts randomly pressing combinations you never thought of. After 30 seconds, the synthesizer produces a terrible screeching noise and then crashes.

The monkey found a bug you missed — a combination of settings that your test suite never reached because it was too unusual to think of manually.

Fuzz testing is the digital monkey. It starts from your known test cases (the corpus — your carefully documented combinations), then **mutates** them: flips bits, swaps characters, inserts random bytes, combines parts of different inputs. It runs millions of combinations per second, guided by code coverage (if pressing button A unlocks a new path, press button A more often).

```go
func FuzzSynth(f *testing.F) {
    f.Add("C4", "major", 120) // your known good inputs
    f.Add("A3", "minor", 80)

    f.Fuzz(func(t *testing.T, note string, scale string, bpm int) {
        // The fuzzer will call this with mutations of the above
        // AND with completely new combinations it generates
        result := synth.Play(note, scale, bpm)
        if result == nil {
            t.Fatal("Play returned nil — should always return something")
        }
    })
}
```

When the monkey crashes the app, the fuzzer saves the exact combination of inputs that caused the crash to `testdata/fuzz/`. From that point on, every `go test` run will replay that combination as a regression test — the monkey's findings become a permanent part of your test suite.

---

## Race Detector: A Referee Watching Two Players to Catch Cheating

Two players are sharing a game board (a shared variable). The rules say they must take turns. But sometimes, when you're not watching, one player moves a piece and the other player moves it at the exact same moment — chaos ensues.

In a real game, you'd notice immediately: pieces are in impossible positions. In software, concurrent writes to shared memory produce subtler corruption that only shows up rarely, under specific timing conditions.

The `-race` flag adds a referee to every memory access. The referee has a notebook tracking who last touched each variable. Whenever two goroutines access the same variable and at least one is a write, without any synchronization between them, the referee blows the whistle:

```
WARNING: DATA RACE
Write at 0x00c0000b6008 by goroutine 7:
  main.(*Counter).Inc()
      /home/user/main.go:15

Previous read at 0x00c0000b6008 by goroutine 8:
  main.(*Counter).Value()
      /home/user/main.go:22
```

The referee tells you exactly which line wrote, which line read, and the goroutine stacks for both — enough information to fix the synchronization.

The caveat: the race detector only catches races that **actually happen** during the test run. It cannot predict races. That's why your tests need to exercise concurrent code paths with enough goroutines that the races are likely to be triggered. `-race` adds ~5-10x overhead, which is why you use it in CI (where time is acceptable to trade for correctness) but not in production.

---

## `t.Helper()`: Telling the Referee Which Player Fouled

When a basketball player commits a foul, the referee points to the player who fouled — not to the coach who told them to play aggressively. The useful information is who actually did something wrong.

In Go tests, when a test helper function calls `t.Error()` or `t.Fatal()`, without `t.Helper()` the failure is reported at the line inside the helper. You see the referee pointing at the assistant coach (the helper) instead of at the player (the test that called the helper with wrong inputs).

```go
// Without t.Helper() — referee points to assistant coach
func checkEqual(t *testing.T, got, want int) {
    if got != want {
        t.Errorf("got %d, want %d", got, want) // reported here — line 5 inside helper
    }
}

// With t.Helper() — referee points to the actual foul
func checkEqual(t *testing.T, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d, want %d", got, want) // reported at the CALLER's line
    }
}

func TestSomething(t *testing.T) {
    checkEqual(t, compute(), 42) // failure reported HERE with t.Helper()
}
```

`t.Helper()` tells the testing framework: "skip over this function when reporting failure locations — go up the call stack to whoever called me."

---

## `TestMain`: The Building Manager Who Opens and Locks Up

Individual employees can handle their own office setup (bring in their coffee mug, set their chair height — this is `t.Cleanup` per test). But some tasks happen once per day for the whole building: the security guard opens the building at 8am, the janitor locks it at 10pm.

`TestMain` is the building manager. It runs once before any test in the package (opens the building — starts a database, a container, loads test fixtures) and once after all tests complete (locks up — closes connections, deletes test data).

```go
func TestMain(m *testing.M) {
    // 8am: open the building
    db := startTestDatabase()
    runMigrations(db)

    // Let everyone work
    exitCode := m.Run()

    // 10pm: lock up
    db.Close()
    cleanupContainers()

    os.Exit(exitCode)
}
```

The critical rule: you must call `os.Exit(m.Run())`. If you forget the `os.Exit`, the program exits with 0 (success) regardless of test results — like the building manager claiming everyone left safely even if some employees are still working. If you forget `m.Run()`, the tests never run — like the building manager opening and closing without letting anyone in.
