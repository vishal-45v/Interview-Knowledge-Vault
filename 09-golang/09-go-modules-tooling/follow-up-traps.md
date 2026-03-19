# Go Modules & Tooling — Follow-Up Traps

---

## Trap 1: go mod tidy Removes Dependencies Needed at Runtime

**The trap:** "I ran `go mod tidy` and now my app fails at startup because a database driver isn't registered."

**The problem:** `go mod tidy` removes packages that aren't **statically** imported. But some packages are imported only for their side effects (registering drivers, codecs, etc.) using blank imports.

```go
// Before go mod tidy: you had this import
import _ "github.com/lib/pq"  // registers postgres driver with database/sql

// If you removed this import from code but kept using database/sql:
db, err := sql.Open("postgres", connStr) // panics: unknown driver "postgres"!

// go mod tidy also removed github.com/lib/pq from go.mod

// Fix: keep the blank import wherever you initialize the driver
// common pattern: put all side-effect imports in main.go or a dedicated init file
```

```go
// cmd/server/main.go — keep all registration imports here
import (
    _ "github.com/lib/pq"         // postgres driver
    _ "image/png"                  // PNG decoder
    _ "net/http/pprof"             // pprof HTTP handlers
    _ "go.uber.org/automaxprocs"   // set GOMAXPROCS from cgroup
)
```

Also relevant: reflection-based code that uses packages not directly imported:

```go
// go mod tidy removes "unused" package, but it's used via reflect or plugin
// Fix: add an explicit import even if the symbol isn't used directly
import _ "github.com/plugin/register"
```

---

## Trap 2: v2+ Modules MUST Change Import Path — v1 and v2 Can Coexist

**The trap:** "I published v2.0.0 of my module but users can't import it with my old import path."

**The problem:** Go treats major versions as different modules. Without the `/v2` suffix in the module path, the toolchain sees v2.0.0 as just another patch release.

```go
// WRONG: published v2.0.0 but go.mod still says:
module github.com/myco/mylib  // should be: github.com/myco/mylib/v2

// Consumers trying to import:
import "github.com/myco/mylib"  // they get v1 behavior even with v2.0.0 tag

// CORRECT v2 module structure:
// go.mod:
module github.com/myco/mylib/v2

// All internal imports must use the v2 path:
import "github.com/myco/mylib/v2/somepackage"

// Consumer code:
import "github.com/myco/mylib/v2"
// can also import v1 simultaneously:
import myv1 "github.com/myco/mylib"
import myv2 "github.com/myco/mylib/v2"
```

**The practical consequence:** If you never change the import path for breaking changes, users can't safely upgrade and downgrade independently. Both v1 and v2 will silently resolve to whatever go.mod says.

---

## Trap 3: go.sum Is a Security Mechanism — Never Skip It

**The trap:** "go.sum has conflicts in a PR merge. I'll just delete it and re-run `go mod tidy`."

**The problem:** `go.sum` contains cryptographic hashes of every dependency. The Go toolchain verifies these against the public `sum.golang.org` checksum database. Deleting and regenerating it doesn't lose functionality — but you should understand WHY it changed.

```bash
# If go.sum has unexpected changes, investigate BEFORE deleting:
git diff go.sum | head -30  # see which packages changed

# A legitimate change: you added a new dependency
# A suspicious change: a package you didn't change has a new hash
#   → possible supply chain attack or accidental version bump
```

```bash
# Verify all module hashes are correct:
go mod verify
# Output: all modules verified (or lists which ones have issues)

# If you must regenerate (e.g., recovering from corruption):
rm go.sum
go mod tidy  # re-fetches and re-verifies all hashes
```

Also: in CI, adding `go mod verify` as a step detects tampering:
```yaml
- run: go mod verify
- run: go mod tidy && git diff --exit-code go.mod go.sum
```

---

## Trap 4: replace Directive Only Works in Top-Level Module

**The trap:** "I added a `replace` in my library's go.mod to fix a dependency. My library users still get the unfixed dependency."

**The problem:** The `replace` directive in a non-root module is **ignored** by consumers. Only the top-level module (the one with the `go build` / `go test` entry point) can use `replace`.

```
# In github.com/myco/mylib's go.mod:
replace github.com/buggy/dep => github.com/fixed/dep v1.2.4-fix
# This has NO EFFECT for consumers of myco/mylib!

# The consumer must add their own replace:
# Consumer's go.mod:
replace github.com/buggy/dep => github.com/fixed/dep v1.2.4-fix
```

**Implications:**
- You cannot "bundle" a fix for your transitive dependencies for your library users
- Library maintainers should NOT use `replace` in published libraries
- For library users: you must add the `replace` yourself, or wait for the upstream to publish a fixed version

The correct fix: open an issue / PR upstream, or publish a fork and update your `require` to point to the fork.

---

## Trap 5: GOFLAGS for Persistent Build Flags

**The trap:** "I always need to pass `-mod=vendor` to every Go command. I keep forgetting."

**The solution:** `GOFLAGS` sets flags that apply to all `go` subcommands.

```bash
# Set in shell profile or CI env:
export GOFLAGS="-mod=vendor"

# Now these all use vendor automatically:
go build ./...   # equivalent to: go build -mod=vendor ./...
go test ./...    # equivalent to: go test -mod=vendor ./...
go vet ./...     # equivalent to: go vet -mod=vendor ./...

# Set multiple flags:
export GOFLAGS="-mod=vendor -count=1"

# Override for a single command:
GOFLAGS="" go get github.com/newdep@latest  # can't use vendor with go get
```

