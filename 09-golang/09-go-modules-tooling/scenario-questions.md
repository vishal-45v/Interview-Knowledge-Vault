# Go Modules & Tooling — Scenario Questions

---

## Scenario 1: Using a Local Fork Temporarily

**"You need to use a fork of a dependency temporarily while your PR to the upstream is pending. How do you do it?"**

**Step 1 — Fork and clone the dependency:**
```bash
# Fork github.com/original/loglib on GitHub → github.com/you/loglib
git clone github.com/you/loglib ../loglib
cd ../loglib
git checkout -b fix/my-bugfix
# Make changes
```

**Step 2 — Add replace directive to go.mod:**
```
module github.com/mycompany/myservice

go 1.22

require (
    github.com/original/loglib v1.3.2
)

// Temporary: use local fork while PR is in review
replace github.com/original/loglib => ../loglib
```

**Step 3 — Verify it works:**
```bash
go mod tidy
go build ./...
```

**When your PR is merged:**
```bash
# Remove the replace directive
# Update to the new upstream version
go get github.com/original/loglib@v1.3.3
go mod tidy
```

**If you want to point at your GitHub fork (not local):**
```
replace github.com/original/loglib => github.com/you/loglib v0.0.0-20240315abcdef12
```

```bash
go mod tidy  # fetches the specific commit from your fork
```

---

## Scenario 2: CI Builds Re-downloading Modules

**"Your CI build takes 3 minutes just to download modules. How do you fix it?"**

**Root cause:** Module downloads are not cached between CI runs.

**Solution 1 — Cache the Go module cache in CI:**

```yaml
# GitHub Actions:
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build     # build cache
      ~/go/pkg/mod          # module download cache
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-

- name: Build
  run: go build ./...
```

```yaml
# GitLab CI:
variables:
  GOPATH: $CI_PROJECT_DIR/.go
  GOCACHE: $CI_PROJECT_DIR/.go-cache

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .go/pkg/mod/
    - .go-cache/

build:
  script:
    - go build ./...
```

**Solution 2 — Use `go mod vendor`:**

```bash
go mod vendor
git add vendor/
git commit -m "vendor dependencies"
```

```dockerfile
# In CI / Dockerfile: no download needed
go build -mod=vendor ./...
```

**Solution 3 — Set up a private module proxy (Athens):**

```bash
# Self-hosted module proxy that caches all downloads
docker run -p 3000:3000 gomods/athens:latest

# Point builds at it:
GOPROXY=http://your-athens:3000,direct go build ./...
```

**Solution 4 — Use `GOFLAGS` to always use vendor:**

```bash
export GOFLAGS=-mod=vendor
# Now go build, go test, etc. all use vendor/ automatically
```

---

## Scenario 3: Cross-Compiling for Linux ARM64 from macOS

**"You need to build a binary for a Linux ARM64 server from your macOS M2 laptop."**

```bash
# Pure Go: just set environment variables
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o bin/app-linux-arm64 ./cmd/app

# Verify the binary format:
file bin/app-linux-arm64
# Output: ELF 64-bit LSB executable, ARM aarch64

# Check no dynamic dependencies:
# (On Linux ARM64 machine)
ldd ./app-linux-arm64
# "not a dynamic executable"
```

**If CGO is required:**
```bash
# Need a cross-compiler
brew install FiloSottile/musl-cross/musl-cross

# Set the C cross-compiler:
CC=aarch64-linux-musl-gcc CGO_ENABLED=1 GOOS=linux GOARCH=arm64 \
    go build -o bin/app-linux-arm64 ./cmd/app
```

**Using Docker for cross-compilation (most reliable for CGO):**
```dockerfile
# Build stage runs on Linux ARM64 (via Docker buildx)
# docker buildx build --platform linux/arm64 -t myapp:arm64 .

FROM --platform=linux/arm64 golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/server

FROM --platform=linux/arm64 gcr.io/distroless/static-debian12
COPY --from=builder /app/server /app/server
ENTRYPOINT ["/app/server"]
```

```bash
# Build multi-platform image:
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
```

---

## Scenario 4: Setting Up a New Go Project from Scratch

**"Start a new Go project with proper module setup, linting, and testing infrastructure."**

