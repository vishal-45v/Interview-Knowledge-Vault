# Go Testing — Theory Questions

## testing Package Fundamentals

**Q1. What is the convention for Go test file and function naming?**

- Test files must end with `_test.go`
- Test functions must start with `Test` followed by a name starting with an uppercase letter: `TestXxx`
- The function must accept a single `*testing.T` parameter
- Test functions must be in the same package as the code (white-box testing) or a `_test` package (black-box testing)

```go
// file: calculator_test.go
package calculator

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

// Black-box testing (package_test suffix)
package calculator_test

import (
    "testing"
    "mypackage/calculator"
)

func TestAddFromOutside(t *testing.T) {
    result := calculator.Add(2, 3)
    // Can only use exported identifiers
}
```

---

**Q2. What is `t.Error` vs `t.Fatal` vs `t.Errorf` vs `t.Fatalf`?**

| Function | Marks test failed | Stops test immediately |
|---|---|---|
| `t.Error(args...)` | Yes | No |
| `t.Errorf(format, args...)` | Yes | No |
| `t.Fatal(args...)` | Yes | Yes |
| `t.Fatalf(format, args...)` | Yes | Yes |
| `t.Log(args...)` | No | No |
| `t.Logf(format, args...)` | No | No |

```go
func TestSomething(t *testing.T) {
    conn, err := openDB()
    if err != nil {
        t.Fatalf("failed to open DB: %v", err) // stop here — no point continuing
    }
    defer conn.Close()

    result := conn.Query("SELECT 1")
    if result == nil {
        t.Error("expected non-nil result") // mark failed but keep going
    }
    if len(result) != 1 {
        t.Errorf("expected 1 result, got %d", len(result)) // keep checking other things
    }
}
```

Use `Fatal` when the test cannot meaningfully continue (failed setup). Use `Error` when you can still check other assertions.

---

**Q3. What are subtests (`t.Run`) and why are they useful?**

`t.Run` creates a named subtest that can be run independently:

```go
func TestMath(t *testing.T) {
    t.Run("addition", func(t *testing.T) {
        if Add(2, 3) != 5 {
            t.Error("expected 5")
        }
    })

    t.Run("subtraction", func(t *testing.T) {
        if Sub(5, 3) != 2 {
            t.Error("expected 2")
        }
    })

    t.Run("division by zero", func(t *testing.T) {
        _, err := Div(5, 0)
        if err == nil {
            t.Error("expected error")
        }
    })
}
```

Benefits:
- Run a specific subtest: `go test -run TestMath/addition`
- Each subtest has its own `*testing.T` for isolated `Fatal` calls
- Parent test fails if any subtest fails
- Cleaner output — failures are attributed to the specific subtest
- Perfect foundation for table-driven tests

---

**Q4. What is the table-driven test pattern? Why is it idiomatic Go?**

Table-driven tests define test cases as a slice of structs, then iterate over them:

```go
func TestIsValidEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "user@example.com", true},
        {"no at sign", "userexample.com", false},
        {"no domain", "user@", false},
        {"empty string", "", false},
        {"with dots", "first.last@sub.example.co.uk", true},
        {"with plus", "user+tag@example.com", true},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got := IsValidEmail(tc.email)
            if got != tc.want {
                t.Errorf("IsValidEmail(%q) = %v; want %v", tc.email, got, tc.want)
            }
        })
    }
}
```

It is idiomatic because:
- Adding a test case is adding one line in the table (low friction)
- All test cases are visible together — easy to see coverage
- Clear what input maps to what output
- `t.Run` gives each case a meaningful name in test output

---

**Q5. What is `t.Parallel()` and how does it work?**

`t.Parallel()` marks a test (or subtest) as safe to run in parallel with other parallel tests:

```go
func TestFeatureA(t *testing.T) {
    t.Parallel() // this test runs in parallel with other parallel tests
    // ... test code
}

func TestFeatureB(t *testing.T) {
    t.Parallel()
    // ... test code
}
```

