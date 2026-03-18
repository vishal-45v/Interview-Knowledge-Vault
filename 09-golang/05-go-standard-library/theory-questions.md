# Go Standard Library — Theory Questions

## fmt Package

**Q1. What are the main formatting functions in the `fmt` package? Describe each.**

```go
// Print to stdout
fmt.Print("hello")                    // no newline, no formatting
fmt.Println("hello", "world")         // space-separated, newline at end
fmt.Printf("name: %s, age: %d\n", name, age) // formatted, no auto newline

// Format to string (S prefix)
s := fmt.Sprintf("id=%d", 42)        // returns string, no output
s := fmt.Sprint("hello", " ", "world") // returns space-separated string
s := fmt.Sprintln("hello")            // returns with newline

// Format to writer (F prefix)
fmt.Fprintf(os.Stderr, "error: %v\n", err)  // writes to any io.Writer
fmt.Fprintln(w, "response line")

// Scan from stdin
var name string
fmt.Scan(&name)        // reads whitespace-separated tokens
fmt.Scanf("%s", &name) // reads with format
fmt.Scanln(&name)      // reads until newline
```

Common verbs: `%v` (default), `%+v` (struct with field names), `%#v` (Go syntax), `%T` (type), `%d` (decimal), `%s` (string), `%q` (quoted string), `%p` (pointer), `%f` (float), `%e` (scientific notation), `%b` (binary), `%x` (hex).

---

**Q2. What is the `Stringer` interface and how does `fmt` use it?**

```go
type Stringer interface {
    String() string
}
```

When `fmt` formats a value with `%v` or `%s`, it checks if the value implements `Stringer`. If it does, it calls `String()` instead of using default struct formatting:

```go
type Color struct {
    R, G, B uint8
}

func (c Color) String() string {
    return fmt.Sprintf("#%02x%02x%02x", c.R, c.G, c.B)
}

c := Color{255, 128, 0}
fmt.Println(c)          // #ff8000
fmt.Printf("%v\n", c)  // #ff8000
fmt.Printf("%s\n", c)  // #ff8000
```

Similarly, `error` (which has `Error() string`) is printed using its `Error()` method.

---

**Q3. What is `fmt.Errorf` and what does `%w` do?**

`fmt.Errorf` creates a new formatted error. The `%w` verb wraps an existing error, making it accessible via `errors.Unwrap()`, `errors.Is()`, and `errors.As()`:

```go
originalErr := errors.New("connection refused")
wrappedErr := fmt.Errorf("dial database at %s: %w", addr, originalErr)

fmt.Println(wrappedErr)                       // "dial database at localhost:5432: connection refused"
fmt.Println(errors.Is(wrappedErr, originalErr)) // true — chain traversal works
```

---

## io Package

**Q4. What is `io.Reader` and why is it so powerful?**

```go
type Reader interface {
    Read(p []byte) bool // returns n int, err error
}
```

Correct signature:
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

`io.Reader` is the fundamental abstraction for reading sequential bytes. Any source of bytes — a file, network connection, string, gzip stream, HTTP response body, in-memory buffer — can implement it. Functions that accept `io.Reader` work with **any** of these sources without modification:

```go
// This function works with files, HTTP responses, strings — anything
func countLines(r io.Reader) (int, error) {
    scanner := bufio.NewScanner(r)
    count := 0
    for scanner.Scan() {
        count++
    }
    return count, scanner.Err()
}

// All of these work:
countLines(os.Stdin)
countLines(resp.Body)
countLines(strings.NewReader("line1\nline2\n"))
countLines(bytes.NewBuffer(data))
```

---

**Q5. What is `io.Writer` and how does it complement `io.Reader`?**

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`io.Writer` is the symmetric counterpart for writing bytes to any destination: files, network connections, buffers, gzip compressors, HTTP response writers, hash functions. Anything that accepts `io.Writer` works with all of these.

```go
// Write JSON to any writer
func writeJSON(w io.Writer, v interface{}) error {
    return json.NewEncoder(w).Encode(v)
}

// Works with file, HTTP response, buffer
writeJSON(file, data)
writeJSON(responseWriter, data)
writeJSON(&buf, data)
```

---

**Q6. What is `io.ReadCloser` and why does it exist?**

```go
type ReadCloser interface {
    Reader
    Closer
}
```

