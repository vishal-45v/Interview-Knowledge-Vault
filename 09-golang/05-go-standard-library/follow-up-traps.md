# Go Standard Library — Follow-Up Traps

## Trap 1: `http.DefaultClient` Has No Timeout

**The trap:** Using `http.Get` or `http.DefaultClient` in production without a timeout. A slow or unresponsive server will cause your goroutine to block indefinitely.

```go
// DANGEROUS — no timeout, can hang forever
resp, err := http.Get("https://slow-service.example.com/data")

// Also dangerous — DefaultClient has no timeout
resp, err := http.DefaultClient.Get(url)
```

**Fix:** Always create a custom `http.Client` with a `Timeout`:

```go
client := &http.Client{
    Timeout: 30 * time.Second, // entire request: connect + TLS + read
}
resp, err := client.Get(url)
```

Or use context for per-request control:
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

---

## Trap 2: Response Body Must Be Closed

**The trap:** Not closing `resp.Body` causes the underlying TCP connection to never return to the connection pool, leaking connections.

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
// BUG: body never closed — connection leaked
data, _ := io.ReadAll(resp.Body)
```

**Fix:** Always `defer resp.Body.Close()` immediately after the nil error check:

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close() // ← close here, not at end of function

data, err := io.ReadAll(resp.Body)
```

**Additional trap:** Always read (drain) the body before closing, even if you don't need it. This allows the connection to be reused:

```go
// If you don't need the body but want connection reuse:
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

---

## Trap 3: `json.Unmarshal` into a Nil Pointer

**The trap:** Passing a nil pointer to `json.Unmarshal` returns an `InvalidUnmarshalError` but doesn't panic — the data is silently lost.

```go
var user *User // nil pointer

err := json.Unmarshal(data, user) // passes nil pointer
// err = &json.InvalidUnmarshalError{Type: reflect.TypeOf(user)}
// user is still nil — data was not decoded

fmt.Println(err) // "json: Unmarshal(nil *main.User)"
```

**Fix:** Always pass a non-nil pointer to a value:

```go
var user User
err := json.Unmarshal(data, &user) // &user is non-nil pointer to User

// Or allocate:
user := new(User) // *User pointing to a zero-value User
err := json.Unmarshal(data, user) // user is non-nil
```

---

## Trap 4: JSON Numbers Decoded as `float64` When Using `interface{}`

**The trap:** When unmarshaling JSON into `interface{}`, all numbers become `float64`, not `int`. This causes silent precision loss for large integers.

```go
var result interface{}
json.Unmarshal([]byte(`{"id": 9007199254740993, "count": 42}`), &result)

m := result.(map[string]interface{})
id := m["id"].(float64)    // float64 — precision LOST for large int64!
count := m["count"].(float64) // 42.0 — works for small numbers

// Type assertion to int fails:
n, ok := m["count"].(int) // ok = false! It's float64, not int
```

**Fix 1:** Use typed structs:
```go
var result struct {
    ID    int64 `json:"id"`
    Count int   `json:"count"`
}
json.Unmarshal(data, &result) // correct types
```

**Fix 2:** Use `json.Decoder` with `UseNumber()`:
```go
dec := json.NewDecoder(strings.NewReader(jsonStr))
dec.UseNumber() // numbers decoded as json.Number, not float64

var result map[string]interface{}
dec.Decode(&result)

id, _ := result["id"].(json.Number).Int64() // safe int64
```

---

## Trap 5: `time.Time` Zero Value and JSON Marshaling

**The trap:** A zero-value `time.Time` marshals to `"0001-01-01T00:00:00Z"` in JSON, which looks like a real timestamp to clients.

```go
type Event struct {
    Name        string    `json:"name"`
    CompletedAt time.Time `json:"completed_at"`
}

e := Event{Name: "deploy"}
data, _ := json.Marshal(e)
// {"name":"deploy","completed_at":"0001-01-01T00:00:00Z"}
// Clients see year 1 — very confusing!
```

**Fix:** Use a pointer + `omitempty` for optional timestamps:
```go
type Event struct {
    Name        string     `json:"name"`
    CompletedAt *time.Time `json:"completed_at,omitempty"` // nil → omitted
}

e := Event{Name: "deploy"} // CompletedAt is nil
data, _ := json.Marshal(e)
// {"name":"deploy"}

now := time.Now()
e.CompletedAt = &now
data, _ = json.Marshal(e)
// {"name":"deploy","completed_at":"2025-03-17T10:00:00Z"}
```

---

## Trap 6: `os.Open` vs `os.Create` — Read-Only vs Create/Truncate

**The trap:** Using `os.Create` when you intend to read, or using `os.Open` when you intend to write.

```go
// os.Open: READ ONLY
f, err := os.Open("config.json")
f.Write([]byte("data")) // error: write /config.json: bad file descriptor

// os.Create: CREATE or TRUNCATE (destroys existing content!)
f, err := os.Create("important.log")
// If important.log existed with 10MB of logs — now it's EMPTY
```

**Fix:** Match the function to your intent:

```go
// Read existing file
f, err := os.Open("input.txt")

// Create new or truncate existing
f, err := os.Create("output.txt")