In table-driven subtests, `t.Parallel()` inside `t.Run` is common — but requires careful variable capture:

```go
func TestProcess(t *testing.T) {
    tests := []struct{ input, want string }{
        {"a", "A"},
        {"b", "B"},
    }
    for _, tc := range tests {
        tc := tc // CRITICAL: capture loop variable before goroutine
        t.Run(tc.input, func(t *testing.T) {
            t.Parallel() // subtests now run concurrently
            got := Process(tc.input)
            if got != tc.want {
                t.Errorf("Process(%q) = %q; want %q", tc.input, got, tc.want)
            }
        })
    }
}
```

Without `tc := tc`, all goroutines would share the same `tc` variable — the classic loop variable capture bug.

---

**Q6. What is `t.Helper()` and why should test helpers call it?**

`t.Helper()` marks the calling function as a test helper. When a helper calls `t.Fatal` or `t.Error`, the failure is reported at the **caller's** line, not inside the helper:

```go
// Without t.Helper() — failure reported at line 8 inside assertEq
func assertEq(t *testing.T, got, want int) {
    if got != want {
        t.Errorf("got %d, want %d", got, want) // line 8 — confusing!
    }
}

// With t.Helper() — failure reported at the test's call site
func assertEq(t *testing.T, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d, want %d", got, want) // reported at assertEq's caller
    }
}

func TestSomething(t *testing.T) {
    result := compute()
    assertEq(t, result, 42) // failure reported HERE — much clearer
}
```

---

**Q7. What is `t.Cleanup()` and how does it differ from `defer`?**

`t.Cleanup(func())` registers a cleanup function that runs when the test (or subtest) completes, whether it passed or failed. It is like `defer` but:
- Works across multiple helper functions (helper can register cleanup, caller doesn't need to know)
- Cleanups run in LIFO order, same as `defer`
- Available in subtests — each `t.Run` has its own cleanup scope

```go
// Helper that sets up AND registers its own cleanup
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("open db: %v", err)
    }
    t.Cleanup(func() {
        db.Close() // automatically called when test ends
    })
    return db
}

func TestUserStore(t *testing.T) {
    db := setupTestDB(t)      // cleanup registered inside helper
    store := NewUserStore(db) // no need to defer db.Close() here
    // test code...
}
```

---

**Q8. What is `t.Setenv()` and when do you use it?**

`t.Setenv(key, value)` sets an environment variable for the duration of a test and automatically restores the original value when the test ends:

```go
func TestWithEnv(t *testing.T) {
    t.Setenv("DATABASE_URL", "postgres://localhost/testdb")
    t.Setenv("LOG_LEVEL", "debug")

    // These env vars are set during this test only
    cfg := LoadConfig()
    if cfg.DatabaseURL != "postgres://localhost/testdb" {
        t.Errorf("unexpected db url: %s", cfg.DatabaseURL)
    }
    // After test ends, DATABASE_URL and LOG_LEVEL are restored to original values
}
```

`t.Setenv` also calls `t.Helper()` internally and marks the test as not parallelizable (environment variables are global state).

---

## Benchmarks

**Q9. How do you write a benchmark in Go?**

Benchmark functions must start with `Benchmark` and accept `*testing.B`:

```go
func BenchmarkFibonacci(b *testing.B) {
    b.ReportAllocs() // also report memory allocations

    for i := 0; i < b.N; i++ { // b.N is set by the testing framework
        Fibonacci(20)
    }
}

// Run benchmarks
// go test -bench=. -benchmem
// go test -bench=BenchmarkFibonacci -benchtime=5s -count=3
```

Output:
```
BenchmarkFibonacci-8    5000000    234 ns/op    0 B/op    0 allocs/op
```

Fields: `name-GOMAXPROCS`, `iterations`, `ns/op`, `bytes/op`, `allocs/op`.

---

**Q10. What is `b.ResetTimer()` and when do you need it?**

When a benchmark has setup code before the measured loop, `b.ResetTimer()` discards the time spent on setup:

```go
func BenchmarkProcess(b *testing.B) {
    // Expensive setup — should NOT be counted in benchmark time
    data := generateLargeDataset()
    setupDatabase()

    b.ResetTimer() // start timing from here

    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

Similarly, `b.StopTimer()` and `b.StartTimer()` can pause/resume timing within the loop:

```go
func BenchmarkSort(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomSlice(1000) // reset data each iteration
        b.StartTimer()

        sort.Ints(data) // measure only sorting, not data generation
    }
}
```

---

**Q11. What is `b.ReportAllocs()` and what does it show?**

`b.ReportAllocs()` enables reporting of memory allocations per operation. It is equivalent to running with `-benchmem`.

```go
func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "a" // many allocations — each += allocates a new string
        }
        _ = s
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteByte('a') // fewer allocations — grows buffer geometrically
        }
        _ = sb.String()
    }
}
```

---

## Examples

**Q12. What are Example functions and what makes them special?**

Example functions (starting with `Example`) serve dual purposes: they are runnable tests AND documentation examples.

```go
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output:
    // 5
}

