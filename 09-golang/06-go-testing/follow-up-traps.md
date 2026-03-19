# Go Testing — Follow-Up Traps

## Trap 1: `t.Parallel()` and Loop Variable Capture

**The trap:** The classic Go closure-over-loop-variable bug is especially dangerous in parallel subtests. Without capturing the loop variable, all goroutines share the same variable and see its final value.

```go
func TestParallel_BUG(t *testing.T) {
    tests := []struct{ name, input string }{
        {"a", "input-a"},
        {"b", "input-b"},
        {"c", "input-c"},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            // BUG: tc is shared by all goroutines
            // By the time goroutines run, the loop has ended
            // All goroutines see tc = {"c", "input-c"}
            process(tc.input)
        })
    }
}

// FIX: capture loop variable before t.Parallel()
func TestParallel_CORRECT(t *testing.T) {
    tests := []struct{ name, input string }{
        {"a", "input-a"},
        {"b", "input-b"},
        {"c", "input-c"},
    }

    for _, tc := range tests {
        tc := tc // capture — creates new variable in each iteration
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            process(tc.input) // uses the captured tc, not the loop variable
        })
    }
}
```

**Note:** In Go 1.22+, the loop variable is re-declared each iteration, so `tc := tc` is no longer needed. But be aware of the Go version your codebase targets.

---

## Trap 2: `t.Fatal` in a Goroutine Panics Instead of Failing the Test

**The trap:** `t.Fatal`, `t.Error`, and `t.FailNow` must only be called from the goroutine running the test function. Calling them from a spawned goroutine causes a panic.

```go
func TestSomething(t *testing.T) {
    go func() {
        result := compute()
        if result != expected {
            t.Errorf("wrong result") // PANIC: t.Errorf called in goroutine
        }
    }()
    time.Sleep(100 * time.Millisecond) // terrible: relying on timing
}

// FIX: use a channel to communicate results back to the test goroutine
func TestSomething_CORRECT(t *testing.T) {
    resultCh := make(chan int, 1)
    go func() {
        resultCh <- compute()
    }()

    select {
    case result := <-resultCh:
        if result != expected {
            t.Errorf("wrong result: got %d, want %d", result, expected)
        }
    case <-time.After(5 * time.Second):
        t.Fatal("timed out waiting for compute")
    }
}
```

---

## Trap 3: `TestMain` — Forgetting to Call `os.Exit(m.Run())`

**The trap:** If you implement `TestMain` but forget to call `os.Exit(m.Run())`, tests either never run or always report success.

```go
// BUG: tests never run
func TestMain(m *testing.M) {
    setupDB()
    m.Run()   // runs tests but ignores exit code
    cleanupDB()
    // No os.Exit — test binary exits with 0 regardless of test failures!
}

// CORRECT
func TestMain(m *testing.M) {
    setupDB()
    code := m.Run()   // run tests, capture exit code
    cleanupDB()
    os.Exit(code)     // propagate test result
}
```

Also: if `setupDB()` fails and you call `os.Exit(1)` without calling `m.Run()`, no tests run and CI reports a failure — but you get no information about which tests would have run.

---

## Trap 4: `b.ResetTimer()` — Forgetting It When Benchmarks Have Setup Code

**The trap:** Expensive setup code before the benchmark loop inflates the benchmark results.

```go
// BUG: DB setup is included in timing
func BenchmarkQuery(b *testing.B) {
    db := setupDatabase() // takes 200ms
    defer db.Close()

    for i := 0; i < b.N; i++ {
        db.Query("SELECT 1") // takes 1ms
    }
    // Result is wrong: the 200ms setup is amortized over all iterations
    // but for b.N=1, it completely dominates
}

// CORRECT
func BenchmarkQuery(b *testing.B) {
    db := setupDatabase()
    defer db.Close()

    b.ResetTimer() // discard time spent on setup

    for i := 0; i < b.N; i++ {
        db.Query("SELECT 1")
    }
}
```

---

## Trap 5: Flaky Tests with `time.Sleep`

**The trap:** Using `time.Sleep` for synchronization is non-deterministic. On a slow CI machine, the sleep may not be long enough. On a fast machine, the sleep wastes time.

```go
// FLAKY: assumes goroutine finishes in 100ms
func TestWorker_Flaky(t *testing.T) {
    w := NewWorker()
    go w.Start()
    w.Submit("job")
    time.Sleep(100 * time.Millisecond) // might not be enough on a slow machine
    if w.ProcessedCount() != 1 {
        t.Error("job not processed")
    }
}

// RELIABLE: use a channel to signal completion
func TestWorker_Reliable(t *testing.T) {
    done := make(chan struct{})
    w := NewWorker(func(job string) {
        close(done)
    })
    go w.Start()
    w.Submit("job")

    select {
    case <-done:
        // success
    case <-time.After(5 * time.Second):
        t.Fatal("job not processed within 5 seconds")
    }
}
```

---

## Trap 6: `t.Fatal()` vs `t.Error()` — Understanding When Tests Stop

**The trap:** Using `t.Error()` when you should use `t.Fatal()`, leading to cascading failures that are confusing to debug.

```go
func TestLogin(t *testing.T) {
    session, err := Login("alice", "wrong-password")

    // BAD: using t.Error — test continues with nil session
    if err == nil {
        t.Error("expected error for wrong password")
    }
    // This line panics because session is nil!
    if session.Token != "" {
        t.Error("session should have no token")
    }

    // GOOD: using t.Fatal — test stops if err is nil
    if err == nil {
        t.Fatal("expected error for wrong password")
    }
    // Only reaches here if err != nil, so session is safe to inspect
    if session != nil {
        t.Error("expected nil session on failed login")
    }
}
```

