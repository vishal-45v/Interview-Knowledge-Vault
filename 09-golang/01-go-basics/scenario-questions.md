# Chapter 01 — Go Basics: Scenario Questions

These are open-ended, situational questions asked in coding rounds and system design interviews. Each question is followed by what the interviewer is looking for.

---

## Functions and Return Values

**S1. You have a function that should return a user and an error. How do you design it in Go?**

Expected answer:
```go
type User struct {
    ID    int
    Name  string
    Email string
}

func GetUserByID(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("invalid user ID: %d", id)
    }
    // Imagine a database call here
    user := &User{ID: id, Name: "Alice", Email: "alice@example.com"}
    return user, nil
}

func main() {
    user, err := GetUserByID(1)
    if err != nil {
        log.Fatalf("failed to get user: %v", err)
    }
    fmt.Printf("User: %s (%s)\n", user.Name, user.Email)
}
```
Interviewer looks for: multiple return values, `nil` for error on success, pointer return for optional result, error checking at call site.

---

**S2. Write a function that parses a comma-separated list of integers from a string and returns the sum and any parsing error.**

```go
func sumCSV(s string) (int, error) {
    if s == "" {
        return 0, nil
    }
    parts := strings.Split(s, ",")
    total := 0
    for i, p := range parts {
        n, err := strconv.Atoi(strings.TrimSpace(p))
        if err != nil {
            return 0, fmt.Errorf("invalid integer at position %d: %q", i, p)
        }
        total += n
    }
    return total, nil
}

// Usage
sum, err := sumCSV("1, 2, 3, 4, 5")
if err != nil {
    log.Fatal(err)
}
fmt.Println(sum) // 15
```

---

## Slices and Maps

**S3. A slice was passed to a function and modified inside. Did the caller see the changes? Explain why or why not.**

Answer: It depends on what modification was done.

```go
// Case 1: Modifying existing elements — caller DOES see changes
func doubleAll(nums []int) {
    for i := range nums {
        nums[i] *= 2  // modifying through the shared backing array
    }
}

s := []int{1, 2, 3}
doubleAll(s)
fmt.Println(s) // [2 4 6] — caller sees changes

// Case 2: Appending — caller does NOT see changes (if cap is exceeded)
func appendElement(nums []int) {
    nums = append(nums, 99) // may create a new backing array
    fmt.Println("inside:", nums) // [1 2 3 99]
}

s := []int{1, 2, 3}
appendElement(s)
fmt.Println("outside:", s) // [1 2 3] — caller does NOT see 99
```

The key insight: a slice header (pointer, len, cap) is copied on function call. Modifying elements via the copied pointer affects the shared backing array. But `append` returns a new header, and since Go is pass-by-value, the caller's header isn't updated.

---

**S4. You need to iterate over a map in sorted key order. How do you do it?**

```go
import "sort"

func sortedMapIteration(m map[string]int) {
    keys := make([]string, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    sort.Strings(keys)

    for _, k := range keys {
        fmt.Printf("%s: %d\n", k, m[k])
    }
}

scores := map[string]int{"charlie": 80, "alice": 95, "bob": 70}
sortedMapIteration(scores)
// alice: 95
// bob: 70
// charlie: 80
```

---

**S5. You need a stack data structure in Go. Implement it using a slice.**

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return top, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int { return len(s.items) }

// Without generics (pre Go 1.18):
type IntStack []int

func (s *IntStack) Push(v int)         { *s = append(*s, v) }
func (s *IntStack) Pop() (int, bool) {
    if len(*s) == 0 { return 0, false }
    n := len(*s)
    v := (*s)[n-1]
    *s = (*s)[:n-1]
    return v, true
}
```

---

**S6. You have a function that builds a large string by concatenating in a loop. It's slow. How do you fix it?**

Problem: string concatenation with `+` in a loop creates a new string each iteration — O(n²) total.

```go
// SLOW — creates a new string each iteration
func buildBad(words []string) string {
    result := ""
    for _, w := range words {
        result += w + " "
    }
    return result
}

// FAST — strings.Builder uses an internal byte buffer
func buildGood(words []string) string {
    var sb strings.Builder
    sb.Grow(estimatedSize(words)) // optional: pre-allocate
    for _, w := range words {
        sb.WriteString(w)
        sb.WriteByte(' ')
    }
    return sb.String()
}

// Also good for joining
func buildJoin(words []string) string {
    return strings.Join(words, " ")
}
```

---

## Closures and Variable Capture

**S7. A developer writes the following code expecting to print 0, 1, 2 but it prints 3, 3, 3. Why? Fix it.**

```go
// Buggy code
funcs := make([]func(), 3)
for i := 0; i < 3; i++ {
    funcs[i] = func() { fmt.Println(i) }
}
for _, f := range funcs {
    f()
}
// Prints: 3 3 3 (pre Go 1.22)

