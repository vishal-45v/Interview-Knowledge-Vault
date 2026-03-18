# Go Modules & Tooling — Analogy Explanations

---

## go.mod: A Shopping List for Your Project

When you go grocery shopping, you write a list: "I need milk (any brand), eggs (dozen, large), bread (whole wheat)." The list captures your **intent** — what you need and roughly what kind.

`go.mod` is your project's shopping list:

```
module github.com/me/myservice  ← "This is my project"

go 1.22                         ← "My kitchen (Go version) can handle this"

require (
    github.com/gin-gonic/gin v1.9.1    ← "I need Gin framework, version 1.9.1"
    github.com/jackc/pgx/v5 v5.5.5    ← "I need pgx database driver v5"
)
```

The shopping list is committed to version control so everyone on the team buys the same things. Without the list, each developer might pick up different versions, causing "it works on my machine" problems.

`go mod tidy` is like auditing your shopping list after cooking: remove items you bought but didn't use, add items you needed but forgot to list.

---

## go.sum: A Checksum Receipt to Verify You Got the Right Items

After shopping, the store gives you a receipt with the weight of each item. When you get home, you can weigh your purchases and compare. If someone swapped your "organic milk" for a cheap fake on the way home, the weight won't match.

`go.sum` is that cryptographic receipt:

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU...=
                                     ↑
                         SHA-256 hash of the exact bytes in the package
```

When the Go toolchain downloads Gin v1.9.1, it computes the SHA-256 hash and checks it against `go.sum`. If someone replaced the code between when you first downloaded it and now (a supply chain attack), the hash won't match, and the build fails with:

> "verifying github.com/gin-gonic/gin@v1.9.1: checksum mismatch"

You would NOT delete the receipt just because reconciling it is annoying. Similarly, never delete `go.sum` without understanding why the hashes changed.

---

## Module Proxy: A Well-Stocked Local Store

Imagine the Go module ecosystem as a vast warehouse network (GitHub, GitLab, etc.) spread around the world. Every time you build, you'd need to fetch packages from warehouses potentially thousands of miles away — slow, unreliable, and dependent on those warehouses staying online.

The **module proxy** (like `proxy.golang.org`) is a well-stocked **local store**:
- It carries copies of the most popular warehouse items (Go modules)
- When you need a package, the store checks its shelves first (fast!)
- If the store doesn't have it, it fetches from the warehouse and caches it (so the next customer is served instantly)
- The original warehouse could go offline or the item could be discontinued — the store still has your copy

`GOPROXY=https://proxy.golang.org,direct` means: "Check the local store first. If not there, go directly to the warehouse."

For companies, running your own store (like Athens or Goproxy) means:
- You control which packages are available (compliance, security)
- You never depend on external internet during builds
- If `github.com` goes down, your CI still works

---

## Vendoring: Bringing All Your Tools With You

Imagine going on a camping trip. You have two choices:

**Without vendor:** Pack light. When you need a hammer, order it from Amazon and wait for delivery. Works great in civilization, terrible in the wilderness.

**With vendor:** Pack all your tools. The camping site has no internet, but you have everything you need in your bag. Heavier upfront, but completely self-sufficient.

`go mod vendor` packs all your dependencies into the `vendor/` directory:

```bash
go mod vendor
# Creates vendor/ with ALL your dependencies copied in

git add vendor/
git commit -m "vendor all dependencies"

# Now builds work with ZERO network access:
go build -mod=vendor ./...
```

The tradeoff:
- Repository gets larger (all that code lives in your repo)
- You can audit every line of third-party code
- Build is reproducible and fast even on air-gapped servers
- Updates require re-running `go mod vendor`

Large companies, especially in regulated industries (finance, healthcare), often require vendoring because all code in production must be audited and approved.

---

## Cross-Compilation: A Factory That Produces Goods for Different Countries

A factory in Japan makes electronics. They don't need to build separate factories in the US, Europe, and Australia to sell there — they configure their production line to produce products with the right power adapters, keyboard layouts, and voltage ratings for each market.

Go's cross-compilation is the same: your Mac laptop is the factory, and it can produce binaries for any target operating system and CPU architecture.

```bash
# Your Mac produces a binary for the Linux warehouse (server):
GOOS=linux GOARCH=amd64 go build ./...

# Same factory, configured for the Raspberry Pi market (ARM):
GOOS=linux GOARCH=arm64 go build ./...

# For the Windows distribution channel:
GOOS=windows GOARCH=amd64 go build ./...
```

The key requirement: `CGO_ENABLED=0` (disable cgo). When you use CGO, you're incorporating local C libraries — like trying to ship a Japanese-format power adapter to the US. You need a **cross-compiler** (a specialist adapter factory) if you must use CGO.

Pure Go code is like universal standards (USB-C) — it works everywhere without modification.

---

## go.work (Workspace): A Shared Workshop for Multiple Projects

Imagine you're building a piece of furniture that requires custom hinges. Normally:
- You order hinges from a supplier (published module version)
- Wait for delivery
- Test fit
- Discover they're the wrong size
- Send back, order again
- Repeat

With a shared workshop (Go workspace), you build the hinges yourself in the same workshop as the furniture:
- Any change to the hinge jig is immediately usable in the furniture project
- No shipping delays, no version numbers, no "let me publish v0.0.1-test and go get it"

```
~/workshop/
├── go.work          ← "These projects share this workshop"
├── furniture/       ← your main project
│   ├── go.mod
│   └── main.go      (imports "github.com/me/hinges")
└── hinges/          ← your shared library
    ├── go.mod
    └── hinge.go     (change here → immediately visible in furniture/)
```

When the hinges are done and ready to ship (publish to GitHub), you close the workshop arrangement and use the proper supply chain (real module versions). In CI, you use `GOWORK=off` to ensure the "production" build uses real published versions, not your in-progress workshop versions.
