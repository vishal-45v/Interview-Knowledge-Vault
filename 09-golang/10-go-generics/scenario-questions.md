# Go Generics — Scenario Questions

---

## Scenario 1: Generic Stack with Push, Pop, and Peek

**"Implement a generic Stack[T] with Push, Pop, and Peek operations."**

```go
package stack

// Stack[T] is a LIFO data structure that works with any type T.
type Stack[T any] struct {
    items []T
}

// Push adds an item to the top of the stack.
func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

// Pop removes and returns the top item.
// Returns the zero value and false if the stack is empty.
func (s *Stack[T]) Pop() (T, bool) {
    if s.IsEmpty() {
        var zero T
        return zero, false
    }
    n := len(s.items) - 1
    item := s.items[n]
    s.items[n] = *new(T) // help GC: zero out the slot before shrinking
    s.items = s.items[:n]
    return item, true
}

// Peek returns the top item without removing it.
func (s *Stack[T]) Peek() (T, bool) {
    if s.IsEmpty() {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

// IsEmpty reports whether the stack has no items.
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

// Len returns the number of items.
func (s *Stack[T]) Len() int { return len(s.items) }

// Usage examples:
func ExampleStack() {
    // Integer stack
    var intStack Stack[int]
    intStack.Push(10)
    intStack.Push(20)
    intStack.Push(30)

    top, _ := intStack.Peek()  // 30 (not removed)
    v, ok := intStack.Pop()    // 30, true
    v, ok = intStack.Pop()     // 20, true
    _ = top; _ = v; _ = ok

    // String stack
    var strStack Stack[string]
    strStack.Push("first")
    strStack.Push("second")
    val, _ := strStack.Pop()   // "second"
    _ = val

    // Struct stack
    type Job struct {
        ID       int
        Priority int
    }
    var jobStack Stack[Job]
    jobStack.Push(Job{ID: 1, Priority: 5})
    jobStack.Push(Job{ID: 2, Priority: 10})
    job, _ := jobStack.Pop() // Job{ID: 2, Priority: 10}
    _ = job
}
```

**Test:**
```go
func TestStack(t *testing.T) {
    s := &Stack[int]{}

    // Empty stack behavior
    _, ok := s.Pop()
    if ok {
        t.Fatal("expected false on empty stack Pop")
    }

    // Push and pop
    s.Push(1)
    s.Push(2)
    s.Push(3)

    if s.Len() != 3 {
        t.Fatalf("expected len 3, got %d", s.Len())
    }

    v, ok := s.Pop()
    if !ok || v != 3 {
        t.Fatalf("expected 3, got %d", v)
    }

    // Peek doesn't remove
    peeked, _ := s.Peek()
    if peeked != 2 {
        t.Fatalf("expected peek 2, got %d", peeked)
    }
    if s.Len() != 2 {
        t.Fatal("peek should not remove element")
    }
}
```

---

## Scenario 2: Generic Map Function

**"Write a generic Map function that transforms a slice of type T into a slice of type U."**

```go
// Map applies fn to each element of slice and returns a new slice.
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// MapWithError: map that can fail
func MapE[T, U any](slice []T, fn func(T) (U, error)) ([]U, error) {
    result := make([]U, 0, len(slice))
    for _, v := range slice {
        u, err := fn(v)
        if err != nil {
            return nil, fmt.Errorf("map at element %v: %w", v, err)
        }
        result = append(result, u)
    }
    return result, nil
}

// MapConcurrent: parallel map using goroutines
func MapConcurrent[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    var wg sync.WaitGroup
    for i, v := range slice {
        wg.Add(1)
        go func(i int, v T) {
            defer wg.Done()
            result[i] = fn(v)
        }(i, v)
    }
    wg.Wait()
    return result
}

// Usage examples:
func examples() {
    // Convert ints to strings
    nums := []int{1, 2, 3, 4, 5}
    strs := Map(nums, strconv.Itoa) // ["1", "2", "3", "4", "5"]
    _ = strs

    // Extract field from struct
    type User struct{ Name string; Age int }
    users := []User{{"Alice", 30}, {"Bob", 25}, {"Charlie", 35}}
    names := Map(users, func(u User) string { return u.Name })
    ages  := Map(users, func(u User) int { return u.Age })
    _ = names; _ = ages

    // Transform with error handling
    strNums := []string{"1", "2", "bad", "4"}
    parsed, err := MapE(strNums, strconv.Atoi)
    if err != nil {
        fmt.Println("error:", err) // "map at element bad: ..."
    }
    _ = parsed

    // Compose: Map of Map
    doubled := Map(Map(nums, func(n int) float64 { return float64(n) }),
        func(f float64) string { return fmt.Sprintf("%.1f", f*2) })
    _ = doubled
}
```

