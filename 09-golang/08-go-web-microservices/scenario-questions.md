# Go Web & Microservices — Scenario Questions

---

## Scenario 1: Build a REST API with CRUD

**"Build a REST API with CRUD operations for a User resource using standard net/http."**

```go
package main

import (
    "encoding/json"
    "net/http"
    "sync"
    "strconv"
    "log"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

type UserStore struct {
    mu      sync.RWMutex
    users   map[int]User
    counter int
}

func NewUserStore() *UserStore {
    return &UserStore{users: make(map[int]User)}
}

func (s *UserStore) Create(u User) User {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.counter++
    u.ID = s.counter
    s.users[u.ID] = u
    return u
}

func (s *UserStore) Get(id int) (User, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    u, ok := s.users[id]
    return u, ok
}

func (s *UserStore) Update(id int, u User) (User, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.users[id]; !ok {
        return User{}, false
    }
    u.ID = id
    s.users[id] = u
    return u, true
}

func (s *UserStore) Delete(id int) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.users[id]; !ok {
        return false
    }
    delete(s.users, id)
    return true
}

func main() {
    store := NewUserStore()
    mux := http.NewServeMux()

    // POST /users
    mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
        var req User
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "invalid JSON", http.StatusBadRequest)
            return
        }
        if req.Name == "" || req.Email == "" {
            http.Error(w, "name and email required", http.StatusUnprocessableEntity)
            return
        }
        created := store.Create(req)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(created)
    })

    // GET /users/{id}
    mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            http.Error(w, "invalid id", http.StatusBadRequest)
            return
        }
        user, ok := store.Get(id)
        if !ok {
            http.Error(w, "user not found", http.StatusNotFound)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    })

    // PUT /users/{id}
    mux.HandleFunc("PUT /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id, _ := strconv.Atoi(r.PathValue("id"))
        var req User
        json.NewDecoder(r.Body).Decode(&req)
        updated, ok := store.Update(id, req)
        if !ok {
            http.Error(w, "user not found", http.StatusNotFound)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(updated)
    })

    // DELETE /users/{id}
    mux.HandleFunc("DELETE /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id, _ := strconv.Atoi(r.PathValue("id"))
        if !store.Delete(id) {
            http.Error(w, "user not found", http.StatusNotFound)
            return
        }
        w.WriteHeader(http.StatusNoContent)
    })

    log.Println("Listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## Scenario 2: JWT Authentication Middleware

**"Implement JWT authentication middleware in Go."**

```go
package middleware

import (
    "context"
    "fmt"
    "net/http"
    "strings"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type contextKey string
const ClaimsContextKey contextKey = "claims"

type Claims struct {
    UserID string   `json:"user_id"`
    Roles  []string `json:"roles"`
    jwt.RegisteredClaims
}

type JWTConfig struct {
    Secret     []byte
    Issuer     string
    Expiration time.Duration
}

func NewJWTMiddleware(cfg JWTConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                w.Header().Set("WWW-Authenticate", `Bearer realm="api"`)
                http.Error(w, `{"error":"missing authorization header"}`, http.StatusUnauthorized)
                return
            }

            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") {
                http.Error(w, `{"error":"invalid authorization format"}`, http.StatusUnauthorized)
                return
            }

            tokenStr := parts[1]
            claims := &Claims{}

            token, err := jwt.ParseWithClaims(tokenStr, claims,
                func(t *jwt.Token) (interface{}, error) {
                    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                        return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
                    }
                    return cfg.Secret, nil
                },
                jwt.WithIssuer(cfg.Issuer),
                jwt.WithExpirationRequired(),
            )

            if err != nil {
                http.Error(w, fmt.Sprintf(`{"error":"invalid token: %s"}`, err.Error()),
                    http.StatusUnauthorized)
                return
            }

            if !token.Valid {
                http.Error(w, `{"error":"token is not valid"}`, http.StatusUnauthorized)
                return
            }

            // Attach claims to context
            ctx := context.WithValue(r.Context(), ClaimsContextKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Helper to extract claims from context
func GetClaims(ctx context.Context) (*Claims, bool) {
    claims, ok := ctx.Value(ClaimsContextKey).(*Claims)
    return claims, ok
}

// Role-based authorization middleware
func RequireRole(role string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := GetClaims(r.Context())
            if !ok {
                http.Error(w, `{"error":"no claims in context"}`, http.StatusForbidden)
                return
            }
            for _, r := range claims.Roles {
                if r == role {
                    next.ServeHTTP(w, r)
                    return
                }
            }
            http.Error(w, `{"error":"insufficient permissions"}`, http.StatusForbidden)
        })
    }
}

