# Chapter 02 — Go OOP & Interfaces: Structured Answers

---

## Answer 1: What is the difference between embedding and inheritance in Go?

**Inheritance** (Java/C++): creates an `is-a` relationship. A `Dog` IS-AN `Animal`. The child class inherits state and behavior, can override methods, and participates in a type hierarchy that enables polymorphism through the hierarchy.

**Embedding** (Go): creates a `has-a` relationship through composition. A `Dog` HAS-A `LivingThing`. The embedded type's methods are promoted (syntactic convenience), but there is no `is-a` relationship, no method overriding, and no polymorphism through the type hierarchy.

```go
// Java-style inheritance (not possible in Go):
// class Dog extends Animal {
//     @Override public String sound() { return "Woof"; }
// }

// Go: composition via embedding
type Animal struct {
    name string
}

func (a Animal) Breathe() string {
    return a.name + " is breathing"
}

func (a Animal) Name() string { return a.name }

type Dog struct {
    Animal          // embedded — promotes Name() and Breathe()
    breed string
}

func (d Dog) Sound() string { return "Woof" }

// Dog "shadows" Breathe if it defines its own, but doesn't "override"
func (d Dog) Breathe() string {
    // Call embedded type's method explicitly
    return d.Animal.Breathe() + " (dog style)"
}

d := Dog{Animal: Animal{name: "Rex"}, breed: "Lab"}
fmt.Println(d.Name())    // "Rex" — promoted from Animal
fmt.Println(d.Breathe()) // "Rex is breathing (dog style)" — Dog's own
fmt.Println(d.Sound())   // "Woof"

// KEY DIFFERENCE: Dog is NOT an Animal interface (unless Animal defines methods matching an interface)
// In Java, Dog IS-A Animal. In Go, Dog HAS-A Animal.
```

**Polymorphism in Go comes from interfaces, not from type hierarchy:**

```go
type Speaker interface {
    Sound() string
}

type Cat struct{ name string }
func (c Cat) Sound() string { return "Meow" }

// Both Dog and Cat implement Speaker independently — no shared base class needed
func makeNoise(s Speaker) {
    fmt.Println(s.Sound())
}

makeNoise(Dog{Animal: Animal{name: "Rex"}}) // Woof
makeNoise(Cat{name: "Whiskers"})            // Meow
```

---

## Answer 2: Explain the nil interface gotcha with a code example

An interface value in Go is internally a pair: `{type, value}`. The interface is nil **only when both** the type pointer and the value pointer are nil.

```go
// Internal representation (conceptual):
// interface{ type: nil,      value: nil }  → nil interface
// interface{ type: *MyError, value: nil }  → non-nil interface with nil value
// interface{ type: *MyError, value: 0x... } → non-nil interface with value

type MyError struct{ msg string }
func (e *MyError) Error() string {
    if e == nil {
        return "nil MyError"
    }
    return e.msg
}

// The trap in real code:
func riskyOperation(shouldFail bool) error {
    var err *MyError  // nil pointer to concrete type

    if shouldFail {
        err = &MyError{msg: "something failed"}
    }

    // When shouldFail=false, err is nil *MyError
    // But returning it as 'error' interface creates:
    // error{ type: *MyError, value: nil }
    // This is NOT a nil error interface!
    return err
}

err := riskyOperation(false)
if err != nil {
    // This block EXECUTES even though we expected no error!
    fmt.Println("unexpected error branch")
    fmt.Println(err.Error()) // prints "nil MyError" — doesn't panic here because we added the nil check in Error()
}

// The correct fix:
func riskyOperationFixed(shouldFail bool) error {
    if shouldFail {
        return &MyError{msg: "something failed"}
    }
    return nil  // directly return untyped nil
}
```

**Detecting a typed nil inside an interface (for defensive code):**

```go
func isNilInterface(i interface{}) bool {
    if i == nil {
        return true // truly nil interface
    }
    // Check if the underlying value is a nil pointer
    v := reflect.ValueOf(i)
    return v.Kind() == reflect.Ptr && v.IsNil()
}

var err *MyError = nil
var iface error = err

fmt.Println(iface == nil)        // false
fmt.Println(isNilInterface(iface)) // true
```

