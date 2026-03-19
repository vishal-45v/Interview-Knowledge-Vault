# Chapter 03 — Go Concurrency: Scenario Questions

---

**S1. Implement a worker pool that processes jobs concurrently with N workers.**

```go
type Job struct {
    ID      int
    Payload string
}

type Result struct {
    JobID  int
    Output string
    Err    error
}

func workerPool(ctx context.Context, numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result, numWorkers)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return // jobs channel closed
                    }
                    output, err := processJob(job)
                    results <- Result{JobID: job.ID, Output: output, Err: err}
                case <-ctx.Done():
                    return // context cancelled
                }
            }
        }(i)
    }

    // Close results when all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

func processJob(j Job) (string, error) {
    time.Sleep(10 * time.Millisecond) // simulate work
    return strings.ToUpper(j.Payload), nil
}

// Usage:
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    jobs := make(chan Job, 100)
    results := workerPool(ctx, 5, jobs)

    // Submit jobs
    go func() {
        defer close(jobs)
        for i := 0; i < 20; i++ {
            jobs <- Job{ID: i, Payload: fmt.Sprintf("job-%d", i)}
        }
    }()

    // Collect results
    for result := range results {
        if result.Err != nil {
            log.Printf("job %d failed: %v", result.JobID, result.Err)
            continue
        }
        fmt.Printf("job %d: %s\n", result.JobID, result.Output)
    }
}
```

---

**S2. You need to cancel all goroutines when one fails. How?**

Use `errgroup` with a context — when any goroutine returns an error, the context is cancelled, signalling all others to stop.

```go
import "golang.org/x/sync/errgroup"

func fetchConcurrently(ctx context.Context, urls []string) ([][]byte, error) {
    // errgroup.WithContext creates a ctx that cancels when any goroutine fails
    g, gCtx := errgroup.WithContext(ctx)
    results := make([][]byte, len(urls))

    for i, url := range urls {
        i, url := i, url // capture loop variables
        g.Go(func() error {
            // Use gCtx so if another goroutine fails, this one sees cancellation
            req, err := http.NewRequestWithContext(gCtx, "GET", url, nil)
            if err != nil {
                return fmt.Errorf("creating request for %s: %w", url, err)
            }
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            defer resp.Body.Close()
            results[i], err = io.ReadAll(resp.Body)
            return err
        })
    }

    // Wait for all goroutines — returns the first non-nil error
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}

// Manual version without errgroup:
func fetchWithCancel(ctx context.Context, urls []string) ([][]byte, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel() // cancel all on return (success or error)

    type indexedResult struct {
        index int
        data  []byte
        err   error
    }

    ch := make(chan indexedResult, len(urls))
    for i, url := range urls {
        go func(idx int, u string) {
            data, err := fetchURL(ctx, u)
            ch <- indexedResult{idx, data, err}
        }(i, url)
    }

    results := make([][]byte, len(urls))
    for range urls {
        r := <-ch
        if r.err != nil {
            cancel() // cancel remaining goroutines
            return nil, r.err
        }
        results[r.index] = r.data
    }
    return results, nil
}
```

---

**S3. Implement a rate limiter using time.Ticker and channels.**

```go
// Simple token bucket rate limiter
type RateLimiter struct {
    tokens chan struct{}
    done   chan struct{}
}

func NewRateLimiter(rps int) *RateLimiter {
    rl := &RateLimiter{
        tokens: make(chan struct{}, rps),
        done:   make(chan struct{}),
    }
    // Pre-fill bucket
    for i := 0; i < rps; i++ {
        rl.tokens <- struct{}{}
    }
    // Refill ticker
    go func() {
        ticker := time.NewTicker(time.Second / time.Duration(rps))
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                select {
                case rl.tokens <- struct{}{}:
                default: // bucket full, discard token
                }
            case <-rl.done:
                return
            }
        }
    }()
    return rl
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    select {
    case <-rl.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (rl *RateLimiter) Stop() { close(rl.done) }

// Usage
limiter := NewRateLimiter(10) // 10 requests per second
defer limiter.Stop()

for i := 0; i < 100; i++ {
    if err := limiter.Wait(ctx); err != nil {
        log.Fatal(err)
    }
    go makeAPICall(i)
}

// Alternative: use golang.org/x/time/rate
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(10), 10) // 10 rps, burst of 10
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := limiter.Wait(ctx); err != nil {
    log.Fatal("rate limit exceeded or context done:", err)
}
makeAPICall()
```