// Token generation helper
func GenerateToken(cfg JWTConfig, userID string, roles []string) (string, error) {
    claims := Claims{
        UserID: userID,
        Roles:  roles,
        RegisteredClaims: jwt.RegisteredClaims{
            Issuer:    cfg.Issuer,
            Subject:   userID,
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(cfg.Expiration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(cfg.Secret)
}
```

---

## Scenario 3: Graceful Shutdown

**"Your microservice needs graceful shutdown. How do you implement it?"**

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type Server struct {
    http     *http.Server
    logger   *slog.Logger
    shutdown []func(ctx context.Context) error // cleanup hooks
}

func NewServer(addr string, handler http.Handler, logger *slog.Logger) *Server {
    return &Server{
        http: &http.Server{
            Addr:         addr,
            Handler:      handler,
            ReadTimeout:  5 * time.Second,
            WriteTimeout: 30 * time.Second,
            IdleTimeout:  120 * time.Second,
        },
        logger: logger,
    }
}

// RegisterShutdownHook adds a cleanup function called during shutdown.
// Examples: close DB pool, flush metrics, deregister from service discovery.
func (s *Server) RegisterShutdownHook(fn func(ctx context.Context) error) {
    s.shutdown = append(s.shutdown, fn)
}

func (s *Server) Run() error {
    // Channel to receive OS signals
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)

    // Start HTTP server in background
    errCh := make(chan error, 1)
    go func() {
        s.logger.Info("server starting", slog.String("addr", s.http.Addr))
        if err := s.http.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    // Wait for signal or fatal error
    select {
    case sig := <-sigCh:
        s.logger.Info("shutdown signal received", slog.String("signal", sig.String()))
    case err := <-errCh:
        return err
    }

    // Graceful shutdown: give in-flight requests 30s to complete
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    s.logger.Info("shutting down HTTP server...")
    if err := s.http.Shutdown(shutdownCtx); err != nil {
        s.logger.Error("http server shutdown error", slog.Any("error", err))
    }

    // Run registered cleanup hooks
    for _, fn := range s.shutdown {
        if err := fn(shutdownCtx); err != nil {
            s.logger.Error("shutdown hook error", slog.Any("error", err))
        }
    }

    s.logger.Info("server stopped cleanly")
    return nil
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"status":"ok"}`))
    })

    db := connectDB()
    srv := NewServer(":8080", mux, logger)

    // Register cleanup
    srv.RegisterShutdownHook(func(ctx context.Context) error {
        logger.Info("closing database pool...")
        return db.Close()
    })

    if err := srv.Run(); err != nil {
        logger.Error("server error", slog.Any("error", err))
        os.Exit(1)
    }
}
```

---

## Scenario 4: Circuit Breaker for External Dependency

**"Implement a circuit breaker for an external HTTP dependency."**

```go
package circuitbreaker

import (
    "context"
    "errors"
    "fmt"
    "net/http"
    "sync"
    "time"
)

type State int

const (
    StateClosed   State = iota // normal operation
    StateOpen                  // failing fast
    StateHalfOpen              // testing recovery
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

type CircuitBreaker struct {
    mu               sync.Mutex
    state            State
    failureCount     int
    successCount     int
    lastFailureTime  time.Time

    // Configuration
    maxFailures      int
    resetTimeout     time.Duration
    halfOpenMaxCalls int
}

func New(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:            StateClosed,
        maxFailures:      maxFailures,
        resetTimeout:     resetTimeout,
        halfOpenMaxCalls: 1,
    }
}

func (cb *CircuitBreaker) Allow() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        return true

    case StateOpen:
        if time.Since(cb.lastFailureTime) > cb.resetTimeout {
            cb.state = StateHalfOpen
            cb.successCount = 0
            return true // allow probe
        }
        return false

    case StateHalfOpen:
        return cb.successCount < cb.halfOpenMaxCalls
    }
    return false
}