---

## Scenario 3: Generic Filter Function

**"Write a generic Filter function that keeps only elements satisfying a predicate."**

```go
// Filter returns elements where fn(element) returns true.
func Filter[T any](slice []T, fn func(T) bool) []T {
    result := make([]T, 0) // don't pre-allocate: unknown how many pass
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// FilterInPlace modifies the slice in place (avoids allocation)
func FilterInPlace[T any](slice *[]T, fn func(T) bool) {
    n := 0
    for _, v := range *slice {
        if fn(v) {
            (*slice)[n] = v
            n++
        }
    }
    *slice = (*slice)[:n]
}

// Partition splits a slice into two: those that pass and those that don't
func Partition[T any](slice []T, fn func(T) bool) (pass, fail []T) {
    for _, v := range slice {
        if fn(v) {
            pass = append(pass, v)
        } else {
            fail = append(fail, v)
        }
    }
    return
}

// Usage examples:
func filterExamples() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Keep even numbers
    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    // [2, 4, 6, 8, 10]

    // Keep numbers > 5
    big := Filter(nums, func(n int) bool { return n > 5 })
    // [6, 7, 8, 9, 10]

    // Filter structs
    type Product struct{ Name string; Price float64; InStock bool }
    products := []Product{
        {"Widget", 9.99, true},
        {"Gadget", 49.99, false},
        {"Doohickey", 4.99, true},
    }
    available := Filter(products, func(p Product) bool { return p.InStock })
    affordable := Filter(products, func(p Product) bool { return p.Price < 10 })
    _ = available; _ = affordable

    // Partition: separate adults and minors
    type Person struct{ Name string; Age int }
    people := []Person{{"Alice", 30}, {"Bob", 17}, {"Charlie", 25}, {"Dave", 15}}
    adults, minors := Partition(people, func(p Person) bool { return p.Age >= 18 })
    _ = adults; _ = minors

    // Chain Filter and Map:
    result := Map(
        Filter(nums, func(n int) bool { return n%2 == 0 }), // even only
        func(n int) string { return strconv.Itoa(n * n) },  // square, as string
    )
    // ["4", "16", "36", "64", "100"]
    _ = result

    // FilterInPlace: no allocation
    data := []int{1, -2, 3, -4, 5}
    FilterInPlace(&data, func(n int) bool { return n > 0 })
    // data = [1, 3, 5]
}
```

---

## Scenario 4: Generic Set

**"Implement a generic Set[T comparable] with Add, Remove, Contains, and set operations."**

```go
// Set is an unordered collection of unique elements.
type Set[T comparable] struct {
    m map[T]struct{}
}

func NewSet[T comparable](items ...T) Set[T] {
    s := Set[T]{m: make(map[T]struct{}, len(items))}
    for _, item := range items {
        s.Add(item)
    }
    return s
}

func (s *Set[T]) Add(item T) {
    if s.m == nil {
        s.m = make(map[T]struct{})
    }
    s.m[item] = struct{}{}
}

func (s *Set[T]) Remove(item T)     { delete(s.m, item) }
func (s Set[T]) Contains(item T) bool { _, ok := s.m[item]; return ok }
func (s Set[T]) Len() int           { return len(s.m) }
func (s Set[T]) IsEmpty() bool      { return len(s.m) == 0 }

func (s Set[T]) ToSlice() []T {
    result := make([]T, 0, len(s.m))
    for k := range s.m {
        result = append(result, k)
    }
    return result
}

// Union returns all elements from both sets
func (s Set[T]) Union(other Set[T]) Set[T] {
    result := NewSet[T]()
    for k := range s.m      { result.Add(k) }
    for k := range other.m  { result.Add(k) }
    return result
}

// Intersection returns elements present in both sets
func (s Set[T]) Intersection(other Set[T]) Set[T] {
    // Iterate the smaller set for efficiency
    small, large := s, other
    if len(s.m) > len(other.m) {
        small, large = other, s
    }
    result := NewSet[T]()
    for k := range small.m {
        if large.Contains(k) {
            result.Add(k)
        }
    }
    return result
}

// Difference returns elements in s but not in other
func (s Set[T]) Difference(other Set[T]) Set[T] {
    result := NewSet[T]()
    for k := range s.m {
        if !other.Contains(k) {
            result.Add(k)
        }
    }
    return result
}

// IsSubset returns true if all elements of s are in other
func (s Set[T]) IsSubset(other Set[T]) bool {
    for k := range s.m {
        if !other.Contains(k) {
            return false
        }
    }
    return true
}

// Usage:
func setExamples() {
    a := NewSet("apple", "banana", "cherry")
    b := NewSet("banana", "cherry", "date")

    union := a.Union(b)           // {apple, banana, cherry, date}
    inter := a.Intersection(b)    // {banana, cherry}
    diff  := a.Difference(b)      // {apple}

    fmt.Println(a.Contains("apple"))  // true
    fmt.Println(a.Contains("date"))   // false
    _ = union; _ = inter; _ = diff

    // Works with any comparable type:
    intSet := NewSet(1, 2, 3, 4, 5)
    intSet.Add(6)
    intSet.Remove(1)
    fmt.Println(intSet.Contains(3)) // true

    type Point struct{ X, Y int }
    pointSet := NewSet(Point{0, 0}, Point{1, 1})
    pointSet.Add(Point{2, 2})
    fmt.Println(pointSet.Len()) // 3
}
```

