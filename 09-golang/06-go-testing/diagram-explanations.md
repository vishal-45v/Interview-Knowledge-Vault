# Go Testing — Diagram Explanations

## Diagram 1: Table-Driven Test Structure

```
TABLE-DRIVEN TEST ANATOMY:

func TestIsValid(t *testing.T) {
    tests := []struct {                 ← test case struct
        name    string                  ← subtest name (used in -run filter)
        input   string                  ← inputs to the function
        want    bool                    ← expected output
        wantErr bool                    ← expected error behavior
    }{
        ┌─────────────────────────────────────────────────────────┐
        │ {"valid email", "a@b.com", true, false},                │ case 1
        │ {"missing @", "ab.com", false, true},                  │ case 2
        │ {"empty", "", false, true},                            │ case 3
        └─────────────────────────────────────────────────────────┘
    }

    for _, tc := range tests {          ← iterate over all cases
        tc := tc                         ← capture for parallel subtests
        t.Run(tc.name, func(t *testing.T) {  ← create named subtest
            got, err := IsValid(tc.input)

            if tc.wantErr && err == nil {
                t.Error("expected error, got nil")
            }
            if !tc.wantErr && err != nil {
                t.Errorf("unexpected error: %v", err)
            }
            if got != tc.want {
                t.Errorf("IsValid(%q) = %v; want %v", tc.input, got, tc.want)
            }
        })
    }
}

─────────────────────────────────────────────────────────────────

GO TEST OUTPUT:

--- FAIL: TestIsValid (0.00s)
    --- PASS: TestIsValid/valid_email (0.00s)    ← each case named
    --- FAIL: TestIsValid/missing_@ (0.00s)      ← clear which failed
        validator_test.go:30: IsValid("ab.com") = true; want false
    --- PASS: TestIsValid/empty (0.00s)

RUN SPECIFIC CASE: go test -run "TestIsValid/valid_email"
RUN BY PATTERN:    go test -run "TestIsValid/valid"
```

---

## Diagram 2: Test Pyramid for Go

```
            ┌──────────────────────────────────────────────┐
            │                                              │
            │            End-to-End Tests                  │
            │     (full system: real DB, real network)     │ FEW
            │     - testcontainers-go                      │
            │     - real HTTP server + client              │
            │     - production-like environment            │
            │                                              │
            ├──────────────────────────────────────────────┤
            │                                              │
            │         Integration Tests                    │
            │   (two or more real components together)     │ SOME
            │   - handler + real store (in-memory DB)      │
            │   - httptest.NewServer + real client         │
            │   - TestMain with test database              │
            │                                              │
            ├──────────────────────────────────────────────┤
            │                                              │
            │              Unit Tests                      │
            │    (one component, all deps mocked)          │ MANY
            │    - table-driven tests                      │
            │    - interface mocks                         │
            │    - httptest.NewRecorder                    │
            │    - t.Parallel() for speed                  │
            │                                              │
            └──────────────────────────────────────────────┘

CHARACTERISTICS:

Unit Tests:
  Speed:       < 1ms per test      ← fast feedback
  Isolation:   complete            ← deterministic, no flakiness
  Coverage:    high                ← test every branch
  Command:     go test ./...

Integration Tests:
  Speed:       10ms - 1s per test  ← moderate
  Isolation:   partial             ← real I/O, possible flakiness
  Coverage:    medium              ← happy paths + key error paths
  Command:     go test -tags=integration ./...

End-to-End Tests:
  Speed:       1s - 30s per test   ← slow
  Isolation:   none                ← real environment
  Coverage:    low (but important) ← smoke test critical flows
  Command:     go test -tags=e2e ./...

GOAL: fast feedback at the bottom, confidence at the top.
```

---

## Diagram 3: httptest.NewServer Flow