---

## Answer 3: How does Go achieve polymorphism without inheritance?

Go uses **interface-based polymorphism**. Any type that has the required methods implicitly satisfies the interface — no class hierarchy needed.

```go
// Step 1: Define a behavioral contract (interface)
type Shape interface {
    Area() float64
    String() string
}

// Step 2: Multiple independent types implement the interface
type Circle struct{ R float64 }
func (c Circle) Area() float64   { return math.Pi * c.R * c.R }
func (c Circle) String() string  { return fmt.Sprintf("Circle(r=%.1f)", c.R) }

type Square struct{ Side float64 }
func (s Square) Area() float64   { return s.Side * s.Side }
func (s Square) String() string  { return fmt.Sprintf("Square(s=%.1f)", s.Side) }

type Triangle struct{ Base, Height float64 }
func (t Triangle) Area() float64   { return 0.5 * t.Base * t.Height }
func (t Triangle) String() string  { return fmt.Sprintf("Triangle(b=%.1f,h=%.1f)", t.Base, t.Height) }

// Step 3: Polymorphic code operates on the interface
func largestShape(shapes []Shape) Shape {
    var largest Shape
    maxArea := -1.0
    for _, s := range shapes {
        if a := s.Area(); a > maxArea {
            maxArea = a
            largest = s
        }
    }
    return largest
}

shapes := []Shape{
    Circle{R: 3},
    Square{Side: 5},
    Triangle{Base: 8, Height: 6},
}

big := largestShape(shapes)
fmt.Printf("Largest: %s with area %.2f\n", big, big.Area())
// Runtime dispatch — the correct Area() is called based on the concrete type
```

**Runtime dispatch mechanism:** Interface values store a pointer to the concrete type's method table (itab) at runtime. When `big.Area()` is called, Go looks up `Area` in the itab and calls the right implementation. This is similar to virtual dispatch in C++ but uses the interface type rather than class hierarchy.

---

## Answer 4: What is the io.Reader interface and why is it so powerful?

```go
// The entire io.Reader interface:
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

One method. One simple contract: "give me bytes from wherever you get them." This simplicity is the source of its power — anything that can produce bytes implements it.

```go
// Standard library types that implement io.Reader:
// *os.File        — file on disk
// *bytes.Buffer   — in-memory buffer
// *strings.Reader — read from a string
// net.Conn        — network connection
// *gzip.Reader    — decompress on the fly
// *bufio.Reader   — buffered reading
// io.LimitReader  — limit bytes read
// io.MultiReader  — read from multiple sources sequentially

// Functions that accept io.Reader work with ANY of the above:
func countWords(r io.Reader) (int, error) {
    scanner := bufio.NewScanner(r)
    scanner.Split(bufio.ScanWords)
    count := 0
    for scanner.Scan() {
        count++
    }
    return count, scanner.Err()
}

// Works with a file:
f, _ := os.Open("document.txt")
n, _ := countWords(f)

// Works with an in-memory string:
n, _ = countWords(strings.NewReader("hello world foo bar"))

// Works with a gzip-compressed network response:
resp, _ := http.Get("https://example.com/data.gz")
gr, _ := gzip.NewReader(resp.Body)
n, _ = countWords(gr)

// Chaining (compose readers):
//
//   HTTP response body → gzip decompressor → buffered reader → your code
//
func processRemoteGzip(url string) error {
    resp, err := http.Get(url)
    if err != nil { return err }
    defer resp.Body.Close()

    gr, err := gzip.NewReader(resp.Body)
    if err != nil { return err }
    defer gr.Close()

    br := bufio.NewReader(gr)
    // br is an io.Reader — pass it to any function accepting io.Reader
    _, err = io.Copy(os.Stdout, br)
    return err
}
```

The key design insight: by depending on `io.Reader` rather than `*os.File`, functions become testable (pass a `strings.Reader`), flexible (swap implementations), and composable (chain transformations).

---

## Answer 5: How do struct tags work? Show JSON marshaling example.

Struct tags are raw string literals following a field declaration. The convention is `key:"options"` pairs separated by spaces. The `reflect` package's `StructTag` type parses them.

```go
import (
    "encoding/json"
    "time"
)

