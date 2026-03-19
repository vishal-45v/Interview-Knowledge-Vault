# Go Web & Microservices — Theory Questions

---

## net/http Standard Library

**Q1. How does Go's net/http server work internally? What is a Handler?**

The `net/http` package implements an HTTP/1.1 and HTTP/2 server. The core abstraction is the `http.Handler` interface:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

When `http.ListenAndServe` starts, it:
1. Creates a `net.Listener` on the given address
2. Accepts connections in a loop
3. Spawns a new goroutine for each accepted connection
4. Reads HTTP requests and dispatches to the registered Handler

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, World!")
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }
    log.Fatal(server.ListenAndServe())
}
```

---

**Q2. What is ServeMux and how does its routing work?**

`http.ServeMux` is Go's built-in HTTP request multiplexer (router). It matches incoming requests to registered handlers based on URL path patterns.

**Routing rules:**
- Exact match: `/foo` matches only `/foo`
- Subtree match: `/foo/` (trailing slash) matches `/foo/`, `/foo/bar`, `/foo/bar/baz`
- Longest prefix wins: `/foo/bar` is chosen over `/foo/` for `/foo/bar/baz`
- Host-specific patterns: `example.com/path`

```go
mux := http.NewServeMux()
mux.HandleFunc("/", homeHandler)        // matches everything not matched elsewhere
mux.HandleFunc("/api/", apiHandler)     // matches /api/ and all sub-paths
mux.HandleFunc("/api/users", usersHandler) // exact match, takes priority
```

Go 1.22 enhanced `ServeMux` with method-based routing and path parameters:

```go
// Go 1.22+
mux.HandleFunc("GET /users/{id}", getUserHandler)
mux.HandleFunc("POST /users", createUserHandler)
mux.HandleFunc("DELETE /users/{id}", deleteUserHandler)
```

---

**Q3. What is the middleware pattern in net/http?**

Middleware wraps a `Handler` to add behavior before/after the handler runs. It follows the decorator pattern:

```go
type Middleware func(http.Handler) http.Handler

// Logging middleware
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Recovery middleware (prevents panics from crashing the server)
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Chaining middlewares:
handler := LoggingMiddleware(RecoveryMiddleware(AuthMiddleware(mux)))
```

---

**Q4. How do popular Go frameworks (Gin, Echo, Chi, Fiber) compare to the standard library?**

| Feature | net/http | Chi | Gin | Echo | Fiber |
|---------|----------|-----|-----|------|-------|
| Path params | Go 1.22+ | Yes | Yes | Yes | Yes |
| Middleware | Manual | Built-in | Built-in | Built-in | Built-in |
| Validation | No | No | binding | validator | validator |
| HTTP/2 | Yes | Yes | Yes | Yes | No (fasthttp) |
| net/http compat | — | Yes | Yes | Yes | No |
| Benchmarks | Baseline | ~1.1x | ~2x | ~2x | ~5x |

**Chi**: thin wrapper over `net/http`, fully compatible, best for projects wanting stdlib-compatible middleware.

**Gin**: fastest among `net/http`-compatible frameworks, large ecosystem, uses `httprouter`.

**Fiber**: uses `fasthttp` (not `net/http`), fastest but incompatible with standard middleware.

```go
// Gin example:
r := gin.Default()
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"id": id})
})
r.Run(":8080")
```

---

**Q5. How do you handle JSON request and response in Go?**

```go
// Request body decoding:
func createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    // json.NewDecoder is preferred for HTTP: streams from body, no buffering
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid JSON: "+err.Error(), http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    // Validate
    if req.Name == "" {
        http.Error(w, "name is required", http.StatusUnprocessableEntity)
        return
    }

    user, err := createUserInDB(r.Context(), &req)
    if err != nil {
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    // Response:
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

---

**Q6. How do you extract path parameters and query parameters?**

```go
// Query parameters:
func searchUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()
    name := query.Get("name")           // "" if missing
    page := query.Get("page")
    limit := query.Get("limit")
    tags := query["tags"]               // []string for repeated params (?tags=a&tags=b)
}

// Path parameters (Go 1.22+):
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintf(w, "User ID: %s", id)
})

// With Gin:
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    name := c.Query("name")
    c.JSON(200, gin.H{"id": id, "name": name})
})
```

---

**Q7. How do you test HTTP handlers in Go?**

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestGetUser(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/123", nil)
    req.Header.Set("Authorization", "Bearer test-token")

    w := httptest.NewRecorder()

    handler := NewServer()
    handler.ServeHTTP(w, req)

    resp := w.Result()
    if resp.StatusCode != http.StatusOK {
        t.Errorf("expected 200, got %d", resp.StatusCode)
    }

    var user User
    json.NewDecoder(resp.Body).Decode(&user)
    if user.ID != "123" {
        t.Errorf("expected ID 123, got %s", user.ID)
    }
}
```

`httptest.NewRecorder()` is an in-memory `http.ResponseWriter` — no actual network I/O.

---

**Q8. What is gRPC in Go and how does it work?**

gRPC is an RPC framework using Protocol Buffers (protobuf) for serialization and HTTP/2 for transport. It provides strongly-typed contracts and supports 4 communication patterns.

**Define service in proto:**
```protobuf
syntax = "proto3";
package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User) {}
    rpc ListUsers(ListUsersRequest) returns (stream User) {}
}

