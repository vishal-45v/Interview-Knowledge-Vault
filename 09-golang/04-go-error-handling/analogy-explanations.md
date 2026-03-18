# Go Error Handling — Analogy Explanations

## Error as a Value: The Receipt for What Went Wrong

Imagine you order a package from an online store. When the package arrives, you get a delivery receipt. If the delivery fails, you get a failure receipt — it says exactly what went wrong: "address not found," "recipient unavailable," "package damaged in transit."

In Go, errors work the same way. When a function completes normally, it returns the result. When something goes wrong, it returns an error receipt — a specific, named, inspectable value that says exactly what failed.

This is different from exceptions (used in Java, Python, C#), which are more like a fire alarm going off in the building. You can hear the alarm but you don't know which floor, which room, or what type of fire until you investigate the stack trace.

Go says: hand the caller a descriptive receipt. They can read it, pass it along, inspect it, add more information to it, or act on it — all at the call site, not buried in a try/catch block somewhere up the stack.

```go
// Receipt-based: caller knows immediately what went wrong
user, err := store.GetUser(id)
if err != nil {
    // Read the receipt, decide what to do
    if errors.Is(err, store.ErrNotFound) {
        return http.StatusNotFound
    }
    return http.StatusInternalServerError
}
```

---

## panic: The Fire Alarm — Only for Truly Exceptional Situations

A fire alarm in a building should only go off when there is an actual fire. If it rings every time someone burns toast, people stop evacuating when it counts.

`panic` in Go is the fire alarm. It is designed for situations where something has gone so wrong that the program's safety cannot be guaranteed — a nil pointer on a required dependency was passed in, an internal invariant has been violated, a data structure is in an impossible state.

If you use `panic` for expected conditions like "file not found" or "user doesn't exist," your code is the person who keeps burning toast and triggering the alarm. Everyone loses trust in the signal.

```go
// Fire alarm — appropriate: core invariant violated at startup
func New(db *sql.DB) *Service {
    if db == nil {
        panic("service: db is nil — this is a programming error, not a runtime condition")
    }
    return &Service{db: db}
}

// Toast burning — NOT appropriate: expected runtime condition
func readFile(path string) []byte {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err) // WRONG — fire alarm for toast
    }
    return data
}
```

The test: ask yourself, "Should a correct, bug-free program ever hit this condition during normal operation?" If yes, return an error. If no (it would mean the programmer wrote wrong code), then panic is appropriate.

---

## recover: The Emergency Exit Plan

Every building has an emergency exit plan — not because fires are expected daily, but because when one does happen, you need a predetermined path to safety that doesn't require thinking.

`recover()` is Go's emergency exit plan for panics. It's the code you write in advance that says "if something catastrophically unexpected happens, at least stop the bleeding and report the situation before shutting down."

The key rule of emergency exit plans: they only work if you set them up **before** the emergency. `recover()` inside a `defer` statement is that setup. If you haven't set it up in advance (deferred it), `recover()` cannot help you mid-panic.

```go
// Emergency exit: set it up before anything can go wrong
func safeParse(data []byte) (result *Doc, err error) {
    defer func() {
        // Emergency exit: if parsing panics, convert to error
        if r := recover(); r != nil {
            err = fmt.Errorf("parse panicked: %v", r)
        }
    }()
    result = dangerousThirdPartyParser(data) // might panic
    return
}
```

Notice that `recover()` must be in the `defer` — you set up the exit route before entering the dangerous area. You cannot call `recover()` *after* a panic has started and expect it to work.

---

## errors.Is: Checking if a Russian Doll Contains a Specific Doll

A Russian nesting doll (Matryoshka) is a set of dolls where each one contains a smaller one inside. When you wrap an error in Go, you create exactly this structure: `outerErr` contains `middleErr` contains `innerErr`.

`errors.Is(err, target)` is like asking: "Does this Russian doll set contain the specific blue doll with a red hat?"

It doesn't care how many layers of dolls are in between. It opens the outer doll, checks — not the one. Opens the next doll, checks — still not. Opens the innermost doll, checks — yes! That's the one.

```go
base := errors.New("connection refused")   // the specific blue doll
layer1 := fmt.Errorf("tcp dial: %w", base)  // wrapped in one doll
layer2 := fmt.Errorf("connect db: %w", layer1) // wrapped in another
layer3 := fmt.Errorf("init app: %w", layer2)   // wrapped in a third

// errors.Is opens each doll until it finds the base
errors.Is(layer3, base) // true — found it at the innermost layer
```

Without `errors.Is`, you would only look at the outermost doll and give up:

```go
layer3 == base // false — only checks the outermost layer
```

---

## errors.As: Checking if Anyone in the Family Has Red Hair

`errors.As` is like asking: "Does anyone in this family tree have red hair?" You're not looking for one specific person — you're looking for anyone who matches a certain description (a type).

You start with grandma (the outermost error), check her — she has brown hair (wrong type). Her daughter — black hair (wrong type). Her grandson — red hair! Found it. You pull him out and study him.

```go
type RateLimitError struct {
    RetryAfter time.Duration
}
func (e *RateLimitError) Error() string { ... }

// Family tree (error chain)
inner := &RateLimitError{RetryAfter: 30 * time.Second}
middle := fmt.Errorf("api call: %w", inner)
outer := fmt.Errorf("sync data: %w", middle)

// "Does anyone in this family have red hair (type *RateLimitError)?"
var rle *RateLimitError
if errors.As(outer, &rle) {
    // Found the red-haired family member — now you can read their specific traits
    fmt.Printf("retry after: %s\n", rle.RetryAfter)
}
```

The contrast with `errors.Is`:
- `errors.Is` asks "Is THIS SPECIFIC PERSON in the family?" (identity)
- `errors.As` asks "Is ANYONE OF THIS TYPE in the family?" (type check)

---

## Sentinel Errors: A "No Entry" Sign Everyone Recognizes

Imagine driving in a city. Some intersections have a standard "No Entry" sign — red circle, white bar. Every driver in every country recognizes this sign. They don't need to read text. They see the shape and act accordingly.

Sentinel errors are the "No Entry" signs of your codebase. They are well-known, standardized values that every caller recognizes and can react to without parsing strings:

```go
// Standard library sentinels — everyone knows these
io.EOF          // "stream is done"
sql.ErrNoRows   // "query found nothing"
os.ErrNotExist  // "file/directory does not exist"

// Your package's sentinels — your callers learn to recognize these
store.ErrNotFound    // "the resource you asked for doesn't exist"
store.ErrConflict    // "this would create a duplicate"
```

Just like traffic signs, the value of a sentinel is its **identity**, not its text. `io.EOF` is recognized because of where it points in memory, not because of what its `Error()` string says.

The downside of sentinels is the same as overusing No Entry signs: if you put them everywhere, they lose their specificity. A sentinel tells you only *that* something happened, not *what resource* or *which record* triggered it. For richer context, you need a custom error type.

---

## Error Wrapping: Building a Layered Context Sandwich

Error wrapping is like making a club sandwich. The bottom layer is the original problem (the bread at the bottom). As the error passes through layers of your application, each layer adds its own context on top — like layers of ingredients.

```
Top layer:       "handleHTTPRequest: "
                 ↓
Middle layer:    "UserService.GetProfile: "
                 ↓
Database layer:  "store.GetUser 42: "
                 ↓
Bottom (bread):  "sql: no rows in result set"
```

When you read the final error, you get a complete, ordered description: `handleHTTPRequest: UserService.GetProfile: store.GetUser 42: sql: no rows in result set`.

This is much more useful than just "sql: no rows in result set" — you know exactly which path through the code produced the problem.

`errors.Is` and `errors.As` let you "reach through" the sandwich layers to find specific ingredients at any level.

---

## The Non-Nil Nil Interface Trap: The Empty Box in a Labeled Wrapper

Imagine you order a product online. You receive a box — it has a label, it has a company name printed on it. When you open it, there's nothing inside.

Is the box something or nothing? It's clearly **something** — it exists, it has properties, it takes up space. But its contents are empty.

A typed nil in an interface is the same: the interface wrapper exists (it has type information), but the value inside is nil.

```go
var err *MyError = nil // empty box labeled "*MyError"
var iface error = err  // wrapped in an interface wrapper

// The wrapper (interface) is not nil — it exists with type info
iface != nil // true! The box exists even though it's empty
```

A truly nil `error` is like receiving nothing at all — no box, no wrapper, nothing:

```go
var iface error = nil // no box, no wrapper, nothing
iface == nil // true
```

The lesson: never return a labeled-but-empty box. When you have nothing to return, return actual nothing (the untyped `nil`).