```
TEST CODE                          TEST SERVER                    HTTP CLIENT
─────────────────────────────────────────────────────────────────────────────

srv := httptest.NewServer(handler)
          │
          ▼
  Starts real HTTP server
  on random port: 127.0.0.1:PORT
  (random unused port)
          │
          ▼
  srv.URL = "http://127.0.0.1:PORT"
          │
          │ pass srv.URL to client
          ▼
                                                    client := NewClient(srv.URL)
                                                              │
                                                    client.GetUser(ctx, 42)
                                                              │
                                                    GET http://127.0.0.1:PORT/users/42
                                                              │
                                  handler receives request
                                  ┌─────────────────────┐
                                  │ r.URL.Path == /users/42 │
                                  │ assert r.Header.Authorization │
                                  │ w.Write(json response) │
                                  └─────────────────────┘
                                              │
                                  Response returned
                                              │
                                              ▼
                                    client returns User{ID: 42}
                                              │
                                              ▼
                            Test asserts User.Name == "Alice"

defer srv.Close()  ← shuts down test server when test ends

─────────────────────────────────────────────────────────────────

NewRecorder vs NewServer:

  httptest.NewRecorder:           httptest.NewServer:
  ┌────────────────────┐         ┌────────────────────────────┐
  │ call handler       │         │ real TCP listener          │
  │ directly           │         │ real HTTP parsing          │
  │ (no network)       │         │ tests HTTP client code     │
  │ tests handler      │         │ tests full request/response │
  │ in isolation       │         │ cycle                      │
  │ very fast          │         │ slightly slower (TCP)      │
  └────────────────────┘         └────────────────────────────┘
         ↑                                ↑
  Use for testing              Use for testing
  HTTP handlers                HTTP clients
```

---

## Diagram 4: Benchmark Execution Loop

```
go test -bench=BenchmarkFoo -benchtime=5s

PHASE 1: Calibration (find N)

  b.N = 1
  │  run loop once → takes 5μs
  ▼
  Too fast (< 1s) → scale up N
  b.N = 100
  │  run loop 100 times → takes 0.5ms
  ▼
  Still too fast → scale up N
  b.N = 10000
  │  run loop 10000 times → takes 50ms
  ▼
  Still too fast → scale up N
  b.N = 1000000
  │  run loop 1,000,000 times → takes 5.0s
  ▼
  Time target reached → REPORT

PHASE 2: Measurement

  for i := 0; i < b.N; i++ {
       │
       ▼
    b.StopTimer() (optional: for per-iteration setup)
       │
       ▼
    setup data (not counted if StopTimer used)
       │
       ▼
    b.StartTimer() (optional)
       │
       ▼
    [YOUR CODE RUNS HERE]  ← measured
       │
       ▼
    (repeat N times)
  }

PHASE 3: Report

  BenchmarkFoo-8    1000000    5000 ns/op    1024 B/op    3 allocs/op
                    ───────    ──────────    ──────────    ────────────
                      N        total_ns/N    heap bytes   heap allocs
                               per iteration per iteration per iteration

─────────────────────────────────────────────────────────────────

KEY BENCHMARK FLAGS:

  -bench=.         run all benchmarks
  -bench=BenchFoo  run matching benchmarks
  -benchtime=5s    run each benchmark for 5 seconds
  -benchtime=100x  run exactly 100 iterations
  -benchmem        show memory stats (same as b.ReportAllocs())
  -count=5         run each benchmark 5 times (for benchstat)
  -cpu=1,2,4       run with GOMAXPROCS=1, 2, 4 (parallelism test)
```

---

## Diagram 5: Fuzz Test Corpus and Mutation Flow

