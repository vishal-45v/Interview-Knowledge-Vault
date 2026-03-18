# Go Web & Microservices — Structured Answers

---

## Answer 1: How Do You Structure a Go REST API Project?

**One-sentence answer:** Use a domain-driven layered structure: handlers call services, services call repositories, with shared types in a core package — wired together in `main.go`.

**Recommended project layout:**

```
myservice/
├── cmd/
│   └── server/
│       └── main.go           # entry point: wiring, config, start
├── internal/
│   ├── handler/              # HTTP layer: parse request, call service, write response
│   │   ├── user.go
│   │   └── user_test.go
│   ├── service/              # business logic layer
│   │   ├── user.go
│   │   └── user_test.go
│   ├── repository/           # data access layer
│   │   ├── user_postgres.go
│   │   └── user_memory.go    # for tests
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── logging.go
│   │   └── recovery.go
│   └── domain/               # shared types, interfaces
│       └── user.go
├── pkg/                      # exportable packages (if any)
├── migrations/
├── go.mod
└── go.sum
```

**Domain layer (shared types & interfaces):**

```go
// internal/domain/user.go
package domain

import "context"

type User struct {
    ID    string
    Name  string
    Email string
}

type UserRepository interface {
    Get(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
}

type UserService interface {
    GetUser(ctx context.Context, id string) (*User, error)
    CreateUser(ctx context.Context, name, email string) (*User, error)
}
```

**Service layer:**

```go
// internal/service/user.go
type userService struct {
    repo   domain.UserRepository
    logger *slog.Logger
}

func NewUserService(repo domain.UserRepository, logger *slog.Logger) domain.UserService {
    return &userService{repo: repo, logger: logger}
}

func (s *userService) CreateUser(ctx context.Context, name, email string) (*domain.User, error) {
    if name == "" || email == "" {
        return nil, &domain.ValidationError{Message: "name and email required"}
    }
    u := &domain.User{ID: uuid.New().String(), Name: name, Email: email}
    return u, s.repo.Create(ctx, u)
}
```

**Handler layer:**

```go
// internal/handler/user.go
type UserHandler struct {
    svc domain.UserService
}

func (h *UserHandler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("POST /users", h.Create)
    mux.HandleFunc("GET /users/{id}", h.Get)
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request")
        return
    }
    user, err := h.svc.CreateUser(r.Context(), req.Name, req.Email)
    if err != nil {
        // map domain errors to HTTP status codes
        var ve *domain.ValidationError
        if errors.As(err, &ve) {
            writeError(w, http.StatusUnprocessableEntity, ve.Message)
            return
        }
        writeError(w, http.StatusInternalServerError, "internal error")
        return
    }
    writeJSON(w, http.StatusCreated, user)
}
```

---

## Answer 2: How Do You Implement Middleware in Go's net/http?

**One-sentence answer:** Middleware is a function that takes an `http.Handler` and returns an `http.Handler`, wrapping the original with additional behavior before/after the call.

**The middleware signature:**

```go
type Middleware func(http.Handler) http.Handler
```

**Pattern: response writer wrapper (to capture status code):**

```go
type loggingResponseWriter struct {
    http.ResponseWriter
    statusCode int
}

func newLoggingResponseWriter(w http.ResponseWriter) *loggingResponseWriter {
    return &loggingResponseWriter{w, http.StatusOK}
}

func (l *loggingResponseWriter) WriteHeader(code int) {
    l.statusCode = code
    l.ResponseWriter.WriteHeader(code)
}

func LoggingMiddleware(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            wrapped := newLoggingResponseWriter(w)

            next.ServeHTTP(wrapped, r)

            logger.Info("http request",
                slog.String("method", r.Method),
                slog.String("path", r.URL.Path),
                slog.Int("status", wrapped.statusCode),
                slog.Duration("duration", time.Since(start)),
            )
        })
    }
}
```

**Middleware chain helper:**

```go
func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Usage:
handler := Chain(
    mux,
    RequestIDMiddleware,
    LoggingMiddleware(logger),
    RecoveryMiddleware(logger),
    JWTMiddleware(jwtSecret),
)
```

**Middleware that modifies the request (adds value to context):**