message GetUserRequest { string id = 1; }
message User { string id = 1; string name = 2; int32 age = 3; }
```

**Implement server:**
```go
type userServer struct {
    pb.UnimplementedUserServiceServer
    db UserDB
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.db.Get(ctx, req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %s not found", req.Id)
    }
    return &pb.User{Id: user.ID, Name: user.Name, Age: int32(user.Age)}, nil
}

// Start server:
lis, _ := net.Listen("tcp", ":50051")
s := grpc.NewServer()
pb.RegisterUserServiceServer(s, &userServer{db: db})
s.Serve(lis)
```

---

**Q9. What are the four gRPC streaming patterns?**

```go
// 1. Unary: single request, single response
rpc GetUser(Request) returns (Response) {}

// 2. Server streaming: single request, stream of responses
rpc ListUsers(Request) returns (stream User) {}
// Client:
stream, _ := client.ListUsers(ctx, req)
for {
    user, err := stream.Recv()
    if err == io.EOF { break }
}

// 3. Client streaming: stream of requests, single response
rpc UploadUsers(stream User) returns (UploadResult) {}
// Client:
stream, _ := client.UploadUsers(ctx)
for _, u := range users {
    stream.Send(u)
}
resp, _ := stream.CloseAndRecv()

// 4. Bidirectional streaming: both sides stream
rpc Chat(stream Message) returns (stream Message) {}
```

---

**Q10. How does graceful shutdown work in a Go HTTP server?**

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: handler}

    // Start server in background
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for OS signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("shutting down server...")

    // Give in-flight requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("server forced to shutdown: %v", err)
    }
    log.Println("server stopped")
}
```

---

**Q11. What is the circuit breaker pattern and why is it used in microservices?**

A circuit breaker prevents cascading failures when a downstream service is slow or unavailable. It has three states:

- **Closed** — requests pass through normally; failures are counted
- **Open** — requests fail immediately (fast-fail) without calling the downstream service; opened after failure threshold
- **Half-Open** — a single probe request is allowed; if it succeeds, circuit closes; if it fails, circuit stays open

```go
// Using github.com/sony/gobreaker:
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "external-api",
    MaxRequests: 1,                  // allowed in half-open state
    Interval:    10 * time.Second,   // rolling window
    Timeout:     30 * time.Second,   // time before trying half-open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

func callExternalAPI(ctx context.Context, id string) (*Response, error) {
    result, err := cb.Execute(func() (interface{}, error) {
        return httpGet(ctx, "https://api.external.com/"+id)
    })
    if err != nil {
        return nil, err // may be gobreaker.ErrOpenState (fast-fail)
    }
    return result.(*Response), nil
}
```

---

**Q12. What is OpenTelemetry and how do you add distributed tracing to a Go service?**

OpenTelemetry (OTel) is a vendor-neutral observability framework for traces, metrics, and logs.

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

// Setup tracer provider (in main):
tp := tracerprovider.New(jaegerExporter)
otel.SetTracerProvider(tp)

// Instrument HTTP server:
handler := otelhttp.NewHandler(mux, "my-service")
http.ListenAndServe(":8080", handler)

// Manual spans in business logic:
func processOrder(ctx context.Context, orderID string) error {
    ctx, span := otel.Tracer("order-service").Start(ctx, "processOrder")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", orderID))

    if err := validateOrder(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }
    return nil
}
```

---

**Q13. What is the health check pattern in Go microservices?**

```go
// Health check types:
// - Liveness: "Is the process alive?" (restart if fails)
// - Readiness: "Is the service ready to accept traffic?" (remove from LB if fails)

type HealthStatus struct {
    Status  string            `json:"status"`
    Checks  map[string]string `json:"checks"`
    Version string            `json:"version"`
}

func livenessHandler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(HealthStatus{Status: "ok"})
}