---

## Scenario 5: Generic Cache with TTL

**"Implement a generic thread-safe cache Cache[K, V] with TTL."**

```go
type entry[V any] struct {
    value     V
    expiresAt time.Time
}

type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]entry[V]
    ttl   time.Duration
}

func NewCache[K comparable, V any](ttl time.Duration) *Cache[K, V] {
    c := &Cache[K, V]{
        items: make(map[K]entry[V]),
        ttl:   ttl,
    }
    go c.cleanupLoop()
    return c
}

func (c *Cache[K, V]) Set(key K, value V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = entry[V]{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    e, ok := c.items[key]
    if !ok || time.Now().After(e.expiresAt) {
        var zero V
        return zero, false
    }
    return e.value, true
}

func (c *Cache[K, V]) Delete(key K) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *Cache[K, V]) cleanupLoop() {
    ticker := time.NewTicker(c.ttl / 2)
    defer ticker.Stop()
    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for k, e := range c.items {
            if now.After(e.expiresAt) {
                delete(c.items, k)
            }
        }
        c.mu.Unlock()
    }
}

// Usage:
userCache := NewCache[string, *User](5 * time.Minute)
userCache.Set("user:123", &User{ID: "123", Name: "Alice"})

if user, ok := userCache.Get("user:123"); ok {
    fmt.Println(user.Name)
}

sessionCache := NewCache[int64, SessionData](30 * time.Minute)
sessionCache.Set(sessionID, SessionData{UserID: "abc", Roles: []string{"admin"}})
```

---

## Scenario 6: Generic Pipeline (Functional Chaining)

**"Build a generic pipeline type that allows chaining Map and Filter operations."**

```go
// Pipeline wraps a slice and allows chaining operations.
type Pipeline[T any] struct {
    data []T
}

func From[T any](data []T) Pipeline[T] {
    return Pipeline[T]{data: data}
}

func (p Pipeline[T]) Filter(fn func(T) bool) Pipeline[T] {
    return From(Filter(p.data, fn))
}

func (p Pipeline[T]) ToSlice() []T { return p.data }
func (p Pipeline[T]) Len() int     { return len(p.data) }

// Map requires a standalone function because methods can't introduce new type params
func PipeMap[T, U any](p Pipeline[T], fn func(T) U) Pipeline[U] {
    return From(Map(p.data, fn))
}

// Usage:
type Employee struct {
    Name       string
    Department string
    Salary     float64
}

employees := []Employee{
    {"Alice", "Engineering", 120000},
    {"Bob", "Marketing", 80000},
    {"Charlie", "Engineering", 95000},
    {"Dave", "HR", 70000},
    {"Eve", "Engineering", 150000},
}

// Find average salary of engineers earning > 100k
engineerSalaries := PipeMap(
    From(employees).
        Filter(func(e Employee) bool { return e.Department == "Engineering" }).
        Filter(func(e Employee) bool { return e.Salary > 100000 }),
    func(e Employee) float64 { return e.Salary },
).ToSlice()

total := Reduce(engineerSalaries, 0.0, func(acc, s float64) float64 { return acc + s })
avg := total / float64(len(engineerSalaries))
fmt.Printf("Average: %.2f\n", avg)
```

---

## Scenario 7: Implementing constraints.Ordered from Scratch

**"Define the Ordered constraint yourself without using the constraints package."**