```go
type contextKey string
const UserIDKey contextKey = "user_id"

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := parseTokenAndGetUserID(r)
        if userID == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), UserIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## Answer 3: How Does gRPC Compare to REST for Go Microservices?

**One-sentence answer:** gRPC is 5–10× faster with strongly typed contracts over HTTP/2, ideal for internal services; REST is universally compatible and easier to debug, ideal for public APIs.

**Code comparison:**

```go
// REST: define and parse manually
// POST /users {"name": "Alice"} → {"id": "123", "name": "Alice"}

type CreateUserRequest struct {
    Name string `json:"name"`
}
type CreateUserResponse struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req) // may fail at runtime
    // ...
}

// gRPC: compile-time type safety via .proto
// UserService.CreateUser(CreateUserRequest) → User

func (s *Server) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    // req.Name is guaranteed to be a string — checked by protobuf
    return &pb.User{Id: "123", Name: req.Name}, nil
}
```

**When to use each:**

| Use case | Recommendation |
|----------|----------------|
| Public API (browsers, mobile) | REST |
| Internal microservice calls | gRPC |
| Streaming (real-time data) | gRPC (bidirectional streaming) |
| Simple CRUD | REST |
| High-throughput data pipeline | gRPC |
| Contract-first development | gRPC |
| Third-party integrations | REST (more universal) |

**Performance benchmark (approximate):**
- Protobuf serialization: ~50 ns vs JSON: ~500 ns (10× faster)
- HTTP/2 multiplexing: multiple gRPC calls on one TCP connection
- Payload size: Protobuf is 3–10× smaller than equivalent JSON

---

## Answer 4: How Do You Implement Graceful Shutdown in a Go HTTP Server?

**One-sentence answer:** Run the server in a goroutine, wait for an OS signal, call `srv.Shutdown(ctx)` with a timeout, and run cleanup hooks.

**Complete implementation:**

```go
func run(ctx context.Context) error {
    cfg := loadConfig()
    db := mustConnectDB(cfg.DatabaseURL)
    logger := slog.Default()

    mux := http.NewServeMux()
    registerRoutes(mux, db, logger)

    srv := &http.Server{
        Addr:         cfg.Addr,
        Handler:      addMiddleware(mux, logger),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  2 * time.Minute,
    }

    // Start server
    go func() {
        logger.Info("server listening", slog.String("addr", cfg.Addr))
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            logger.Error("server error", slog.Any("error", err))
        }
    }()

    // Wait for termination signal
    <-ctx.Done()
    logger.Info("shutdown initiated")

    // Allow 30 seconds for in-flight requests to complete
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        logger.Error("graceful shutdown failed", slog.Any("error", err))
        srv.Close() // force close
    }

    // Cleanup resources
    logger.Info("closing database pool")
    if err := db.Close(); err != nil {
        logger.Error("db close error", slog.Any("error", err))
    }

    logger.Info("shutdown complete")
    return nil
}