type Article struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Content     string    `json:"content"`
    AuthorEmail string    `json:"author_email"`
    Internal    string    `json:"-"`                   // never in JSON
    ViewCount   int       `json:"view_count,omitempty"` // omit if 0
    Tags        []string  `json:"tags,omitempty"`       // omit if nil/empty
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at,omitempty"`
}

// Marshaling (Go → JSON)
a := Article{
    ID:    1,
    Title: "Understanding Go Interfaces",
    Content: "Interfaces in Go...",
    AuthorEmail: "alice@example.com",
    Internal: "sensitive data",
    CreatedAt: time.Now(),
}

b, err := json.Marshal(a)
// Output:
// {
//   "id": 1,
//   "title": "Understanding Go Interfaces",
//   "content": "Interfaces in Go...",
//   "author_email": "alice@example.com",
//   "created_at": "2026-03-17T10:30:00Z"
// }
// "internal" is omitted (json:"-")
// "view_count" is omitted (omitempty, zero value)
// "tags" is omitted (omitempty, nil)
// "updated_at" is omitted (omitempty, zero time)

// Unmarshaling (JSON → Go)
jsonStr := `{"id":2,"title":"Goroutines","view_count":150}`
var a2 Article
json.Unmarshal([]byte(jsonStr), &a2)
fmt.Println(a2.ID)         // 2
fmt.Println(a2.Title)      // Goroutines
fmt.Println(a2.ViewCount)  // 150
fmt.Println(a2.Content)    // "" — missing from JSON, zero value

// Custom JSON handling with MarshalJSON/UnmarshalJSON:
type SafeString struct {
    value string
}

func (s SafeString) MarshalJSON() ([]byte, error) {
    // Redact the value in JSON output
    return json.Marshal("[REDACTED]")
}
```

**Reading tags via reflection:**

```go
t := reflect.TypeOf(Article{})
field, _ := t.FieldByName("AuthorEmail")
tag := field.Tag
fmt.Println(tag.Get("json")) // "author_email"
```

---

## Answer 6: What is the difference between interface{} and a concrete type for performance?

When a value is stored in an `interface{}`, it may be **boxed**: if the value is larger than a pointer, it gets allocated on the heap. Interface calls also involve an indirect function call through the itab.

```go
// Benchmark: interface{} vs concrete type
func BenchmarkConcreteAdd(b *testing.B) {
    x := 0
    for i := 0; i < b.N; i++ {
        x = concreteAdd(x, 1)  // direct call, no allocation
    }
}

func BenchmarkInterfaceAdd(b *testing.B) {
    var x interface{} = 0
    for i := 0; i < b.N; i++ {
        x = interfaceAdd(x, 1) // type assertion + allocation possible
    }
}

// Typical results:
// BenchmarkConcreteAdd  1000000000   0.28 ns/op   0 allocs/op
// BenchmarkInterfaceAdd   50000000  22.3  ns/op   1 alloc/op

// Why the difference:
// 1. Interface call: indirect dispatch through itab (extra memory access)
// 2. Small non-pointer values stored in interface{} may be heap-allocated
// 3. Type assertion has a runtime check

// When it matters vs doesn't:
// - Hot path code (millions of calls): prefer concrete types or generics
// - I/O, database, HTTP handlers: interface overhead is negligible
// - Generic containers: use generics (Go 1.18+) for type safety without overhead
```

---

## Answer 7: How do you implement the Stringer interface?

```go
// fmt.Stringer is defined as:
// type Stringer interface { String() string }

// Basic implementation
type Color struct {
    R, G, B uint8
}

func (c Color) String() string {
    return fmt.Sprintf("#%02X%02X%02X", c.R, c.G, c.B)
}

// More complex: implement for a custom error type too
type ValidationError struct {
    Field   string
    Message string
    Value   interface{}
}

