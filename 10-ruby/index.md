# Ruby Interview Mastery — Section Index

## Purpose

This section prepares backend engineers for Ruby technical interviews at all levels — from junior engineers answering "what is a symbol?" to senior engineers explaining the GVL, Ractor isolation, or metaclass hierarchies. Every chapter combines theory, real-world scenarios, traps, structured answers, analogies, and visual diagrams.

---

## Chapter Overview

| # | Chapter | Core Focus | Questions |
|---|---------|-----------|-----------|
| 01 | [Ruby Basics](./01-ruby-basics/) | Variables, types, strings, arrays, hashes, control flow, object fundamentals | 30+ theory, 20+ scenario |
| 02 | [OOP & Modules](./02-oop-modules/) | Classes, inheritance, mixins, method lookup chain, visibility, duck typing | 30+ theory, 15+ scenario |
| 03 | [Blocks, Procs & Lambdas](./03-blocks-procs-lambdas/) | Closures, yield, Proc vs Lambda, currying, &:symbol, lazy enumerators | 25+ theory, 15+ scenario |
| 04 | [Collections & Enumerable](./04-collections-enumerable/) | map/select/reduce, Enumerable, Set, Comparable, lazy evaluation | 25+ theory, 15+ scenario |
| 05 | [Concurrency & Threading](./05-concurrency-threading/) | GVL, Thread, Mutex, Fiber, Ractor, fork, race conditions | 25+ theory, 15+ scenario |
| 06 | [Metaprogramming](./06-metaprogramming/) | method_missing, define_method, open classes, DSLs, respond_to_missing? | 25+ theory, 15+ scenario |
| 07 | [Error Handling](./07-error-handling/) | raise/rescue/ensure/retry, custom exceptions, exception hierarchy | 20+ theory, 12+ scenario |
| 08 | [Ruby & Rails Patterns](./08-ruby-rails-patterns/) | Service objects, decorators, observers, presenters, concerns | 20+ theory, 15+ scenario |
| 09 | [Performance & Memory](./09-performance-memory/) | Object allocation, GC, frozen strings, lazy evaluation, profiling | 20+ theory, 12+ scenario |
| 10 | [Ruby Internals](./10-ruby-internals/) | MRI internals, VALUE type, YARV bytecode, object model, C extensions | 20+ theory, 10+ scenario |

---

## Study Tracks

### Beginner Track (Chapters 01 → 02 → 03)
Start here if you are new to Ruby or interviewing for junior roles. Focus on:
- Chapter 01: Understand Ruby's object model, every-type-is-an-object, nil/false falsy distinction
- Chapter 02: Master classes, modules, inheritance, and method lookup
- Chapter 03: Understand blocks and procs — Ruby code that takes blocks is everywhere

### Intermediate Track (Chapters 04 → 05 → 07)
For mid-level engineers. Focus on:
- Chapter 04: Enumerable mastery — map/select/reduce fluency is expected at every level
- Chapter 05: Concurrency basics — GVL, Thread, Mutex, Fiber
- Chapter 07: Error handling — rescue hierarchies, ensure, retry patterns

### Advanced Track (Chapters 06 → 08 → 09 → 10)
For senior/staff engineers. Focus on:
- Chapter 06: Metaprogramming — DSLs, open classes, define_method
- Chapter 08: Design patterns in Ruby/Rails context
- Chapter 09: Performance and memory profiling
- Chapter 10: Ruby internals — MRI, YARV, GC internals, VALUE type

---

## How Each Chapter Is Structured

Every chapter contains exactly 6 files:

| File | Purpose |
|------|---------|
| `theory-questions.md` | Direct technical questions with concise answers |
| `scenario-questions.md` | Real-world code scenarios requiring analysis |
| `follow-up-traps.md` | Gotchas, edge cases, and interviewer follow-up traps |
| `structured-answers.md` | Long-form STAR-style answers with code examples |
| `analogy-explanations.md` | Plain-English analogies for non-technical explanations |
| `diagram-explanations.md` | ASCII diagrams illustrating structures and flows |

---

## Quick Reference: Key Ruby Concepts by Chapter

```ruby
# Chapter 01 — The basics every Rubyist must know
:symbol == :symbol          # => true (same object_id)
"string" == "string"        # => true (same value, different objects)
nil.to_s   # => ""
nil.to_a   # => []
nil.to_i   # => 0
0 == false # => false   (0 is TRUTHY in Ruby!)

# Chapter 02 — Method lookup
Dog.ancestors
# => [Dog, Animal, Comparable, Object, Kernel, BasicObject]

# Chapter 03 — Lambda vs Proc
lam = lambda { |x| x * 2 }
prc = Proc.new { |x| x * 2 }
lam.lambda?  # => true
prc.lambda?  # => false

# Chapter 04 — Enumerable power
(1..10).lazy.select(&:odd?).map { |n| n ** 2 }.first(3)
# => [1, 9, 25]

# Chapter 05 — Concurrency
mutex = Mutex.new
mutex.synchronize { @counter += 1 }  # thread-safe increment
```
