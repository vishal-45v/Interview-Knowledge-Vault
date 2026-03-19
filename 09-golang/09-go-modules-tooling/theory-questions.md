# Go Modules & Tooling — Theory Questions

---

## Go Modules Fundamentals

**Q1. What is a Go module and what does go.mod contain?**

A Go module is a collection of Go packages versioned together as a unit. It was introduced in Go 1.11 to replace the GOPATH-based dependency management. Every module has a `go.mod` file at its root.

```
module github.com/mycompany/myservice

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/jackc/pgx/v5 v5.5.5
    golang.org/x/crypto v0.21.0
)

require (
    // indirect dependencies (transitive, required at compile time)
    github.com/bytedance/sonic v1.11.3 // indirect
    github.com/gin-contrib/sse v0.1.0 // indirect
)
```

Fields explained:
- `module` — the module path (acts as the import prefix)
- `go` — minimum Go version required
- `require` — direct and indirect dependencies with versions
- `replace` — override a dependency with a local path or different version
- `exclude` — exclude specific versions (rare)
- `retract` — mark versions as retracted by the module author

---

**Q2. What is go.sum and what does it contain?**

`go.sum` is a security file containing cryptographic hashes (SHA-256) of every dependency's source code and go.mod file.

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

Each entry has two lines:
1. Hash of the module's zip file (`h1:...` format)
2. Hash of just the go.mod file

The `go` tool verifies these hashes when downloading modules, preventing supply chain attacks where a module version is replaced with malicious code. **Never delete or manually edit go.sum.**

---

**Q3. What is the difference between GOPATH mode and module mode?**

| Feature | GOPATH mode | Module mode |
|---------|------------|-------------|
| Code location | Must be in `$GOPATH/src` | Anywhere on filesystem |
| Versioning | No versioning | Semantic versioning |
| Reproducibility | Not guaranteed | go.sum ensures it |
| Activation | Default before Go 1.13 | Default since Go 1.16 |
| Vendor | Optional | `go mod vendor` available |
| go.mod | Not present | Required |

Module mode is activated by having a `go.mod` file in the project root (or any parent directory). Set `GO111MODULE=on/off/auto` to override.

---

**Q4. What does `go mod tidy` do?**

`go mod tidy` ensures `go.mod` and `go.sum` exactly match the imports in your code:

1. **Adds** missing required packages (packages you import but don't have in go.mod)
2. **Removes** unused packages (packages in go.mod that aren't imported anywhere)
3. **Updates** go.sum with hashes for all added packages
4. **Promotes** indirect dependencies that are directly imported

```bash
# After adding a new import in code:
go mod tidy
# After removing an import from code:
go mod tidy
# In CI: verify go.mod and go.sum are clean:
go mod tidy && git diff --exit-code go.mod go.sum
```

Warning: `go mod tidy` may remove a package that is needed at runtime (loaded via plugin or reflection) but not via static import. Keep such packages imported with a blank identifier if needed.

---

**Q5. What does `go get` do and when do you use it?**

```bash
# Add a new dependency
go get github.com/gin-gonic/gin@v1.9.1

# Upgrade to latest version
go get github.com/gin-gonic/gin@latest

# Upgrade to a specific commit
go get github.com/somepackage@abc1234

# Downgrade
go get github.com/somepackage@v1.2.3

# Remove a dependency (sets to none)
go get github.com/somepackage@none
```

`go get` modifies `go.mod` and `go.sum`. After running `go get`, you should run `go mod tidy` to clean up any indirect dependencies.

---

**Q6. What is `go mod vendor` and when do you use it?**

`go mod vendor` copies all dependencies into a `vendor/` directory in your project.

```bash
go mod vendor
# Creates: vendor/modules.txt, vendor/<module>/<package>/*
```

Benefits:
- **Offline builds**: no network access needed
- **Audit**: all third-party code is visible in your repo
- **Reproducibility**: immune to proxy/network issues
- **Security**: immutable snapshot of dependencies

Build using vendor:
```bash
go build -mod=vendor ./...
go test -mod=vendor ./...
```

When to use:
- Large enterprises with air-gapped CI environments
- Projects requiring strict audit of all dependencies
- When your module proxy is unreliable

---

**Q7. What is semantic versioning in Go modules? Why do v2+ modules require an import path change?**

Go modules follow SemVer: `vMAJOR.MINOR.PATCH`

- **PATCH** (v1.0.0 → v1.0.1): bug fixes, backward compatible
- **MINOR** (v1.0.0 → v1.1.0): new features, backward compatible
- **MAJOR** (v1.0.0 → v2.0.0): breaking changes

For v2+, Go requires changing the module path to include the major version:

```go
// go.mod for v2:
module github.com/mycompany/mylib/v2

// Imports in code using v2:
import "github.com/mycompany/mylib/v2/somepackage"

// vs v1 import:
import "github.com/mycompany/mylib/somepackage"
```

**Why?** A program can import both v1 and v2 of the same library simultaneously (they're considered different modules). This allows gradual migration. Without the path change, the compiler couldn't distinguish them.

---

**Q8. What is the `replace` directive in go.mod?**

`replace` overrides a module's source to point elsewhere:

```
# Use a local fork instead of the published version
replace github.com/original/pkg => ../my-fork

# Replace with a specific version of a fork on GitHub
replace github.com/original/pkg v1.2.3 => github.com/my-fork/pkg v1.2.4-patch

# Pin to an exact commit (useful for bugfixes not yet released)
replace github.com/original/pkg => github.com/original/pkg v0.0.0-20240101abcdef12
```

**Important limitation:** `replace` only applies to the **top-level module**. If your library uses `replace`, consumers of your library won't get the replacement — they need to add the replace directive themselves.

---

**Q9. What is the Go module proxy and how does GOPROXY work?**

The module proxy is an HTTP server that caches and serves Go modules. `GOPROXY` is a comma-separated list of proxy URLs.

```bash
# Default value:
GOPROXY=https://proxy.golang.org,direct

# Meaning:
# 1. Try proxy.golang.org (Google's public proxy)
# 2. If not found there, go direct to VCS (GitHub, etc.)
# 3. "off" would mean: fail if not in any proxy

# Private modules (skip proxy):
GONOSUMCHECK=github.com/mycompany/*
GOPRIVATE=github.com/mycompany/*
GONOSUMDB=github.com/mycompany/*

# Disable proxy (go direct always):
GOPROXY=direct

# Corporate proxy:
GOPROXY=https://athens.mycompany.com,direct
```

The proxy provides:
- Faster downloads (CDN-cached)
- Availability even if source VCS goes down
- `go.sum` database verification (sum.golang.org)

---

**Q10. What is the Go workspace (`go work`) introduced in Go 1.18?**

A Go workspace allows working with multiple modules simultaneously without needing replace directives.

```
myworkspace/
├── go.work          ← workspace file
├── service-a/
│   ├── go.mod       (module github.com/me/service-a)
│   └── main.go
├── service-b/
│   ├── go.mod       (module github.com/me/service-b)
│   └── main.go
└── shared-lib/
    ├── go.mod       (module github.com/me/shared-lib)
    └── lib.go
```

```
// go.work:
go 1.22

use (
    ./service-a
    ./service-b
    ./shared-lib
)
```

With `go.work`, changes to `shared-lib` are immediately visible to `service-a` and `service-b` without publishing a new version or using `replace` directives. Perfect for mono-repos or local development across modules.

---

**Q11. What are build tags and how do you use them?**

Build tags (build constraints) conditionally include or exclude files during compilation.

**New syntax (Go 1.17+):**
```go
//go:build linux && amd64
// +build linux amd64  ← old syntax (still supported for compatibility)

package mypackage
```

**Common uses:**

```go
// Platform-specific implementation:
// file: db_postgres.go
//go:build !windows
package storage

// file: db_sqlite.go
//go:build windows
package storage

// Custom tags for test utilities:
//go:build integration
package tests

// file: client_real.go
//go:build !mock
package api

// file: client_mock.go
//go:build mock
package api
```

Build with tags:
```bash
go build -tags integration ./...
go test -tags "integration e2e" ./...
```

---

**Q12. What is `go generate` and what is it used for?**

`go generate` runs arbitrary commands listed in `//go:generate` comments. It is NOT run automatically by `go build` — you must invoke it explicitly.

```go
//go:generate protoc --go_out=. --go-grpc_out=. proto/service.proto
//go:generate mockgen -destination=mocks/db_mock.go -package=mocks . UserRepository
//go:generate stringer -type=Color
//go:generate go run github.com/99designs/gqlgen generate
```

```bash
go generate ./...  # run all generators in all packages
go generate ./internal/...
```

Common use cases:
- Generate protobuf stubs
- Generate mock implementations for testing
- Generate `String()` methods for enum types
- Generate type-safe SQL query code (sqlc)
- Generate GraphQL resolvers

---

**Q13. What is `golangci-lint` and which linters are most important?**

`golangci-lint` is a meta-linter that runs many static analysis tools simultaneously.

```bash
# Install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run
golangci-lint run ./...
```

Key linters enabled by default:
- `errcheck` — finds unchecked errors
- `gosimple` — suggests code simplifications
- `govet` — `go vet` checks (shadowed variables, printf format strings)
- `ineffassign` — detects assignments whose values are never used
- `staticcheck` — comprehensive static analysis
- `unused` — finds unexported identifiers never used

Essential additional linters to enable:
```yaml
# .golangci.yml
linters:
  enable:
    - gocyclo      # cyclomatic complexity
    - godot        # comment formatting
    - misspell     # spelling in comments/strings
    - revive       # general Go style
    - gosec        # security issues
    - exhaustive   # exhaustive switch statements
    - noctx        # HTTP requests without context
    - bodyclose    # http response body close check
```

---

**Q14. How do you cross-compile a Go binary?**

Go has excellent cross-compilation built in — just set `GOOS` and `GOARCH`:

```bash
# Build for Linux AMD64 (from any platform)
GOOS=linux GOARCH=amd64 go build -o bin/app-linux-amd64 ./cmd/app

# Build for macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o bin/app-darwin-arm64 ./cmd/app

# Build for Windows
GOOS=windows GOARCH=amd64 go build -o bin/app-windows.exe ./cmd/app

# Build for Linux ARM64 (Raspberry Pi 4, cloud ARM instances)
GOOS=linux GOARCH=arm64 go build -o bin/app-linux-arm64 ./cmd/app

# Check all supported targets:
go tool dist list
```

**CGO and cross-compilation:**

```bash
# CGO requires a cross-compiler for each target — disable for pure Go:
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ./...

# Static binary (no libc dependency):
CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' ./...
```

---

**Q15. What is the `go:embed` directive?**

`go:embed` (Go 1.16) embeds files or directories into the compiled binary at build time.

```go
import "embed"

// Embed a single file
//go:embed config/defaults.yaml
var defaultConfig []byte

// Embed a string
//go:embed VERSION
var version string

// Embed an entire directory as fs.FS
//go:embed templates/*
var templateFiles embed.FS

// Embed multiple patterns
//go:embed static/*.css static/*.js
var staticFiles embed.FS

func loadTemplate(name string) (string, error) {
    b, err := templateFiles.ReadFile("templates/" + name)
    return string(b), err
}
```

The embedded files become part of the binary — no need to ship separate files. Useful for:
- HTML templates, CSS, JavaScript for web servers
- Default configuration files
- SQL migration files
- TLS certificates
- Swagger/OpenAPI specs

---

**Q16. What is `go vet` and `staticcheck`?**

`go vet` is built into the Go toolchain and catches suspicious code patterns:

```bash
go vet ./...
```

Checks include:
- Printf format string mismatches (`fmt.Sprintf("%d", "string")`)
- Unreachable code after return
- Incorrect mutex copying
- Incorrect use of `sync.Mutex` in copy
- Tests that always pass (test function not starting with `Test`)

`staticcheck` is a more comprehensive tool:

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...
```

Additional checks:
- Deprecated API usage
- Incorrect context usage
- Concurrent access issues
- Unnecessary type conversions
- Dead code

---

**Q17. What are Dockerfile best practices for Go services?**

Use multi-stage builds to produce minimal final images:

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy go.mod first for layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -o server \
    ./cmd/server

# Stage 2: Minimal runtime image
FROM gcr.io/distroless/static-debian12

WORKDIR /app
COPY --from=builder /app/server /app/server

# Non-root user (distroless has nonroot user by default)
USER nonroot:nonroot

EXPOSE 8080
ENTRYPOINT ["/app/server"]
```

Key practices:
- `CGO_ENABLED=0` — no libc dependency, runs in scratch/distroless
- `-ldflags="-s -w"` — strip debug symbols (reduces binary size ~30%)
- `distroless/static` — only contains CA certs and tzdata, no shell
- Copy go.mod before source code — Docker caches `go mod download`
- Build binary in alpine, run in distroless — final image ~10-20MB vs ~300MB (golang:alpine)

---

**Q18. What does `go tool trace` do and how is it different from pprof?**

`go tool trace` provides a detailed timeline of runtime events:

```bash
# Collect trace
curl "http://localhost:6060/debug/pprof/trace?seconds=10" > trace.out
go tool trace trace.out
```

Shows:
- Goroutine scheduling (when created, when runnable, when blocking, on which P)
- GC events with timing
- System calls
- Network I/O events
- Heap size over time
- Processor (P) utilization per CPU

vs pprof:
- **pprof**: "Where does CPU time/memory go?" (statistical sampling)
- **trace**: "What happened and when, exactly?" (event-based timeline)

Use pprof first to find hot spots. Use trace to diagnose scheduling issues, GC pauses, and P starvation.

---

**Q19. What is `CGO_ENABLED=0` and when do you need it?**

`CGO_ENABLED=0` disables cgo (the mechanism for calling C code from Go). This makes the binary:

1. **Statically linked** — no external shared library dependencies
2. **Fully cross-compilable** — no need for a C cross-compiler
3. **Runnable in distroless/scratch containers** — no libc required

```bash
# Required for: Docker scratch/distroless, cross-compilation
CGO_ENABLED=0 GOOS=linux go build ./...

# Check if binary has dependencies:
ldd ./myapp
# "not a dynamic executable" = fully static (good for containers)

# Some stdlib packages require cgo:
# - net (DNS resolution on some platforms) — use pure Go fallback
# - os/user (getpwuid_r) — use pure Go fallback
```

Pure Go fallbacks are activated when `CGO_ENABLED=0`:
```go
// net package uses pure Go DNS by default when CGO disabled
```

---

**Q20. What are `init()` functions and how do they affect module loading?**

`init()` functions run automatically after all package-level variables are initialized, before `main()`.

```go
package mypackage

var db *sql.DB

func init() {
    // Called automatically, before any code using this package runs
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal("failed to connect:", err)
    }
}
```

**Ordering rules:**
1. Within a package: init() functions run in the order they appear in the file, files in alphabetical order
2. Across packages: dependencies' init() run before the importing package's init()
3. A package's init() runs only once, even if imported multiple times

**The blank import trick:**
```go
// Import solely for its side-effect (init() function):
import _ "github.com/lib/pq"            // registers postgres driver
import _ "net/http/pprof"               // registers pprof HTTP handlers
import _ "image/png"                    // registers PNG decoder
```

**Warning:** Overusing init() for complex logic makes code hard to test and understand. Prefer explicit initialization in `main()` or constructor functions.

---

**Q21. What is the difference between `go build` flags: `-ldflags`, `-tags`, `-trimpath`?**

```bash
# -ldflags: pass flags to the linker
go build -ldflags="-s -w" ./...
# -s: strip symbol table (smaller binary)
# -w: strip DWARF debug info (smaller binary)
# Result: ~30% smaller binary

# Inject version info at build time:
go build -ldflags="-X main.version=1.2.3 -X main.commit=$(git rev-parse HEAD)" ./...

# In code:
var version string  // set by -X flag

# -tags: conditional compilation
go build -tags "netgo osusergo" ./...  # pure Go networking

# -trimpath: remove local filesystem paths from binary (security/reproducibility)
go build -trimpath ./...
# Without: binary contains /home/user/projects/myapp/cmd/main.go
# With: binary contains only module-relative paths
```

---

**Q22. How does `go install` differ from `go build`?**

```bash
# go build: compiles and places binary in current directory (or -o path)
go build -o ./bin/myapp ./cmd/myapp

# go install: compiles and places binary in $GOPATH/bin or $GOBIN
go install ./cmd/myapp
go install github.com/some/tool@latest  # install without needing source

# Check where binaries go:
go env GOBIN  # usually $GOPATH/bin or ~/go/bin
```

`go install` with a module path + version is the recommended way to install Go CLI tools without affecting your module's dependencies.