`io.ReadCloser` combines reading with closing. It is used when the underlying resource must be explicitly closed to release resources (connections, file descriptors). The most common example is `http.Response.Body`:

```go
resp, err := http.Get("https://example.com")
if err != nil {
    return err
}
defer resp.Body.Close() // MUST close — returns connection to pool

body, err := io.ReadAll(resp.Body)
```

---

**Q7. What does `io.TeeReader` do?**

`io.TeeReader` wraps a reader and copies everything read from it to a writer simultaneously — like a T-junction in a pipe:

```go
var buf bytes.Buffer
tee := io.TeeReader(resp.Body, &buf)

// Reading from tee also writes to buf
n, err := io.Copy(io.Discard, tee) // reads all

// buf now contains exactly what was read
fmt.Println("body was:", buf.String())
```

Useful for: logging request/response bodies while still processing them, computing checksums while reading, debugging.

---

**Q8. What does `io.Copy` do and what are its performance characteristics?**

```go
func Copy(dst Writer, src Reader) (written int64, err error)
```

`io.Copy` copies from src to dst using an internal 32KB buffer. It is the idiomatic way to transfer data between readers and writers without loading everything into memory:

```go
// Copy a file
src, _ := os.Open("input.txt")
defer src.Close()
dst, _ := os.Create("output.txt")
defer dst.Close()

n, err := io.Copy(dst, src)
fmt.Printf("copied %d bytes\n", n)
```

If either end implements `WriterTo` or `ReaderFrom`, `io.Copy` delegates to those for potentially more efficient transfer (e.g., zero-copy with `sendfile` on Linux).

---

## bufio Package

**Q9. What is the `bufio` package for and why is it important?**

`bufio` wraps an `io.Reader` or `io.Writer` with an in-memory buffer. Instead of making a syscall for every byte, it batches reads and writes into larger chunks:

```go
// Without bufio: one syscall per Read call to the underlying reader
// With bufio: reads a chunk at a time, serves subsequent calls from buffer

r := bufio.NewReader(file)      // default 4096-byte buffer
line, err := r.ReadString('\n') // reads efficiently

w := bufio.NewWriter(file)      // default 4096-byte buffer
w.WriteString("hello\n")        // written to buffer, not file yet
w.Flush()                       // flush buffer to file
```

---

**Q10. How does `bufio.Scanner` work and what is its token-size limitation?**

`bufio.Scanner` reads line by line (or by any split function):

```go
scanner := bufio.NewScanner(r)
for scanner.Scan() {
    line := scanner.Text() // current line (without newline)
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}
```

**Default token size limit: 64KB**. If a single line is longer than 64KB, `scanner.Scan()` returns false and `scanner.Err()` returns `bufio.ErrTooLong`.

To increase it:
```go
scanner := bufio.NewScanner(r)
scanner.Buffer(make([]byte, 1024*1024), 1024*1024) // 1MB max token
```

---

## os Package

**Q11. What is the difference between `os.Open` and `os.Create`?**

```go
// os.Open: opens for reading ONLY. Returns *os.File
f, err := os.Open("input.txt")
// equivalent to: os.OpenFile("input.txt", os.O_RDONLY, 0)

// os.Create: creates or TRUNCATES for read-write. Returns *os.File
f, err := os.Create("output.txt")
// equivalent to: os.OpenFile("output.txt", os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0666)

// For append without truncate:
f, err := os.OpenFile("log.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
```

Common mistake: using `os.Create` when you only want to create if it doesn't exist — `os.Create` will **truncate** an existing file.

---

**Q12. How do you work with environment variables and program arguments in Go?**

```go
// Environment variables
val := os.Getenv("DATABASE_URL")      // returns "" if not set
val, ok := os.LookupEnv("DATABASE_URL") // returns ("", false) if not set
os.Setenv("FOO", "bar")
os.Unsetenv("FOO")
envs := os.Environ() // []string{"KEY=value", ...}

// Program arguments
args := os.Args        // []string: os.Args[0] is program name
args := os.Args[1:]    // all flags/arguments passed

// Exit
os.Exit(0)   // success
os.Exit(1)   // failure — does NOT run deferred functions
```

---

## net/http Package

**Q13. How does the `net/http` server work? Explain `Handler`, `ServeMux`, and `ListenAndServe`.**