// Append to existing (or create if not exists)
f, err := os.OpenFile("log.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

// Read-write without truncating (file must exist)
f, err := os.OpenFile("data.bin", os.O_RDWR, 0)

// Create only if not exists (fail if exists)
f, err := os.OpenFile("lock", os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0600)
```

---

## Trap 7: `bufio.Scanner` Panics on Lines Longer Than 64KB

**The trap:** `bufio.Scanner` has a default max token size of 64KB. Lines longer than this cause `scanner.Scan()` to return false, `scanner.Err()` returns `bufio.ErrTooLong`, but the truncated call chain can look like a panic in some versions or simply cause silent data loss if `Err()` is not checked.

```go
scanner := bufio.NewScanner(bigFile)
for scanner.Scan() {
    // If a line is > 64KB, this loop silently stops!
}
if err := scanner.Err(); err != nil {
    // err = bufio.ErrTooLong — easy to miss if you don't check
    log.Fatal(err)
}
```

**Fix:** Set a larger buffer when dealing with potentially long lines:
```go
scanner := bufio.NewScanner(bigFile)
const maxLineSize = 1024 * 1024 // 1MB
buf := make([]byte, maxLineSize)
scanner.Buffer(buf, maxLineSize)

for scanner.Scan() {
    // now handles lines up to 1MB
}
```

---

## Trap 8: `http.Get` Does Not Support Custom Headers — Use `http.NewRequest`

**The trap:** `http.Get`, `http.Post`, `http.PostForm` are convenience functions that do not allow setting custom headers (Authorization, Content-Type, Accept, etc.).

```go
// WRONG — no way to set Authorization header
resp, err := http.Get("https://api.example.com/data")

// CORRECT — build request manually, set headers
req, err := http.NewRequest(http.MethodGet, "https://api.example.com/data", nil)
if err != nil {
    return err
}
req.Header.Set("Authorization", "Bearer "+token)
req.Header.Set("Accept", "application/json")

resp, err := client.Do(req)
```

For context support, use `http.NewRequestWithContext`:
```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
req.Header.Set("Authorization", "Bearer "+token)
resp, err := client.Do(req)
```

---

## Trap 9: `fmt.Println` vs `log.Println` — Differences

**The trap:** Treating them as equivalent.

```go
fmt.Println("server started")    // writes to stdout, no timestamp, no prefix
log.Println("server started")    // writes to stderr, with timestamp: 2025/03/17 10:00:00 server started
```

Key differences:
| | `fmt.Println` | `log.Println` |
|---|---|---|
| Output | `os.Stdout` | `os.Stderr` |
| Timestamp | No | Yes (by default) |
| File/line | No | Optional (`log.SetFlags`) |
| Thread-safe | No | Yes (mutex-protected) |
| Goroutine safe | No | Yes |

In production code, use `log` or `log/slog`, not `fmt.Println`, for diagnostic output. `fmt.Println` is for user-facing output (CLI programs).

---

## Trap 10: `sort.Slice` Is Not Stable — Use `sort.SliceStable`

**The trap:** Using `sort.Slice` when you need to preserve the original order of equal elements.

```go
type Task struct {
    Priority int
    Name     string
    Order    int // original insertion order
}

tasks := []Task{
    {1, "A", 0},
    {2, "B", 1},
    {1, "C", 2}, // same priority as A
}

// sort.Slice — NOT stable: A and C might swap even though priority is equal
sort.Slice(tasks, func(i, j int) bool {
    return tasks[i].Priority < tasks[j].Priority
})
// Order of A and C is UNDEFINED

// sort.SliceStable — preserves original order for equal elements
sort.SliceStable(tasks, func(i, j int) bool {
    return tasks[i].Priority < tasks[j].Priority
})
// A always comes before C (they have equal priority, original order preserved)
```

Use `sort.SliceStable` whenever:
- You sort the same slice multiple times by different keys
- The user expects stable ordering
- You need deterministic output for equal elements

---

## Trap 11: Not Draining Response Body Before Close Prevents Connection Reuse

**The trap:** Closing `resp.Body` without reading it fully prevents the HTTP/1.1 connection from being reused, because the server might be mid-stream.

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close() // ← closes without draining

if resp.StatusCode != 200 {
    return fmt.Errorf("HTTP %d", resp.StatusCode)
    // Body not read — this close does NOT return connection to pool
}
```

**Fix:** Always drain before closing for connection reuse:

```go
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()

if resp.StatusCode != 200 {
    return fmt.Errorf("HTTP %d", resp.StatusCode)
    // Now the body IS drained, connection IS reused
}
```

---

## Trap 12: `context.WithCancel` Leak — Always Call the Cancel Function

**The trap:** Not calling the cancel function returned by `context.WithCancel`, `context.WithTimeout`, or `context.WithDeadline`. This leaks the context's goroutine and associated resources until the parent context is cancelled.

```go
// LEAK — cancel never called
func fetchData(url string) ([]byte, error) {
    ctx, _ := context.WithTimeout(context.Background(), 5*time.Second)
    // ↑ _ discards cancel — resource leak!
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    resp, err := client.Do(req)
    // ...
}

// CORRECT — always defer cancel
func fetchData(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel() // ← always call cancel, even if timeout fires first
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    resp, err := client.Do(req)
    // ...
}
```

The `vet` tool and `staticcheck` will warn about uncalled cancel functions.
