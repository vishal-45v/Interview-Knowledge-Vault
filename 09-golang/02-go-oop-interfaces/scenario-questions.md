# Chapter 02 — Go OOP & Interfaces: Scenario Questions

---

**S1. Design a Shape interface with Area() and Perimeter() methods. Implement it for Circle and Rectangle.**

```go
import "math"

type Shape interface {
    Area() float64
    Perimeter() float64
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

type Triangle struct {
    A, B, C float64 // side lengths
}

func (t Triangle) Perimeter() float64 { return t.A + t.B + t.C }
func (t Triangle) Area() float64 {
    s := t.Perimeter() / 2 // Heron's formula
    return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
}

// Polymorphic function using the interface
func totalArea(shapes []Shape) float64 {
    total := 0.0
    for _, s := range shapes {
        total += s.Area()
    }
    return total
}

func printShapeInfo(s Shape) {
    fmt.Printf("Type: %T, Area: %.2f, Perimeter: %.2f\n",
        s, s.Area(), s.Perimeter())
}

shapes := []Shape{
    Circle{Radius: 5},
    Rectangle{Width: 4, Height: 6},
    Triangle{A: 3, B: 4, C: 5},
}

for _, s := range shapes {
    printShapeInfo(s)
}
fmt.Printf("Total area: %.2f\n", totalArea(shapes))
```

---

**S2. You have a Logger interface. Implement a ConsoleLogger and a NoopLogger for tests.**

```go
type Logger interface {
    Info(msg string, args ...any)
    Warn(msg string, args ...any)
    Error(msg string, args ...any)
}

// ConsoleLogger — writes to stderr with timestamps
type ConsoleLogger struct {
    prefix string
}

func NewConsoleLogger(prefix string) *ConsoleLogger {
    return &ConsoleLogger{prefix: prefix}
}

func (l *ConsoleLogger) log(level, msg string, args ...any) {
    ts := time.Now().Format("2006-01-02T15:04:05")
    formatted := fmt.Sprintf(msg, args...)
    fmt.Fprintf(os.Stderr, "%s [%s] %s: %s\n", ts, level, l.prefix, formatted)
}

func (l *ConsoleLogger) Info(msg string, args ...any)  { l.log("INFO", msg, args...) }
func (l *ConsoleLogger) Warn(msg string, args ...any)  { l.log("WARN", msg, args...) }
func (l *ConsoleLogger) Error(msg string, args ...any) { l.log("ERROR", msg, args...) }

// NoopLogger — discards all log output (ideal for tests)
type NoopLogger struct{}

func (NoopLogger) Info(msg string, args ...any)  {}
func (NoopLogger) Warn(msg string, args ...any)  {}
func (NoopLogger) Error(msg string, args ...any) {}

// CapturingLogger — records logs for test assertions
type CapturingLogger struct {
    mu   sync.Mutex
    Logs []string
}

func (c *CapturingLogger) log(level, msg string, args ...any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.Logs = append(c.Logs, fmt.Sprintf("[%s] "+msg, append([]any{level}, args...)...))
}

func (c *CapturingLogger) Info(msg string, args ...any)  { c.log("INFO", msg, args...) }
func (c *CapturingLogger) Warn(msg string, args ...any)  { c.log("WARN", msg, args...) }
func (c *CapturingLogger) Error(msg string, args ...any) { c.log("ERROR", msg, args...) }

// Service using the logger
type UserService struct {
    logger Logger
    store  UserStore
}

func NewUserService(store UserStore, logger Logger) *UserService {
    return &UserService{store: store, logger: logger}
}

func (s *UserService) CreateUser(name, email string) error {
    s.logger.Info("creating user: %s", email)
    if err := s.store.Insert(name, email); err != nil {
        s.logger.Error("failed to create user %s: %v", email, err)
        return err
    }
    s.logger.Info("created user: %s", email)
    return nil
}

// In tests:
func TestCreateUser(t *testing.T) {
    logger := &CapturingLogger{}
    store := &MockUserStore{}
    svc := NewUserService(store, logger)

    err := svc.CreateUser("Alice", "alice@example.com")
    if err != nil {
        t.Fatal(err)
    }
    if len(logger.Logs) < 2 {
        t.Errorf("expected at least 2 log entries, got %d", len(logger.Logs))
    }
}
```