func (cb *CircuitBreaker) RecordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.failureCount = 0
    if cb.state == StateHalfOpen {
        cb.successCount++
        if cb.successCount >= cb.halfOpenMaxCalls {
            cb.state = StateClosed
        }
    }
}

func (cb *CircuitBreaker) RecordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.failureCount++
    cb.lastFailureTime = time.Now()
    if cb.failureCount >= cb.maxFailures || cb.state == StateHalfOpen {
        cb.state = StateOpen
    }
}

// ExternalAPIClient wraps an HTTP client with a circuit breaker
type ExternalAPIClient struct {
    baseURL string
    client  *http.Client
    cb      *CircuitBreaker
}

func NewExternalAPIClient(baseURL string) *ExternalAPIClient {
    return &ExternalAPIClient{
        baseURL: baseURL,
        client:  &http.Client{Timeout: 5 * time.Second},
        cb:      New(5, 30*time.Second),
    }
}

func (c *ExternalAPIClient) Get(ctx context.Context, path string) (*http.Response, error) {
    if !c.cb.Allow() {
        return nil, fmt.Errorf("external API %s: %w", path, ErrCircuitOpen)
    }

    req, _ := http.NewRequestWithContext(ctx, "GET", c.baseURL+path, nil)
    resp, err := c.client.Do(req)
    if err != nil {
        c.cb.RecordFailure()
        return nil, fmt.Errorf("external API request: %w", err)
    }

    if resp.StatusCode >= 500 {
        c.cb.RecordFailure()
        return resp, fmt.Errorf("external API returned %d", resp.StatusCode)
    }

    c.cb.RecordSuccess()
    return resp, nil
}
```

---

## Scenario 5: Middleware Chain with Logging and Auth

**"Build a middleware chain with logging, recovery, authentication, and request ID injection."**

```go
package main

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "log/slog"
    "net/http"
    "runtime/debug"
    "time"
)

type contextKey int
const requestIDKey contextKey = 1

func generateRequestID() string {
    b := make([]byte, 8)
    rand.Read(b)
    return hex.EncodeToString(b)
}

// RequestID injects a unique ID into every request context
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = generateRequestID()
        }
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// statusRecorder wraps ResponseWriter to capture status code
type statusRecorder struct {
    http.ResponseWriter
    status int
    bytes  int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status
    r.ResponseWriter.WriteHeader(status)
}

func (r *statusRecorder) Write(b []byte) (int, error) {
    n, err := r.ResponseWriter.Write(b)
    r.bytes += n
    return n, err
}

// Logger logs request details after each response
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            rec := &statusRecorder{ResponseWriter: w, status: 200}

            next.ServeHTTP(rec, r)

            requestID, _ := r.Context().Value(requestIDKey).(string)
            logger.Info("request",
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
                slog.Int("status", rec.status),
                slog.Int("bytes", rec.bytes),
                slog.Duration("latency", time.Since(start)),
                slog.String("request_id", requestID),
                slog.String("remote_addr", r.RemoteAddr),
            )
        })
    }
}

// Recovery catches panics and returns 500
func RecoveryMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    stack := debug.Stack()
                    logger.Error("panic recovered",
                        slog.Any("panic", err),
                        slog.String("stack", string(stack)),
                    )
                    http.Error(w, `{"error":"internal server error"}`,
                        http.StatusInternalServerError)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}

func Chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    // Apply in reverse so the first middleware is outermost
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func main() {
    logger := slog.Default()
    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/users", listUsersHandler)

    jwtCfg := JWTConfig{Secret: []byte("supersecret"), Issuer: "my-service"}

    handler := Chain(mux,
        RequestIDMiddleware,
        LoggingMiddleware(logger),
        RecoveryMiddleware(logger),
        NewJWTMiddleware(jwtCfg),
    )

    http.ListenAndServe(":8080", handler)
}
```

---

## Scenario 6: Structured Request Validation

**"Implement request validation in a Go HTTP handler without using a framework."**

```go
package validation

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "unicode/utf8"
)

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ValidationErrors []ValidationError

func (v ValidationErrors) Error() string {
    msgs := make([]string, len(v))
    for i, e := range v {
        msgs[i] = e.Field + ": " + e.Message
    }
    return strings.Join(msgs, "; ")
}

type CreateUserRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
    Password string `json:"password"`
}

func (r *CreateUserRequest) Validate() ValidationErrors {
    var errs ValidationErrors

    if strings.TrimSpace(r.Name) == "" {
        errs = append(errs, ValidationError{"name", "is required"})
    } else if utf8.RuneCountInString(r.Name) > 100 {
        errs = append(errs, ValidationError{"name", "must be at most 100 characters"})
    }

    if r.Email == "" {
        errs = append(errs, ValidationError{"email", "is required"})
    } else if !strings.Contains(r.Email, "@") {
        errs = append(errs, ValidationError{"email", "is not a valid email address"})
    }

    if r.Age < 0 || r.Age > 150 {
        errs = append(errs, ValidationError{"age", "must be between 0 and 150"})
    }

    if len(r.Password) < 8 {
        errs = append(errs, ValidationError{"password", "must be at least 8 characters"})
    }

    return errs
}

// DecodeAndValidate decodes JSON body and validates
func DecodeAndValidate[T interface{ Validate() ValidationErrors }](
    r *http.Request,
) (T, error) {
    var req T
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields() // stricter parsing
    if err := dec.Decode(&req); err != nil {
        return req, fmt.Errorf("invalid JSON: %w", err)
    }
    if errs := req.Validate(); len(errs) > 0 {
        return req, errs
    }
    return req, nil
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    req, err := DecodeAndValidate[CreateUserRequest](r)
    if err != nil {
        w.Header().Set("Content-Type", "application/json")
        var ve ValidationErrors
        if errors.As(err, &ve) {
            w.WriteHeader(http.StatusUnprocessableEntity)
            json.NewEncoder(w).Encode(map[string]interface{}{
                "error":  "validation failed",
                "fields": ve,
            })
        } else {
            w.WriteHeader(http.StatusBadRequest)
            json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        }
        return
    }
    // use req...
}
```

---

## Scenario 7: File Upload Handler

**"Implement a file upload endpoint in Go that handles large files without buffering in memory."**

```go
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    // Limit total request size to prevent abuse
    r.Body = http.MaxBytesReader(w, r.Body, 100<<20) // 100MB max

    // ParseMultipartForm with disk spill threshold (32MB in memory, rest to disk)
    if err := r.ParseMultipartForm(32 << 20); err != nil {
        http.Error(w, "file too large or invalid form", http.StatusBadRequest)
        return
    }
    defer r.MultipartForm.RemoveAll() // clean up temp files

    file, header, err := r.FormFile("file")
    if err != nil {
        http.Error(w, "no file uploaded", http.StatusBadRequest)
        return
    }
    defer file.Close()

    // Validate file type by reading first 512 bytes (not by extension!)
    buf := make([]byte, 512)
    n, _ := file.Read(buf)
    contentType := http.DetectContentType(buf[:n])
    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
    }
    if !allowedTypes[contentType] {
        http.Error(w, "unsupported file type: "+contentType, http.StatusUnsupportedMediaType)
        return
    }

    // Seek back to beginning after sniffing type
    if seeker, ok := file.(io.Seeker); ok {
        seeker.Seek(0, io.SeekStart)
    }

    // Save to disk (stream — never fully buffered in memory)
    savePath := filepath.Join("/uploads", filepath.Base(header.Filename))
    dst, err := os.Create(savePath)
    if err != nil {
        http.Error(w, "failed to save file", http.StatusInternalServerError)
        return
    }
    defer dst.Close()

    written, err := io.Copy(dst, file) // streams from multipart → disk
    if err != nil {
        http.Error(w, "failed to write file", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "filename": header.Filename,
        "size":     written,
        "type":     contentType,
    })
}
```

---

## Scenario 8: Distributed Tracing with OpenTelemetry

**"Add distributed tracing to a Go HTTP service that calls a downstream gRPC service."**

```go
package main