func ExampleReverse() {
    fmt.Println(Reverse("hello"))
    fmt.Println(Reverse(""))
    // Output:
    // olleh
    //
}
```

The `// Output:` comment is a test assertion — `go test` runs the function and compares printed output to the comment. If they don't match, the test fails.

They appear in `go doc` and `pkg.go.dev` documentation as runnable examples.

---

## Mocking

**Q13. How do you mock dependencies in Go tests?**

Go mocking is interface-based. Design your code to depend on interfaces, then provide fake implementations in tests:

```go
// Production interface
type UserRepository interface {
    GetUser(ctx context.Context, id int64) (*User, error)
    CreateUser(ctx context.Context, user *User) error
}

// Production implementation
type PostgresUserRepo struct{ db *sql.DB }

// Test mock
type MockUserRepo struct {
    GetUserFunc    func(ctx context.Context, id int64) (*User, error)
    CreateUserFunc func(ctx context.Context, user *User) error
    calls          []string
}

func (m *MockUserRepo) GetUser(ctx context.Context, id int64) (*User, error) {
    m.calls = append(m.calls, fmt.Sprintf("GetUser(%d)", id))
    return m.GetUserFunc(ctx, id)
}

func (m *MockUserRepo) CreateUser(ctx context.Context, user *User) error {
    m.calls = append(m.calls, "CreateUser")
    return m.CreateUserFunc(ctx, user)
}

// In test
func TestUserService_GetProfile(t *testing.T) {
    mock := &MockUserRepo{
        GetUserFunc: func(ctx context.Context, id int64) (*User, error) {
            if id == 42 {
                return &User{ID: 42, Name: "Alice"}, nil
            }
            return nil, ErrNotFound
        },
    }
    svc := NewUserService(mock)
    profile, err := svc.GetProfile(context.Background(), 42)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if profile.Name != "Alice" {
        t.Errorf("expected Alice, got %s", profile.Name)
    }
}
```

---

**Q14. What is the `testify` library and what does `assert` vs `require` mean?**

