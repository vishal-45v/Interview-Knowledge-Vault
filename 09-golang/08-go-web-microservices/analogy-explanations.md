# Go Web & Microservices — Analogy Explanations

---

## HTTP Handler: A Post Office Clerk

An HTTP handler is like a post office clerk. When a customer (HTTP request) walks in, the clerk:

1. Looks at the address on the package (request URL and method)
2. Checks the ID (authentication headers)
3. Opens the package (reads and decodes the request body)
4. Does the work (business logic — saves to DB, calls a service)
5. Seals the response envelope (sets headers, status code)
6. Hands it back (writes response body)

The post office building is the `http.ServeMux`. Its job is to look at each incoming address and route it to the correct clerk. Clerk A handles "packages addressed to `/users`", Clerk B handles "packages addressed to `/orders`".

```go
// The clerk (handler) — one function per "desk"
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    // Look at the address on the package:
    id := r.PathValue("id")

    // Do the work:
    user := db.FindUser(id)

    // Seal and hand back the response:
    json.NewEncoder(w).Encode(user)
})
```

The clerk doesn't need to understand routing — that's the post office's job. The clerk just focuses on their specific task.

---

## Middleware Chain: A Security Checkpoint Line at an Airport

When you pass through an airport, you go through multiple security checks in a specific order:

1. **Ticket check** — verify you have a valid boarding pass (authentication)
2. **ID check** — verify your identity matches your ticket (authorization)
3. **Luggage scan** — check your bags (request body inspection/logging)
4. **Metal detector** — final security check (rate limiting, validation)
5. **Gate** — the actual destination (your handler)

Middleware works exactly the same way. Each checkpoint wraps the next:

```
Request → RequestID → Logging → Recovery → Auth → RateLimit → Handler
Response ←    ←          ←         ←        ←       ←           ←
```