---

**S4. You have a function that calls an external API. Add a 5-second timeout.**

```go
// Using context (preferred approach)
func callAPIWithTimeout(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("API call timed out after 5s")
        }
        return nil, fmt.Errorf("API call failed: %w", err)
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

// Using http.Client timeout (simpler for single calls)
func callWithClientTimeout(url string) ([]byte, error) {
    client := &http.Client{Timeout: 5 * time.Second}
    resp, err := client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// The context approach is better because:
// 1. You can propagate the parent context (request-scoped cancellation)
// 2. You can cancel early if the caller context is cancelled
// 3. Timeout applies to the whole chain, not just the HTTP call
func handleHTTPRequest(w http.ResponseWriter, r *http.Request) {
    // Use the request's context — includes client disconnect detection
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    data, err := callAPIWithContext(ctx, "https://api.example.com/data")
    if err != nil {
        http.Error(w, err.Error(), http.StatusGatewayTimeout)
        return
    }
    w.Write(data)
}
```

---

**S5. Detect and fix a goroutine leak in this code.**

```go
// BUGGY: goroutine leak
func process(input string) string {
    resultCh := make(chan string)  // unbuffered — NO BUFFER

    go func() {
        result := heavyComputation(input) // takes up to 10 seconds
        resultCh <- result                // BLOCKS if nobody reads
    }()

    select {
    case result := <-resultCh:
        return result
    case <-time.After(2 * time.Second):
        return "timeout"
        // BUG: goroutine is still blocked trying to send to resultCh
        // resultCh goes out of scope but goroutine lives forever
    }
}

// FIX 1: Use a buffered channel (goroutine can send and exit)
func processFixed1(input string) string {
    resultCh := make(chan string, 1) // buffered — goroutine won't block

    go func() {
        resultCh <- heavyComputation(input) // won't block, just fills buffer
    }()

    select {
    case result := <-resultCh:
        return result
    case <-time.After(2 * time.Second):
        return "timeout"
        // Goroutine will send to buffer and exit cleanly
    }
}

// FIX 2: Use context for cancellation (proper approach)
func processFixed2(ctx context.Context, input string) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    resultCh := make(chan string, 1)

    go func() {
        // Pass ctx to computation so it can be cancelled
        result, err := heavyComputationCtx(ctx, input)
        if err != nil {
            return // context cancelled, don't send
        }
        select {
        case resultCh <- result:
        case <-ctx.Done(): // if nobody is reading anymore, exit
        }
    }()

    select {
    case result := <-resultCh:
        return result, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}
```

---

**S6. Implement a semaphore to limit concurrent access to a resource.**

```go
// Semaphore using a buffered channel
type Semaphore struct {
    permits chan struct{}
}

func NewSemaphore(maxConcurrent int) *Semaphore {
    return &Semaphore{permits: make(chan struct{}, maxConcurrent)}
}

func (s *Semaphore) Acquire(ctx context.Context) error {
    select {
    case s.permits <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (s *Semaphore) Release() {
    <-s.permits
}

// Usage: limit database connections to 10 concurrent
sem := NewSemaphore(10)

for _, item := range items {
    item := item
    go func() {
        if err := sem.Acquire(ctx); err != nil {
            return
        }
        defer sem.Release()
        processWithDB(item)
    }()
}

// Alternative: golang.org/x/sync/semaphore
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10)
if err := sem.Acquire(ctx, 1); err != nil {
    return err
}
defer sem.Release(1)
```

---

**S7. Implement a pub/sub system using goroutines and channels.**

