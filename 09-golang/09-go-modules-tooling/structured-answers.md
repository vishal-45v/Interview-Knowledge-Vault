# Go Modules & Tooling — Structured Answers

---

## Answer 1: What Is a Go Module and How Is It Different from GOPATH?

**One-sentence answer:** A Go module is a versioned, self-contained unit of code defined by `go.mod`, allowing dependency management anywhere on the filesystem without the rigid GOPATH directory requirement.

**GOPATH (the old way):**

```bash
# All Go code had to live under $GOPATH/src:
$GOPATH/
├── bin/           # compiled binaries
├── pkg/           # compiled packages
└── src/
    ├── github.com/
    │   └── myname/
    │       └── myproject/   # ← your code MUST be here
    └── github.com/
        └── somelib/
            └── ...           # ← all dependencies MUST be here too
```

Problems: no versioning (always `master`), import paths tied to directory structure, all projects shared the same dependency tree.

**Module mode (current):**

```bash
# Code anywhere:
~/projects/myproject/
├── go.mod         # defines module path and dependencies
├── go.sum         # cryptographic lock file
├── main.go
└── internal/

# go.mod example:
module github.com/myname/myproject

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1   // pinned version
)
```

Key improvements:
- Code lives anywhere on the filesystem
- Each project has isolated, versioned dependencies
- Reproducible builds via go.sum hashes
- Multiple versions of the same library can coexist

---

## Answer 2: What Is go.sum and Why Is It Important for Security?

**One-sentence answer:** `go.sum` records the SHA-256 hash of every dependency's source code, which the Go toolchain verifies on every download to prevent supply chain attacks where a published module version is silently replaced with malicious code.

**Format:**

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

Two entries per module:
1. `h1:...` hash of the module zip file (source code)
2. `/go.mod h1:...` hash of just the go.mod file (used for dependency resolution)

**Security model:**

```bash
# When you run go build:
# 1. Download module from GOPROXY
# 2. Compute SHA-256 of the downloaded content
# 3. Compare with go.sum entry
# 4. Also verify against sum.golang.org (public transparency log)
# If they don't match → BUILD FAILS with:
# "verifying github.com/evil/pkg@v1.0.0: checksum mismatch"

# This prevents:
# - BGP hijacking redirecting your traffic to a malicious server
# - A maintainer replacing a published version with backdoored code
# - Man-in-the-middle attacks
```

**Best practices:**

```bash
# Always commit go.sum to version control
git add go.sum && git commit

# Verify in CI:
go mod verify  # checks all cached modules match go.sum

# Never do this in CI:
GONOSUMCHECK=* go build  # disables verification — security risk!

# For private modules (can't verify against public sum DB):
GONOSUMDB=gitlab.mycompany.com/* go get ...
GONOSUMCHECK=gitlab.mycompany.com/* go build ...
```

---

## Answer 3: How Do You Use a Local Fork of a Dependency?

**One-sentence answer:** Add a `replace` directive in `go.mod` pointing to your local fork path or GitHub fork, then remove it when the upstream fix is published.

**Step-by-step:**

```bash
# 1. Clone and modify the dependency
git clone https://github.com/original/pkg ~/workspace/pkg-fork
cd ~/workspace/pkg-fork
git checkout -b fix/my-bugfix
# make your changes

# 2. Add replace directive to your project's go.mod
cd ~/workspace/myproject
```

```
# go.mod:
module github.com/me/myproject

go 1.22

require (
    github.com/original/pkg v1.3.2  // keep the original version listed
)

// Use local fork:
replace github.com/original/pkg => ../pkg-fork

// OR use a fork on GitHub (specific commit):
replace github.com/original/pkg => github.com/me/pkg-fork v0.0.0-20240315abc123
```

```bash
go mod tidy  # updates go.sum for the replacement

# Build and test with your fork:
go build ./...
go test ./...
```

**When upstream merges your PR:**
```bash
# Remove the replace directive from go.mod
# Update to the new upstream version
go get github.com/original/pkg@v1.3.3
go mod tidy
```

**Important limitation:** `replace` only works in your top-level module. If you publish a library that uses `replace`, your library's consumers do NOT get the replacement — they must add it themselves.

---

## Answer 4: How Do You Cross-Compile a Go Binary?