---

**S3. How would you implement a plugin system using interfaces in Go?**

```go
// Plugin defines what every plugin must implement
type Plugin interface {
    Name() string
    Version() string
    Execute(ctx context.Context, input map[string]any) (map[string]any, error)
}

// Registry holds registered plugins
type PluginRegistry struct {
    mu      sync.RWMutex
    plugins map[string]Plugin
}

func NewRegistry() *PluginRegistry {
    return &PluginRegistry{plugins: make(map[string]Plugin)}
}

func (r *PluginRegistry) Register(p Plugin) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    if _, exists := r.plugins[p.Name()]; exists {
        return fmt.Errorf("plugin %q already registered", p.Name())
    }
    r.plugins[p.Name()] = p
    return nil
}

func (r *PluginRegistry) Execute(name string, ctx context.Context, input map[string]any) (map[string]any, error) {
    r.mu.RLock()
    p, ok := r.plugins[name]
    r.mu.RUnlock()
    if !ok {
        return nil, fmt.Errorf("plugin %q not found", name)
    }
    return p.Execute(ctx, input)
}

// Concrete plugin implementation
type UpperCasePlugin struct{}

func (UpperCasePlugin) Name() string    { return "uppercase" }
func (UpperCasePlugin) Version() string { return "1.0.0" }
func (UpperCasePlugin) Execute(ctx context.Context, input map[string]any) (map[string]any, error) {
    text, ok := input["text"].(string)
    if !ok {
        return nil, fmt.Errorf("input 'text' must be a string")
    }
    return map[string]any{"result": strings.ToUpper(text)}, nil
}

// Usage
registry := NewRegistry()
registry.Register(UpperCasePlugin{})

result, err := registry.Execute("uppercase", context.Background(), map[string]any{"text": "hello"})
// result["result"] = "HELLO"
```

---

**S4. A function returns an interface. Explain the nil interface trap and how to avoid it.**

```go
// TRAP: returns interface containing a typed nil
type Validator interface {
    Validate() error
}

type AgeValidator struct {
    age int
}

func (v *AgeValidator) Validate() error {
    if v.age < 0 || v.age > 150 {
        return fmt.Errorf("invalid age: %d", v.age)
    }
    return nil
}

func getValidator(requiresAgeValidation bool) Validator {
    if !requiresAgeValidation {
        var v *AgeValidator = nil // typed nil pointer
        return v // DANGER: interface{type=*AgeValidator, value=nil} — NOT nil!
    }
    return &AgeValidator{age: 25}
}

v := getValidator(false)
if v != nil { // TRUE even though the pointer is nil
    v.Validate() // PANIC: nil pointer dereference
}

// FIX: return untyped nil
func getValidatorFixed(requiresAgeValidation bool) Validator {
    if !requiresAgeValidation {
        return nil // returns interface{type=nil, value=nil} — truly nil
    }
    return &AgeValidator{age: 25}
}

// Or use the comma-ok pattern with type assertion:
func safeValidate(v Validator) error {
    if v == nil {
        return nil
    }
    // Also check for typed nil via reflection (more robust):
    rv := reflect.ValueOf(v)
    if rv.Kind() == reflect.Ptr && rv.IsNil() {
        return nil
    }
    return v.Validate()
}
```

---

**S5. Use embedding to compose a full-featured HTTP middleware with logging and timing.**

```go
type ResponseWriter interface {
    http.ResponseWriter
    Status() int
    BytesWritten() int
}

// Wraps http.ResponseWriter to capture status code
type responseRecorder struct {
    http.ResponseWriter
    status       int
    bytesWritten int
}

func newResponseRecorder(w http.ResponseWriter) *responseRecorder {
    return &responseRecorder{ResponseWriter: w, status: http.StatusOK}
}

func (r *responseRecorder) WriteHeader(code int) {
    r.status = code
    r.ResponseWriter.WriteHeader(code)
}

func (r *responseRecorder) Write(b []byte) (int, error) {
    n, err := r.ResponseWriter.Write(b)
    r.bytesWritten += n
    return n, err
}

func (r *responseRecorder) Status() int      { return r.status }
func (r *responseRecorder) BytesWritten() int { return r.bytesWritten }

// Middleware using the embedded writer
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rec := newResponseRecorder(w)

        next.ServeHTTP(rec, r)

        log.Printf(
            "method=%s path=%s status=%d bytes=%d duration=%s",
            r.Method, r.URL.Path,
            rec.Status(), rec.BytesWritten(),
            time.Since(start),
        )
    })
}
```