```bash
# 1. Initialize the module
mkdir myservice && cd myservice
go mod init github.com/mycompany/myservice

# 2. Add initial dependencies
go get github.com/gin-gonic/gin@latest
go get github.com/jackc/pgx/v5@latest
go get go.uber.org/zap@latest

# 3. Create basic structure
mkdir -p cmd/server internal/{handler,service,repository,domain} pkg

# 4. Set up golangci-lint
cat > .golangci.yml << 'EOF'
run:
  timeout: 5m
  go: "1.22"

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - staticcheck
    - bodyclose
    - noctx
    - gosec
    - misspell
    - revive

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
EOF

# 5. Set up Makefile
cat > Makefile << 'EOF'
.PHONY: build test lint tidy

build:
    CGO_ENABLED=0 go build -ldflags="-s -w" -o bin/server ./cmd/server

test:
    go test -race -count=1 ./...

lint:
    golangci-lint run ./...

tidy:
    go mod tidy
    go mod verify

generate:
    go generate ./...
EOF

# 6. Set up CI (.github/workflows/ci.yml) and pre-commit hooks
```

---

## Scenario 5: Debugging Module Version Conflicts

**"Your build fails with 'ambiguous import' or a dependency requires a newer version of another package. How do you resolve it?"**

```bash
# Step 1: See full dependency graph
go mod graph | head -50

# Step 2: Find who requires the conflicting version
go mod why github.com/conflicting/package
# Output: explains import path from your code to the package

# Step 3: View all versions of a dependency in use
go list -m all | grep conflicting

# Step 4: Force a minimum version
go get github.com/conflicting/package@v2.1.0
go mod tidy

# Step 5: If a transitive dep requires a very old version that causes issues:
# Add an explicit minimum version requirement
```

```
// go.mod — explicit version override:
require (
    github.com/indirect/dep v1.5.0  // normally indirect but we need minimum 1.5.0
)
```

**Exclude a broken version:**
```
exclude github.com/buggy/package v1.2.3
```

**Debugging with -v flag:**
```bash
go build -v ./...  # shows which packages are compiled
go get -v github.com/some/package  # verbose download info
go list -m -json all | jq '.Path, .Version' | head -40
```

---

## Scenario 6: Embedding Static Files

**"Your web server needs to serve HTML, CSS, and JavaScript. Embed them in the binary."**

```
Project structure:
mywebapp/
├── cmd/server/main.go
├── internal/web/
│   ├── embed.go
│   └── handler.go
└── web/
    ├── static/
    │   ├── app.css
    │   └── app.js
    └── templates/
        ├── index.html
        └── error.html
```

```go
// internal/web/embed.go
package web

import (
    "embed"
    "io/fs"
)

//go:embed ../../web/static
var staticFiles embed.FS

//go:embed ../../web/templates
var templateFiles embed.FS

// GetStaticFS returns a file system rooted at web/static
func GetStaticFS() fs.FS {
    sub, _ := fs.Sub(staticFiles, "web/static")
    return sub
}

// GetTemplateFS returns a file system rooted at web/templates
func GetTemplateFS() fs.FS {
    sub, _ := fs.Sub(templateFiles, "web/templates")
    return sub
}
```

```go
// internal/web/handler.go
package web

import (
    "html/template"
    "net/http"
)

func RegisterRoutes(mux *http.ServeMux) {
    // Serve static files from embedded FS
    staticHandler := http.FileServer(http.FS(GetStaticFS()))
    mux.Handle("/static/", http.StripPrefix("/static/", staticHandler))

    // Parse templates from embedded FS
    tmpl := template.Must(template.ParseFS(GetTemplateFS(), "*.html"))

    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        tmpl.ExecuteTemplate(w, "index.html", nil)
    })
}
```

---

## Scenario 7: Multi-Module Workspace

**"You have a monorepo with 3 services and a shared library. How do you set up local development so changes to the library are immediately reflected?"**

```
mycompany/
├── go.work              ← workspace root
├── shared/
│   ├── go.mod           (module github.com/mycompany/shared)
│   └── logger/
├── service-api/
│   ├── go.mod           (module github.com/mycompany/service-api)
│   └── main.go
├── service-worker/
│   ├── go.mod           (module github.com/mycompany/service-worker)
│   └── main.go
└── service-notifier/
    ├── go.mod           (module github.com/mycompany/service-notifier)
    └── main.go
```

```bash
# Initialize workspace
cd mycompany
go work init ./shared ./service-api ./service-worker ./service-notifier
```

