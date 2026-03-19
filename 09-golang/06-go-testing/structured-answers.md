# Go Testing — Structured Answers

## Answer 1: What is a table-driven test and why is it idiomatic Go?

A table-driven test defines all test cases upfront as a slice of structs, then iterates over them with a single test loop. It is idiomatic Go because it minimizes boilerplate while maximizing coverage visibility.

```go
package calc_test

import "testing"

func Add(a, b int) int { return a + b }

func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"zero", 0, 0, 0},
        {"mixed signs", -1, 1, 0},
        {"large numbers", 1000000, 999999, 1999999},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got := Add(tc.a, tc.b)
            if got != tc.want {
                t.Errorf("Add(%d, %d) = %d; want %d", tc.a, tc.b, got, tc.want)
            }
        })
    }
}
```

**Why idiomatic Go:**
1. **Adding a test case = adding a line** in the table. There is no `TestAdd_NegativeNumbers`, `TestAdd_Zero` function proliferation.
2. **The complete test matrix is visible** in one place — reviewers can see all cases being tested.
3. **`t.Run` gives each case a name** — failed output clearly says `TestAdd/mixed_signs` failed, not just `TestAdd`.
4. **The test logic is written once** — the loop body, not duplicated per case.
5. **Order of inputs is declarative** — the struct field names document what is input vs expected output.

The standard library tests (`strings`, `fmt`, `math`) are almost entirely table-driven.

---

## Answer 2: How do you mock HTTP dependencies in Go tests using `httptest`?

Go provides `net/http/httptest` for testing HTTP code without making real network calls.

**`httptest.NewRecorder`** — for testing handlers in isolation:

```go
package handlers_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestGetUserHandler(t *testing.T) {
    // Create request and recorder
    req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
    req.Header.Set("Authorization", "Bearer test-token")
    rr := httptest.NewRecorder()

    // Call handler directly — no network
    GetUserHandler(rr, req)

    // Inspect response
    resp := rr.Result()
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d; want %d", resp.StatusCode, http.StatusOK)
    }

    var user User
    json.NewDecoder(resp.Body).Decode(&user)
    if user.Name != "Alice" {
        t.Errorf("name = %q; want Alice", user.Name)
    }
}
```

**`httptest.NewServer`** — for testing HTTP clients:

```go
func TestAPIClient_GetUser(t *testing.T) {
    // Control what the "server" returns
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Assert the client sends correct request
        if r.URL.Path != "/users/42" {
            t.Errorf("unexpected path: %s", r.URL.Path)
        }
        if r.Header.Get("X-API-Key") == "" {
            http.Error(w, "no api key", http.StatusUnauthorized)
            return
        }
        // Return test data
        json.NewEncoder(w).Encode(User{ID: 42, Name: "Alice"})
    })

    srv := httptest.NewServer(handler)
    defer srv.Close()

    // Point client at the test server
    client := NewAPIClient(srv.URL, "test-api-key")
    user, err := client.GetUser(context.Background(), 42)

    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("Name = %q; want Alice", user.Name)
    }
}
```

Use `NewRecorder` when testing handlers directly. Use `NewServer` when testing the HTTP client layer.

---

## Answer 3: How do you write and run benchmarks in Go?

```go
package stringutil_test

import (
    "strings"
    "testing"
)

// Function under benchmark
func joinWithBuilder(strs []string) string {
    var sb strings.Builder
    for i, s := range strs {
        if i > 0 {
            sb.WriteByte(',')
        }
        sb.WriteString(s)
    }
    return sb.String()
}

func joinWithJoin(strs []string) string {
    return strings.Join(strs, ",")
}

// Benchmarks
func BenchmarkJoinBuilder(b *testing.B) {
    data := make([]string, 100)
    for i := range data {
        data[i] = fmt.Sprintf("item-%d", i)
    }
    b.ResetTimer()  // don't count data generation
    b.ReportAllocs() // show memory allocation stats

    for i := 0; i < b.N; i++ {
        _ = joinWithBuilder(data)
    }
}

func BenchmarkJoinStrings(b *testing.B) {
    data := make([]string, 100)
    for i := range data {
        data[i] = fmt.Sprintf("item-%d", i)
    }
    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        _ = joinWithJoin(data)
    }
}
```

**Running benchmarks:**