// Fix 1: Shadow the variable inside the loop
for i := 0; i < 3; i++ {
    i := i // new variable per iteration
    funcs[i] = func() { fmt.Println(i) }
}

// Fix 2: Pass as parameter
for i := 0; i < 3; i++ {
    func(n int) {
        funcs[n] = func() { fmt.Println(n) }
    }(i)
}

// Fix 3: Go 1.22+ — loop variables are per-iteration by default
// No fix needed in Go 1.22+
```

---

## Defer

**S8. What will the following code print? Explain defer ordering.**

```go
func main() {
    fmt.Println("start")
    defer fmt.Println("deferred 1")
    defer fmt.Println("deferred 2")
    defer fmt.Println("deferred 3")
    fmt.Println("end")
}
```

Output:
```
start
end
deferred 3
deferred 2
deferred 1
```

Reason: defers are pushed onto a stack (LIFO). When `main` returns, they execute in reverse order of their registration.

---

**S9. A developer uses defer inside a loop to close database rows. What's wrong with this pattern and how do you fix it?**

```go
// PROBLEMATIC — defer runs at function return, not at end of each iteration
func processAll(db *sql.DB, ids []int) error {
    for _, id := range ids {
        rows, err := db.Query("SELECT * FROM users WHERE id = ?", id)
        if err != nil {
            return err
        }
        defer rows.Close() // This only runs when processAll returns!
        // process rows...
    }
    return nil
}

// FIX 1: Use a helper function — defer runs when the helper returns
func processOne(db *sql.DB, id int) error {
    rows, err := db.Query("SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        return err
    }
    defer rows.Close() // now runs after each processOne call
    // process rows...
    return nil
}

func processAll(db *sql.DB, ids []int) error {
    for _, id := range ids {
        if err := processOne(db, id); err != nil {
            return err
        }
    }
    return nil
}

// FIX 2: Close explicitly without defer
func processAll2(db *sql.DB, ids []int) error {
    for _, id := range ids {
        rows, err := db.Query("SELECT * FROM users WHERE id = ?", id)
        if err != nil {
            return err
        }
        // process rows...
        rows.Close() // explicit close at end of iteration
    }
    return nil
}
```

---

## Pointers

**S10. When would you pass a struct by pointer vs by value in Go?**

```go
type SmallConfig struct {
    Debug bool
    Port  int
}

type LargeRequest struct {
    Headers map[string]string
    Body    []byte // potentially megabytes
    Params  [100]string // large array
}

// Small struct — pass by value is fine, no aliasing issues
func processConfig(cfg SmallConfig) {
    // cfg is a copy, modifications don't affect caller
}

// Large struct — pass by pointer to avoid copying
func processRequest(req *LargeRequest) error {
    // req is a pointer, no copying of the large data
    return nil
}

// When the function needs to modify the caller's data
func incrementAge(p *Person) {
    p.Age++ // modifies the original
}

// When implementing methods that need to mutate state
type Cache struct {
    data map[string]string
}

func (c *Cache) Set(key, val string) {
    if c.data == nil {
        c.data = make(map[string]string)
    }
    c.data[key] = val
}
```

---

**S11. Implement a function that safely converts a string to an integer, returning a default value on failure.**

```go
func parseIntOrDefault(s string, defaultVal int) int {
    n, err := strconv.Atoi(strings.TrimSpace(s))
    if err != nil {
        return defaultVal
    }
    return n
}

// More general approach with named return for documentation
func tryParseInt(s string) (value int, ok bool) {
    n, err := strconv.Atoi(strings.TrimSpace(s))
    if err != nil {
        return 0, false
    }
    return n, true
}
```

---

## Append Gotcha

**S12. Two slices share a backing array. Show a scenario where appending to one corrupts the other, and explain how to prevent it.**

```go
original := make([]int, 3, 6) // len=3, cap=6
original[0], original[1], original[2] = 1, 2, 3

// Both slices share the same backing array
a := original[:3]
b := original[:3]

// Appending to 'a' does NOT allocate (cap has room)
a = append(a, 10)
fmt.Println(a)        // [1 2 3 10]
fmt.Println(b)        // [1 2 3] — b doesn't see 10 (its len is still 3)

// But if we then access original[3]:
fmt.Println(original[:4]) // [1 2 3 10] — backing array was modified!

// DANGEROUS: modifications through 'a' affect what 'b' can see via re-slicing
// Fix: use full slice expression to limit capacity, forcing new allocation on append
a := original[:3:3] // three-index slice: pointer, len=3, cap=3
b := original[:3:3]