func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    if err := run(ctx); err != nil {
        log.Fatal(err)
    }
}
```

Key: `signal.NotifyContext` (Go 1.16+) simplifies signal handling by returning a context that is cancelled on signal receipt.

---

## Answer 5: How Do You Do Request Validation in Go?

**One-sentence answer:** Implement a `Validate()` method on your request struct, decode JSON with `json.NewDecoder`, then call `Validate()` and map validation errors to HTTP 422 responses.

```go
type CreateProductRequest struct {
    Name     string   `json:"name"`
    Price    float64  `json:"price"`
    SKU      string   `json:"sku"`
    Tags     []string `json:"tags"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func (r CreateProductRequest) Validate() []FieldError {
    var errs []FieldError

    if strings.TrimSpace(r.Name) == "" {
        errs = append(errs, FieldError{"name", "required"})
    } else if len(r.Name) > 200 {
        errs = append(errs, FieldError{"name", "max 200 characters"})
    }

    if r.Price <= 0 {
        errs = append(errs, FieldError{"price", "must be greater than 0"})
    }

    if r.SKU == "" {
        errs = append(errs, FieldError{"sku", "required"})
    } else if !regexp.MustCompile(`^[A-Z]{2}\d{6}$`).MatchString(r.SKU) {
        errs = append(errs, FieldError{"sku", "must match format XX000000"})
    }

    return errs
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    var req CreateProductRequest
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(&req); err != nil {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON: " + err.Error()})
        return
    }

    if errs := req.Validate(); len(errs) > 0 {
        writeJSON(w, http.StatusUnprocessableEntity, map[string]interface{}{
            "error":  "validation failed",
            "fields": errs,
        })
        return
    }
    // proceed...
}
```

For complex projects, consider `github.com/go-playground/validator` for struct tag-based validation.

---

## Answer 6: How Do You Implement JWT Authentication Middleware?

**One-sentence answer:** Parse and validate the `Authorization: Bearer <token>` header, verify the signature and claims, then attach the decoded claims to the request context.

```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID string   `json:"user_id"`
    Roles  []string `json:"roles"`
    jwt.RegisteredClaims
}

func JWTMiddleware(secret []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractBearerToken(r)
            if token == "" {
                writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "missing token"})
                return
            }

            claims := &Claims{}
            parsed, err := jwt.ParseWithClaims(token, claims,
                func(t *jwt.Token) (interface{}, error) {
                    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                        return nil, fmt.Errorf("unexpected alg: %v", t.Header["alg"])
                    }
                    return secret, nil
                },
                jwt.WithExpirationRequired(),
            )

            if err != nil || !parsed.Valid {
                writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid token"})
                return
            }

            ctx := context.WithValue(r.Context(), claimsKey{}, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func extractBearerToken(r *http.Request) string {
    h := r.Header.Get("Authorization")
    if len(h) > 7 && strings.EqualFold(h[:7], "bearer ") {
        return h[7:]
    }
    return ""
}

// Usage in handlers:
func protectedHandler(w http.ResponseWriter, r *http.Request) {
    claims := r.Context().Value(claimsKey{}).(*Claims)
    fmt.Fprintf(w, "Hello, user %s", claims.UserID)
}
```

---

## Answer 7: How Do You Add Distributed Tracing with OpenTelemetry?

**One-sentence answer:** Initialize a TracerProvider with an exporter, instrument HTTP/gRPC transports, and create spans in business logic using `otel.Tracer().Start(ctx, "span-name")`.

```go
// Step 1: Initialize in main
func initOTel(ctx context.Context, serviceName string) (func(), error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
            attribute.String("deployment.environment", os.Getenv("ENV")),
        )),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return func() { tp.Shutdown(ctx) }, nil
}

// Step 2: Instrument HTTP server (auto-creates span per request)
handler := otelhttp.NewHandler(mux, "api-server")

// Step 3: Instrument outgoing HTTP calls (auto-propagates trace)
httpClient := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}

// Step 4: Add manual spans in business logic
func (s *OrderService) placeOrder(ctx context.Context, order *Order) error {
    ctx, span := otel.Tracer("order-service").Start(ctx, "place-order",
        trace.WithAttributes(
            attribute.String("order.id", order.ID),
            attribute.Float64("order.total", order.Total),
        ),
    )
    defer span.End()

    if err := s.inventory.Reserve(ctx, order.Items); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "inventory reservation failed")
        return err
    }

    span.AddEvent("inventory reserved")
    return s.db.Save(ctx, order)
}
```

---

## Answer 8: How Do You Implement a Health Check Endpoint?

**One-sentence answer:** Expose `/healthz` (liveness — process is alive) and `/readyz` (readiness — all dependencies are healthy) endpoints, with readiness checking DB, cache, and external dependencies.

```go
type HealthChecker struct {
    db     *sql.DB
    redis  *redis.Client
    checks []func(ctx context.Context) (string, error)
}

func (h *HealthChecker) Liveness(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "alive"})
}

func (h *HealthChecker) Readiness(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    type checkResult struct {
        Status  string `json:"status"`
        Message string `json:"message,omitempty"`
    }

    results := make(map[string]checkResult)
    httpStatus := http.StatusOK

    // Database check
    if err := h.db.PingContext(ctx); err != nil {
        results["database"] = checkResult{"unhealthy", err.Error()}
        httpStatus = http.StatusServiceUnavailable
    } else {
        results["database"] = checkResult{Status: "healthy"}
    }

    // Redis check
    if err := h.redis.Ping(ctx).Err(); err != nil {
        results["redis"] = checkResult{"unhealthy", err.Error()}
        httpStatus = http.StatusServiceUnavailable
    } else {
        results["redis"] = checkResult{Status: "healthy"}
    }

    overallStatus := "healthy"
    if httpStatus != http.StatusOK {
        overallStatus = "unhealthy"
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(httpStatus)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "status":  overallStatus,
        "checks":  results,
        "version": version.BuildVersion,
    })
}

// Register:
mux.HandleFunc("GET /healthz", health.Liveness)
mux.HandleFunc("GET /readyz", health.Readiness)
```

---

## Answer 9: How Do You Handle File Uploads in Go?

**One-sentence answer:** Use `r.ParseMultipartForm` to parse the form with a memory threshold (rest spills to temp files), then stream the file content to its destination with `io.Copy`.

```go
func uploadFile(w http.ResponseWriter, r *http.Request) {
    // 1. Enforce total request size limit
    r.Body = http.MaxBytesReader(w, r.Body, 10<<20) // 10MB total

    // 2. Parse with 2MB in-memory threshold; rest goes to disk temp files
    if err := r.ParseMultipartForm(2 << 20); err != nil {
        http.Error(w, "request too large or malformed", http.StatusBadRequest)
        return
    }
    defer r.MultipartForm.RemoveAll() // cleanup temp files

    // 3. Get the file
    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "no file in 'upload' field", http.StatusBadRequest)
        return
    }
    defer file.Close()

    // 4. Detect content type (first 512 bytes)
    buf := make([]byte, 512)
    n, _ := file.Read(buf)
    mimeType := http.DetectContentType(buf[:n])
    if _, ok := allowedMIME[mimeType]; !ok {
        http.Error(w, "file type not allowed: "+mimeType, http.StatusUnsupportedMediaType)
        return
    }
    // Seek back after sniffing
    file.(io.Seeker).Seek(0, io.SeekStart)

    // 5. Stream to destination (S3, disk, etc.)
    objectKey := uuid.New().String() + filepath.Ext(header.Filename)
    if err := s3client.Upload(r.Context(), objectKey, file, header.Size); err != nil {
        http.Error(w, "upload failed", http.StatusInternalServerError)
        return
    }

    writeJSON(w, http.StatusCreated, map[string]string{
        "key":  objectKey,
        "size": strconv.FormatInt(header.Size, 10),
        "type": mimeType,
    })
}

var allowedMIME = map[string]bool{
    "image/jpeg": true,
    "image/png":  true,
    "image/gif":  true,
    "application/pdf": true,
}
```

---

## Answer 10: What Is the Standard Way to Do Dependency Injection in Go Services?

**One-sentence answer:** Use constructor functions that accept interfaces, wire dependencies in `main.go`, and optionally use `google/wire` to generate the wiring code for larger services.

**Manual DI (recommended for most projects):**

```go
// Interfaces defined in domain layer
type UserRepo interface {
    Get(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, u *User) error
}

type EmailSender interface {
    SendWelcome(ctx context.Context, to, name string) error
}

// Service accepts interfaces — easily testable
type UserService struct {
    repo   UserRepo
    email  EmailSender
    logger *slog.Logger
}

func NewUserService(repo UserRepo, email EmailSender, logger *slog.Logger) *UserService {
    return &UserService{repo: repo, email: email, logger: logger}
}

// main.go wires concrete implementations
func main() {
    db := mustOpenDB(os.Getenv("DATABASE_URL"))
    emailClient := smtp.NewClient(os.Getenv("SMTP_HOST"))
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    userRepo := postgres.NewUserRepo(db)
    userSvc := NewUserService(userRepo, emailClient, logger)
    userHandler := handler.NewUserHandler(userSvc, logger)

    mux := http.NewServeMux()
    userHandler.RegisterRoutes(mux)
    http.ListenAndServe(":8080", mux)
}

// Test uses fake implementations
func TestCreateUser(t *testing.T) {
    svc := NewUserService(
        &fakeUserRepo{},
        &fakeEmailSender{},
        slog.Default(),
    )
    user, err := svc.CreateUser(context.Background(), "Alice", "alice@example.com")
    // assert...
}
```

**With google/wire for large projects:**

```go
// wire_gen.go (auto-generated by wire CLI)
func InitializeServer(addr string) (*http.Server, error) {
    db := provideDB()
    userRepo := postgres.NewUserRepo(db)
    emailClient := smtp.NewClient(os.Getenv("SMTP_HOST"))
    logger := provideLogger()
    userSvc := NewUserService(userRepo, emailClient, logger)
    userHandler := handler.NewUserHandler(userSvc, logger)
    server := provideServer(addr, userHandler)
    return server, nil
}
```