```go
// Handler interface — anything that handles HTTP requests
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// HandlerFunc adapter — makes a function satisfy Handler
type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

// ServeMux — request router (multiplexer)
mux := http.NewServeMux()
mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
})
mux.Handle("/api/users", userHandler) // userHandler implements http.Handler

// ListenAndServe — starts the server
// Blocks until error
err := http.ListenAndServe(":8080", mux)
```

For production, use a custom `http.Server` with timeouts (see Q14 below).

---

**Q14. What are the critical timeouts on `http.Server` and why does `http.DefaultServer` lack them?**

`http.DefaultServer` (used by `http.ListenAndServe`) has **no timeouts**. This means:
- A slow client can hold a connection open indefinitely
- A request handler that takes forever is never cancelled
- A server can run out of goroutines under a slow-loris attack

Always use a custom server:

```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadTimeout:       5 * time.Second,   // time to read request headers + body
    ReadHeaderTimeout: 2 * time.Second,   // time to read request headers only
    WriteTimeout:      10 * time.Second,  // time to write response
    IdleTimeout:       120 * time.Second, // keep-alive connection idle time
    MaxHeaderBytes:    1 << 20,           // 1MB max header size
}
log.Fatal(server.ListenAndServe())
```

---

**Q15. How do you configure `http.Client` for production use?**

`http.DefaultClient` has **no timeout** — a request to a slow server can hang forever. Configure explicitly:

```go
client := &http.Client{
    Timeout: 30 * time.Second, // total request timeout (connect + read + write)
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,  // TCP connection timeout
            KeepAlive: 30 * time.Second, // TCP keep-alive
        }).DialContext,
        MaxIdleConns:          100,             // total idle connections in pool
        MaxIdleConnsPerHost:   10,              // idle connections per host
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   10 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
        ResponseHeaderTimeout: 10 * time.Second,
    },
}
```

---

## encoding/json Package

**Q16. How does `encoding/json` work with struct tags?**

JSON struct tags control how fields are serialized and deserialized:

```go
type User struct {
    ID        int64   `json:"id"`
    Name      string  `json:"name"`
    Email     string  `json:"email"`
    Password  string  `json:"-"`           // never serialized
    CreatedAt time.Time `json:"created_at"`
    Address   *Address `json:"address,omitempty"` // omit if nil
    Score     float64  `json:"score,omitempty"`   // omit if 0.0
}

// Marshal — Go struct → JSON bytes
data, err := json.Marshal(user)

// Unmarshal — JSON bytes → Go struct
var u User
err := json.Unmarshal(data, &u)

// Streaming — for large data
enc := json.NewEncoder(w)
enc.SetIndent("", "  ") // pretty print
err := enc.Encode(user)

dec := json.NewDecoder(r)
err := dec.Decode(&u)
```

---

**Q17. How do you implement custom JSON marshaling?**

Implement `json.Marshaler` and/or `json.Unmarshaler`:

```go
type Duration struct {
    time.Duration
}

// Custom marshal: output as string "1h30m"
func (d Duration) MarshalJSON() ([]byte, error) {
    return json.Marshal(d.Duration.String())
}

// Custom unmarshal: parse "1h30m" → time.Duration
func (d *Duration) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    dur, err := time.ParseDuration(s)
    if err != nil {
        return fmt.Errorf("parse duration %q: %w", s, err)
    }
    d.Duration = dur
    return nil
}

type Config struct {
    Timeout Duration `json:"timeout"`
}
// {"timeout": "30s"} ↔ Config{Timeout: Duration{30 * time.Second}}
```

---

## context Package

**Q18. How does `context` integrate with standard library HTTP calls?**

Context is passed through `http.Request` and used for cancellation and deadlines:

```go
// Server side: request has a context attached automatically
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // cancelled when client disconnects or deadline exceeded

    result, err := db.QueryContext(ctx, "SELECT ...")
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected
            return
        }
        http.Error(w, "db error", 500)
        return
    }
    _ = result
}

// Client side: pass context to control request lifetime
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}
resp, err := client.Do(req)
```

---

## time Package

**Q19. What are the key time operations in Go?**