---

**S6. Design a payment processor using interfaces that allows easy swapping of payment providers.**

```go
type PaymentMethod string

const (
    CreditCard PaymentMethod = "credit_card"
    PayPal     PaymentMethod = "paypal"
    Crypto     PaymentMethod = "crypto"
)

type PaymentRequest struct {
    Amount      float64
    Currency    string
    Description string
    CustomerID  string
}

type PaymentResult struct {
    TransactionID string
    Status        string
    ProcessedAt   time.Time
}

type PaymentProcessor interface {
    Charge(ctx context.Context, req PaymentRequest) (*PaymentResult, error)
    Refund(ctx context.Context, transactionID string, amount float64) error
    Supports(method PaymentMethod) bool
}

// Stripe implementation
type StripeProcessor struct {
    apiKey string
    client *http.Client
}

func (s *StripeProcessor) Charge(ctx context.Context, req PaymentRequest) (*PaymentResult, error) {
    // Make API call to Stripe
    return &PaymentResult{
        TransactionID: "stripe_" + generateID(),
        Status:        "succeeded",
        ProcessedAt:   time.Now(),
    }, nil
}

func (s *StripeProcessor) Refund(ctx context.Context, txID string, amount float64) error {
    // Stripe refund API call
    return nil
}

func (s *StripeProcessor) Supports(m PaymentMethod) bool {
    return m == CreditCard
}

// Service uses the interface — not the concrete type
type OrderService struct {
    processor PaymentProcessor
}

func (o *OrderService) PlaceOrder(ctx context.Context, items []Item) error {
    total := calculateTotal(items)
    result, err := o.processor.Charge(ctx, PaymentRequest{
        Amount:      total,
        Currency:    "USD",
        Description: fmt.Sprintf("Order for %d items", len(items)),
    })
    if err != nil {
        return fmt.Errorf("payment failed: %w", err)
    }
    log.Printf("Order paid, transaction: %s", result.TransactionID)
    return nil
}
```

---

**S7. Implement the io.Reader interface for a custom data source that generates Fibonacci numbers as bytes.**

```go
type FibReader struct {
    a, b uint64
    buf  bytes.Buffer
}

func NewFibReader() *FibReader {
    return &FibReader{a: 0, b: 1}
}

func (r *FibReader) Read(p []byte) (int, error) {
    // Fill buffer until we have enough bytes
    for r.buf.Len() < len(p) {
        r.buf.WriteString(strconv.FormatUint(r.a, 10))
        r.buf.WriteByte('\n')
        r.a, r.b = r.b, r.a+r.b
        if r.a > 1e18 { // prevent overflow
            return r.buf.Read(p)
        }
    }
    return r.buf.Read(p)
}

// Usage — any function accepting io.Reader works with FibReader
fib := NewFibReader()
scanner := bufio.NewScanner(fib)
for i := 0; i < 10; i++ {
    scanner.Scan()
    fmt.Println(scanner.Text())
}
// 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

---

**S8. How would you implement method chaining (builder pattern) in Go?**

```go
type QueryBuilder struct {
    table      string
    conditions []string
    orderBy    string
    limit      int
    offset     int
    args       []any
}

func NewQuery(table string) *QueryBuilder {
    return &QueryBuilder{table: table, limit: -1}
}

func (q *QueryBuilder) Where(condition string, args ...any) *QueryBuilder {
    q.conditions = append(q.conditions, condition)
    q.args = append(q.args, args...)
    return q
}

func (q *QueryBuilder) OrderBy(field string) *QueryBuilder {
    q.orderBy = field
    return q
}

func (q *QueryBuilder) Limit(n int) *QueryBuilder {
    q.limit = n
    return q
}

func (q *QueryBuilder) Offset(n int) *QueryBuilder {
    q.offset = n
    return q
}

