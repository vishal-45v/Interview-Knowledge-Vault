# Go Modules & Tooling — Diagram Explanations

---

## Diagram 1: Go Module Dependency Resolution

```
YOUR PROJECT: github.com/me/myservice
              go.mod requires:
              - gin v1.9.1
              - pgx v5.5.5

              go build ./...
                    │
                    ▼
         ┌──────────────────────┐
         │  Go Toolchain        │
         │  reads go.mod        │
         │  checks go.sum       │
         └──────────┬───────────┘
                    │
           For each required module:
                    │
         ┌──────────▼───────────┐
         │  Local Module Cache  │
         │  ~/.cache/go/mod/    │
         │                      │
         │  gin@v1.9.1 here? ──────── YES ──► use cached copy
         │                 NO           verify hash against go.sum
         └──────────┬───────────┘                    │
                    │                                 ▼
                    ▼                        ┌─────────────────┐
         GOPROXY chain:                      │ Build succeeds  │
                    │
         ┌──────────▼───────────────────────────────────┐
         │  1st: proxy.golang.org (or your corp proxy)  │
         │       GET /gin-gonic/gin/@v/v1.9.1.info      │
         │       GET /gin-gonic/gin/@v/v1.9.1.zip       │
         │                                              │
         │  Module found in proxy cache? YES ──────────────► download
         │                               NO            │
         └──────────────────────────────┬──────────────┘
                                        │
         ┌──────────────────────────────▼──────────────┐
         │  2nd: "direct" (VCS — github.com)           │
         │       git clone https://github.com/         │
         │              gin-gonic/gin                   │
         │       extract module zip                     │
         └──────────────────────────────┬──────────────┘
                                        │
         ┌──────────────────────────────▼──────────────┐
         │  Checksum verification                       │
         │  compute SHA-256 of downloaded zip           │
         │  compare with go.sum                         │
         │  also verify with sum.golang.org             │
         │                                              │
         │  Match? YES ──► cache in ~/.cache/go/mod/   │
         │         NO  ──► FATAL: checksum mismatch     │
         └──────────────────────────────────────────────┘

GOPROXY FALLBACK RULES:
  proxy1,proxy2,direct  → try each in order; next on any error
  proxy1|proxy2|direct  → try each in order; next on 404/410 only
  off                   → fail if not already in local cache
```

---

## Diagram 2: Multi-Stage Docker Build for Go

```
Dockerfile execution flow:

 ┌────────────────────────────────────────────────────────────────────┐
 │  STAGE 1: builder (golang:1.22-alpine)                             │
 │                                                                    │
 │  Layer 1: FROM golang:1.22-alpine                                  │
 │           Alpine Linux + Go toolchain (~250MB)                     │
 │           │                                                        │
 │  Layer 2: COPY go.mod go.sum ./                                    │
 │           RUN go mod download                                      │
 │           (cached unless go.mod/go.sum changed — fast on re-build)│
 │           │                                                        │
 │  Layer 3: COPY . .                                                 │
 │           (copies all source code)                                 │
 │           │                                                        │
 │  Layer 4: RUN CGO_ENABLED=0 go build -o server ./cmd/server       │
 │           Produces: /build/server (statically linked binary)       │
 │           Binary size: ~15MB (with -ldflags="-s -w")              │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘
                                │
                    COPY --from=builder /build/server /server
                                │
 ┌────────────────────────────────────────────────────────────────────┐
 │  STAGE 2: runtime (gcr.io/distroless/static-debian12)              │
 │                                                                    │
 │  Layer 1: FROM distroless/static-debian12:nonroot                  │
 │           Contents: CA certs, tzdata, no shell, no package manager│
 │           Base size: ~2MB                                          │
 │           │                                                        │
 │  Layer 2: COPY --from=builder /build/server /server               │
 │           Just the binary: +15MB                                   │
 │           │                                                        │
 │  FINAL IMAGE SIZE: ~17MB vs ~300MB (golang:alpine with all tools) │
 └────────────────────────────────────────────────────────────────────┘

LAYER CACHE OPTIMIZATION:
  ┌─────────────────────────────────────────────────────────────┐
  │  What changed?      │ Layers rebuilt                        │
  ├─────────────────────┼───────────────────────────────────────┤
  │  Source code only   │ COPY . . and RUN go build (fast!)     │
  │  go.mod/go.sum      │ go mod download + COPY + build        │
  │  Go version         │ Everything                            │
  └─────────────────────┴───────────────────────────────────────┘
```

