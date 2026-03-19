# Chapter 02 — Go OOP & Interfaces: Theory Questions

---

## Structs

**Q1. How do you define a struct in Go and what are its key characteristics?**
A struct is a composite type that groups named fields of potentially different types. Each field has a name and a type. Struct fields are accessed with dot notation. Structs are value types — assigning one struct to another copies all fields.

```go
type Employee struct {
    ID         int
    Name       string
    Department string
    salary     float64 // unexported — only accessible within the package
}

e := Employee{ID: 1, Name: "Alice", Department: "Engineering"}
e.salary = 95000.0 // accessible within same package
```

**Q2. What is struct embedding in Go?**
Struct embedding is placing one struct type inside another without giving it an explicit field name. The embedded type's fields and methods are "promoted" to the outer type — they become accessible as if they were defined on the outer type directly.

```go
type Address struct {
    Street string
    City   string
    Zip    string
}

type Person struct {
    Name    string
    Age     int
    Address // embedded — no field name, just the type
}

p := Person{
    Name: "Alice",
    Age:  30,
    Address: Address{Street: "123 Main St", City: "Springfield"},
}

// Promoted field access:
fmt.Println(p.City)   // same as p.Address.City — promoted
fmt.Println(p.Street) // same as p.Address.Street
```

**Q3. What is the difference between an embedded field and a named field?**
```go
// Named field — explicit access required
type Named struct {
    addr Address
}
n := Named{}
fmt.Println(n.addr.City) // must use n.addr.City

// Embedded field — promoted access
type Embedded struct {
    Address
}
e := Embedded{}
fmt.Println(e.City)         // promoted: works
fmt.Println(e.Address.City) // also works — both are valid
```
Named fields give explicit namespacing. Embedded fields promote for convenience but can cause name collisions if multiple embedded types have the same field or method name.

**Q4. What is an anonymous field in Go?**
An anonymous field (also called an embedded field) is a field declared with only a type name and no explicit name. The field name is implicitly the type name.

```go
type Logger struct {
    *log.Logger // anonymous field — pointer embedding
}
// Field name is "Logger" (the type name, without the package prefix)
l := Logger{Logger: log.New(os.Stdout, "", 0)}
l.Println("hello") // promoted from *log.Logger
```

---

## Methods

**Q5. How do you define a method in Go?**
A method is a function with a receiver argument. The receiver appears between `func` and the method name.

```go
type Circle struct {
    Radius float64
}

// Value receiver method
func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

// Pointer receiver method
func (c *Circle) Scale(factor float64) {
    c.Radius *= factor
}

c := Circle{Radius: 5.0}
fmt.Println(c.Area())  // 78.54
c.Scale(2)
fmt.Println(c.Radius)  // 10.0
```

**Q6. What is the method set of a type in Go?**
The method set of a type determines which methods can be called on values of that type, and which interfaces the type satisfies.

- Method set of `T`: all methods with value receiver `T`
- Method set of `*T`: all methods with receiver `T` OR `*T`

This means `*T` always has a superset of the methods of `T`. A `*T` can satisfy interfaces that require methods with value receivers, but a `T` cannot satisfy interfaces that require methods with pointer receivers.

**Q7. Can you add methods to built-in types like `int` or `string` in Go?**
No, you cannot add methods to types from another package (including built-in types). You must define a named type based on the built-in type and add methods to that named type.

```go
// Error: cannot define new methods on non-local type int
// func (n int) IsEven() bool { return n%2 == 0 } // COMPILE ERROR

// Correct: define a named type
type MyInt int

func (n MyInt) IsEven() bool { return n%2 == 0 }
func (n MyInt) String() string { return fmt.Sprintf("MyInt(%d)", int(n)) }

x := MyInt(42)
fmt.Println(x.IsEven()) // true
```

---

## Interfaces

**Q8. What is an interface in Go?**
An interface is a type that specifies a method set. A value of interface type can hold any concrete value that implements the interface — that is, any type with all the methods the interface requires.