`testify/assert` functions continue the test if assertion fails. `testify/require` functions call `t.FailNow()` if assertion fails (equivalent to `t.Fatal`).

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestLogin(t *testing.T) {
    user, err := Login("alice", "password")

    // require: if err != nil, test stops here — no point checking user
    require.NoError(t, err, "login should not fail")
    require.NotNil(t, user, "user should be returned")

    // assert: test continues even if these fail
    assert.Equal(t, "alice", user.Name)
    assert.True(t, user.Active)
    assert.WithinDuration(t, time.Now(), user.LastLogin, time.Second)
}
```

Common assertion functions:
- `assert.Equal(t, expected, actual)`
- `assert.NoError(t, err)`
- `assert.Error(t, err)`
- `assert.Nil(t, v)`
- `assert.NotNil(t, v)`
- `assert.True(t, condition)`
- `assert.Len(t, slice, n)`
- `assert.Contains(t, collection, element)`
- `assert.ErrorIs(t, err, target)` — uses `errors.Is` under the hood

---

## httptest Package

**Q15. How does `httptest.NewRecorder()` work?**

`httptest.ResponseRecorder` captures the response written by an HTTP handler so you can inspect it in tests:

```go
func TestHealthHandler(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    rr := httptest.NewRecorder()

    HealthHandler(rr, req)

    // Inspect the recorded response
    if rr.Code != http.StatusOK {
        t.Errorf("expected 200, got %d", rr.Code)
    }

    var resp map[string]string
    json.NewDecoder(rr.Body).Decode(&resp)
    if resp["status"] != "ok" {
        t.Errorf("expected status ok, got %v", resp)
    }

    ct := rr.Header().Get("Content-Type")
    if ct != "application/json" {
        t.Errorf("expected JSON content type, got %s", ct)
    }
}
```

---

**Q16. How does `httptest.NewServer()` work and when do you use it instead of `NewRecorder`?**

`httptest.NewServer` starts a real HTTP server on a random local port. Use it when you need:
- To test actual HTTP client code (not just handlers)
- Middleware that inspects `http.Request` details
- Connection pooling or keep-alive behavior

```go
func TestHTTPClient(t *testing.T) {
    // Start a real test server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"id": 1, "name": "Alice"}`))
    }))
    defer server.Close() // stops server when test ends

    client := NewAPIClient(server.URL) // use the test server URL
    user, err := client.GetUser(context.Background(), "token-abc")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}
```

---

## Fuzz Testing

**Q17. What is fuzz testing (Go 1.18+) and how do you write a fuzz test?**

Fuzz testing automatically generates random inputs to find crashes and panics. The fuzzer learns from the corpus and from code coverage to generate increasingly interesting inputs.

```go
func FuzzParseURL(f *testing.F) {
    // Seed corpus — known interesting inputs
    f.Add("http://example.com/path?q=1")
    f.Add("")
    f.Add("https://user:pass@host:8080/path#fragment")

    f.Fuzz(func(t *testing.T, s string) {
        // This function is called with s being each corpus entry,
        // then with generated mutations
        u, err := url.Parse(s)
        if err != nil {
            return // errors are fine — we're looking for panics
        }
        // Invariant: re-serializing should not panic
        _ = u.String()
    })
}
```

Run: `go test -fuzz=FuzzParseURL -fuzztime=30s`

When the fuzzer finds a failing input, it saves it to `testdata/fuzz/FuzzParseURL/`. On subsequent `go test` runs, these saved inputs are always tested (regression test).

---

**Q18. What does the `-race` flag do?**

`-race` enables Go's race detector, which instruments memory accesses at runtime to detect concurrent reads and writes without synchronization:

```go
// Run all tests with race detection:
// go test -race ./...

func TestConcurrentCounter(t *testing.T) {
    var counter int
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE: concurrent unsynchronized write
        }()
    }
    wg.Wait()
}
// go test -race → reports the race with goroutine stacks
```

The race detector adds ~5-10x overhead. Always use `-race` in CI. The race detector only catches races that actually occur during the run — it is not a static analysis tool.

---

**Q19. What is `TestMain` and when do you need it?**

`TestMain` allows you to run setup and teardown code around the entire test binary:

```go
func TestMain(m *testing.M) {
    // Setup — runs once before ALL tests in this package
    db := setupTestDatabase()
    setupTestFixtures(db)

    // Run all tests
    exitCode := m.Run()

    // Teardown — runs once after ALL tests complete
    db.Close()
    cleanupTestFiles()

    os.Exit(exitCode) // MUST call os.Exit with the code from m.Run()
}
```

Use `TestMain` for:
- Database migration/seeding before any tests
- Starting a test server once (shared across tests)
- Setting up external resources (test containers, mock servers)
- Expensive operations that should run once per package, not per test

---

**Q20. What is `-cover` and `-coverprofile`?**

```bash
# Basic coverage
go test -cover ./...
# PASS
# coverage: 78.3% of statements