---

## Diagram 3: GOPROXY Chain (direct → proxy → off)

```
GOPROXY=https://proxy.golang.org,direct

go get github.com/example/pkg@v1.2.3
              │
              ▼
     ┌────────────────┐
     │ Check local    │
     │ module cache   │──── FOUND ──────────────────────► Use it
     │ ~/.cache/go/   │
     └────────┬───────┘
              │ NOT FOUND
              ▼
     ┌────────────────────────────────────────────────────────┐
     │  proxy.golang.org                                      │
     │  GET /example/pkg/@v/v1.2.3.info                      │
     └────────────────────────────────────────────────────────┘
              │
    ┌─────────┴──────────┐
   200 OK             404 Not Found
    │                    │
    ▼                    ▼
Download ──────────  Try next in chain: "direct"
and cache             │
    │                  ▼
    ▼         ┌─────────────────────────────────────────────┐
 Verify       │  Direct VCS access                          │
 go.sum ──────│  git clone https://github.com/example/pkg   │
              │  extract module                              │
              └─────────────────────────────────────────────┘

COMMON CONFIGURATIONS:
┌───────────────────────────────────────────────────────────────────┐
│ Setting                 │ Behavior                                 │
├───────────────────────────────────────────────────────────────────┤
│ proxy.golang.org,direct │ Default: proxy first, then VCS          │
│ direct                  │ Always go to VCS (no proxy)              │
│ off                     │ Fail if not in local cache               │
│ corp-proxy,direct       │ Corporate proxy first, then public VCS   │
│ corp-proxy,proxy.golang.org,direct │ Try internal, then public, then direct │
└───────────────────────────────────────────────────────────────────┘

PRIVATE MODULE BYPASS (GOPRIVATE):
  GOPRIVATE=gitlab.myco.com/*

  This sets both:
  - GONOSUMDB=gitlab.myco.com/*  (skip sum.golang.org verification)
  - GONOPROXY=gitlab.myco.com/*  (skip proxy, go direct)

  So private modules always go directly to your internal git server.
```

---

## Diagram 4: Go Workspace (go.work) Layout

```
~/workspace/                 ← workspace root (has go.work)
├── go.work
│   ┌─────────────────────────────────────────────────┐
│   │ go 1.22                                         │
│   │                                                 │
│   │ use (                                           │
│   │     ./platform                                  │
│   │     ./service-a                                 │
│   │     ./service-b                                 │
│   │ )                                               │
│   └─────────────────────────────────────────────────┘
│
├── platform/                ← shared library module
│   ├── go.mod               module github.com/me/platform
│   ├── logger/
│   │   └── logger.go
│   └── config/
│       └── config.go
│
├── service-a/               ← microservice
│   ├── go.mod               module github.com/me/service-a
│   │   require github.com/me/platform v0.0.0   ← resolved by workspace!
│   └── main.go
│       import "github.com/me/platform/logger"  ← uses LOCAL platform/
│
└── service-b/               ← another microservice
    ├── go.mod               module github.com/me/service-b
    │   require github.com/me/platform v0.0.0
    └── main.go

RESOLUTION WITHOUT WORKSPACE (normal mode):
  service-a imports github.com/me/platform → fetches from proxy.golang.org
  (must be published and versioned)

RESOLUTION WITH WORKSPACE (go.work present):
  service-a imports github.com/me/platform → uses ./platform/ directly
  Changes to platform/ are IMMEDIATELY visible in service-a — no publish needed!

KEY COMMANDS:
  go work init ./platform ./service-a ./service-b   → create go.work
  go work use ./new-service                          → add module to workspace
  go work sync                                       → sync workspace module graph
  GOWORK=off go build ./...                          → disable workspace (use go.mod versions)

CI BEST PRACTICE:
  ┌─────────────────────────────────────────────────────────┐
  │ Development: use go.work (local sources, fast iteration) │
  │ CI/Production: GOWORK=off (use published, verified vers) │
  └─────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Cross-Compilation Targets (GOOS × GOARCH Matrix)

```
Cross-compilation: GOOS × GOARCH

                  GOARCH
         amd64   arm64   arm    386    mips64
        ┌──────┬───────┬──────┬──────┬────────┐
