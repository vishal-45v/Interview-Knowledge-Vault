# Go Testing — Scenario Questions

## HTTP and External Dependencies

**Scenario 1: You have a service that calls an external HTTP API. Write tests without making real network calls.**

Use `httptest.NewServer` to provide a real HTTP server under test control:

```go
package service_test

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestWeatherService_GetTemperature(t *testing.T) {
    tests := []struct {
        name           string
        city           string
        serverResponse interface{}
        serverStatus   int
        wantTemp       float64
        wantErr        bool
    }{
        {
            name:           "successful response",
            city:           "London",
            serverResponse: map[string]interface{}{"temp": 15.5, "unit": "celsius"},
            serverStatus:   http.StatusOK,
            wantTemp:       15.5,
        },
        {
            name:         "city not found",
            city:         "Atlantis",
            serverStatus: http.StatusNotFound,
            wantErr:      true,
        },
        {
            name:         "server error",
            city:         "Paris",
            serverStatus: http.StatusInternalServerError,
            wantErr:      true,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            // Create a test server that returns the configured response
            srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                // Verify the request is correct
                assert.Equal(t, "/weather", r.URL.Path)
                assert.Equal(t, tc.city, r.URL.Query().Get("city"))
                assert.Equal(t, "Bearer test-token", r.Header.Get("Authorization"))

                w.WriteHeader(tc.serverStatus)
                if tc.serverResponse != nil {
                    json.NewEncoder(w).Encode(tc.serverResponse)
                }
            }))
            defer srv.Close()

            client := NewWeatherClient(srv.URL, "test-token")
            temp, err := client.GetTemperature(context.Background(), tc.city)

            if tc.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tc.wantTemp, temp)
        })
    }
}
```

---

**Scenario 2: Test an HTTP handler that reads from a database.**

Use interface-based mocking to isolate the handler from the database:

```go
package handler_test

import (
    "context"
    "encoding/json"
    "errors"
    "net/http"
    "net/http/httptest"
    "testing"
)

type MockUserStore struct {
    users map[int64]*User
}

func (m *MockUserStore) GetUser(ctx context.Context, id int64) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func TestGetUserHandler(t *testing.T) {
    store := &MockUserStore{
        users: map[int64]*User{
            1: {ID: 1, Name: "Alice", Email: "alice@example.com"},
        },
    }
    handler := NewUserHandler(store)

    t.Run("existing user", func(t *testing.T) {
        req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
        req = setPathParam(req, "id", "1") // using gorilla/mux or chi
        rr := httptest.NewRecorder()

        handler.GetUser(rr, req)

        if rr.Code != http.StatusOK {
            t.Fatalf("expected 200, got %d: %s", rr.Code, rr.Body.String())
        }

        var got User
        json.NewDecoder(rr.Body).Decode(&got)
        if got.Name != "Alice" {
            t.Errorf("expected Alice, got %s", got.Name)
        }
    })

    t.Run("user not found", func(t *testing.T) {
        req := httptest.NewRequest(http.MethodGet, "/users/999", nil)
        req = setPathParam(req, "id", "999")
        rr := httptest.NewRecorder()

        handler.GetUser(rr, req)

        if rr.Code != http.StatusNotFound {
            t.Errorf("expected 404, got %d", rr.Code)
        }
    })
}
```

---

## Table-Driven Tests

**Scenario 3: Write a table-driven test for a function that validates email addresses.**

```go
package validator_test

import (
    "testing"
    "myapp/validator"
)

func TestIsValidEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantOK  bool
        wantErr string // empty if no error expected
    }{
        // Valid emails
        {name: "simple valid", email: "user@example.com", wantOK: true},
        {name: "with dots", email: "first.last@sub.example.co.uk", wantOK: true},
        {name: "with plus", email: "user+tag@example.com", wantOK: true},
        {name: "with numbers", email: "user123@example.com", wantOK: true},
        {name: "with hyphen", email: "user-name@my-domain.com", wantOK: true},

        // Invalid emails
        {name: "empty string", email: "", wantOK: false, wantErr: "email is empty"},
        {name: "no at sign", email: "userexample.com", wantOK: false, wantErr: "missing @"},
        {name: "no domain", email: "user@", wantOK: false, wantErr: "missing domain"},
        {name: "no local", email: "@example.com", wantOK: false, wantErr: "missing local part"},
        {name: "spaces", email: "user name@example.com", wantOK: false, wantErr: "contains spaces"},
        {name: "double at", email: "user@@example.com", wantOK: false},
        {name: "no tld", email: "user@localhost", wantOK: false},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            ok, err := validator.IsValidEmail(tc.email)

            if ok != tc.wantOK {
                t.Errorf("IsValidEmail(%q) = %v; want %v", tc.email, ok, tc.wantOK)
            }

            if tc.wantErr != "" {
                if err == nil {
                    t.Errorf("expected error containing %q, got nil", tc.wantErr)
                } else if !strings.Contains(err.Error(), tc.wantErr) {
                    t.Errorf("expected error containing %q, got %q", tc.wantErr, err.Error())
                }
            }
        })
    }
}
```