a = append(a, 10) // NOW allocates a new backing array because cap==len
// a and b now have independent backing arrays
```

---

**S13. Write a function that removes duplicates from an integer slice without using a second slice of the same capacity.**

```go
func removeDuplicates(nums []int) []int {
    if len(nums) == 0 {
        return nums
    }
    seen := make(map[int]bool)
    result := nums[:0] // reuse the backing array, zero length
    for _, n := range nums {
        if !seen[n] {
            seen[n] = true
            result = append(result, n)
        }
    }
    return result
}

// For a sorted slice, O(1) space:
func removeDuplicatesSorted(nums []int) []int {
    if len(nums) == 0 {
        return nums
    }
    j := 0
    for i := 1; i < len(nums); i++ {
        if nums[i] != nums[j] {
            j++
            nums[j] = nums[i]
        }
    }
    return nums[:j+1]
}
```

---

**S14. Implement a simple cache using a map with a mutex for thread safety.**

```go
import "sync"

type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

func NewCache() *Cache {
    return &Cache{items: make(map[string]string)}
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}
```

---

**S15. Write a function using variadic arguments that accepts a format string and produces a log line with a timestamp prefix.**

```go
import (
    "fmt"
    "time"
)

func logf(format string, args ...any) string {
    ts := time.Now().Format("2006-01-02 15:04:05")
    msg := fmt.Sprintf(format, args...)
    return fmt.Sprintf("[%s] %s", ts, msg)
}

// Usage
line := logf("User %d logged in from %s", 42, "192.168.1.1")
// [2026-03-17 10:30:00] User 42 logged in from 192.168.1.1
```

---

**S16. You need to filter a slice of integers. Write a generic-style filter function (using both the pre-1.18 and post-1.18 approaches).**

```go
// Pre-generics: specific to int
func filterInts(nums []int, predicate func(int) bool) []int {
    result := make([]int, 0, len(nums))
    for _, n := range nums {
        if predicate(n) {
            result = append(result, n)
        }
    }
    return result
}

// Post-generics (Go 1.18+)
func filter[T any](items []T, predicate func(T) bool) []T {
    result := make([]T, 0, len(items))
    for _, item := range items {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}

// Usage
evens := filterInts([]int{1, 2, 3, 4, 5}, func(n int) bool { return n%2 == 0 })
// [2 4]

words := filter([]string{"apple", "banana", "cherry"}, func(s string) bool {
    return len(s) > 5
})
// ["banana", "cherry"]
```

---

**S17. How would you implement a set data structure in Go?**

```go
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{items: make(map[T]struct{})}
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
    delete(s.items, item)
}

func (s *Set[T]) Contains(item T) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set[T]) Len() int { return len(s.items) }

// Pre-generics idiom
type StringSet map[string]struct{}

func (s StringSet) Add(v string)      { s[v] = struct{}{} }
func (s StringSet) Contains(v string) bool { _, ok := s[v]; return ok }
```

Note: `struct{}` uses zero bytes — it's the idiomatic way to use a map as a set.

---

**S18. Write a function that finds the two numbers in a slice that add up to a target sum.**

```go
func twoSum(nums []int, target int) (int, int, bool) {
    seen := make(map[int]int) // value -> index
    for i, n := range nums {
        complement := target - n
        if j, ok := seen[complement]; ok {
            return j, i, true
        }
        seen[n] = i
    }
    return 0, 0, false
}

i, j, found := twoSum([]int{2, 7, 11, 15}, 9)
if found {
    fmt.Printf("indices: %d and %d\n", i, j) // indices: 0 and 1
}
```

---

**S19. Explain what happens step by step when you call `append` on a slice with `len == cap`.**

```go
s := make([]int, 3, 3) // len=3, cap=3, backing array: [0, 0, 0]

// Before append: pointer=A, len=3, cap=3

s = append(s, 42)
// 1. append sees len(3) == cap(3)
// 2. allocate new backing array, typically 2x size: cap=6
// 3. copy existing 3 elements to new array
// 4. write 42 at index 3
// 5. return new slice header: pointer=B (new array), len=4, cap=6
// Old array at pointer A is now unreferenced and eligible for GC
```

---

**S20. You need to implement a frequency counter for words in a string. Write it in Go.**

```go
func wordFrequency(text string) map[string]int {
    freq := make(map[string]int)
    words := strings.Fields(text) // splits on any whitespace
    for _, word := range words {
        word = strings.ToLower(strings.Trim(word, ".,!?\"'"))
        if word != "" {
            freq[word]++
        }
    }
    return freq
}

// Top N words by frequency
func topN(freq map[string]int, n int) []string {
    type wordCount struct {
        word  string
        count int
    }
    pairs := make([]wordCount, 0, len(freq))
    for w, c := range freq {
        pairs = append(pairs, wordCount{w, c})
    }
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].count > pairs[j].count
    })
    result := make([]string, 0, n)
    for i := 0; i < n && i < len(pairs); i++ {
        result = append(result, pairs[i].word)
    }
    return result
}
```