```go
// Current time
now := time.Now()                    // local time
utc := time.Now().UTC()              // UTC

// Durations
d := 5 * time.Second                 // time.Duration (int64 nanoseconds)
d := 2*time.Hour + 30*time.Minute
d := time.Since(startTime)           // elapsed time
d := time.Until(deadline)            // time remaining

// Arithmetic
later := now.Add(24 * time.Hour)
diff := laterTime.Sub(earlierTime)   // returns Duration

// Formatting and parsing (reference time: Mon Jan 2 15:04:05 MST 2006)
s := now.Format("2006-01-02T15:04:05Z07:00")  // ISO 8601
t, err := time.Parse("2006-01-02", "2025-03-17")

// Timers and tickers
timer := time.NewTimer(5 * time.Second)
<-timer.C       // blocks until 5 seconds pass
timer.Stop()    // cancel

ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
for t := range ticker.C {
    fmt.Println("tick at", t)
}

// time.After (convenience, creates timer)
select {
case result := <-ch:
    // got result
case <-time.After(5 * time.Second):
    // timed out
}
```

---

**Q20. What is the zero value of `time.Time` and why does it matter for JSON?**

The zero value of `time.Time` is January 1, year 1, 00:00:00 UTC. When marshaled to JSON, it becomes `"0001-01-01T00:00:00Z"`.

This can cause unexpected behavior in JSON responses when time fields are not initialized:

```go
type Event struct {
    Name      string    `json:"name"`
    CompletedAt time.Time `json:"completed_at"`           // zero value marshals as "0001-01-01T00:00:00Z"
    CompletedAt *time.Time `json:"completed_at,omitempty"` // nil pointer omitted
}

// Safe pattern: use pointer + omitempty for optional times
type Event struct {
    Name        string     `json:"name"`
    CompletedAt *time.Time `json:"completed_at,omitempty"`
}
```

---

## log/slog Package (Go 1.21+)

**Q21. What is `log/slog` and how does it differ from the `log` package?**

`log/slog` provides **structured logging** — each log entry is a set of key-value pairs, not just a message string. This makes logs machine-parseable.

```go
// Old log package — unstructured
log.Printf("user %d logged in from %s", userID, ip)
// Output: 2025/03/17 10:00:00 user 42 logged in from 192.168.1.1

// slog — structured
slog.Info("user logged in", "user_id", userID, "ip", ip)
// Output (text): 2025/03/17T10:00:00.000Z INFO user logged in user_id=42 ip=192.168.1.1
// Output (JSON): {"time":"2025-03-17T10:00:00Z","level":"INFO","msg":"user logged in","user_id":42,"ip":"192.168.1.1"}

// JSON handler for production
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

// Levels: Debug, Info, Warn, Error
slog.Debug("processing item", "id", itemID)
slog.Error("db query failed", "err", err, "query", query)

// Groups for namespacing
logger.With(slog.Group("request",
    "method", r.Method,
    "path", r.URL.Path,
)).Info("request received")
```

---

## strings and strconv Packages

**Q22. What are the most commonly used `strings` package functions?**

```go
import "strings"

strings.Contains("seafood", "foo")       // true
strings.HasPrefix("seafood", "sea")      // true
strings.HasSuffix("seafood", "food")     // true
strings.Index("seafood", "foo")          // 3
strings.Count("cheese", "e")             // 3
strings.Replace("oink oink oink", "oink", "moo", 2)  // "moo moo oink" (2 replacements)
strings.ReplaceAll("oink oink", "oink", "moo")        // "moo moo"
strings.Split("a,b,c", ",")              // ["a","b","c"]
strings.Join([]string{"a","b"}, "-")     // "a-b"
strings.TrimSpace("  hello  ")           // "hello"
strings.Trim("##hello##", "#")           // "hello"
strings.ToLower("HELLO")                 // "hello"
strings.ToUpper("hello")                 // "HELLO"
strings.Fields("  foo bar  baz  ")       // ["foo","bar","baz"]

// Builder — efficient string concatenation
var sb strings.Builder
for i := 0; i < 1000; i++ {
    sb.WriteString("item ")
    sb.WriteString(strconv.Itoa(i))
    sb.WriteByte('\n')
}
result := sb.String()
```

---

**Q23. What are the key `strconv` functions?**