---

**Scenario 4: Write a table-driven test for a function that parses configuration from environment variables.**

```go
func TestLoadConfig(t *testing.T) {
    tests := []struct {
        name    string
        env     map[string]string
        want    Config
        wantErr bool
    }{
        {
            name: "all required vars set",
            env: map[string]string{
                "DB_HOST": "localhost",
                "DB_PORT": "5432",
                "DB_NAME": "mydb",
            },
            want: Config{
                DBHost: "localhost",
                DBPort: 5432,
                DBName: "mydb",
            },
        },
        {
            name:    "missing DB_HOST",
            env:     map[string]string{"DB_PORT": "5432", "DB_NAME": "mydb"},
            wantErr: true,
        },
        {
            name:    "invalid port",
            env:     map[string]string{"DB_HOST": "localhost", "DB_PORT": "notaport", "DB_NAME": "mydb"},
            wantErr: true,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            // Set environment for this test only
            for k, v := range tc.env {
                t.Setenv(k, v)
            }

            got, err := LoadConfig()
            if (err != nil) != tc.wantErr {
                t.Fatalf("LoadConfig() error = %v, wantErr = %v", err, tc.wantErr)
            }
            if !tc.wantErr {
                if got.DBHost != tc.want.DBHost {
                    t.Errorf("DBHost = %q, want %q", got.DBHost, tc.want.DBHost)
                }
                if got.DBPort != tc.want.DBPort {
                    t.Errorf("DBPort = %d, want %d", got.DBPort, tc.want.DBPort)
                }
            }
        })
    }
}
```

---

## Concurrent Code Testing

**Scenario 5: You have a concurrent function. How do you test it for race conditions?**

```go
package cache_test

import (
    "sync"
    "testing"
)

// Cache is the type under test
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

// Test with race detector: go test -race
func TestCache_ConcurrentReadWrite(t *testing.T) {
    c := &Cache{data: make(map[string]string)}
    var wg sync.WaitGroup
    const goroutines = 100

    // Launch concurrent writers
    for i := 0; i < goroutines; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", n)
            c.Set(key, fmt.Sprintf("value-%d", n))
        }(i)
    }

    // Launch concurrent readers
    for i := 0; i < goroutines; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            c.Get(fmt.Sprintf("key-%d", n))
        }(i)
    }

    wg.Wait()
    // No assertions needed — the race detector catches violations
    // Run as: go test -race -run TestCache_ConcurrentReadWrite
}

// Test correctness under concurrency
func TestCache_ConcurrentSetGet_Correctness(t *testing.T) {
    c := &Cache{data: make(map[string]string)}
    var wg sync.WaitGroup

    // All goroutines set the same key — final value should be a valid value
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            c.Set("shared-key", fmt.Sprintf("value-%d", n))
        }(i)
    }
    wg.Wait()

    val, ok := c.Get("shared-key")
    if !ok {
        t.Error("expected key to be present")
    }
    if !strings.HasPrefix(val, "value-") {
        t.Errorf("unexpected value: %s", val)
    }
}
```

---

## Benchmarks

**Scenario 6: Write a benchmark comparing two sorting implementations.**

```go
package sort_test

import (
    "math/rand"
    "sort"
    "testing"
)

func generateRandomInts(n int) []int {
    s := make([]int, n)
    for i := range s {
        s[i] = rand.Intn(1000000)
    }
    return s
}

// Benchmark standard sort.Ints (quicksort)
func BenchmarkStdSort(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomInts(10000)
        b.StartTimer()

        sort.Ints(data)
    }
}

// Benchmark custom bucket sort
func BenchmarkBucketSort(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomInts(10000)
        b.StartTimer()

        BucketSort(data)
    }
}

// Benchmark at different sizes
func BenchmarkSort_Sizes(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}
    for _, size := range sizes {
        b.Run(fmt.Sprintf("std-n=%d", size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                b.StopTimer()
                data := generateRandomInts(size)
                b.StartTimer()
                sort.Ints(data)
            }
        })

        b.Run(fmt.Sprintf("bucket-n=%d", size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                b.StopTimer()
                data := generateRandomInts(size)
                b.StartTimer()
                BucketSort(data)
            }
        })
    }
}
```