Each guard at each checkpoint has the power to:
- **Let you through** (call `next.ServeHTTP(w, r)`)
- **Turn you away** (return an error, don't call next)
- **Stamp your passport** (add something to the request context)
- **Check what happened** after you passed through (wrap the response writer)

The key insight: each checkpoint sees the request BEFORE passing it on, and sees the response AFTER it comes back. This is why the wrapping order matters — the outermost guard is the first to check and the last to see the response.

```go
// RequestID is added outermost because ALL subsequent middleware needs it
handler := Chain(mux,
    RequestIDMiddleware,   // stamp: give passenger a tracking number
    LoggingMiddleware,     // log: record entry time (has tracking number)
    RecoveryMiddleware,    // catch panics: safety net
    JWTMiddleware,         // ID check: is this a valid traveler?
)
```

---

## Graceful Shutdown: A Restaurant That Closes for the Night

Imagine a restaurant at closing time. A bad owner would:
- Flip the "CLOSED" sign and immediately lock all the doors
- Throw out customers who are mid-meal
- Turn off the kitchen while orders are still being cooked

A good owner does a **graceful shutdown**:
1. Stop the host from seating new customers (stop accepting new connections)
2. Let existing customers finish their meals (allow in-flight requests to complete)
3. Give everyone a reasonable amount of time (shutdown timeout, e.g., 30 seconds)
4. After everyone is done, close the kitchen (close DB pool, flush metrics)
5. Lock the doors (process exits)

```go
// "Flip the sign" — stop accepting new customers
srv.Shutdown(ctx) // stops accepting new connections

// "Let current customers finish" — in-flight requests drain
// srv.Shutdown waits until all goroutines handling requests finish

// "Close the kitchen" — cleanup hooks
db.Close()
metrics.Flush()
```

The `context.WithTimeout(context.Background(), 30*time.Second)` is the restaurant's policy: "We wait up to 30 seconds. If a customer is still here after 30 minutes, we politely but firmly ask them to leave" (`srv.Close()`).

---

## gRPC vs REST: A Scripted Phone Call vs a Freeform Letter

**REST/JSON** is like exchanging letters:
- You write a letter in plain text (JSON)
- No strict format — the recipient tries to parse what you wrote
- If you misspell a field, they just ignore it (or crash trying to use it)
- Anyone with a pen can write a letter (universal — `curl`, browsers, anything)
- The envelope (HTTP/1.1) goes back and forth for every letter

**gRPC/Protobuf** is like a scripted phone call:
- Both sides have the **exact same script** (the `.proto` file)
- If you say something that's not in the script, the compiler catches it immediately
- The conversation is compressed (binary protobuf — 3–10× smaller than text)
- You use the same phone line for multiple conversations simultaneously (HTTP/2 multiplexing)
- Not every phone can make this call — browsers need a special adapter (`grpc-web`)

```protobuf
// The "script" both sides share
service OrderService {
    rpc PlaceOrder(PlaceOrderRequest) returns (Order) {}
    // Both client and server are generated from this — compile-time contract
}
```

Use letters (REST) for communication with the outside world — universal, readable. Use the scripted phone call (gRPC) for internal communication where speed and type-safety matter more than universality.

---

## Circuit Breaker: A Power Safety Switch

In electrical engineering, a **circuit breaker** is a safety switch. When current becomes dangerously high (a short circuit or overload), the breaker trips — cutting power immediately to protect the rest of the circuit.

Without a circuit breaker: one faulty appliance causes the entire house to catch fire.

In microservices, the equivalent is a downstream service that starts failing:
- Without a circuit breaker: your service keeps making requests to the failing service, each one waiting until timeout (e.g., 30 seconds). Under load, all your goroutines pile up waiting. Your service becomes slow and eventually unavailable too. This is a **cascading failure** — one broken service takes down the entire chain.
- With a circuit breaker: after 5 failures, the breaker **trips (opens)**. New requests immediately return an error without even trying to call the downstream service. Your goroutines are freed instantly. After 30 seconds, the breaker tries one request (half-open). If it succeeds, the circuit closes (normal operation). If it fails, it stays open.

```
Normal traffic:   [A] → [B] → [C: healthy]  ✓
                           ↑
C starts failing: [A] → [B] → [C: slow/error] ✗ × 5
Circuit opens:    [A] → [B] → [breaker OPEN] → fast fail
                  No goroutines waiting for C! B stays healthy.
30s later:        [A] → [B] → probe request → [C: recovered?]
                                               ↓ success → circuit closes
```

The circuit breaker saves the rest of your system by "tripping" before the damage spreads.

---

## Dependency Injection: Hiring from a Staffing Agency

When a business needs a delivery driver, it has two options:

**Option 1 (no DI):** Train your own driver in-house. The business directly creates and controls the driver.
```go
func NewOrderService() *OrderService {
    db := postgres.Connect("hardcoded-connection-string") // creates own dependency
    return &OrderService{db: db}
}
// Problem: testing requires a real PostgreSQL connection!
```

**Option 2 (DI):** Hire from a staffing agency. The business says "I need someone who can drive a van" (interface). The agency provides any driver who meets the requirements.

```go
// "I need someone who can drive" (interface)
type Database interface {
    Query(ctx context.Context, sql string) (Rows, error)
}

func NewOrderService(db Database) *OrderService {
    return &OrderService{db: db} // accepts anyone who can "drive"
}

// In production: send a PostgreSQL driver
svc := NewOrderService(postgres.NewDB(connString))

// In tests: send a fake driver
svc := NewOrderService(&fakeDB{records: testData})
```

The business (OrderService) doesn't care WHERE the driver came from — only that they can drive (satisfy the interface). This makes the service testable, flexible, and easy to swap implementations.

`main.go` is the "staffing agency coordinator" — it knows which concrete driver (postgres, redis, smtp) to provide to each department (service, handler) based on the current environment.

---

## Health Checks: Car Dashboard Warning Lights

Your car has two types of indicators:

**Is the car running? (Liveness)**
- The engine light — is the engine turning? (Is the process alive?)
- If the engine stalls: restart the car (Kubernetes: restart the pod)

**Is the car ready to drive? (Readiness)**
- The fuel gauge — enough fuel? (DB connection healthy?)
- The temperature gauge — engine warmed up? (dependencies ready?)
- If any of these are off: don't send passengers in this car (Kubernetes: remove from load balancer)

```go
// Liveness: "Is the engine running?"
// Kubernetes restarts the pod if this fails
GET /healthz → {"status": "alive"}

// Readiness: "Is the car ready for passengers?"
// Kubernetes removes from load balancer rotation if this fails
GET /readyz  → {"status": "healthy", "checks": {"database": "healthy", "redis": "healthy"}}
```

A car that's alive but not ready (warming up, fueling) should still answer liveness checks (don't restart it!) but fail readiness checks (don't send it passengers yet). This is why the two endpoints are separate.