```bash
# Run all benchmarks
go test -bench=. -benchmem

# Run specific benchmark, 5 seconds per bench
go test -bench=BenchmarkJoin -benchtime=5s -benchmem

# Run multiple times for stable results
go test -bench=. -benchmem -count=5

# Compare with benchstat
go test -bench=. -count=10 > old.txt
# make optimization
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt
```

**Reading output:**

```
BenchmarkJoinBuilder-8    500000    2345 ns/op    1024 B/op    3 allocs/op
                 ^               ^           ^          ^          ^
              GOMAXPROCS    iterations   ns per op   bytes/op   allocs/op
```

---

## Answer 4: How do you use `testify/assert` vs `testify/require`?

Both packages provide rich assertion functions with clear failure messages. The difference is in test control flow.

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestLogin(t *testing.T) {
    user, token, err := Login("alice@example.com", "correctpassword")

    // require: stops the test immediately on failure (calls t.FailNow)
    // Use when subsequent assertions would panic or be meaningless
    require.NoError(t, err)         // if err != nil, stop here
    require.NotNil(t, user)         // if user is nil, next line would panic
    require.NotEmpty(t, token)      // token must exist

    // assert: marks test as failed but continues (calls t.Fail)
    // Use when multiple independent checks are valuable
    assert.Equal(t, "alice@example.com", user.Email)
    assert.True(t, user.Active)
    assert.Equal(t, "alice", user.DisplayName)

    // Both accept an optional message format
    assert.Equal(t, 200, user.Score, "new user should start with 200 points")

    // Useful assertions
    assert.Error(t, err)                           // err must be non-nil
    assert.ErrorIs(t, err, ErrNotFound)            // uses errors.Is
    assert.ErrorAs(t, err, &myErrType)             // uses errors.As
    assert.Len(t, user.Roles, 2)                   // slice/map length
    assert.Contains(t, user.Roles, "admin")        // contains element
    assert.ElementsMatch(t, got, want)             // same elements, any order
    assert.JSONEq(t, expectedJSON, actualJSON)      // JSON semantic equality
    assert.WithinDuration(t, expected, actual, d)  // time comparison
}
```

**Rule of thumb:** Start with `require` for setup/preconditions where nil pointer derefs or meaningless tests would follow. Use `assert` for checking multiple independent properties.

---

## Answer 5: How do you test concurrent code for race conditions?

The primary tool is the `-race` flag, which instruments the binary to detect concurrent memory accesses at runtime.

```go
// Code under test — intentionally buggy
type UnsafeCounter struct {
    count int // not protected
}

func (c *UnsafeCounter) Inc() {
    c.count++ // DATA RACE
}

func (c *UnsafeCounter) Value() int {
    return c.count // DATA RACE
}

// Test that triggers the race
func TestUnsafeCounter_Race(t *testing.T) {
    c := &UnsafeCounter{}
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Inc()
        }()
    }
    wg.Wait()
    // Run with: go test -race -run TestUnsafeCounter_Race
    // Output: WARNING: DATA RACE with goroutine stacks
}

// Fixed version — also test that it's correct
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func TestSafeCounter(t *testing.T) {
    c := &SafeCounter{}
    var wg sync.WaitGroup
    const n = 1000

    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Inc()
        }()
    }
    wg.Wait()

    if got := c.Value(); got != n {
        t.Errorf("counter = %d; want %d", got, n)
    }
}
```

**In CI:** Always run `go test -race ./...`. The race detector has 5-10x overhead but is worth it for correctness. It only detects races that actually occur during the run, so write tests that exercise concurrent code paths.

---

## Answer 6: What is fuzz testing and when should you use it?

Fuzz testing automatically generates inputs to your function, trying to trigger panics, crashes, and invariant violations. The fuzzer uses code coverage to guide mutation toward unexplored code paths.

```go
package parser_test

import (
    "testing"
    "myapp/parser"
)