func (q *QueryBuilder) Build() (string, []any) {
    sql := "SELECT * FROM " + q.table
    if len(q.conditions) > 0 {
        sql += " WHERE " + strings.Join(q.conditions, " AND ")
    }
    if q.orderBy != "" {
        sql += " ORDER BY " + q.orderBy
    }
    if q.limit >= 0 {
        sql += fmt.Sprintf(" LIMIT %d", q.limit)
    }
    if q.offset > 0 {
        sql += fmt.Sprintf(" OFFSET %d", q.offset)
    }
    return sql, q.args
}

// Fluent usage
sql, args := NewQuery("users").
    Where("age > ?", 18).
    Where("status = ?", "active").
    OrderBy("name").
    Limit(10).
    Offset(20).
    Build()
```

---

**S9. Design a middleware chain for an HTTP server using function types and interfaces.**

```go
// Middleware is a function that wraps an http.Handler
type Middleware func(http.Handler) http.Handler

// Chain applies middlewares in order (first listed runs first)
func Chain(middlewares ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        // Apply in reverse so first middleware is outermost
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}

// Auth middleware
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        // validate token...
        next.ServeHTTP(w, r)
    })
}

// Rate limit middleware
func RateLimitMiddleware(rps int) Middleware {
    limiter := rate.NewLimiter(rate.Limit(rps), rps)
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "too many requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Composing the chain
mux := http.NewServeMux()
mux.HandleFunc("/api/users", handleUsers)

handler := Chain(
    LoggingMiddleware,
    AuthMiddleware,
    RateLimitMiddleware(100),
)(mux)

http.ListenAndServe(":8080", handler)
```

---

**S10. How do you retroactively make a third-party type satisfy your interface?**

```go
// Third-party type you can't modify
// package thirdparty
type RemoteCache struct{ ... }
func (c *RemoteCache) Fetch(key string) ([]byte, error) { ... }
func (c *RemoteCache) Store(key string, val []byte) error { ... }

// Your interface
type Cache interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
}

// Adapter — wraps the third-party type
type RemoteCacheAdapter struct {
    cache *thirdparty.RemoteCache
}

func (a *RemoteCacheAdapter) Get(key string) ([]byte, error) {
    return a.cache.Fetch(key)
}

func (a *RemoteCacheAdapter) Set(key string, val []byte) error {
    return a.cache.Store(key, val)
}

// Compile-time check
var _ Cache = (*RemoteCacheAdapter)(nil)

// Now RemoteCache works anywhere Cache is expected
func NewRemoteCacheAdapter(c *thirdparty.RemoteCache) Cache {
    return &RemoteCacheAdapter{cache: c}
}
```

---

**S11. Implement a type-safe event bus using interfaces.**

```go
type EventHandler interface {
    Handle(event any) error
}

type EventHandlerFunc func(event any) error

func (f EventHandlerFunc) Handle(event any) error { return f(event) }

type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]EventHandler
}

func NewEventBus() *EventBus {
    return &EventBus{handlers: make(map[string][]EventHandler)}
}

func (b *EventBus) Subscribe(eventType string, h EventHandler) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.handlers[eventType] = append(b.handlers[eventType], h)
}

func (b *EventBus) Publish(eventType string, event any) []error {
    b.mu.RLock()
    handlers := b.handlers[eventType]
    b.mu.RUnlock()

    var errs []error
    for _, h := range handlers {
        if err := h.Handle(event); err != nil {
            errs = append(errs, err)
        }
    }
    return errs
}

// Usage
type UserCreated struct {
    UserID int
    Email  string
}

bus := NewEventBus()
bus.Subscribe("user.created", EventHandlerFunc(func(e any) error {
    uc := e.(UserCreated)
    fmt.Printf("Sending welcome email to %s\n", uc.Email)
    return nil
}))
bus.Subscribe("user.created", EventHandlerFunc(func(e any) error {
    uc := e.(UserCreated)
    fmt.Printf("Setting up default preferences for user %d\n", uc.UserID)
    return nil
}))

bus.Publish("user.created", UserCreated{UserID: 1, Email: "alice@example.com"})
```

---

**S12. How would you implement the Stringer interface for a complex nested struct?**

```go
type Address struct {
    Street  string
    City    string
    Country string
}

func (a Address) String() string {
    return fmt.Sprintf("%s, %s, %s", a.Street, a.City, a.Country)
}