# Generate coverage profile
go test -coverprofile=coverage.out ./...

# View per-function coverage
go tool cover -func=coverage.out

# Open HTML coverage report in browser
go tool cover -html=coverage.out

# Set minimum threshold in CI
go test -cover ./... | grep -E "coverage: [0-9]+" | \
    awk '{if ($2 < 80) {print "Coverage below 80%!"; exit 1}}'
```

Coverage modes:
- `-covermode=set` (default) — was each statement executed at least once?
- `-covermode=count` — how many times was each statement executed?
- `-covermode=atomic` — like count but safe for parallel tests

---

**Q21. What are golden file tests and when do you use them?**

Golden file tests compare function output to a stored "golden" file. Useful for testing complex, multi-line output (templates, formatters, code generators):

```go
func TestRenderTemplate(t *testing.T) {
    data := TemplateData{Name: "Alice", Items: []string{"foo", "bar"}}
    got := RenderTemplate(data)

    // Path to golden file
    goldenPath := filepath.Join("testdata", "expected_output.txt")

    if *update { // go test -update flag
        os.WriteFile(goldenPath, []byte(got), 0644)
        return
    }

    expected, err := os.ReadFile(goldenPath)
    if err != nil {
        t.Fatalf("read golden file: %v", err)
    }

    if got != string(expected) {
        t.Errorf("output mismatch:\ngot:\n%s\nwant:\n%s", got, expected)
    }
}

var update = flag.Bool("update", false, "update golden files")
```

---

**Q22. What are the most important `go test` flags?**

```bash
go test ./...                     # run all tests in all packages
go test -v ./...                  # verbose: print each test name and result
go test -run TestUser             # run tests matching pattern
go test -run "TestUser/create"    # run specific subtest
go test -race ./...               # race detection
go test -count=1 ./...            # disable test result caching
go test -timeout 30s ./...        # overall timeout
go test -short ./...              # skip slow tests (check t.Short())
go test -bench=. -benchmem        # run benchmarks with memory stats
go test -fuzz=FuzzFoo             # run fuzz test
go test -cover -coverprofile=c.out # coverage
go test -parallel 4               # max parallel tests
go test -cpu 1,2,4                # run with different GOMAXPROCS
```

---

**Q23. What is the `testcontainers-go` library and what problem does it solve?**

`testcontainers-go` starts real Docker containers during tests, providing actual databases, queues, and services — not mocks:

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestWithRealDB(t *testing.T) {
    ctx := context.Background()

    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections"),
        ),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    db, err := sql.Open("pgx", connStr)
    require.NoError(t, err)
    defer db.Close()

    // Now test against a real PostgreSQL database
    store := NewStore(db)
    // ...
}
```

---

**Q24. What does `t.Short()` enable?**

`t.Short()` returns true when tests are run with the `-short` flag. Use it to skip slow integration tests:

```go
func TestIntegration_FetchFromAPI(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // This only runs in full test mode
    resp, err := http.Get("https://external.api.com/data")
    // ...
}
```

In CI, run unit tests with `-short` on every commit, and full integration tests only on pull requests or scheduled runs.

---

**Q25. What are the differences between white-box and black-box testing in Go?**

```go
// White-box: same package as code — can access unexported identifiers
package calculator

import "testing"

func TestInternalHelper(t *testing.T) {
    // Can test unexported functions
    result := internalHelperFunc(42)
    // ...
}

// Black-box: package_test suffix — only exported identifiers
package calculator_test

import (
    "testing"
    "mypackage/calculator"
)

func TestPublicAPI(t *testing.T) {
    // Can only access exported types/functions
    result := calculator.Add(2, 3)
    // ...
}
```

**When to use each:**
- White-box: when testing internal logic, complex private functions, or wanting full coverage of internals
- Black-box: when testing the public API as a consumer would, ensuring the public interface is sufficient

You can have both in the same directory — Go handles them both.