Run: `go test -bench=BenchmarkSort -benchmem -benchtime=5s`

---

**Scenario 7: How do you prevent a benchmark from being optimized away by the compiler?**

```go
// PROBLEM: compiler may optimize away this computation since the result is unused
func BenchmarkCompute(b *testing.B) {
    for i := 0; i < b.N; i++ {
        expensiveCompute(100) // result discarded — may be compiled out!
    }
}

// SOLUTION 1: assign to a package-level sink variable
var resultSink int

func BenchmarkCompute(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = expensiveCompute(100)
    }
    resultSink = r // prevents optimization
}

// SOLUTION 2: use b.ReportMetric or the result in a meaningful way
func BenchmarkComputeString(b *testing.B) {
    b.ReportAllocs()
    var s string
    for i := 0; i < b.N; i++ {
        s = buildString(100)
    }
    _ = s
}
```

---

## Mocking and Dependency Injection

**Scenario 8: How do you test a service that sends emails? You don't want real emails sent.**

```go
package service_test

import (
    "context"
    "testing"
)

// Interface — the contract for sending email
type Mailer interface {
    SendWelcome(ctx context.Context, to, name string) error
    SendPasswordReset(ctx context.Context, to, token string) error
}

// Mock implementation for testing
type MockMailer struct {
    calls    []MailCall
    SendErr  error // configurable error
}

type MailCall struct {
    Method string
    To     string
    Params map[string]string
}

func (m *MockMailer) SendWelcome(ctx context.Context, to, name string) error {
    m.calls = append(m.calls, MailCall{
        Method: "SendWelcome",
        To:     to,
        Params: map[string]string{"name": name},
    })
    return m.SendErr
}

func (m *MockMailer) SendPasswordReset(ctx context.Context, to, token string) error {
    m.calls = append(m.calls, MailCall{
        Method: "SendPasswordReset",
        To:     to,
        Params: map[string]string{"token": token},
    })
    return m.SendErr
}

func (m *MockMailer) AssertCalledWith(t *testing.T, method, to string) {
    t.Helper()
    for _, c := range m.calls {
        if c.Method == method && c.To == to {
            return
        }
    }
    t.Errorf("expected %s to be called with to=%q, got calls: %+v", method, to, m.calls)
}

func TestUserService_Register(t *testing.T) {
    mailer := &MockMailer{}
    svc := NewUserService(mockStore, mailer)

    err := svc.Register(context.Background(), "alice@example.com", "Alice")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    mailer.AssertCalledWith(t, "SendWelcome", "alice@example.com")
    if len(mailer.calls) != 1 {
        t.Errorf("expected 1 mail sent, got %d", len(mailer.calls))
    }
}

func TestUserService_Register_MailError(t *testing.T) {
    mailer := &MockMailer{SendErr: errors.New("smtp: connection refused")}
    svc := NewUserService(mockStore, mailer)

    err := svc.Register(context.Background(), "alice@example.com", "Alice")
    // Decide: should registration fail if email fails?
    // This tests whatever the current behavior is
    if err != nil {
        t.Logf("registration failed due to mail error: %v", err)
    }
}
```

---

**Scenario 9: How do you test a function that depends on `time.Now()`?**

Inject a clock interface instead of calling `time.Now()` directly:

```go
// Clock interface — makes time injectable
type Clock interface {
    Now() time.Time
}

// Real clock for production
type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }

// Fake clock for tests
type FakeClock struct {
    current time.Time
}
func (f *FakeClock) Now() time.Time { return f.current }
func (f *FakeClock) Advance(d time.Duration) { f.current = f.current.Add(d) }

// Service uses Clock interface
type SessionService struct {
    clock   Clock
    sessions map[string]session
}

func (s *SessionService) IsExpired(token string) bool {
    sess := s.sessions[token]
    return s.clock.Now().After(sess.ExpiresAt)
}

// Test with full control over time
func TestSessionService_Expiry(t *testing.T) {
    clock := &FakeClock{current: time.Date(2025, 3, 17, 12, 0, 0, 0, time.UTC)}
    svc := NewSessionService(clock)

    svc.Create("token-123", 1*time.Hour)

    // Before expiry
    if svc.IsExpired("token-123") {
        t.Error("session should not be expired yet")
    }

    // Advance time past expiry
    clock.Advance(2 * time.Hour)

    if !svc.IsExpired("token-123") {
        t.Error("session should be expired")
    }
}
```