```go
type Animal interface {
    Sound() string
    Name() string
}

type Dog struct{ name string }
func (d Dog) Sound() string { return "Woof" }
func (d Dog) Name() string  { return d.name }

type Cat struct{ name string }
func (c Cat) Sound() string { return "Meow" }
func (c Cat) Name() string  { return c.name }

// Both Dog and Cat implicitly implement Animal
func makeNoise(a Animal) {
    fmt.Printf("%s says %s\n", a.Name(), a.Sound())
}

makeNoise(Dog{name: "Rex"})
makeNoise(Cat{name: "Whiskers"})
```

**Q9. What does "implicit interface implementation" mean in Go?**
In Go, there is no `implements` keyword. A type automatically implements an interface if it has all the required methods with matching signatures. No declaration of intent is needed — this is called structural typing or duck typing. The compiler checks implementation at the point of assignment or function call.

```go
type Stringer interface {
    String() string
}

// MyType implements Stringer without any declaration
type MyType struct{ value int }
func (m MyType) String() string { return fmt.Sprintf("MyType{%d}", m.value) }

var s Stringer = MyType{42} // compiles because MyType has String()
```

**Q10. What is the empty interface `interface{}` (or `any`)?**
The empty interface has no methods, so every type in Go satisfies it. It's the most general type — equivalent to `Object` in Java. Since Go 1.18, `any` is a built-in alias for `interface{}`.

```go
// any can hold any value
var v any = 42
v = "hello"
v = []int{1, 2, 3}

// Common uses:
func printAnything(v any) {
    fmt.Printf("%v (%T)\n", v, v)
}

// Container for heterogeneous data
data := map[string]any{
    "name": "Alice",
    "age":  30,
    "tags": []string{"admin", "user"},
}
```

**Q11. How do you compose interfaces in Go?**
Interfaces can embed other interfaces. The composed interface requires all methods from all embedded interfaces.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter requires both Read and Write methods
type ReadWriter interface {
    Reader
    Writer
}

// Closer with both reading and closing
type ReadCloser interface {
    Reader
    io.Closer
}
```

**Q12. What is the `fmt.Stringer` interface?**
`fmt.Stringer` is defined in the `fmt` package and is the most commonly implemented interface:
```go
type Stringer interface {
    String() string
}
```
When the `fmt` package encounters a value that implements `Stringer`, it calls `String()` for formatting. It's Go's equivalent of Java's `toString()`.

```go
type Point struct{ X, Y int }