```
INITIAL STATE:

  f.Add("http://example.com")      ← seed corpus entry 1
  f.Add("")                         ← seed corpus entry 2
  f.Add("ftp://user:pass@host")     ← seed corpus entry 3

  testdata/fuzz/FuzzParseURL/
  └── (empty — no saved findings yet)

─────────────────────────────────────────────────────────────────

FUZZING LOOP (go test -fuzz=FuzzParseURL):

  Pull from corpus queue
       │
  Mutate input:
  ├─► flip a bit: "http://example.com" → "http://example\x00com"
  ├─► delete char: "http://example.com" → "http://exmple.com"
  ├─► insert random: "http://example.com" → "http://\xffexample.com"
  ├─► combine two corpus entries
  └─► use coverage guidance: focus on inputs that reach new code paths
       │
       ▼
  Execute f.Fuzz(func(t *testing.T, input string) {
       │
       ├─► Panics?  ──────────────────────────────────────────────► SAVE + REPORT
       │
       ├─► t.Fatal/t.Error called?  ────────────────────────────► SAVE + REPORT
       │
       └─► Executes new code path (coverage increase)?
                │
                └─► Add to corpus queue (interesting input)
       │
       ▼
  Repeat billions of times per second

─────────────────────────────────────────────────────────────────

WHEN A FAILING INPUT IS FOUND:

  testdata/fuzz/FuzzParseURL/
  └── 1234abc  ← saved failing input (hex hash)
      (content: the exact byte sequence that caused failure)

  Future runs:
  go test -run=FuzzParseURL   ← replays ALL saved inputs as regression tests
  go test -fuzz=FuzzParseURL  ← resumes fuzzing (adds to corpus)

─────────────────────────────────────────────────────────────────

INVARIANT TESTING (what to check in fuzz body):

  f.Fuzz(func(t *testing.T, input string) {
      result, err := parse(input)
      if err != nil { return }  ← parsing errors: OK

      // These invariants must ALWAYS hold for valid parses:
      ① serialized := result.String()
        re-parsed, err := parse(serialized)
        if err != nil { t.Fatal("re-parse failed") }     ← round-trip
        if re-parsed != result { t.Error("round-trip changed value") }

      ② if result.Port < 0 || result.Port > 65535 {
          t.Error("port out of range")                    ← bounds check
        }

      ③ // should never panic after successful parse
        _ = result.Hostname()                             ← no-panic invariant
  })
```

---

## Diagram 6: Test with Mock Dependency (Interface → Mock)

```
PRODUCTION CODE:

  UserService
  ┌──────────────────────────────────────────────┐
  │  deps:                                       │
  │    repo: UserRepository (interface)          │
  │    mail: Mailer (interface)                  │
  │                                              │
  │  func Register(email, name string) error {   │
  │      user := &User{...}                      │
  │      repo.Save(user)         ◄── interface   │
  │      mail.SendWelcome(email) ◄── interface   │
  │  }                                           │
  └──────────────────────────────────────────────┘

─────────────────────────────────────────────────────────────────

PRODUCTION WIRING:

  UserService {
      repo: &PostgresUserRepo{db: realDB}   ← real database
      mail: &SMTPMailer{host: "smtp.gmail"} ← real email
  }

─────────────────────────────────────────────────────────────────

TEST WIRING:

  UserService {
      repo: &MockUserRepo{                  ← in-memory, no DB
          SaveFunc: func(u *User) error {
              recordedUsers = append(recordedUsers, u)
              return nil
          },
      },
      mail: &MockMailer{                    ← records calls, no SMTP
          calls: []MailCall{},
      },
  }

─────────────────────────────────────────────────────────────────

TEST EXECUTION FLOW:

  Test calls svc.Register("alice@example.com", "Alice")
                │
                ▼
  UserService.Register executes
                │
          ┌─────┴──────────────────────────────┐
          │                                    │
          ▼                                    ▼
  MockUserRepo.Save("alice@...")        MockMailer.SendWelcome("alice@...")
  ┌─────────────────────┐               ┌─────────────────────┐
  │ record call         │               │ record call         │
  │ return nil (success)│               │ return nil (success)│
  └─────────────────────┘               └─────────────────────┘
          │                                    │
          └──────────────────┬─────────────────┘
                             ▼
                    Register returns nil (success)
                             │
                             ▼
  Test assertions:
  ✓ err == nil
  ✓ MockUserRepo.recordedUsers has 1 entry with email "alice@example.com"
  ✓ MockMailer.calls has 1 call: {method:"SendWelcome", to:"alice@example.com"}

─────────────────────────────────────────────────────────────────

INTERFACE-BASED MOCKING SUMMARY:

  Real System:
  UserService → PostgresUserRepo → TCP → PostgreSQL → disk
  UserService → SMTPMailer → TCP → SMTP server → email

  Test System:
  UserService → MockUserRepo → (in-memory map)
  UserService → MockMailer   → (records calls only)

  Result: tests run in microseconds, deterministically,
          without network or disk access.
```