import (
    "context"
    "log"
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("localhost:4318"),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("1.0.0"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

type OrderService struct {
    inventoryClient pb.InventoryServiceClient
}

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []OrderItem) (*Order, error) {
    // Create a child span for business logic
    ctx, span := otel.Tracer("order-service").Start(ctx, "CreateOrder")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.id", userID),
        attribute.Int("order.item_count", len(items)),
    )

    // This call automatically propagates trace context via gRPC metadata
    resp, err := s.inventoryClient.CheckStock(ctx, &pb.StockRequest{Items: toProtoItems(items)})
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "inventory check failed")
        return nil, fmt.Errorf("inventory check: %w", err)
    }

    if !resp.Available {
        span.SetAttributes(attribute.Bool("order.stock_available", false))
        return nil, ErrOutOfStock
    }

    order := &Order{UserID: userID, Items: items, Status: "pending"}
    span.SetAttributes(attribute.String("order.id", order.ID))
    return order, nil
}

func main() {
    ctx := context.Background()
    tp, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(ctx)

    // gRPC connection with OTel instrumentation (auto-propagates traces)
    conn, _ := grpc.Dial("inventory:50051",
        grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
        grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
    )

    svc := &OrderService{inventoryClient: pb.NewInventoryServiceClient(conn)}
    mux := http.NewServeMux()
    mux.HandleFunc("POST /orders", svc.handleCreateOrder)

    // Instrument entire HTTP server (adds trace to each request)
    handler := otelhttp.NewHandler(mux, "order-service-http")
    http.ListenAndServe(":8080", handler)
}
```

---

## Scenario 9: Reading Request Body Twice

**"A middleware needs to log the request body, but the handler also needs to read it. How do you handle this?"**

```go
// Problem: http.Request.Body is an io.ReadCloser — once read, it's drained.
// Naively reading it in middleware leaves nothing for the handler.

func LogBodyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Read and log the body
        bodyBytes, err := io.ReadAll(r.Body)
        r.Body.Close()
        if err != nil {
            http.Error(w, "failed to read body", 500)
            return
        }

        log.Printf("request body: %s", string(bodyBytes))

        // Restore the body for the handler
        r.Body = io.NopCloser(bytes.NewReader(bodyBytes))

        next.ServeHTTP(w, r)
    })
}

// Alternative: use io.TeeReader to read and replay simultaneously
func TeeBodyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var logBuf bytes.Buffer
        // TeeReader reads from Body and simultaneously writes to logBuf
        tee := io.TeeReader(r.Body, &logBuf)
        r.Body = io.NopCloser(tee)

        next.ServeHTTP(w, r)

        // After handler runs, logBuf contains what was read
        log.Printf("request body: %s", logBuf.String())
    })
}
```

---

## Scenario 10: HTTP Client with Retry and Timeout

**"Build a resilient HTTP client with timeouts, retries, and exponential backoff."**

```go
package httpclient

import (
    "context"
    "fmt"
    "io"
    "math"
    "net/http"
    "time"
)

type RetryConfig struct {
    MaxRetries  int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    RetryStatus []int // status codes to retry on (e.g., 429, 503)
}

type Client struct {
    http   *http.Client
    retry  RetryConfig
}

func New(timeout time.Duration, retry RetryConfig) *Client {
    return &Client{
        http: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 20,
                IdleConnTimeout:     90 * time.Second,
            },
        },
        retry: retry,
    }
}

func (c *Client) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= c.retry.MaxRetries; attempt++ {
        if attempt > 0 {
            delay := c.backoffDelay(attempt)
            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }

        // Clone request for retry (body may have been consumed)
        cloned := req.Clone(ctx)

        resp, err := c.http.Do(cloned)
        if err != nil {
            lastErr = err
            continue // retry on transport error
        }

        if c.shouldRetry(resp.StatusCode) {
            io.Copy(io.Discard, resp.Body) // drain body before closing
            resp.Body.Close()
            lastErr = fmt.Errorf("server returned %d", resp.StatusCode)
            continue
        }

        return resp, nil // success
    }

    return nil, fmt.Errorf("after %d retries: %w", c.retry.MaxRetries, lastErr)
}

func (c *Client) backoffDelay(attempt int) time.Duration {
    delay := time.Duration(float64(c.retry.BaseDelay) * math.Pow(2, float64(attempt-1)))
    if delay > c.retry.MaxDelay {
        delay = c.retry.MaxDelay
    }
    return delay
}

func (c *Client) shouldRetry(status int) bool {
    for _, s := range c.retry.RetryStatus {
        if s == status {
            return true
        }
    }
    return false
}
```