GOOS    │      │       │      │      │        │
linux   │  ✓   │   ✓   │  ✓   │  ✓   │   ✓    │  ← most common server targets
darwin  │  ✓   │   ✓   │      │      │        │  ← macOS Intel + Apple Silicon
windows │  ✓   │   ✓   │      │  ✓   │        │  ← Windows desktop/server
android │      │   ✓   │  ✓   │      │        │  ← Android apps (via gomobile)
ios     │      │   ✓   │      │      │        │  ← iOS apps (via gomobile)
plan9   │  ✓   │       │      │  ✓   │        │  ← Plan 9 OS
freebsd │  ✓   │   ✓   │      │  ✓   │        │  ← BSD servers
        └──────┴───────┴──────┴──────┴────────┘

See all: go tool dist list

COMMON PRODUCTION TARGETS:
┌──────────────────────────────────────────────────────────────────────┐
│ GOOS    GOARCH   │ Use case                                          │
├─────────────────────────────────────────────────────────────────────┤
│ linux   amd64    │ x86-64 servers, most cloud VMs, CI runners        │
│ linux   arm64    │ AWS Graviton, Azure Ampere, Raspberry Pi 4+       │
│ darwin  arm64    │ Apple Silicon (M1/M2/M3) developer machines       │
│ darwin  amd64    │ Intel Mac developer machines                      │
│ windows amd64    │ Windows servers, desktop apps                     │
└──────────────────────────────────────────────────────────────────────┘

BUILD COMMAND PATTERN:
  CGO_ENABLED=0 GOOS={target_os} GOARCH={target_arch} \
    go build -o bin/app-{target_os}-{target_arch} ./cmd/app

MULTI-PLATFORM RELEASE SCRIPT:
  ┌───────────────────────────────────────────────────────────────────┐
  │ for platform in                                                   │
  │     linux/amd64                                                   │
  │     linux/arm64                                                   │
  │     darwin/amd64                                                  │
  │     darwin/arm64                                                  │
  │     windows/amd64                                                 │
  │ do                                                                │
  │     GOOS=$(cut -d/ -f1 <<< $platform)                            │
  │     GOARCH=$(cut -d/ -f2 <<< $platform)                          │
  │     out="app-$GOOS-$GOARCH"                                      │
  │     [ "$GOOS" = "windows" ] && out="$out.exe"                    │
  │     CGO_ENABLED=0 GOOS=$GOOS GOARCH=$GOARCH \                    │
  │         go build -ldflags="-s -w" -o "bin/$out" ./cmd/app        │
  │ done                                                              │
  └───────────────────────────────────────────────────────────────────┘

CGO AND CROSS-COMPILATION:
  ┌──────────────────┬────────────────────────────────────────────────┐
  │ CGO_ENABLED=0    │ Pure Go: cross-compile to any target easily    │
  │ CGO_ENABLED=1    │ Needs C cross-compiler for each target         │
  │                  │ (e.g., aarch64-linux-musl-gcc for arm64/linux) │
  └──────────────────┴────────────────────────────────────────────────┘
```