func (p Point) String() string {
    return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

p := Point{3, 4}
fmt.Println(p)        // (3, 4) — String() called automatically
fmt.Printf("%v\n", p) // (3, 4)
fmt.Printf("%s\n", p) // (3, 4)
```

**Q13. What are `io.Reader` and `io.Writer`? Why are they important?**

```go
// From the io package:
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

These are Go's core streaming interfaces. Because they're defined in terms of simple byte buffers, any data source or destination can implement them:
- `os.File` implements both Reader and Writer
- `net.Conn` (network connection) implements both
- `bytes.Buffer` implements both
- `strings.Reader` implements Reader
- `http.ResponseWriter` implements Writer
- `gzip.Writer` wraps any Writer with compression

This composability is the power of Go interfaces — you can chain: file → gzip → network, and every layer just sees a Reader or Writer.

**Q14. What is duck typing in Go?**
Duck typing is the principle that "if it walks like a duck and quacks like a duck, it's a duck." In Go, if a type has all the methods an interface requires, it is that interface — no explicit declaration needed. This allows retrofitting: you can define an interface after writing the concrete type, and the concrete type satisfies the interface without any modification.

```go
// Written by someone else, in a different package:
type ThirdPartyDB struct{ ... }
func (db *ThirdPartyDB) Query(sql string) (*Rows, error) { ... }
func (db *ThirdPartyDB) Exec(sql string) error { ... }

// YOU define an interface that matches what you need:
type Database interface {
    Query(sql string) (*Rows, error)
    Exec(sql string) error
}

// ThirdPartyDB implicitly satisfies Database — no changes needed to it
var db Database = &ThirdPartyDB{...} // works!
```

---

## Type Assertions

**Q15. What is a type assertion in Go?**
A type assertion extracts the concrete value stored inside an interface variable.

```go
var i interface{} = "hello world"

// Asserting the concrete type
s := i.(string)         // s = "hello world"
                         // panics if i is not a string

// Safe form with comma-ok idiom
s, ok := i.(string)     // s = "hello world", ok = true
n, ok := i.(int)        // n = 0, ok = false — no panic

// Pattern: extract then use
if s, ok := i.(string); ok {
    fmt.Println(strings.ToUpper(s))
}
```

**Q16. What is a type switch in Go?**
A type switch runs different code depending on the dynamic type of an interface value. It's a multi-branch type assertion.

```go
func process(v interface{}) {
    switch val := v.(type) {
    case nil:
        fmt.Println("nil value")
    case int:
        fmt.Printf("integer: %d\n", val)
    case string:
        fmt.Printf("string: %q\n", val)
    case []byte:
        fmt.Printf("bytes: %x\n", val)
    case error:
        fmt.Printf("error: %v\n", val)
    default:
        fmt.Printf("unknown: %T = %v\n", val, val)
    }
}
```

---

## Struct Tags

**Q17. What are struct tags and how are they used?**
Struct tags are string literals attached to struct fields that provide metadata used by reflection-based libraries like `encoding/json`, `database/sql`, and `gopkg.in/yaml.v3`.

```go
type User struct {
    ID        int       `json:"id" db:"user_id"`
    Name      string    `json:"name" db:"name"`
    Email     string    `json:"email" db:"email"`
    Password  string    `json:"-"`                 // "-" means omit from JSON
    CreatedAt time.Time `json:"created_at,omitempty" db:"created_at"`
}

u := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
b, _ := json.Marshal(u)
// {"id":1,"name":"Alice","email":"alice@example.com"}
// Password omitted (json:"-"), CreatedAt omitted (omitempty + zero value)
```

Common tag conventions:
- `json:"field_name"` — JSON key name
- `json:"name,omitempty"` — omit if zero value
- `json:"-"` — never include in JSON
- `db:"column_name"` — database column name (sqlx, gorm)
- `yaml:"field_name"` — YAML key name
- `validate:"required,email"` — validation rules (go-validator)

**Q18. What happens if a struct tag is malformed?**
Malformed tags don't cause a compile error. The `reflect` package's `StructTag.Lookup()` method will fail to parse them correctly and return empty strings. Some libraries may silently ignore malformed tags, others may return errors at runtime. Use `go vet` to detect malformed struct tags at development time.

---

## Embedding vs Inheritance

**Q19. What is the key difference between embedding in Go and inheritance in Java?**

| Aspect | Java Inheritance | Go Embedding |
|--------|-----------------|--------------|
| Relationship | `is-a` (Dog is-an Animal) | `has-a` (Dog has a LivingThing) |
| Polymorphism | Via class hierarchy | Via interfaces only |
| Method override | Yes — `@Override` | No — outer type can shadow embedded method |
| Multiple | Single inheritance | Multiple embedding allowed |
| `super` equivalent | `super.Method()` | `s.EmbeddedType.Method()` |
| Runtime dispatch | Virtual method table | Compile-time method promotion |

```go
// Java: class Dog extends Animal { ... }
// -- Dog IS-AN Animal, polymorphism via class

// Go: struct Dog with embedded Animal
type Animal struct{ name string }
func (a Animal) Breathe() string { return a.name + " breathes" }

type Dog struct {
    Animal // embedded — promoted Breathe() method
    breed string
}
func (d Dog) Fetch() string { return d.name + " fetches!" }

d := Dog{Animal: Animal{name: "Rex"}, breed: "Lab"}
d.Breathe() // promoted from Animal
d.Fetch()   // Dog's own method

// But Dog is NOT an Animal interface unless Animal defines methods
// and Dog happens to have all of them
```

**Q20. What happens when two embedded structs have a method with the same name?**

```go
type A struct{}
func (A) Hello() string { return "Hello from A" }

type B struct{}
func (B) Hello() string { return "Hello from B" }

type C struct {
    A
    B
}

c := C{}
// c.Hello() — COMPILE ERROR: ambiguous selector c.Hello
// Must qualify explicitly:
fmt.Println(c.A.Hello()) // Hello from A
fmt.Println(c.B.Hello()) // Hello from B

// If C defines its own Hello, it shadows both:
func (c C) Hello() string { return "Hello from C" }
c.Hello() // Hello from C — no ambiguity
```

---

## Interfaces and nil

**Q21. What is the nil interface gotcha in Go?**
This is Go's most infamous trap. An interface value has two components: a type pointer and a value pointer. An interface is `nil` only when BOTH are nil. If you store a nil pointer of a concrete type into an interface, the interface is NOT nil — it has a non-nil type component.

```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func mayFail() error {
    var err *MyError = nil // nil pointer of concrete type
    // ...
    return err // DANGER: returns interface{type=*MyError, value=nil}
}

err := mayFail()
if err != nil { // TRUE — the interface is NOT nil!
    fmt.Println("there is an error") // This prints!
    fmt.Println(err.Error()) // PANIC: nil pointer dereference
}
```

Fix: return `nil` directly, not a typed nil:
```go
func mayFail() error {
    // ...
    return nil // returns interface{type=nil, value=nil} — truly nil
}
```

**Q22. What is the difference between `interface{}` and `any` in Go?**
They are identical. `any` was introduced in Go 1.18 as a built-in alias for `interface{}`. Using `any` is preferred for readability in modern Go code:
```go
// Pre-1.18
func process(v interface{}) { ... }

// Post-1.18 (preferred)
func process(v any) { ... }
```

**Q23. Can you compare two interface values in Go?**
Yes, with `==`, but with a caveat: if the underlying concrete type is not comparable (e.g., a slice), the comparison panics at runtime.

```go
var a, b interface{}

a, b = 1, 1
fmt.Println(a == b) // true

a, b = []int{1, 2}, []int{1, 2}
// fmt.Println(a == b) // PANIC at runtime: comparing uncomparable type []int
```

---

## Dependency Injection and Mocking

**Q24. How does Go achieve dependency injection using interfaces?**
Go doesn't use a DI framework by default. Instead, accept interface types as function or struct parameters, and pass concrete implementations at the call site:

```go
type EmailSender interface {
    Send(to, subject, body string) error
}

type UserService struct {
    emailer EmailSender
    db      Database
}

func NewUserService(emailer EmailSender, db Database) *UserService {
    return &UserService{emailer: emailer, db: db}
}

// In tests, inject a mock:
type MockEmailSender struct {
    sent []string
}
func (m *MockEmailSender) Send(to, subject, body string) error {
    m.sent = append(m.sent, to)
    return nil
}
```

**Q25. How do you perform mocking in Go tests without a mocking library?**
Define an interface matching the dependency, then write a hand-crafted mock struct that implements it:

```go
type Store interface {
    GetUser(id int) (*User, error)
    SaveUser(u *User) error
}

// Mock implementation for tests
type MockStore struct {
    users map[int]*User
    saveErr error
}

func (m *MockStore) GetUser(id int) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, fmt.Errorf("user %d not found", id)
    }
    return u, nil
}

func (m *MockStore) SaveUser(u *User) error {
    return m.saveErr // return configured error or nil
}

// In test:
func TestProcessUser(t *testing.T) {
    store := &MockStore{
        users: map[int]*User{1: {ID: 1, Name: "Alice"}},
    }
    err := ProcessUser(store, 1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}
```

---

**Total: 25 theory questions covering structs, methods, interfaces, embedding, type assertions, struct tags, and DI patterns.**