---

**Scenario 10: How do you test a goroutine-based worker without relying on `time.Sleep`?**

Use channels to synchronize instead of sleeping:

```go
package worker_test

import (
    "context"
    "testing"
    "time"
)

func TestWorker_ProcessesJobs(t *testing.T) {
    processed := make(chan string, 10)

    w := NewWorker(func(job string) error {
        processed <- job
        return nil
    })

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go w.Start(ctx)

    // Submit jobs
    w.Submit("job-1")
    w.Submit("job-2")
    w.Submit("job-3")

    // Wait for all 3 to be processed — with timeout instead of sleep
    timeout := time.After(5 * time.Second)
    var got []string
    for len(got) < 3 {
        select {
        case j := <-processed:
            got = append(got, j)
        case <-timeout:
            t.Fatalf("timed out waiting for jobs; got %d: %v", len(got), got)
        }
    }

    // Verify all 3 were processed
    expected := map[string]bool{"job-1": true, "job-2": true, "job-3": true}
    for _, j := range got {
        if !expected[j] {
            t.Errorf("unexpected job: %s", j)
        }
    }
}
```

---

## Fuzz Testing

**Scenario 11: Write a fuzz test for a base64-decode function.**

```go
package encoding_test

import (
    "encoding/base64"
    "testing"
)

func FuzzBase64Decode(f *testing.F) {
    // Seed corpus — representative inputs
    f.Add("aGVsbG8=")          // "hello"
    f.Add("")                   // empty
    f.Add("dGVzdA==")          // "test"
    f.Add("YQ==")               // "a"
    f.Add("!invalid@base64#")  // invalid input

    f.Fuzz(func(t *testing.T, input string) {
        decoded, err := base64.StdEncoding.DecodeString(input)
        if err != nil {
            // Decoding invalid input is expected — not a bug
            return
        }

        // Invariant: re-encoding the decoded value should give back the original
        reencoded := base64.StdEncoding.EncodeToString(decoded)
        if reencoded != input {
            t.Errorf("round-trip failed: %q → %q (decoded: %q)", input, reencoded, decoded)
        }
    })
}

// Fuzz test for custom JSON parser
func FuzzParseJSON(f *testing.F) {
    f.Add(`{"name":"alice","age":30}`)
    f.Add(`{}`)
    f.Add(`null`)
    f.Add(`[]`)

    f.Fuzz(func(t *testing.T, data string) {
        var result interface{}
        err := json.Unmarshal([]byte(data), &result)
        if err != nil {
            return // parsing errors are fine
        }
        // Invariant: marshaling back should not panic
        _, _ = json.Marshal(result)
    })
}
```

---

**Scenario 12: Use `TestMain` to set up a shared test database.**

```go
package store_test

import (
    "context"
    "database/sql"
    "os"
    "testing"
    "time"

    _ "github.com/lib/pq"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
    // Setup
    db, err := setupTestDB()
    if err != nil {
        fmt.Fprintf(os.Stderr, "setup test db: %v\n", err)
        os.Exit(1)
    }
    testDB = db

    // Run all tests
    code := m.Run()

    // Teardown
    testDB.Close()

    os.Exit(code)
}

func setupTestDB() (*sql.DB, error) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        dsn = "postgres://postgres:password@localhost/testdb?sslmode=disable"
    }

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("open: %w", err)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, fmt.Errorf("ping: %w", err)
    }

    // Run migrations
    if err := runMigrations(db); err != nil {
        db.Close()
        return nil, fmt.Errorf("migrations: %w", err)
    }

    return db, nil
}

// Individual tests use testDB
func TestUserStore_Create(t *testing.T) {
    store := NewUserStore(testDB)

    // Clean up after test
    t.Cleanup(func() {
        testDB.Exec("DELETE FROM users WHERE email = $1", "test@example.com")
    })

    user, err := store.Create(context.Background(), "test@example.com", "Test User")
    if err != nil {
        t.Fatalf("create user: %v", err)
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID")
    }
}
```

---