func FuzzParseConfig(f *testing.F) {
    // Seed corpus: known interesting inputs
    f.Add(`{"host":"localhost","port":5432}`)   // valid
    f.Add(`{}`)                                   // empty
    f.Add(`{"port":-1}`)                          // invalid value
    f.Add("")                                     // empty string
    f.Add(`{`)                                    // malformed JSON

    f.Fuzz(func(t *testing.T, input string) {
        // The fuzzer will mutate 'input' to generate new test cases.
        // Our job: define invariants that must always hold.

        cfg, err := parser.ParseConfig([]byte(input))
        if err != nil {
            // Parsing errors are expected — we only care about panics
            return
        }

        // Invariant 1: serializing and re-parsing should give same result
        data, err := cfg.Marshal()
        if err != nil {
            t.Fatalf("marshal failed after successful parse: %v", err)
        }
        cfg2, err := parser.ParseConfig(data)
        if err != nil {
            t.Fatalf("re-parse of marshaled config failed: %v", err)
        }
        if cfg.Host != cfg2.Host || cfg.Port != cfg2.Port {
            t.Errorf("round-trip failed: original=%+v, re-parsed=%+v", cfg, cfg2)
        }

        // Invariant 2: port must be in valid range or config is invalid
        if cfg.Port < 0 || cfg.Port > 65535 {
            t.Errorf("invalid port %d should have caused parse error", cfg.Port)
        }
    })
}
```

**Running the fuzzer:**
```bash
# Fuzz for 30 seconds
go test -fuzz=FuzzParseConfig -fuzztime=30s

# When a failing input is found, it's saved to:
# testdata/fuzz/FuzzParseConfig/<hash>
# Subsequent go test runs always replay it

# Run just the corpus (no fuzzing)
go test -run=FuzzParseConfig
```

**When to use fuzz testing:**
- Parsers (JSON, XML, CSV, binary protocols)
- Security-sensitive code (input validation, deserialization)
- Any function that takes untrusted external input
- Functions with complex invariants that are hard to enumerate manually

---

## Answer 7: How do you set up and tear down test state in Go?

Go provides several mechanisms, ordered from most to least scoped:

```go
// 1. defer — scoped to test function, runs before test returns
func TestWithDefer(t *testing.T) {
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatal(err)
    }
    defer os.Remove(f.Name()) // always cleaned up
    defer f.Close()
    // use f...
}

// 2. t.Cleanup — like defer, but works across helper calls
func createTempFile(t *testing.T) *os.File {
    t.Helper()
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        f.Close()
        os.Remove(f.Name()) // called when test ends, from inside the helper
    })
    return f
}

func TestWithHelper(t *testing.T) {
    f := createTempFile(t) // cleanup registered inside helper
    // use f — no need to defer cleanup here
}

// 3. t.Setenv — sets env var, auto-restores on test end
func TestWithEnv(t *testing.T) {
    t.Setenv("CONFIG_PATH", "/tmp/test-config.json")
    // env var is restored after test
}

// 4. TestMain — runs once per package (not per test)
func TestMain(m *testing.M) {
    // once-per-package setup
    pool := startContainerPool()

    code := m.Run()

    // once-per-package teardown
    pool.Cleanup()
    os.Exit(code)
}

// 5. Subtests with their own cleanup
func TestSuite(t *testing.T) {
    // setup shared by all subtests in this function
    db := setupInMemoryDB(t)
    t.Cleanup(func() { db.Close() })

    t.Run("create", func(t *testing.T) {
        // setup specific to this subtest
        t.Cleanup(func() {
            db.Exec("DELETE FROM users WHERE test = true")
        })
        // test...
    })
    t.Run("read", func(t *testing.T) { /* ... */ })
}
```

---

## Answer 8: How do you achieve dependency injection for testability in Go?

The key principle: depend on interfaces, not concrete types. Inject dependencies through constructor parameters.

```go
// UNTESTABLE: hard dependency on concrete type
type OrderService struct{}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    db := sql.Open("postgres", os.Getenv("DATABASE_URL")) // hard dependency
    // ...
}

// TESTABLE: interface-based dependencies injected via constructor
type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id int64) (*Order, error)
}

type PaymentGateway interface {
    Charge(ctx context.Context, amount int, cardToken string) (string, error)
}

type OrderService struct {
    repo    OrderRepository
    payment PaymentGateway
    clock   func() time.Time // injectable time function
}

func NewOrderService(repo OrderRepository, payment PaymentGateway) *OrderService {
    return &OrderService{
        repo:    repo,
        payment: payment,
        clock:   time.Now, // default to real time
    }
}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    chargeID, err := s.payment.Charge(ctx, req.Amount, req.CardToken)
    if err != nil {
        return nil, fmt.Errorf("charge card: %w", err)
    }
    order := &Order{
        CustomerID: req.CustomerID,
        ChargeID:   chargeID,
        CreatedAt:  s.clock(),
    }
    if err := s.repo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("save order: %w", err)
    }
    return order, nil
}