type ContactInfo struct {
    Email string
    Phone string
}

func (c ContactInfo) String() string {
    parts := []string{}
    if c.Email != "" {
        parts = append(parts, "email:"+c.Email)
    }
    if c.Phone != "" {
        parts = append(parts, "phone:"+c.Phone)
    }
    return strings.Join(parts, ", ")
}

type Person struct {
    Name    string
    Age     int
    Address Address
    Contact ContactInfo
}

func (p Person) String() string {
    return fmt.Sprintf(
        "Person{Name:%q Age:%d Address:%s Contact:{%s}}",
        p.Name, p.Age, p.Address, p.Contact,
    )
}

p := Person{
    Name: "Alice", Age: 30,
    Address: Address{"123 Main St", "Portland", "USA"},
    Contact: ContactInfo{Email: "alice@example.com"},
}
fmt.Println(p)
// Person{Name:"Alice" Age:30 Address:123 Main St, Portland, USA Contact:{email:alice@example.com}}
```

---

**S13. Implement a retry decorator using interfaces.**

```go
type Operation interface {
    Do(ctx context.Context) error
}

type OperationFunc func(ctx context.Context) error

func (f OperationFunc) Do(ctx context.Context) error { return f(ctx) }

type RetryDecorator struct {
    operation Operation
    attempts  int
    delay     time.Duration
}

func WithRetry(op Operation, attempts int, delay time.Duration) Operation {
    return &RetryDecorator{operation: op, attempts: attempts, delay: delay}
}

func (r *RetryDecorator) Do(ctx context.Context) error {
    var lastErr error
    for i := 0; i < r.attempts; i++ {
        if i > 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(r.delay):
            }
        }
        lastErr = r.operation.Do(ctx)
        if lastErr == nil {
            return nil
        }
        log.Printf("attempt %d/%d failed: %v", i+1, r.attempts, lastErr)
    }
    return fmt.Errorf("all %d attempts failed: %w", r.attempts, lastErr)
}

// Usage
op := OperationFunc(func(ctx context.Context) error {
    return callExternalAPI(ctx)
})

reliable := WithRetry(op, 3, 500*time.Millisecond)
if err := reliable.Do(context.Background()); err != nil {
    log.Fatal(err)
}
```

---

**S14. How would you use struct embedding to build a base repository with common CRUD operations?**

```go
type BaseRepository struct {
    db *sql.DB
}

func (r *BaseRepository) exec(ctx context.Context, query string, args ...any) error {
    _, err := r.db.ExecContext(ctx, query, args...)
    return err
}

func (r *BaseRepository) queryRow(ctx context.Context, query string, args ...any) *sql.Row {
    return r.db.QueryRowContext(ctx, query, args...)
}

type UserRepository struct {
    BaseRepository // embedded — gets exec, queryRow
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{BaseRepository: BaseRepository{db: db}}
}

func (r *UserRepository) Create(ctx context.Context, u *User) error {
    return r.exec(ctx, // promoted from BaseRepository
        "INSERT INTO users (name, email) VALUES (?, ?)",
        u.Name, u.Email,
    )
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    var u User
    row := r.queryRow(ctx, "SELECT id, name, email FROM users WHERE id = ?", id)
    err := row.Scan(&u.ID, &u.Name, &u.Email)
    if err == sql.ErrNoRows {
        return nil, nil
    }
    return &u, err
}
```

---

**S15. How do you ensure a type implements an interface at compile time without instantiating it?**

```go
// Pattern: assign nil pointer to interface variable
// This costs nothing at runtime (optimized away)
// but fails to compile if the interface is not satisfied

type MyService struct{}

func (s *MyService) ServeHTTP(w http.ResponseWriter, r *http.Request) {}
func (s *MyService) Shutdown(ctx context.Context) error { return nil }

// Place near the type definition to catch interface drift early
var _ http.Handler = (*MyService)(nil)    // assert implements http.Handler
var _ io.Closer = (*MyService)(nil)        // assert implements io.Closer -- would fail!

// Common interfaces to assert against:
// var _ fmt.Stringer = (*MyType)(nil)
// var _ error = (*MyError)(nil)
// var _ json.Marshaler = (*MyType)(nil)
// var _ sort.Interface = (*MySlice)(nil)
```