```go
type PubSub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan string
}

func NewPubSub() *PubSub {
    return &PubSub{subscribers: make(map[string][]chan string)}
}

func (ps *PubSub) Subscribe(topic string, bufSize int) <-chan string {
    ch := make(chan string, bufSize)
    ps.mu.Lock()
    ps.subscribers[topic] = append(ps.subscribers[topic], ch)
    ps.mu.Unlock()
    return ch
}

func (ps *PubSub) Unsubscribe(topic string, ch <-chan string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    subs := ps.subscribers[topic]
    for i, sub := range subs {
        if sub == ch {
            ps.subscribers[topic] = append(subs[:i], subs[i+1:]...)
            close(sub)
            return
        }
    }
}

func (ps *PubSub) Publish(topic, message string) {
    ps.mu.RLock()
    subs := make([]chan string, len(ps.subscribers[topic]))
    copy(subs, ps.subscribers[topic])
    ps.mu.RUnlock()

    for _, ch := range subs {
        select {
        case ch <- message:
        default: // subscriber buffer full — skip (or handle differently)
        }
    }
}

// Usage
ps := NewPubSub()

orders := ps.Subscribe("orders", 10)
alerts := ps.Subscribe("orders", 10) // two subscribers to same topic

go func() {
    for msg := range orders {
        fmt.Println("Order processor received:", msg)
    }
}()
go func() {
    for msg := range alerts {
        fmt.Println("Alert system received:", msg)
    }
}()

ps.Publish("orders", "order-1234 placed")
ps.Publish("orders", "order-5678 placed")
```

---

**S8. Write a concurrent map-reduce implementation.**

```go
// Map: apply a function to each element concurrently
func mapConcurrent[T, R any](ctx context.Context, items []T, fn func(T) R, workers int) []R {
    results := make([]R, len(items))
    jobs := make(chan int, len(items))
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for idx := range jobs {
                results[idx] = fn(items[idx])
            }
        }()
    }

    for i := range items {
        jobs <- i
    }
    close(jobs)
    wg.Wait()
    return results
}

// Reduce: aggregate results sequentially
func reduce[T, R any](items []T, initial R, fn func(R, T) R) R {
    acc := initial
    for _, item := range items {
        acc = fn(acc, item)
    }
    return acc
}

// Example: parallel word count
func wordCountMapReduce(files []string) map[string]int {
    // Map: count words in each file
    counts := mapConcurrent(context.Background(), files, func(path string) map[string]int {
        data, _ := os.ReadFile(path)
        counts := make(map[string]int)
        for _, word := range strings.Fields(string(data)) {
            counts[strings.ToLower(word)]++
        }
        return counts
    }, runtime.NumCPU())

    // Reduce: merge all word counts
    return reduce(counts, make(map[string]int), func(acc, m map[string]int) map[string]int {
        for word, count := range m {
            acc[word] += count
        }
        return acc
    })
}
```

---

**S9. Implement a circuit breaker pattern in Go.**

```go
type State int

const (
    StateClosed   State = iota // normal operation
    StateOpen                  // failing — reject requests
    StateHalfOpen              // probe — allow one request
)

type CircuitBreaker struct {
    mu            sync.Mutex
    state         State
    failures      int
    threshold     int
    timeout       time.Duration
    lastFailure   time.Time
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:     StateClosed,
        threshold: threshold,
        timeout:   timeout,
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    state := cb.state

    if state == StateOpen {
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            state = StateHalfOpen
        } else {
            cb.mu.Unlock()
            return fmt.Errorf("circuit breaker open")
        }
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold || state == StateHalfOpen {
            cb.state = StateOpen
        }
        return err
    }

    // Success
    cb.failures = 0
    cb.state = StateClosed
    return nil
}

// Usage
cb := NewCircuitBreaker(5, 30*time.Second)

for i := 0; i < 100; i++ {
    err := cb.Call(func() error {
        return callExternalService()
    })
    if err != nil {
        log.Printf("request %d failed: %v", i, err)
        time.Sleep(100 * time.Millisecond)
    }
}
```

---

**S10. Use WaitGroup to download multiple files concurrently and collect errors.**