**One-sentence answer:** Set `GOOS` and `GOARCH` environment variables before `go build`, and set `CGO_ENABLED=0` for pure Go binaries (required for cross-compilation without a C cross-compiler).

```bash
# Common targets:
CGO_ENABLED=0 GOOS=linux   GOARCH=amd64  go build -o app-linux-amd64   ./cmd/app
CGO_ENABLED=0 GOOS=linux   GOARCH=arm64  go build -o app-linux-arm64   ./cmd/app
CGO_ENABLED=0 GOOS=darwin  GOARCH=arm64  go build -o app-darwin-arm64  ./cmd/app
CGO_ENABLED=0 GOOS=darwin  GOARCH=amd64  go build -o app-darwin-amd64  ./cmd/app
CGO_ENABLED=0 GOOS=windows GOARCH=amd64  go build -o app-windows.exe   ./cmd/app

# See all supported combinations:
go tool dist list

# Add version info:
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w -X main.version=$(git describe --tags)" \
    -o app-linux-amd64 \
    ./cmd/app

# Makefile for multi-platform release:
PLATFORMS := linux/amd64 linux/arm64 darwin/amd64 darwin/arm64 windows/amd64

release:
    @for platform in $(PLATFORMS); do \
        GOOS=$$(echo $$platform | cut -d'/' -f1); \
        GOARCH=$$(echo $$platform | cut -d'/' -f2); \
        ext=""; [ "$$GOOS" = "windows" ] && ext=".exe"; \
        CGO_ENABLED=0 GOOS=$$GOOS GOARCH=$$GOARCH go build \
            -ldflags="-s -w" \
            -o bin/app-$$GOOS-$$GOARCH$$ext \
            ./cmd/app; \
    done
```

---

## Answer 5: How Do You Build a Minimal Docker Image for a Go Service?

**One-sentence answer:** Use a multi-stage Dockerfile: compile a statically linked binary (`CGO_ENABLED=0`) in a golang builder stage, then copy only the binary into a `distroless/static` or `scratch` base image.

```dockerfile
# syntax=docker/dockerfile:1.7

# ─── Stage 1: Build ───────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder

# apk installs CA certs and timezone data (needed in final stage)
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /build

# Layer cache: download dependencies before copying all source
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy source and build
COPY . .
ARG VERSION=dev
ARG COMMIT=unknown
RUN CGO_ENABLED=0 GOOS=linux go build \
        -ldflags="-s -w \
            -X main.version=${VERSION} \
            -X main.commit=${COMMIT}" \
        -trimpath \
        -o server \
        ./cmd/server

# ─── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

# Copy timezone data and TLS certificates
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the binary
COPY --from=builder /build/server /server

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Build:**
```bash
docker build \
    --build-arg VERSION=$(git describe --tags) \
    --build-arg COMMIT=$(git rev-parse --short HEAD) \
    -t myservice:latest .

docker image ls myservice:latest
# REPOSITORY   TAG     IMAGE ID       SIZE
# myservice    latest  abc123def456   12.4MB
```

**Size comparison:**
- `golang:1.22` (full): ~850 MB
- `golang:1.22-alpine` (builder): ~250 MB
- `distroless/static` (runtime): ~2 MB base + binary

---

## Answer 6: What Is `go:embed` and How Do You Use It?

**One-sentence answer:** `//go:embed` is a compiler directive that embeds files from the filesystem into the binary at compile time, so the running program carries its own static assets without requiring external files.

```go
package main

import (
    "embed"
    "io/fs"
    "net/http"
    "text/template"
)

// 1. Embed a single file as []byte
//go:embed configs/default.yaml
var defaultConfig []byte

// 2. Embed a single file as string
//go:embed VERSION
var buildVersion string

// 3. Embed an entire directory tree as fs.FS
//go:embed web/static
var staticFS embed.FS

// 4. Embed multiple patterns
//go:embed migrations/*.sql
var migrationFiles embed.FS

func main() {
    // Use string/byte embeds directly
    fmt.Printf("Version: %s\n", buildVersion)
    cfg := parseConfig(defaultConfig)

    // Serve static files from embedded FS
    sub, _ := fs.Sub(staticFS, "web/static")
    http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(sub))))

    // Parse templates from embedded FS
    tmpl := template.Must(template.ParseFS(staticFS, "web/static/templates/*.html"))
    tmpl.Execute(w, data)

    // Read migration files
    entries, _ := migrationFiles.ReadDir("migrations")
    for _, entry := range entries {
        sql, _ := migrationFiles.ReadFile("migrations/" + entry.Name())
        db.Exec(string(sql))
    }
}
```