**Scenario 13: Write a test that verifies middleware injects values into context.**

```go
package middleware_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestAuthMiddleware_InjectsUser(t *testing.T) {
    var capturedUser *User

    // Handler that reads from context — verifies middleware worked
    downstream := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        capturedUser = UserFromContext(r.Context())
        w.WriteHeader(http.StatusOK)
    })

    // Wrap with auth middleware
    handler := AuthMiddleware(downstream)

    t.Run("valid token injects user", func(t *testing.T) {
        capturedUser = nil
        req := httptest.NewRequest(http.MethodGet, "/", nil)
        req.Header.Set("Authorization", "Bearer valid-token-for-user-42")
        rr := httptest.NewRecorder()

        handler.ServeHTTP(rr, req)

        if rr.Code != http.StatusOK {
            t.Errorf("expected 200, got %d", rr.Code)
        }
        if capturedUser == nil {
            t.Fatal("expected user in context, got nil")
        }
        if capturedUser.ID != 42 {
            t.Errorf("expected user ID 42, got %d", capturedUser.ID)
        }
    })

    t.Run("missing token returns 401", func(t *testing.T) {
        capturedUser = nil
        req := httptest.NewRequest(http.MethodGet, "/", nil)
        // No Authorization header
        rr := httptest.NewRecorder()

        handler.ServeHTTP(rr, req)

        if rr.Code != http.StatusUnauthorized {
            t.Errorf("expected 401, got %d", rr.Code)
        }
        if capturedUser != nil {
            t.Error("expected nil user in context")
        }
    })
}
```

---

**Scenario 14: How do you test a function that modifies global state?**

Use `t.Cleanup` to restore state, or redesign to avoid global state:

```go
// If you cannot avoid global state — restore it in cleanup
func TestWithGlobalConfig(t *testing.T) {
    // Save original
    originalTimeout := globalConfig.Timeout
    originalDebug := globalConfig.Debug

    // Restore originals when test ends
    t.Cleanup(func() {
        globalConfig.Timeout = originalTimeout
        globalConfig.Debug = originalDebug
    })

    // Set test values
    globalConfig.Timeout = 1 * time.Second
    globalConfig.Debug = true

    // Run test
    result := functionThatUsesGlobalConfig()
    if result != "expected" {
        t.Errorf("got %q", result)
    }
}

// Better: inject config as parameter
type Config struct {
    Timeout time.Duration
    Debug   bool
}

func ProcessWithConfig(cfg Config, input string) string {
    // uses cfg, not global state — trivially testable
}

func TestProcessWithConfig(t *testing.T) {
    cfg := Config{Timeout: 1 * time.Second, Debug: true}
    result := ProcessWithConfig(cfg, "input")
    if result != "expected" {
        t.Errorf("got %q", result)
    }
}
```

---

**Scenario 15: How do you test that a function returns the correct errors?**

```go
func TestCreateOrder_Errors(t *testing.T) {
    tests := []struct {
        name      string
        input     CreateOrderRequest
        setupMock func(*MockStore)
        wantErr   error      // exact sentinel error
        wantErrAs interface{} // type check
    }{
        {
            name:    "empty customer ID",
            input:   CreateOrderRequest{CustomerID: 0},
            wantErr: ErrInvalidInput,
        },
        {
            name:  "customer not found",
            input: CreateOrderRequest{CustomerID: 999},
            setupMock: func(m *MockStore) {
                m.GetCustomerErr = ErrNotFound
            },
            wantErr: ErrNotFound,
        },
        {
            name:  "database error",
            input: CreateOrderRequest{CustomerID: 1, Items: validItems},
            setupMock: func(m *MockStore) {
                m.SaveOrderErr = errors.New("pq: deadlock detected")
            },
            wantErrAs: &DBError{},
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            mock := &MockStore{}
            if tc.setupMock != nil {
                tc.setupMock(mock)
            }
            svc := NewOrderService(mock)

            _, err := svc.CreateOrder(context.Background(), tc.input)

            if err == nil {
                t.Fatal("expected error, got nil")
            }

            if tc.wantErr != nil && !errors.Is(err, tc.wantErr) {
                t.Errorf("expected errors.Is(%v), got: %v", tc.wantErr, err)
            }

            if tc.wantErrAs != nil {
                if !errors.As(err, &tc.wantErrAs) {
                    t.Errorf("expected errors.As(%T), got: %T: %v", tc.wantErrAs, err, err)
                }
            }
        })
    }
}
```