func (e ValidationError) Error() string  { return e.String() }
func (e ValidationError) String() string {
    return fmt.Sprintf("validation failed for field %q (value=%v): %s",
        e.Field, e.Value, e.Message)
}

// For a collection type:
type IntList []int

func (l IntList) String() string {
    strs := make([]string, len(l))
    for i, v := range l {
        strs[i] = strconv.Itoa(v)
    }
    return "[" + strings.Join(strs, ", ") + "]"
}

list := IntList{1, 2, 3, 4, 5}
fmt.Println(list) // [1, 2, 3, 4, 5]

// CAUTION: pointer vs value receiver with fmt
type Point struct{ X, Y int }

// If String() has a value receiver:
func (p Point) String() string { return fmt.Sprintf("(%d,%d)", p.X, p.Y) }
fmt.Println(Point{1, 2})  // (1,2) — works
fmt.Println(&Point{1, 2}) // (1,2) — also works (pointer gets value methods)

// If String() has a pointer receiver:
func (p *Point) String() string { return fmt.Sprintf("(%d,%d)", p.X, p.Y) }
fmt.Println(&Point{1, 2}) // (1,2) — works
fmt.Println(Point{1, 2})  // {1 2} — does NOT use String() !
// (non-pointer Point doesn't satisfy fmt.Stringer with pointer receiver)
```

---

## Answer 8: How do you do dependency injection in Go using interfaces?

Go's approach to DI is simple: accept interfaces in constructors or function parameters; provide concrete implementations from the composition root (main or test).

```go
// Define narrow interfaces — only the methods you need
type UserStore interface {
    Create(ctx context.Context, u *User) error
    GetByID(ctx context.Context, id int) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
}

type EmailService interface {
    SendWelcome(ctx context.Context, email, name string) error
}

type TokenGenerator interface {
    Generate(userID int) (string, error)
}

// Service depends on interfaces, not concrete types
type AuthService struct {
    store  UserStore
    mailer EmailService
    tokens TokenGenerator
    logger Logger
}

// Constructor takes interfaces — easy to mock in tests
func NewAuthService(
    store UserStore,
    mailer EmailService,
    tokens TokenGenerator,
    logger Logger,
) *AuthService {
    return &AuthService{
        store:  store,
        mailer: mailer,
        tokens: tokens,
        logger: logger,
    }
}

func (s *AuthService) Register(ctx context.Context, email, name, password string) (string, error) {
    // Check existing user
    existing, err := s.store.GetByEmail(ctx, email)
    if err != nil {
        return "", fmt.Errorf("checking email: %w", err)
    }
    if existing != nil {
        return "", fmt.Errorf("email already registered")
    }

    // Create user
    user := &User{Email: email, Name: name}
    if err := s.store.Create(ctx, user); err != nil {
        return "", fmt.Errorf("creating user: %w", err)
    }

    // Send welcome email (failure is non-fatal)
    if err := s.mailer.SendWelcome(ctx, email, name); err != nil {
        s.logger.Warn("failed to send welcome email: %v", err)
    }

    // Generate auth token
    token, err := s.tokens.Generate(user.ID)
    if err != nil {
        return "", fmt.Errorf("generating token: %w", err)
    }
    return token, nil
}

// main.go — wire up concrete implementations
func main() {
    db := mustConnectDB()
    store := postgres.NewUserStore(db)
    mailer := sendgrid.NewEmailService(os.Getenv("SENDGRID_KEY"))
    tokens := jwt.NewTokenGenerator(os.Getenv("JWT_SECRET"))
    logger := zap.NewProductionLogger()

    authSvc := NewAuthService(store, mailer, tokens, logger)
    // ...
}

// In tests — inject mocks/stubs
func TestRegister(t *testing.T) {
    store := &MockUserStore{}
    mailer := &NoopEmailService{}
    tokens := &FakeTokenGenerator{token: "test-token"}
    logger := &NoopLogger{}

    svc := NewAuthService(store, mailer, tokens, logger)
    token, err := svc.Register(context.Background(), "alice@example.com", "Alice", "pass")
    // assert token == "test-token", err == nil
}
```

---

## Answer 9: What are promoted methods and how does embedding create them?

When you embed type `B` in type `A`, all exported methods of `B` are **promoted** to `A`. This means you can call them directly on an `A` value without qualifying through `B`.

```go
type Closer struct{}
func (c Closer) Close() error {
    fmt.Println("closing")
    return nil
}