**What can be embedded:**
- Individual files: `//go:embed file.txt`
- Glob patterns: `//go:embed *.json`
- Directories: `//go:embed dir/` (includes all files recursively)

**What cannot be embedded:**
- Files outside the module root
- Files matching `.` (dotfiles) or `_` prefix (unless using `all:` prefix)
- Generated files not present at build time

```go
// Embed dotfiles (e.g., .gitkeep, .htaccess):
//go:embed all:web/static
var staticFS embed.FS
```

---

## Answer 7: How Do You Set Up a Multi-Module Go Workspace?

**One-sentence answer:** Create a `go.work` file in the root of your workspace using `go work init` listing all module directories, allowing changes to shared modules to be immediately visible without publishing versions.

```bash
# Project structure:
~/workspace/
├── go.work
├── platform/           # shared infrastructure code
│   └── go.mod          (module github.com/myco/platform)
├── user-service/       # microservice
│   └── go.mod          (module github.com/myco/user-service)
└── order-service/      # microservice
    └── go.mod          (module github.com/myco/order-service)

# Initialize workspace:
cd ~/workspace
go work init ./platform ./user-service ./order-service
```

```
// go.work:
go 1.22

use (
    ./platform
    ./user-service
    ./order-service
)

// Optional: replace for external deps across all modules:
replace github.com/external/buggy => ./local-fix
```

```go
// user-service/go.mod:
module github.com/myco/user-service

require github.com/myco/platform v0.0.0  // resolved from workspace

// user-service/main.go:
import "github.com/myco/platform/logger"
// Changes to ../platform/logger/ are immediately visible here!
```

**Add a new module to workspace:**
```bash
go work use ./new-service
```

**Disable workspace for CI (use published versions):**
```bash
GOWORK=off go test ./...  # ignores go.work, uses go.mod versions
```

---

## Answer 8: What Is the Go Module Proxy and How Does GOPROXY Work?

**One-sentence answer:** The Go module proxy is an HTTP caching server for module downloads; `GOPROXY` is a comma-separated list of proxy URLs the toolchain tries in order, enabling fast, reliable, and auditable dependency resolution.

**How resolution works:**

```
go get github.com/gin-gonic/gin@v1.9.1
         │
         ▼
GOPROXY=https://proxy.golang.org,direct

Step 1: Request to proxy.golang.org
  GET https://proxy.golang.org/github.com/gin-gonic/gin/@v/v1.9.1.info
  GET https://proxy.golang.org/github.com/gin-gonic/gin/@v/v1.9.1.mod
  GET https://proxy.golang.org/github.com/gin-gonic/gin/@v/v1.9.1.zip

Step 2: If proxy returns 404/410: try "direct"
  git clone https://github.com/gin-gonic/gin
  Extract module zip locally

Step 3: Verify against sum.golang.org (checksum database)
  GET https://sum.golang.org/lookup/github.com/gin-gonic/gin@v1.9.1
  Compare hash with go.sum entry
```

**Configuration:**

```bash
# Default: try Google's proxy, fall back to direct VCS
GOPROXY=https://proxy.golang.org,direct

# Direct only (no proxy — useful for fully private environments):
GOPROXY=direct

# Fail if not in proxy (no direct VCS access allowed):
GOPROXY=https://proxy.golang.org

# Corporate proxy with fallback:
GOPROXY=https://athens.corp.com,https://proxy.golang.org,direct

# Private modules bypass proxy (go direct to VCS):
GOPRIVATE=gitlab.myco.com/*,github.com/myco/*
# GOPRIVATE sets both GONOSUMDB and GONOPROXY
```

**Module proxy API:**
```bash
# List versions:
curl https://proxy.golang.org/github.com/gin-gonic/gin/@v/list

# Get module info:
curl https://proxy.golang.org/github.com/gin-gonic/gin/@v/v1.9.1.info

# Download module zip (what go uses):
curl https://proxy.golang.org/github.com/gin-gonic/gin/@v/v1.9.1.zip -o gin.zip
```
