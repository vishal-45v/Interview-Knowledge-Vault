# Java 8 Functional — Follow-up Trap Questions

---

## Trap 1: Stream Can Only Be Used Once

```java
Stream<String> stream = list.stream();
stream.filter(s -> s.startsWith("A")).count();  // OK
stream.filter(s -> s.length() > 3).count();     // IllegalStateException!
// Stream has already been consumed
```

---

## Trap 2: forEach vs map — forEach Has No Return Value

```java
// forEach: terminal operation, returns void, for side effects only
list.stream().forEach(s -> process(s));  // OK for side effects

// map: intermediate operation, transforms elements
list.stream().map(s -> transform(s)).collect(toList());  // OK

// WRONG: using map for side effects (ignoring the stream result)
list.stream().map(s -> { process(s); return s; });  // Stream not consumed — nothing happens!
```

---

## Trap 3: Checked Exceptions in Lambdas

```java
// Does not compile — IOException is checked
list.stream()
    .map(path -> Files.readAllBytes(path))  // Compilation error!
    .collect(toList());

// Fix: Wrap in unchecked
list.stream()
    .map(path -> {
        try {
            return Files.readAllBytes(path);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    })
    .collect(toList());
```

---

## Trap 4: Parallel Stream and Order

```java
// Parallel stream doesn't guarantee encounter order for unordered sources
List<Integer> nums = List.of(1, 2, 3, 4, 5);
nums.parallelStream()
    .forEachOrdered(System.out::println);  // Ordered ✓ but slower
    //.forEach(...)                         // May print in any order

// findFirst() vs findAny():
nums.parallelStream().findFirst()  // Returns first in encounter order (slower)
nums.parallelStream().findAny()    // Returns any (faster in parallel)
```

---

## Trap 5: Optional Is Not Serializable

Optional is not designed for class fields or method return types that need serialization. It's designed only as a return type for methods that might not return a value.

```java
class User {
    private Optional<String> middleName;  // BAD — not serializable, not recommended as field
}

// Optional as return type (good):
public Optional<User> findByEmail(String email) { ... }

// Optional as field (bad):
private Optional<String> middleName;  // Use nullable String or another approach
```

---

## Trap 6: Collectors.toMap() with Duplicate Keys

```java
List<User> users = ...;

// Throws IllegalStateException if two users have same email!
Map<String, User> byEmail = users.stream()
    .collect(Collectors.toMap(User::getEmail, u -> u));

// Fix: Provide merge function for duplicates
Map<String, User> byEmail = users.stream()
    .collect(Collectors.toMap(
        User::getEmail,
        u -> u,
        (u1, u2) -> u1  // Keep first on duplicate key
    ));
```

---

## Trap 7: Infinite Stream Without Limit

```java
Stream.iterate(0, n -> n + 1)  // Infinite stream
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());  // Hangs forever! Never terminates

// Must add limit() or findFirst() or takeWhile()
Stream.iterate(0, n -> n + 1)
    .filter(n -> n % 2 == 0)
    .limit(10)  // Take only first 10 even numbers
    .collect(Collectors.toList());  // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

---

## Trap 8: reduce() Identity Value

```java
// Correct identity for addition:
int sum = IntStream.range(1, 6).reduce(0, Integer::sum);  // 15

// Wrong identity for multiplication:
int product = IntStream.range(1, 6).reduce(0, (a, b) -> a * b);  // 0! (0 * 1 * 2 ... = 0)
// Correct:
int product = IntStream.range(1, 6).reduce(1, (a, b) -> a * b);  // 120

// Rule: Identity value must be the neutral element for the operation
// Addition: 0  |  Multiplication: 1  |  AND: true  |  OR: false  |  Concat: ""
```