```go
func downloadAll(ctx context.Context, urls []string, destDir string) []error {
    var (
        wg   sync.WaitGroup
        mu   sync.Mutex
        errs []error
    )

    // Use a semaphore to limit concurrent downloads
    sem := make(chan struct{}, 5) // max 5 concurrent

    for _, url := range urls {
        url := url // capture
        wg.Add(1)
        go func() {
            defer wg.Done()

            // Acquire semaphore
            select {
            case sem <- struct{}{}:
            case <-ctx.Done():
                mu.Lock()
                errs = append(errs, ctx.Err())
                mu.Unlock()
                return
            }
            defer func() { <-sem }()

            if err := downloadFile(ctx, url, destDir); err != nil {
                mu.Lock()
                errs = append(errs, fmt.Errorf("downloading %s: %w", url, err))
                mu.Unlock()
            }
        }()
    }

    wg.Wait()
    return errs
}

func downloadFile(ctx context.Context, url, destDir string) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    filename := filepath.Base(url)
    f, err := os.Create(filepath.Join(destDir, filename))
    if err != nil {
        return err
    }
    defer f.Close()

    _, err = io.Copy(f, resp.Body)
    return err
}
```

---

**S11. Implement a ticker-based heartbeat with graceful shutdown.**

```go
type HeartbeatService struct {
    interval time.Duration
    endpoint string
    done     chan struct{}
    wg       sync.WaitGroup
}

func NewHeartbeat(endpoint string, interval time.Duration) *HeartbeatService {
    return &HeartbeatService{
        interval: interval,
        endpoint: endpoint,
        done:     make(chan struct{}),
    }
}

func (h *HeartbeatService) Start() {
    h.wg.Add(1)
    go func() {
        defer h.wg.Done()
        ticker := time.NewTicker(h.interval)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                if err := h.ping(); err != nil {
                    log.Printf("heartbeat failed: %v", err)
                }
            case <-h.done:
                log.Println("heartbeat service stopping")
                return
            }
        }
    }()
}

func (h *HeartbeatService) Stop() {
    close(h.done)
    h.wg.Wait()
}

func (h *HeartbeatService) ping() error {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    req, _ := http.NewRequestWithContext(ctx, "GET", h.endpoint+"/health", nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("unhealthy: status %d", resp.StatusCode)
    }
    return nil
}

// Main with graceful shutdown
func main() {
    hb := NewHeartbeat("https://api.example.com", 30*time.Second)
    hb.Start()

    // Wait for OS signal
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig

    log.Println("shutting down...")
    hb.Stop()
    log.Println("shutdown complete")
}
```

---

**S12. Write a concurrent bounded cache with TTL-based expiration.**

```go
type entry struct {
    value     interface{}
    expiresAt time.Time
}

type TTLCache struct {
    mu      sync.RWMutex
    items   map[string]entry
    ttl     time.Duration
    maxSize int
}

func NewTTLCache(maxSize int, ttl time.Duration) *TTLCache {
    c := &TTLCache{
        items:   make(map[string]entry),
        ttl:     ttl,
        maxSize: maxSize,
    }
    go c.cleanup()
    return c
}

func (c *TTLCache) Set(key string, value interface{}) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    if len(c.items) >= c.maxSize {
        return false // cache full
    }
    c.items[key] = entry{value: value, expiresAt: time.Now().Add(c.ttl)}
    return true
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    e, ok := c.items[key]
    c.mu.RUnlock()
    if !ok || time.Now().After(e.expiresAt) {
        return nil, false
    }
    return e.value, true
}

func (c *TTLCache) cleanup() {
    ticker := time.NewTicker(c.ttl / 2)
    defer ticker.Stop()
    for range ticker.C {
        now := time.Now()
        c.mu.Lock()
        for k, e := range c.items {
            if now.After(e.expiresAt) {
                delete(c.items, k)
            }
        }
        c.mu.Unlock()
    }
}
```

---

**S13. Implement the producer-consumer pattern with backpressure.**

```go
func producerConsumer(ctx context.Context) {
    // Bounded channel provides backpressure:
    // When buffer is full, producer blocks — this IS the backpressure
    queue := make(chan Item, 100) // buffer of 100 items

    // Producer
    go func() {
        defer close(queue)
        for i := 0; ; i++ {
            select {
            case <-ctx.Done():
                return
            case queue <- Item{ID: i, Data: generateData(i)}:
                // Blocks when queue is full — natural backpressure
            }
        }
    }()

    // Multiple consumers
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for item := range queue {
                if err := consume(item); err != nil {
                    log.Printf("worker %d: consume error: %v", workerID, err)
                }
            }
        }(i)
    }
    wg.Wait()
}
```