```go
// Ordered is satisfied by all types that support < > <= >=
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~string
}

// ClampOrdered clamps v to [min, max]
func Clamp[T Ordered](v, min, max T) T {
    if v < min { return min }
    if v > max { return max }
    return v
}

// SortedUnique returns a sorted slice with duplicates removed
func SortedUnique[T Ordered](slice []T) []T {
    if len(slice) == 0 {
        return nil
    }
    cp := make([]T, len(slice))
    copy(cp, slice)
    slices.Sort(cp)
    // remove duplicates
    n := 1
    for i := 1; i < len(cp); i++ {
        if cp[i] != cp[i-1] {
            cp[n] = cp[i]
            n++
        }
    }
    return cp[:n]
}

// Works with custom types:
type Temperature float64
type Priority int

temps := []Temperature{98.6, 37.0, 100.4, 98.6}
sorted := SortedUnique(temps) // [37.0, 98.6, 100.4]

p := Clamp(Priority(15), Priority(1), Priority(10)) // 10
_ = sorted; _ = p
```

---

## Scenario 8: Generic LRU Cache

**"Implement a generic LRU (Least Recently Used) cache."**

```go
import "container/list"

type lruEntry[K comparable, V any] struct {
    key   K
    value V
}

type LRUCache[K comparable, V any] struct {
    capacity int
    list     *list.List
    items    map[K]*list.Element
}

func NewLRUCache[K comparable, V any](capacity int) *LRUCache[K, V] {
    return &LRUCache[K, V]{
        capacity: capacity,
        list:     list.New(),
        items:    make(map[K]*list.Element, capacity),
    }
}

func (c *LRUCache[K, V]) Get(key K) (V, bool) {
    el, ok := c.items[key]
    if !ok {
        var zero V
        return zero, false
    }
    c.list.MoveToFront(el)  // mark as recently used
    return el.Value.(*lruEntry[K, V]).value, true
}

func (c *LRUCache[K, V]) Put(key K, value V) {
    if el, ok := c.items[key]; ok {
        c.list.MoveToFront(el)
        el.Value.(*lruEntry[K, V]).value = value
        return
    }
    // Evict if at capacity
    if c.list.Len() >= c.capacity {
        oldest := c.list.Back()
        if oldest != nil {
            entry := c.list.Remove(oldest).(*lruEntry[K, V])
            delete(c.items, entry.key)
        }
    }
    el := c.list.PushFront(&lruEntry[K, V]{key: key, value: value})
    c.items[key] = el
}

func (c *LRUCache[K, V]) Len() int { return c.list.Len() }

// Usage:
cache := NewLRUCache[string, []byte](100)
cache.Put("html/index", []byte("<html>...</html>"))
cache.Put("css/main", []byte("body { margin: 0; }"))

data, ok := cache.Get("html/index") // moves to front (most recent)
_ = data; _ = ok
```

---

## Scenario 9: Generic Observer/Event System

**"Implement a generic event bus that supports type-safe event subscriptions."**

```go
type EventBus[E any] struct {
    mu          sync.RWMutex
    subscribers []func(E)
}

func (b *EventBus[E]) Subscribe(fn func(E)) func() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.subscribers = append(b.subscribers, fn)
    // Return unsubscribe function
    idx := len(b.subscribers) - 1
    return func() {
        b.mu.Lock()
        defer b.mu.Unlock()
        b.subscribers = append(b.subscribers[:idx], b.subscribers[idx+1:]...)
    }
}

func (b *EventBus[E]) Publish(event E) {
    b.mu.RLock()
    subs := make([]func(E), len(b.subscribers))
    copy(subs, b.subscribers)
    b.mu.RUnlock()

    for _, fn := range subs {
        fn(event)
    }
}

// Usage:
type OrderPlaced struct {
    OrderID string
    UserID  string
    Total   float64
}

type PaymentProcessed struct {
    OrderID   string
    AmountUSD float64
    Status    string
}

orderBus   := &EventBus[OrderPlaced]{}
paymentBus := &EventBus[PaymentProcessed]{}

// Subscribe — type-safe!
unsub := orderBus.Subscribe(func(e OrderPlaced) {
    fmt.Printf("New order %s for user %s: $%.2f\n", e.OrderID, e.UserID, e.Total)
})
defer unsub()

paymentBus.Subscribe(func(e PaymentProcessed) {
    fmt.Printf("Payment %s for order %s\n", e.Status, e.OrderID)
})

orderBus.Publish(OrderPlaced{
    OrderID: "ORD-001",
    UserID:  "USR-42",
    Total:   99.99,
})
```