func readinessHandler(db *sql.DB, cache *redis.Client) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        checks := map[string]string{}
        status := "ok"

        if err := db.PingContext(r.Context()); err != nil {
            checks["database"] = "unhealthy: " + err.Error()
            status = "degraded"
        } else {
            checks["database"] = "healthy"
        }

        if err := cache.Ping(r.Context()).Err(); err != nil {
            checks["cache"] = "unhealthy: " + err.Error()
            status = "degraded"
        } else {
            checks["cache"] = "healthy"
        }

        w.Header().Set("Content-Type", "application/json")
        if status != "ok" {
            w.WriteHeader(http.StatusServiceUnavailable)
        }
        json.NewEncoder(w).Encode(HealthStatus{Status: status, Checks: checks})
    }
}
```

---

**Q14. What is rate limiting middleware in Go?**

```go
import "golang.org/x/time/rate"

// Per-IP rate limiter
type RateLimiter struct {
    limiters sync.Map
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{rate: r, burst: burst}
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    v, ok := rl.limiters.Load(ip)
    if !ok {
        limiter := rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters.Store(ip, limiter)
        return limiter
    }
    return v.(*rate.Limiter)
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        limiter := rl.getLimiter(ip)
        if !limiter.Allow() {
            http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

**Q15. What is the difference between go-chi/chi, gorilla/mux, and the standard library?**

| Feature | stdlib ServeMux | gorilla/mux | chi |
|---------|----------------|-------------|-----|
| Path variables | Go 1.22+ | Yes | Yes |
| Regex routes | No | Yes | No |
| Middleware | Manual wrap | Manual wrap | Built-in stack |
| net/http compat | Native | Yes | Yes |
| Route groups | No | Sub-routers | Yes (`r.Group`) |
| Active maintenance | Yes | Maintenance mode | Active |
| HTTP method routing | Go 1.22+ | Yes | Yes |

```go
// chi example:
r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)

r.Route("/api/v1", func(r chi.Router) {
    r.Use(AuthMiddleware)
    r.Get("/users/{id}", getUser)
    r.Post("/users", createUser)
    r.Put("/users/{id}", updateUser)
    r.Delete("/users/{id}", deleteUser)
})
```

---

**Q16. What is WebSocket support in Go?**

```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return true // or validate origin
    },
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }
    defer conn.Close()

    for {
        messageType, msg, err := conn.ReadMessage()
        if err != nil {
            break // client disconnected
        }
        // Echo back
        if err := conn.WriteMessage(messageType, msg); err != nil {
            break
        }
    }
}
```

---

**Q17. What is the GOPROXY and how does configuration management work in Go services?**

For configuration, the most common patterns are:

```go
// Using environment variables (12-factor app):
type Config struct {
    Port        int           `env:"PORT" envDefault:"8080"`
    DatabaseURL string        `env:"DATABASE_URL" envRequired:"true"`
    LogLevel    string        `env:"LOG_LEVEL" envDefault:"info"`
    Timeout     time.Duration `env:"TIMEOUT" envDefault:"30s"`
}

// Using viper (supports env, flags, config files):
import "github.com/spf13/viper"

viper.SetConfigName("config")
viper.SetConfigType("yaml")
viper.AddConfigPath(".")
viper.AutomaticEnv()  // override with env vars
viper.ReadInConfig()

type Config struct {
    Port     int    `mapstructure:"port"`
    Database DBConfig `mapstructure:"database"`
}
var cfg Config
viper.Unmarshal(&cfg)
```

---

**Q18. What is service discovery in Go microservices?**

Service discovery allows services to find each other without hardcoded addresses.

```go
// Using Consul:
import consulapi "github.com/hashicorp/consul/api"

func registerService(client *consulapi.Client, name, addr string, port int) error {
    return client.Agent().ServiceRegister(&consulapi.AgentServiceRegistration{
        ID:      fmt.Sprintf("%s-%s-%d", name, addr, port),
        Name:    name,
        Address: addr,
        Port:    port,
        Check: &consulapi.AgentServiceCheck{
            HTTP:     fmt.Sprintf("http://%s:%d/health", addr, port),
            Interval: "10s",
            Timeout:  "2s",
        },
    })
}

func discoverService(client *consulapi.Client, name string) (string, error) {
    services, _, err := client.Health().Service(name, "", true, nil)
    if err != nil || len(services) == 0 {
        return "", fmt.Errorf("service %s not found", name)
    }
    svc := services[0].Service
    return fmt.Sprintf("%s:%d", svc.Address, svc.Port), nil
}
```

---

**Q19. How does JWT authentication middleware work in Go?**

```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func JWTMiddleware(secret []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if !strings.HasPrefix(authHeader, "Bearer ") {
                http.Error(w, "missing token", http.StatusUnauthorized)
                return
            }
            tokenStr := strings.TrimPrefix(authHeader, "Bearer ")

            claims := &Claims{}
            token, err := jwt.ParseWithClaims(tokenStr, claims,
                func(t *jwt.Token) (interface{}, error) {
                    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                        return nil, fmt.Errorf("unexpected signing method")
                    }
                    return secret, nil
                })

            if err != nil || !token.Valid {
                http.Error(w, "invalid token", http.StatusUnauthorized)
                return
            }

            // Store claims in context
            ctx := context.WithValue(r.Context(), claimsKey{}, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

---

**Q20. What is the CORS problem and how do you handle it in Go?**

CORS (Cross-Origin Resource Sharing) headers tell browsers whether a cross-origin request is allowed.

```go
func CORSMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            allowed := false
            for _, o := range allowedOrigins {
                if o == "*" || o == origin {
                    allowed = true
                    break
                }
            }

            if allowed {
                // CRITICAL: headers must be set BEFORE writing status code
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
                w.Header().Set("Access-Control-Max-Age", "86400")
            }

            // Handle preflight
            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusNoContent)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

---

**Q21. What is request context and why should you use r.Context() instead of context.Background()?**

```go
// r.Context() carries:
// - Cancellation when client disconnects
// - Request-scoped values (auth info, trace IDs)
// - Deadline from server's ReadTimeout

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // inherits cancellation from client

    // If client disconnects, ctx.Done() fires
    result, err := db.QueryContext(ctx, "SELECT ...")
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return // client disconnected, don't bother responding
        }
        http.Error(w, "db error", 500)
        return
    }
    _ = result
}

// BAD: using context.Background() loses cancellation
func badHandler(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background() // will continue even if client disconnected!
    result, err := db.QueryContext(ctx, "SELECT ...") // waste of resources
    _ = result; _ = err
}
```

---

**Q22. What is http.Flusher and when do you use it?**

`http.Flusher` is an interface that allows flushing buffered data to the client — needed for Server-Sent Events (SSE) or long-polling responses.

```go
func streamHandler(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-r.Context().Done():
            return
        case t := <-ticker.C:
            fmt.Fprintf(w, "data: %s\n\n", t.Format(time.RFC3339))
            flusher.Flush() // send buffered data to client immediately
        }
    }
}
```

---

**Q23. What is structured logging in Go and which libraries are used?**

```go
// Standard library (Go 1.21): slog
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.Info("request handled",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.Int("status", 200),
    slog.Duration("latency", time.Since(start)),
)

// Using zap (high performance):
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

logger.Info("request handled",
    zap.String("method", r.Method),
    zap.Int("status", 200),
    zap.Duration("latency", time.Since(start)),
)
```

---

**Q24. What is dependency injection in Go and how is it done idiomatically?**

Go uses constructor-based dependency injection — no reflection or DI framework required:

```go
type UserService struct {
    db     UserRepository
    cache  CacheRepository
    logger *slog.Logger
}

func NewUserService(
    db UserRepository,
    cache CacheRepository,
    logger *slog.Logger,
) *UserService {
    return &UserService{db: db, cache: cache, logger: logger}
}

func main() {
    db := postgres.NewRepository(dbConn)
    cache := redis.NewRepository(redisClient)
    logger := slog.Default()

    userSvc := NewUserService(db, cache, logger)
    // Wire up handlers...
}
```

For larger projects, `google/wire` generates the wiring code:

```go
// wire.go
var Set = wire.NewSet(
    NewUserService,
    NewUserHandler,
    postgres.NewRepository,
    redis.NewRepository,
)
```

---

**Q25. What is gRPC vs REST for internal microservice communication?**

| Aspect | REST/JSON | gRPC/Protobuf |
|--------|-----------|---------------|
| Protocol | HTTP/1.1 (usually) | HTTP/2 |
| Serialization | JSON (text) | Protobuf (binary) |
| Schema | Optional (OpenAPI) | Required (.proto) |
| Type safety | Runtime | Compile-time |
| Streaming | Limited (SSE) | Built-in |
| Browser support | Native | Needs grpc-web |
| Throughput | Baseline | 5–10× faster |
| Payload size | Larger (text) | 3–10× smaller |
| Debugging | Easy (curl) | Harder (need grpcurl) |

Use gRPC for high-throughput internal service communication. Use REST for public APIs where browser and curl compatibility matters.
