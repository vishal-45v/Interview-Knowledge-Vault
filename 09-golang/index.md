# 09 — Golang Interview Preparation

## Purpose

This section prepares you for Go-specific interview questions at all levels — from junior developer positions to senior/staff engineer roles at companies that use Go for backend services, microservices, and infrastructure tooling. Coverage spans language fundamentals, idiomatic Go patterns, concurrency primitives, performance, testing, and modern features like generics.

Go interviews typically test:
- Language mechanics and how the runtime works under the hood
- Idiomatic code — Go favors composition over inheritance, explicit error handling, and simplicity
- Concurrency — goroutines, channels, and the sync package are a core differentiator
- System design using Go's strengths (microservices, CLI tools, high-throughput servers)

---

## Chapter Index

| # | Chapter | Description | Questions |
|---|---------|-------------|-----------|
| 01 | [Go Basics](./01-go-basics/) | Types, variables, slices, maps, pointers, functions, closures, defer, for loops, packages | 65+ |
| 02 | [OOP & Interfaces](./02-go-oop-interfaces/) | Structs, methods, interfaces, embedding, type assertions, duck typing, nil interface trap | 52+ |
| 03 | [Concurrency](./03-go-concurrency/) | Goroutines, channels, select, sync package, context, GMP scheduler, worker pools, race conditions | 77+ |
| 04 | [Error Handling](./04-go-error-handling/) | error interface, sentinel errors, error wrapping (errors.Is/As), custom error types, panic/recover | 45+ |
| 05 | [Standard Library](./05-go-standard-library/) | fmt, os, io, net/http, encoding/json, time, strings, strconv, bufio, sort | 40+ |
| 06 | [Testing](./06-go-testing/) | testing package, table-driven tests, benchmarks, testify, mocking, fuzz testing, test coverage | 38+ |
| 07 | [Performance & Memory](./07-go-performance-memory/) | Escape analysis, GC tuning, pprof, memory allocations, benchmarking, inlining, stack vs heap | 42+ |
| 08 | [Web & Microservices](./08-go-web-microservices/) | net/http, mux routers, middleware, REST APIs, gRPC, service patterns, graceful shutdown | 48+ |
| 09 | [Modules & Tooling](./09-go-modules-tooling/) | go mod, go.sum, versioning, build tags, go generate, linters, go vet, staticcheck | 32+ |
| 10 | [Generics](./10-go-generics/) | Type parameters, constraints, type sets, generic functions, generic types, when to use generics | 35+ |

---

## Recommended Study Order

### Beginner Track (0–1 years Go experience)
1. **Chapter 01** — Nail the fundamentals first. Slices, maps, closures, and defer are tested constantly.
2. **Chapter 02** — Interfaces are Go's most important design concept. The nil interface trap trips up most people.
3. **Chapter 04** — Error handling is idiomatic Go. Understand `errors.Is`, `errors.As`, and wrapping.
4. **Chapter 05** — Know the standard library basics: `fmt`, `os`, `net/http`, `encoding/json`.

### Intermediate Track (1–3 years Go experience)
5. **Chapter 03** — Concurrency is where Go shines and where most interviews go deep.
6. **Chapter 06** — Testing discipline is expected at mid-level. Table-driven tests and benchmarks.
7. **Chapter 09** — Modules and tooling are everyday work knowledge.

### Senior/Staff Track (3+ years)
8. **Chapter 07** — Performance, escape analysis, GC, pprof profiling.
9. **Chapter 08** — Web/microservice architecture patterns in Go.
10. **Chapter 10** — Generics are newer (Go 1.18+) but increasingly tested.

---

## How to Use This Section

Each chapter contains:
- **theory-questions.md** — Direct knowledge questions with expected answer depth
- **scenario-questions.md** — "How would you..." situational questions asked in design/coding rounds
- **follow-up-traps.md** — Counterintuitive edge cases interviewers use to separate candidates
- **structured-answers.md** — Full reference answers with working Go code
- **analogy-explanations.md** — Plain-English explanations for when you need to communicate concepts clearly
- **diagram-explanations.md** — ASCII diagrams for visual learners and whiteboard explanations

---

## Key Go Interview Themes Across All Chapters

| Theme | Appears In |
|-------|-----------|
| Implicit interface satisfaction | Ch 02, 03, 04, 08 |
| Goroutine leaks | Ch 03, 08 |
| Zero values and nil safety | Ch 01, 02, 04 |
| Composition over inheritance | Ch 02 |
| Explicit error handling | Ch 04, 08 |
| escape analysis / heap allocation | Ch 07 |
| Context propagation | Ch 03, 08 |
| Benchmarking and profiling | Ch 06, 07 |