type Logger struct{}
func (l Logger) Log(msg string) {
    fmt.Println("log:", msg)
}

type Service struct {
    Closer  // promotes Close()
    Logger  // promotes Log()
    name string
}

s := Service{name: "myservice"}
s.Log("starting")   // equivalent to s.Logger.Log("starting")
s.Close()           // equivalent to s.Closer.Close()

// Interface satisfaction via promoted methods:
var _ io.Closer = Service{} // Service satisfies io.Closer via promoted Close()

// Promoting pointer methods:
type Buffer struct{ data []byte }
func (b *Buffer) Write(p []byte) (int, error) {
    b.data = append(b.data, p...)
    return len(p), nil
}

type Writer struct {
    *Buffer // pointer embedding — promotes Write() as pointer method
}

w := Writer{Buffer: &Buffer{}}
w.Write([]byte("hello"))  // promoted Write() on *Buffer
var _ io.Writer = &Writer{Buffer: &Buffer{}} // *Writer satisfies io.Writer
```

**Shadowing:** If `A` defines a method with the same name as a promoted method from `B`, `A`'s method takes precedence. The `B` method is still accessible as `a.B.Method()`.

---

## Answer 10: How do you mock dependencies in Go tests using interfaces?

Three levels of mocking in Go:

**Level 1: Hand-crafted mock structs (no library)**

```go
type MockUserStore struct {
    users    map[int]*User
    createFn func(*User) error // injectable behavior
    calls    []string          // track method calls
}

func (m *MockUserStore) Create(ctx context.Context, u *User) error {
    m.calls = append(m.calls, "Create")
    if m.createFn != nil {
        return m.createFn(u)
    }
    if m.users == nil {
        m.users = make(map[int]*User)
    }
    u.ID = len(m.users) + 1
    m.users[u.ID] = u
    return nil
}

func (m *MockUserStore) GetByID(ctx context.Context, id int) (*User, error) {
    m.calls = append(m.calls, "GetByID")
    u, ok := m.users[id]
    if !ok {
        return nil, nil
    }
    return u, nil
}

// Test using the mock
func TestCreateUser(t *testing.T) {
    store := &MockUserStore{}
    svc := NewUserService(store, &NoopLogger{})

    user, err := svc.CreateUser(context.Background(), "Alice", "alice@example.com")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.ID == 0 {
        t.Error("expected user to have an ID")
    }
    if len(store.calls) == 0 || store.calls[0] != "Create" {
        t.Error("expected Create to be called")
    }
}
```

**Level 2: Using testify/mock for more complex scenarios**

```go
// go get github.com/stretchr/testify/mock
type MockStore struct {
    mock.Mock
}

func (m *MockStore) GetByID(ctx context.Context, id int) (*User, error) {
    args := m.Called(ctx, id)
    if u := args.Get(0); u != nil {
        return u.(*User), args.Error(1)
    }
    return nil, args.Error(1)
}

func TestService(t *testing.T) {
    store := new(MockStore)
    store.On("GetByID", mock.Anything, 1).Return(&User{ID: 1, Name: "Alice"}, nil)
    store.On("GetByID", mock.Anything, 99).Return(nil, errors.New("not found"))

    svc := NewService(store)
    user, err := svc.GetUser(context.Background(), 1)
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    store.AssertExpectations(t)
}
```

**Level 3: Functional mocks (simple, lightweight)**

```go
// For simple interfaces with one or two methods,
// define a function-based mock type:
type GetUserFunc func(ctx context.Context, id int) (*User, error)

func (f GetUserFunc) GetUser(ctx context.Context, id int) (*User, error) {
    return f(ctx, id)
}

// In test — inline the behavior:
store := GetUserFunc(func(ctx context.Context, id int) (*User, error) {
    return &User{ID: id, Name: "Test User"}, nil
})
```