Rule: use `t.Fatal` when the test cannot reasonably continue (e.g., nil pointer dereference would follow). Use `t.Error` to accumulate multiple independent failures.

---

## Trap 7: Test Binary Runs in the Package Directory

**The trap:** Assuming relative file paths in tests refer to the project root. `go test` changes the working directory to the package's source directory.

```go
// BUG: assumes cwd is the project root
func TestLoadConfig(t *testing.T) {
    cfg, err := LoadConfig("config/test.json") // relative path — where?
    // This works from project root but fails when run from package directory
}

// CORRECT: use testdata/ directory convention
// File: mypackage/testdata/config.json
func TestLoadConfig(t *testing.T) {
    // "testdata/" is always relative to the package directory — reliable
    cfg, err := LoadConfig("testdata/config.json")
    if err != nil {
        t.Fatalf("load config: %v", err)
    }
}

// OR: use runtime.Caller to get the test file's directory
func TestLoadConfig_WithPath(t *testing.T) {
    _, thisFile, _, _ := runtime.Caller(0)
    dir := filepath.Dir(thisFile)
    configPath := filepath.Join(dir, "testdata", "config.json")

    cfg, err := LoadConfig(configPath)
    // ...
}
```

The `testdata/` directory is the Go convention for test fixtures. `go test` ignores it for building but it's accessible by tests.

---

## Trap 8: `-count=1` to Disable Test Caching

**The trap:** Go caches test results. If you run `go test` twice with no code changes, the second run shows cached results (`(cached)` in output). This can mask timing-dependent failures.

```bash
$ go test ./...
ok  mypackage  0.5s
$ go test ./...
ok  mypackage  0.5s (cached)  ← not actually re-run!
```

In CI, you almost always want fresh test runs:

```bash
# Force re-run — do not use cached results
go test -count=1 ./...

# Cache is stored in $GOCACHE. Clear it:
go clean -testcache
```

Tests that access network, files, time, or environment variables are NOT cached (Go detects these). But pure unit tests are cached — `-count=1` ensures they always run.

---

## Trap 9: Subtests and `t.Run` — Parent Fails if Any Subtest Fails

**The trap:** Expecting parent test to pass if you `t.Skip` in a subtest, or not realizing that any subtest failure causes the parent to fail.

```go
func TestComplex(t *testing.T) {
    t.Run("case-1", func(t *testing.T) {
        // This fails
        t.Error("case 1 failed")
    })

    t.Run("case-2", func(t *testing.T) {
        // This passes
    })

    // TestComplex is FAILED because case-1 failed
    // Running just "case-2": go test -run TestComplex/case-2 → PASS
}
```

Also: a `t.Skip()` in a subtest does not stop the parent from running other subtests:

```go
func TestWithDB(t *testing.T) {
    if os.Getenv("DATABASE_URL") == "" {
        t.Skip("skipping: DATABASE_URL not set") // Skips entire test
    }

    t.Run("create", func(t *testing.T) { /* ... */ })
    t.Run("read", func(t *testing.T) { /* ... */ })
}
```

---

## Trap 10: `init()` Runs in Test Binaries

**The trap:** Package `init()` functions run when the test binary loads the package. If `init()` has side effects (connects to DB, starts goroutines, reads files), tests may fail or behave unexpectedly in test environments.

```go
// In production code — problematic in tests
func init() {
    db = mustConnectDB(os.Getenv("DATABASE_URL"))
    // In test environment, DATABASE_URL may not be set → panic
}

// Better: explicit initialization
var db *sql.DB

func MustInit() {
    db = mustConnectDB(os.Getenv("DATABASE_URL"))
}
```

In tests, you can use `TestMain` to control when (and if) initialization happens.

---

## Trap 11: `httptest.NewRecorder().Code` Defaults to 200 Even Without `WriteHeader`

**The trap:** `httptest.ResponseRecorder.Code` starts at 200. If the handler never calls `WriteHeader`, the recorded code is 200 — even if the handler returned an error early.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Handler does nothing — no WriteHeader call
}

func TestHandler(t *testing.T) {
    rr := httptest.NewRecorder()
    handler(rr, httptest.NewRequest("GET", "/", nil))

    // rr.Code == 200 even though WriteHeader was never called!
    // This is accurate — implicit 200 — but can mask missing WriteHeader calls
}
```

Be explicit in handlers and be aware in tests:

```go
// Instead of checking just the code, also check the body:
if rr.Code != http.StatusOK {
    t.Errorf("expected 200, got %d: body=%s", rr.Code, rr.Body.String())
}
```

---

## Trap 12: Mutating Slice/Map Input in Tests Without Copying

**The trap:** Test cases share a slice or map value. If the function under test mutates it, subsequent test cases (or reuses of the same test case struct) see the mutated version.

```go
var baseItems = []string{"a", "b", "c"}

tests := []struct {
    name  string
    items []string
}{
    {name: "first", items: baseItems},
    {name: "second", items: baseItems}, // same slice!
}

for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
        // If Sort mutates tc.items...
        Sort(tc.items) // mutates baseItems!
        // "second" test case now receives pre-sorted items
    })
}

// FIX: copy in each test case
for _, tc := range tests {
    tc := tc
    t.Run(tc.name, func(t *testing.T) {
        items := make([]string, len(tc.items))
        copy(items, tc.items) // work on a copy
        Sort(items)
        // tc.items is unchanged
    })
}
```