**Warning:** Some flags don't work with all subcommands. `-mod=vendor` breaks `go get`. Use carefully.

---

## Trap 6: CGO_ENABLED=0 Required for Static Binaries and Cross-Compilation

**The trap:** "My Go binary works on my machine but fails in a Docker container with 'exec format error' or 'cannot execute binary'."

**The problem:** By default, some Go stdlib packages (`net`, `os/user`) use cgo to call C library functions. The resulting binary is dynamically linked and requires `libc` on the target system. Alpine Linux uses musl libc, not glibc — compatibility issues arise.

```bash
# Check if binary is dynamically linked:
ldd ./myapp
# Bad: linux-vdso.so.1, libc.so.6, etc. (won't work in scratch container)
# Good: "not a dynamic executable"

# Fix: disable cgo
CGO_ENABLED=0 go build ./...

# OR: use pure Go implementations via tags
go build -tags "netgo osusergo" ./...  # pure Go network and user lookup

# For Docker from-scratch:
FROM scratch
# Requires: CGO_ENABLED=0 binary only!
COPY myapp /myapp
ENTRYPOINT ["/myapp"]
```

Also relevant: cross-compilation silently requires `CGO_ENABLED=0`:
```bash
# This FAILS silently if CGO is enabled and you don't have a cross-compiler:
GOOS=linux GOARCH=arm64 go build ./...
# "cannot find cgo compiler gcc for arm64"

# This works:
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./...
```

---

## Trap 7: init() Ordering Across Packages Is Alphabetical Within Files

**The trap:** "My package's init() depends on another package's init() having run. This randomly fails."

**The problem:** Within a single package:
- Files are processed alphabetically
- Each file's `init()` functions run in order of appearance

```go
// file: a_init.go
func init() {
    db = connectDB()  // runs first (a before b alphabetically)
}

// file: b_setup.go
func init() {
    setupCache(db)  // runs second — db is ready ✓
}
```

But **across packages**: Go guarantees that a package's `init()` runs after all its imported packages' `init()` functions have run. This is deterministic.

**The trap is relying on file-level init() ordering for complex initialization:**

```go
// DON'T do this: file ordering dependency is fragile
// init() in 01_config.go must run before init() in 02_server.go
// but "01_" prefix is not a reliable mechanism

// DO this instead: use a single init() or explicit setup in main()
func main() {
    cfg := loadConfig()
    db := initDB(cfg)
    srv := initServer(db)
    srv.Run()
}
```

---

## Trap 8: //go:build vs // +build — Old vs New Tag Syntax

**The trap:** "My //go:build tag is being ignored / I see both syntaxes and don't know which to use."

**History:**
- Before Go 1.17: `// +build` syntax (space-separated, comma for OR)
- Go 1.17+: `//go:build` syntax (Go expression syntax, more readable)
- Both syntaxes are supported for now; `gofmt` auto-adds the old one as a comment

```go
// OLD syntax (still valid, but deprecated style):
// +build linux darwin
// +build amd64

// NEW syntax (Go 1.17+, preferred):
//go:build (linux || darwin) && amd64

// IMPORTANT: The //go:build line must be IMMEDIATELY followed by a blank line,
// then the package declaration:

//go:build linux

package main  // CORRECT: blank line between tag and package

//go:build linux
package main  // WRONG: no blank line — tag is ignored!
```

**Auto-migration:**
```bash
gofmt -w .  # converts old // +build to //go:build style automatically
```

Starting Go 1.17, `go fix` and `gofmt` handle the migration. Use `//go:build` for all new code.

---

## Trap 9: go mod vendor Breaks go get and go mod tidy

**The trap:** "I run `go get github.com/new/dep` but nothing seems to change."

**The problem:** When `GOFLAGS=-mod=vendor` is set or when `-mod=vendor` is explicitly passed, `go get` doesn't update the vendor directory.

```bash
# If running with vendor mode:
export GOFLAGS=-mod=vendor
go get github.com/new/dep@latest
# Updates go.mod and go.sum BUT NOT vendor/

# vendor/ is now out of sync with go.mod!
# Build will succeed (using go.mod version) but vendor/ is stale

# Fix: after go get, update vendor:
GOFLAGS="" go mod vendor  # regenerate vendor from go.mod
# OR
go mod vendor             # always regenerate after go get

# CI check to ensure vendor is in sync:
go mod vendor && git diff --exit-code vendor/
```

---

## Trap 10: Module Path Must Match Repository URL Exactly

**The trap:** "I published my module as `github.com/me/mylib` but named it `mylib` in go.mod. Imports don't work."

**The problem:** The module path in `go.mod` is authoritative — it must match the import paths used in code AND the canonical repository URL.

```go
// go.mod:
module mylib  // WRONG! This is not importable from outside

// Consumer trying to import:
import "mylib/somepackage"  // only works if they have GOPATH set up just right

// go.mod correct:
module github.com/yourusername/mylib  // matches repository URL

// Consumer can now:
go get github.com/yourusername/mylib@latest
import "github.com/yourusername/mylib/somepackage"
```

For private modules, the path is still a URL-like string even if the server is internal:

```
module gitlab.mycompany.com/team/service  // internal GitLab

// Configure GOPRIVATE so the toolchain doesn't try to proxy it:
GOPRIVATE=gitlab.mycompany.com/*
```