```go
import "strconv"

// Integer ↔ string
s := strconv.Itoa(42)                  // "42"
n, err := strconv.Atoi("42")          // 42, nil

// More control
n64, err := strconv.ParseInt("0xFF", 16, 64)    // hex parse
n64, err := strconv.ParseInt("-128", 10, 8)     // base 10, 8-bit
s := strconv.FormatInt(255, 16)                 // "ff"

// Float
f, err := strconv.ParseFloat("3.14", 64)
s := strconv.FormatFloat(3.14159, 'f', 2, 64)  // "3.14"

// Bool
b, err := strconv.ParseBool("true")   // true, nil
s := strconv.FormatBool(true)          // "true"

// Quote/unquote
s := strconv.Quote("hello\nworld")    // "\"hello\\nworld\""
```

---

## sort Package

**Q24. How do you sort slices and custom types in Go?**

```go
import "sort"

// Built-in type slices
ints := []int{5, 2, 4, 1, 3}
sort.Ints(ints)     // [1 2 3 4 5]

strs := []string{"banana", "apple", "cherry"}
sort.Strings(strs)  // [apple banana cherry]

// Custom sort with sort.Slice (not stable)
people := []Person{{Name: "Alice", Age: 30}, {Name: "Bob", Age: 25}}
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age // sort by Age ascending
})

// Stable sort preserves original order for equal elements
sort.SliceStable(people, func(i, j int) bool {
    return people[i].Name < people[j].Name
})

// sort.Search — binary search (returns index where f first becomes true)
idx := sort.Search(len(ints), func(i int) bool {
    return ints[i] >= 3
})
// idx is the position where 3 would be inserted

// sort.Interface — implement for full control
type ByLength []string
func (s ByLength) Len() int           { return len(s) }
func (s ByLength) Less(i, j int) bool { return len(s[i]) < len(s[j]) }
func (s ByLength) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

sort.Sort(ByLength(strs))
```

---

## filepath Package

**Q25. What does the `filepath` package provide and when should you use it instead of `path`?**

`filepath` works with OS-specific path separators (`/` on Unix, `\` on Windows). `path` always uses `/` and is for URL paths.

```go
import "path/filepath"

// Building paths
p := filepath.Join("dir", "subdir", "file.txt")  // "dir/subdir/file.txt" (or backslash on Windows)

// Splitting
dir := filepath.Dir("/home/user/file.txt")    // "/home/user"
base := filepath.Base("/home/user/file.txt")  // "file.txt"
ext := filepath.Ext("file.txt")               // ".txt"

// Cleaning/normalizing
clean := filepath.Clean("dir/../other/./file.txt") // "other/file.txt"

// Walking a directory tree
err := filepath.WalkDir(".", func(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err
    }
    if !d.IsDir() && filepath.Ext(path) == ".go" {
        fmt.Println(path)
    }
    return nil
})

// Glob matching
matches, err := filepath.Glob("*.go")
```

---

**Q26. What is `io.ReadAll` and what replaced `ioutil.ReadAll`?**

In Go 1.16, `ioutil` was deprecated. Its functions moved to `io` and `os`:

```go
// Old (deprecated)
import "io/ioutil"
data, err := ioutil.ReadAll(r)
data, err := ioutil.ReadFile("file.txt")
err := ioutil.WriteFile("file.txt", data, 0644)
files, err := ioutil.ReadDir(".")

// New (Go 1.16+)
import "io"
import "os"
data, err := io.ReadAll(r)
data, err := os.ReadFile("file.txt")
err := os.WriteFile("file.txt", data, 0644)
entries, err := os.ReadDir(".")
```

---

**Q27. How does `sync/atomic` compare to mutex-protected variables?**

`sync/atomic` provides lock-free atomic operations on primitive types, using CPU-level atomic instructions:

```go
import "sync/atomic"

var counter int64

// Atomic increment — no lock needed
atomic.AddInt64(&counter, 1)

// Atomic read — safe to read while others write
val := atomic.LoadInt64(&counter)

// Compare-and-swap — used for lock-free data structures
old := atomic.LoadInt64(&counter)
swapped := atomic.CompareAndSwapInt64(&counter, old, old*2)

// Go 1.19+: typed atomic (safer API)
var atomicCounter atomic.Int64
atomicCounter.Add(1)
current := atomicCounter.Load()
```

Use `sync/atomic` for simple counters and flags. Use `sync.Mutex` when you need to protect more complex invariants or need to lock multiple variables together.