```
// go.work:
go 1.22

use (
    ./shared
    ./service-api
    ./service-worker
    ./service-notifier
)
```

```go
// service-api/go.mod:
module github.com/mycompany/service-api

go 1.22

require (
    github.com/mycompany/shared v0.0.0  // resolved by go.work
)
```

```go
// service-api/main.go:
import "github.com/mycompany/shared/logger"
// Changes to shared/logger/ are immediately visible here — no publish needed
```

```bash
# Build any service (workspace resolves shared automatically):
cd service-api && go build ./...

# Run tests across all modules:
cd mycompany && go test ./...

# In CI (no workspace — use real published versions):
GOWORK=off go test ./...
```

---

## Scenario 8: Build Tags for Platform-Specific Code

**"Your service uses different implementations for Linux (epoll) and macOS (kqueue). Set up platform-specific builds."**

```go
// poller_linux.go
//go:build linux

package poller

import "syscall"

type Poller struct {
    epfd int
}

func New() (*Poller, error) {
    fd, err := syscall.EpollCreate1(0)
    return &Poller{epfd: fd}, err
}

func (p *Poller) Add(fd int) error {
    return syscall.EpollCtl(p.epfd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{
        Events: syscall.EPOLLIN,
        Fd:     int32(fd),
    })
}
```

```go
// poller_darwin.go
//go:build darwin

package poller

import "syscall"

type Poller struct {
    kq int
}

func New() (*Poller, error) {
    kq, err := syscall.Kqueue()
    return &Poller{kq: kq}, err
}

func (p *Poller) Add(fd int) error {
    _, err := syscall.Kevent(p.kq, []syscall.Kevent_t{{
        Ident:  uint64(fd),
        Filter: syscall.EVFILT_READ,
        Flags:  syscall.EV_ADD,
    }}, nil, nil)
    return err
}
```

```go
// poller_test.go (platform-agnostic test)
package poller

import "testing"

func TestPollerAddAndRemove(t *testing.T) {
    p, err := New()
    if err != nil {
        t.Fatal(err)
    }
    // test using the platform-appropriate implementation
}
```

---

## Scenario 9: Injecting Build Information

**"Your service needs to report its version, commit hash, and build time at startup and in /health responses."**

```go
// internal/version/version.go
package version

var (
    // Set by -ldflags at build time
    Version   = "dev"
    Commit    = "unknown"
    BuildTime = "unknown"
    GoVersion = runtime.Version()
)

func Info() map[string]string {
    return map[string]string{
        "version":    Version,
        "commit":     Commit,
        "build_time": BuildTime,
        "go_version": GoVersion,
    }
}
```

```makefile
# Makefile
VERSION := $(shell git describe --tags --always --dirty)
COMMIT  := $(shell git rev-parse --short HEAD)
BUILDTIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

build:
    go build \
        -ldflags="-s -w \
          -X github.com/mycompany/myservice/internal/version.Version=$(VERSION) \
          -X github.com/mycompany/myservice/internal/version.Commit=$(COMMIT) \
          -X github.com/mycompany/myservice/internal/version.BuildTime=$(BUILDTIME)" \
        -o bin/server ./cmd/server
```

```go
// In main.go or health handler:
func main() {
    log.Printf("starting %s commit=%s built=%s",
        version.Version, version.Commit, version.BuildTime)
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(map[string]interface{}{
        "status":  "ok",
        "version": version.Info(),
    })
}
```

---

## Scenario 10: Minimal Docker Image

**"Build the smallest possible Docker image for a Go HTTP server."**

```dockerfile
# syntax=docker/dockerfile:1.7

# Stage 1: Compile
FROM golang:1.22-alpine AS builder

# Install ca-certificates (needed for HTTPS in distroless)
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /build

# Dependency cache layer (only re-runs when go.mod/go.sum change)
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a \
    -ldflags="-s -w -extldflags '-static'" \
    -o server \
    ./cmd/server

# Stage 2: Minimal runtime
FROM scratch

# Copy timezone data and CA certs from builder
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /build/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

```bash
docker build -t myapp:latest .
docker image ls myapp:latest
# Size: ~8-15MB (vs ~300MB for golang:alpine)

# Or use distroless (includes useful utilities, better security):
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /build/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
# Size: ~5MB + your binary
```
