# Java 8 Functional — Structured Answers

---

## Q1: Stream Pipeline Evaluation — Lazy vs Eager

Stream operations are lazy — they don't execute until a terminal operation is called.

```java
// This does nothing:
Stream<String> stream = list.stream()
    .filter(s -> { System.out.println("filter: " + s); return s.length() > 3; })
    .map(s -> { System.out.println("map: " + s); return s.toUpperCase(); });
// No output yet — no terminal operation

stream.findFirst();  // Now filter and map execute, but only until first match!
// filter: first element (if match found, stops here)
```

Short-circuit evaluation: `findFirst()`, `findAny()`, `anyMatch()`, `limit()` can stop early.

---

## Q2: reduce() vs collect()

reduce(): Combines all elements into a single result using a BinaryOperator.
Best for: numerical aggregation, combining values.

```java
// reduce: sum
int sum = IntStream.of(1, 2, 3, 4, 5).reduce(0, Integer::sum);  // 15

// reduce: concatenate (inefficient — creates N intermediate strings)
String concat = Stream.of("a", "b", "c").reduce("", String::concat);
```

collect(): Mutable reduction — accumulates into a mutable container (List, Map, StringBuilder).
Best for: building collections, grouping, joining.

```java
// collect: build list
List<String> result = stream.collect(Collectors.toList());

// collect: join strings (efficient — one StringBuilder)
String joined = Stream.of("a", "b", "c").collect(Collectors.joining(", "));  // "a, b, c"
```

---

## Q3: Parallel Stream — When to Use

Parallel streams use ForkJoinPool.commonPool(). They're beneficial when:
1. Large data sets (thousands of elements)
2. Computationally expensive per-element operations (CPU-bound)
3. Operations are stateless and order-independent
4. No shared mutable state

Avoid parallel streams when:
1. Small data sets (overhead exceeds benefit)
2. I/O-bound operations (use async/reactive instead)
3. Order matters and you need findFirst()
4. Operations have side effects on shared state

---

## Q4: Optional Best Practices

Use Optional as return type for methods that might not return a value.
Don't use Optional.get() without isPresent() — defeats the purpose.
Prefer: map(), flatMap(), orElse(), orElseGet(), orElseThrow(), ifPresent().

```java
// Bad:
Optional<User> user = findUser(id);
if (user.isPresent()) {
    return user.get().getName();  // Old null-check pattern
}
return "Unknown";

// Good:
return findUser(id).map(User::getName).orElse("Unknown");

// Chaining:
return findUser(id)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("City unknown");
```