// Test with mock implementations
func TestCreateOrder_Success(t *testing.T) {
    repo := &MockOrderRepo{}
    payment := &MockPayment{chargeID: "ch_123"}
    fixedTime := time.Date(2025, 3, 17, 12, 0, 0, 0, time.UTC)

    svc := NewOrderService(repo, payment)
    svc.clock = func() time.Time { return fixedTime }

    order, err := svc.CreateOrder(context.Background(), CreateOrderRequest{
        CustomerID: 1,
        Amount:     1999,
        CardToken:  "tok_test",
    })

    require.NoError(t, err)
    assert.Equal(t, "ch_123", order.ChargeID)
    assert.Equal(t, fixedTime, order.CreatedAt)
    assert.Len(t, repo.savedOrders, 1)
}
```

---

## Answer 9: What is `TestMain` and when do you need it?

`TestMain` is a special function that replaces the default `go test` entry point. If present in a test package, the testing framework calls `TestMain(m *testing.M)` instead of running tests directly.

```go
package store_test

import (
    "database/sql"
    "fmt"
    "os"
    "testing"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
    // SETUP: runs once before any test in this package
    var err error
    testDB, err = sql.Open("postgres", os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        fmt.Fprintf(os.Stderr, "open db: %v\n", err)
        os.Exit(1)
    }
    if err := runMigrations(testDB); err != nil {
        fmt.Fprintf(os.Stderr, "migrate: %v\n", err)
        os.Exit(1)
    }

    // RUN: execute all Test*, Benchmark*, Example* functions
    exitCode := m.Run()

    // TEARDOWN: runs after all tests complete (even on failure)
    testDB.Close()
    dropTestDatabase()

    // MUST exit with m.Run()'s code — propagates test pass/fail
    os.Exit(exitCode)
}
```

**When to use `TestMain`:**
- **Database setup**: run migrations, seed data once per package
- **Test containers**: start Docker containers (Postgres, Redis) once, reuse across tests
- **Mock servers**: start one test HTTP server used by all tests
- **Flag parsing**: register custom test flags with `flag.Parse()`
- **Package-level teardown**: delete temp files, release external resources

**What NOT to use it for:**
- Per-test setup — use `t.Cleanup` instead
- Test isolation — all tests share `TestMain`'s setup, so if tests mutate shared state, use `t.Cleanup` to reset it

---

## Answer 10: How do you measure and improve test coverage?

**Measuring coverage:**

```bash
# Basic coverage report
go test -cover ./...
# ok  myapp/store  0.532s  coverage: 72.4% of statements
# ok  myapp/handler  0.125s  coverage: 85.1% of statements

# Detailed coverage profile
go test -coverprofile=coverage.out ./...

# Per-function breakdown
go tool cover -func=coverage.out
# myapp/store/store.go:CreateUser    87.5%
# myapp/store/store.go:GetUser       100.0%
# myapp/store/store.go:deleteUser    0.0%   ← untested!

# Interactive HTML coverage report
go tool cover -html=coverage.out
# Opens browser showing which lines are covered (green) and uncovered (red)

# Coverage for a specific package only
go test -cover -coverprofile=coverage.out myapp/store
```

**Improving coverage methodically:**

```go
// Look at the HTML report to find uncovered paths.
// Common uncovered areas:

// 1. Error paths — hard to trigger with happy-path tests
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Query().Get("id"))
    if err != nil {
        // Error path — test by passing ?id=notanumber
        http.Error(w, "bad id", http.StatusBadRequest)
        return
    }
    // ...
}

// Test the error path:
func TestGetUserHandler_BadID(t *testing.T) {
    req := httptest.NewRequest("GET", "/users?id=abc", nil)
    rr := httptest.NewRecorder()
    getUserHandler(rr, req)
    if rr.Code != http.StatusBadRequest {
        t.Errorf("expected 400, got %d", rr.Code)
    }
}

// 2. Use coverage thresholds in CI
# .github/workflows/test.yml
- run: |
    go test -coverprofile=coverage.out ./...
    COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d %)
    if [ "$COVERAGE" -lt "80" ]; then
      echo "Coverage $COVERAGE% is below minimum 80%"
      exit 1
    fi
```

**What coverage doesn't measure:**
- Whether test assertions are actually correct (you can have 100% coverage with no assertions)
- Concurrency bugs — `go test -race` catches those
- Performance regressions — benchmarks catch those
- Integration correctness — high unit coverage doesn't mean the system works end-to-end

Target 80% as a floor, but focus on testing the critical paths, error handling, and edge cases thoroughly over achieving an arbitrary high number.
