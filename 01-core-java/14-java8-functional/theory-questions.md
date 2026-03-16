# Java 8 Functional Programming — Theory Questions

> 32 theory questions covering lambdas, streams, Optional, functional interfaces, and method references.

---

## Q1: What is a lambda expression in Java?

**Answer:**

A lambda is an anonymous function — a concise way to represent a single-method interface (functional interface).

```java
// Syntax: (parameters) -> expression
//         (parameters) -> { statements; }

// No parameters:
Runnable r = () -> System.out.println("Hello");

// Single parameter (parens optional):
Consumer<String> print = s -> System.out.println(s);

// Multiple parameters:
Comparator<Integer> compare = (a, b) -> Integer.compare(a, b);

// Block body:
Function<String, Integer> parse = s -> {
    if (s == null) return -1;
    return Integer.parseInt(s);
};
```

**How lambdas work under the hood:** Java uses `invokedynamic` bytecode instruction (Java 7+) to create a class implementing the functional interface at runtime. This is more efficient than anonymous classes because the JVM can optimize the implementation strategy.

---

## Q2: What are the built-in functional interfaces in `java.util.function`?

**Answer:**

| Interface | Signature | Description |
|-----------|-----------|-------------|
| `Function<T,R>` | `R apply(T t)` | Maps T to R |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Maps T,U to R |
| `Consumer<T>` | `void accept(T t)` | Consumes T |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | Consumes T,U |
| `Supplier<T>` | `T get()` | Produces T |
| `Predicate<T>` | `boolean test(T t)` | Tests T |
| `BiPredicate<T,U>` | `boolean test(T t, U u)` | Tests T,U |
| `UnaryOperator<T>` | `T apply(T t)` | T → T |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | T,T → T |
| `Runnable` | `void run()` | No params, no return |
| `Callable<T>` | `T call()` | No params, returns T |

```java
Function<String, Integer> length = String::length;
Function<String, String> upper = String::toUpperCase;
Function<String, String> lengthThenUpper = length.andThen(i -> i + " chars").compose(upper);
// andThen: apply this, then apply andThen's function
// compose: apply compose's function first, then this

Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> notNull = Objects::nonNull;
Predicate<String> validString = notNull.and(notEmpty);   // &&
Predicate<String> emptyOrNull = notNull.negate().or(s -> s.isEmpty()); // !notNull || isEmpty
```

---

## Q3: What are method references and what are the four types?

**Answer:**

Method references are shorthand for lambdas that simply call a method.

```java
// Type 1 — Static method reference:  ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;
// Equivalent: s -> Integer.parseInt(s)

// Type 2 — Instance method of arbitrary instance:  ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;
// Equivalent: s -> s.toUpperCase()

// Type 3 — Instance method of specific instance:  instance::instanceMethod
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat;
// Equivalent: s -> prefix.concat(s)

// Type 4 — Constructor reference:  ClassName::new
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<Integer, ArrayList<String>> sizedList = ArrayList::new;  // ArrayList(int)
// Equivalent: () -> new ArrayList<String>()

// Common uses in streams:
List<String> names = users.stream()
    .map(User::getName)           // Type 2 — instance method
    .sorted(String::compareTo)    // Type 2 — used as Comparator
    .collect(Collectors.toList());

List<User> users = names.stream()
    .map(User::new)               // Type 4 — User(String name) constructor
    .collect(Collectors.toList());
```

---

## Q4: Explain the Stream API and its key operations.

**Answer:**

A Stream represents a sequence of elements supporting sequential and parallel aggregate operations. Streams are lazy — operations are only executed when a terminal operation is invoked.

**Stream pipeline: Source → Intermediate operations → Terminal operation**

```java
// Source: collection, array, Stream.of(), Stream.iterate(), Stream.generate()
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eve");

// Intermediate operations (lazy, return Stream):
names.stream()
    .filter(n -> n.length() > 3)          // filter — keep matching elements
    .map(String::toUpperCase)              // map — transform each element
    .sorted()                              // sorted — natural order
    .distinct()                            // deduplicate
    .limit(3)                              // take first N
    .skip(1)                               // skip first N
    .peek(n -> System.out.println(n))      // side effect without consuming
    .flatMap(n -> Arrays.stream(n.split("")))); // flatten nested streams

// Terminal operations (eager, produce result):
long count = stream.count();
Optional<String> first = stream.findFirst();
Optional<String> any = stream.findAny();   // parallel-friendly
boolean allMatch = stream.allMatch(predicate);
boolean anyMatch = stream.anyMatch(predicate);
boolean noneMatch = stream.noneMatch(predicate);
Optional<String> min = stream.min(Comparator.naturalOrder());
Optional<String> max = stream.max(Comparator.naturalOrder());
List<String> list = stream.collect(Collectors.toList());
String joined = stream.collect(Collectors.joining(", "));
Object[] arr = stream.toArray();
stream.forEach(System.out::println);
Optional<String> reduced = stream.reduce((a, b) -> a + "," + b);
String accumulated = stream.reduce("", (a, b) -> a + b);
```

---

## Q5: What is the difference between `map()` and `flatMap()`?

**Answer:**

**`map(f)`:** Applies function `f` to each element. Returns `Stream<R>` where R is the return type of f. Result is a 1-to-1 transformation.

**`flatMap(f)`:** Applies function `f` to each element where f returns a Stream. Then FLATTENS all those streams into one. 1-to-many transformation.

```java
// map — each element → one element:
List<String> names = Arrays.asList("Alice", "Bob");
List<String[]> letters = names.stream()
    .map(name -> name.split(""))    // each name → String[]
    .collect(Collectors.toList());
// Result: [["A","l","i","c","e"], ["B","o","b"]] — a list of arrays!

// flatMap — each element → many elements, then flatten:
List<String> allLetters = names.stream()
    .flatMap(name -> Arrays.stream(name.split("")))  // each name → stream of letters
    .collect(Collectors.toList());
// Result: ["A","l","i","c","e","B","o","b"] — flat list!

// Real-world example — flatten nested collections:
List<Order> orders = customer.getOrders();
List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .collect(Collectors.toList());

// flatMap with Optional:
Optional<String> userId = Optional.of("user123");
Optional<User> user = userId.flatMap(id -> userRepo.findById(id));
// vs map which would give Optional<Optional<User>>
```

---

## Q6: What is `Optional` and when should you use it?

**Answer:**

`Optional<T>` is a container that may or may not contain a non-null value. It's designed to avoid returning null from methods.

```java
// Creating Optional:
Optional<String> present = Optional.of("hello");     // must be non-null, else NPE
Optional<String> empty = Optional.empty();
Optional<String> nullable = Optional.ofNullable(maybeNull); // null-safe

// Consuming Optional:
optional.isPresent()           // true if has value
optional.isEmpty()             // true if no value (Java 11+)
optional.get()                 // returns value or throws NoSuchElementException
optional.orElse("default")     // value or default (always evaluates default)
optional.orElseGet(() -> computeDefault())  // lazy default (only computed if empty)
optional.orElseThrow()         // throws NoSuchElementException if empty (Java 10+)
optional.orElseThrow(() -> new RuntimeException("Not found"))
optional.ifPresent(v -> use(v))
optional.ifPresentOrElse(v -> use(v), () -> handleEmpty())  // Java 9+

// Transforming Optional:
optional.map(String::toUpperCase)          // transform if present
optional.flatMap(s -> findSomethingElse(s)) // flatMap avoids Optional<Optional<T>>
optional.filter(s -> s.length() > 3)       // empty if predicate fails

// Real-world usage:
public Optional<User> findByEmail(String email) {
    return userRepository.findByEmail(email);  // repository returns Optional
}

// Chaining:
String username = findByEmail("user@example.com")
    .map(User::getUsername)
    .map(String::toLowerCase)
    .orElse("anonymous");
```

**When NOT to use Optional:**
- As a field type (use null checks instead, Optional not Serializable)
- As a method parameter (use overloads or nullable parameter instead)
- For collections (use empty collections instead of Optional<List<...>>)

---

## Q7: What are default methods in functional interfaces?

**Answer:**

Default methods in functional interfaces allow composing and chaining functions.

```java
// Function composition:
Function<Integer, Integer> times2 = x -> x * 2;
Function<Integer, Integer> plus3 = x -> x + 3;

Function<Integer, Integer> times2ThenPlus3 = times2.andThen(plus3);
// f.andThen(g): g(f(x))
System.out.println(times2ThenPlus3.apply(5));  // (5*2)+3 = 13

Function<Integer, Integer> plus3ThenTimes2 = times2.compose(plus3);
// f.compose(g): f(g(x))
System.out.println(plus3ThenTimes2.apply(5));  // (5+3)*2 = 16

// Predicate chaining:
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> startsWithA = s -> s.startsWith("A");

Predicate<String> longAndStartsWithA = isLong.and(startsWithA);
Predicate<String> longOrStartsWithA = isLong.or(startsWithA);
Predicate<String> notLong = isLong.negate();

// Consumer chaining:
Consumer<String> log = System.out::println;
Consumer<String> save = this::saveToDb;
Consumer<String> logThenSave = log.andThen(save);
```

---

## Q8: What is the difference between `reduce()` and `collect()`?

**Answer:**

**`reduce()`:** Combines stream elements into a single value using a binary operator. Immutable reduction — creates new objects for each combination.

**`collect()`:** Mutable reduction — accumulates into a container (Collection, StringBuilder, etc.) for better performance.

```java
// reduce — creates intermediate results (less efficient for String building):
Optional<Integer> sum = numbers.stream().reduce((a, b) -> a + b);
Integer total = numbers.stream().reduce(0, Integer::sum);  // with identity

// collect — mutable accumulator (efficient):
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = stream.collect(Collectors.joining(", ", "[", "]"));

// Custom collector using Collector.of():
Collector<String, StringBuilder, String> stringCollector = Collector.of(
    StringBuilder::new,          // supplier
    StringBuilder::append,       // accumulator
    StringBuilder::append,       // combiner (for parallel)
    StringBuilder::toString      // finisher
);

// Summary statistics:
IntSummaryStatistics stats = numbers.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));
stats.getCount(); stats.getSum(); stats.getMin(); stats.getMax(); stats.getAverage();
```

---

## Q9: Explain `Collectors.groupingBy()` with examples.

**Answer:**

`groupingBy()` groups stream elements by a classifier function into a `Map<K, List<V>>`.

```java
// Simple grouping:
Map<String, List<Employee>> byDepartment = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// With downstream collector:
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

Map<String, Optional<Employee>> highestPaidByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));

// Multi-level grouping:
Map<String, Map<String, List<Employee>>> byDeptAndRole = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getRole)
    ));

// Transforming values:
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
```

---

## Q10: What is `Collectors.partitioningBy()`?

**Answer:**

`partitioningBy()` splits elements into two groups: true and false based on a Predicate.

```java
Map<Boolean, List<Integer>> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));

List<Integer> evens = partitioned.get(true);
List<Integer> odds = partitioned.get(false);

// With downstream:
Map<Boolean, Long> countEvenOdd = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0, Collectors.counting()));

// Practical use — split passing/failing students:
Map<Boolean, List<Student>> gradResults = students.stream()
    .collect(Collectors.partitioningBy(s -> s.getGrade() >= 60));
```

---

## Q11: How does parallel streams work?

**Answer:**

Parallel streams use the ForkJoin common pool to process elements concurrently, splitting the stream into sub-streams and merging results.

```java
// Sequential:
list.stream().filter(...).map(...).collect(Collectors.toList());

// Parallel:
list.parallelStream().filter(...).map(...).collect(Collectors.toList());
// or:
list.stream().parallel().filter(...).map(...).collect(Collectors.toList());

// When parallel is beneficial:
// 1. Large data (typically 10K+ elements)
// 2. CPU-intensive operations
// 3. Operations that are easily splittable (ArrayList is better than LinkedList)
// 4. No shared mutable state

// When parallel is harmful:
// 1. Small collections (overhead > benefit)
// 2. I/O-bound operations (would exhaust common pool threads)
// 3. Operations with shared mutable state (race conditions)
// 4. Ordered operations (findFirst is sequential anyway)
```

**Thread safety in parallel streams:**
```java
// UNSAFE — concurrent modification:
List<String> results = new ArrayList<>();
stream.parallel().forEach(results::add);  // race condition!

// SAFE — use collect():
List<String> results = stream.parallel()
    .collect(Collectors.toList());  // thread-safe aggregation
```

---

## Q12: What is the `Stream.generate()` and `Stream.iterate()` for infinite streams?

**Answer:**

```java
// Stream.generate() — produces infinite stream from Supplier:
Stream<Double> randoms = Stream.generate(Math::random);
Stream<String> constants = Stream.generate(() -> "hello");
Stream<UUID> uuids = Stream.generate(UUID::randomUUID);

// Must use limit or findFirst to terminate!
List<Double> tenRandoms = Stream.generate(Math::random).limit(10).collect(Collectors.toList());

// Stream.iterate() — produces infinite stream from seed + function:
Stream<Integer> naturals = Stream.iterate(1, n -> n + 1);  // 1, 2, 3, 4, ...
Stream<Integer> powers = Stream.iterate(1, n -> n * 2);    // 1, 2, 4, 8, 16, ...
Stream<LocalDate> dates = Stream.iterate(
    LocalDate.now(),
    date -> date.plusDays(1)   // infinite daily dates
);

// Java 9+ iterate with predicate (like a for loop):
Stream.iterate(1, n -> n <= 100, n -> n + 1)  // 1 to 100
    .forEach(System.out::println);

// Fibonacci sequence:
Stream.iterate(new int[]{0, 1}, arr -> new int[]{arr[1], arr[0] + arr[1]})
    .limit(10)
    .map(arr -> arr[0])
    .forEach(System.out::println);
```

---

## Q13: What are primitive streams and why do they exist?

**Answer:**

`IntStream`, `LongStream`, and `DoubleStream` avoid boxing/unboxing overhead for primitive operations.

```java
// Boxed stream — autoboxing overhead:
Stream<Integer> boxed = Stream.of(1, 2, 3, 4, 5);
int sum = boxed.reduce(0, Integer::sum);  // unboxing on every reduce step

// Primitive stream — no boxing:
IntStream primitives = IntStream.of(1, 2, 3, 4, 5);
int sum = primitives.sum();  // no boxing/unboxing

// Range generation:
IntStream.range(0, 10)         // 0 to 9 (exclusive end)
IntStream.rangeClosed(1, 10)   // 1 to 10 (inclusive end)

// Conversion:
IntStream intStream = list.stream().mapToInt(Integer::intValue);  // Stream<Integer> → IntStream
Stream<Integer> back = intStream.boxed();  // IntStream → Stream<Integer>

// Primitive stream operations:
IntStream.rangeClosed(1, 100).sum();        // 5050
IntStream.rangeClosed(1, 100).average();    // OptionalDouble(50.5)
IntStream.rangeClosed(1, 100).min();        // OptionalInt(1)
IntStream.rangeClosed(1, 100).max();        // OptionalInt(100)
IntStream.rangeClosed(1, 100).summaryStatistics();

// chars() on String returns IntStream:
"hello".chars()
    .filter(c -> c == 'l')
    .count();  // 2
```

---

## Q14: What is the difference between `Stream.of()` and `Arrays.stream()`?

**Answer:**

```java
// Arrays.stream() — creates stream directly from array:
int[] arr = {1, 2, 3};
IntStream intStream = Arrays.stream(arr);   // returns IntStream (primitive)
int sum = intStream.sum();  // no boxing

// Stream.of() — creates stream of the array as single element OR varargs:
Stream<int[]> wrongStream = Stream.of(arr);  // ONE element: the int[] array!
Stream<Integer> boxedStream = Stream.of(1, 2, 3);  // varargs — boxed

// Subarray:
IntStream sub = Arrays.stream(arr, 1, 3);  // elements at indices 1, 2

// For Object arrays, both work similarly:
String[] strings = {"a", "b", "c"};
Stream<String> s1 = Arrays.stream(strings);
Stream<String> s2 = Stream.of(strings);  // equivalent for Object arrays
```

---

## Q15: How do you handle checked exceptions in streams?

**Answer:**

Stream operations don't allow checked exceptions. Common workarounds:

```java
// Problem:
List<String> urls = Arrays.asList("http://a.com", "http://b.com");
urls.stream()
    .map(url -> new URL(url))  // MalformedURLException is checked!
    .collect(Collectors.toList());

// Approach 1 — wrap in RuntimeException:
urls.stream()
    .map(url -> {
        try { return new URL(url); }
        catch (MalformedURLException e) { throw new RuntimeException(e); }
    })
    .collect(Collectors.toList());

// Approach 2 — utility method:
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> f) {
        return t -> {
            try { return f.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
}

urls.stream()
    .map(ThrowingFunction.wrap(URL::new))
    .collect(Collectors.toList());

// Approach 3 — return Optional for partial processing:
urls.stream()
    .map(url -> {
        try { return Optional.of(new URL(url)); }
        catch (MalformedURLException e) { return Optional.<URL>empty(); }
    })
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

## Q16: What is `Collectors.toUnmodifiableList()` (Java 10+)?

**Answer:**

Returns an unmodifiable list, unlike `Collectors.toList()` which returns a modifiable `ArrayList`.

```java
// toList() — modifiable ArrayList:
List<String> modifiable = stream.collect(Collectors.toList());
modifiable.add("extra");  // allowed

// toUnmodifiableList() — unmodifiable:
List<String> immutable = stream.collect(Collectors.toUnmodifiableList());
immutable.add("extra");  // UnsupportedOperationException

// Java 16+ — Stream.toList():
List<String> immutable2 = stream.toList();  // shortest form, unmodifiable
```

---

## Q17: Explain `Stream.distinct()` and its performance implications.

**Answer:**

`distinct()` removes duplicate elements using `equals()` and `hashCode()`. For ordered streams, it maintains first-occurrence order.

```java
List<Integer> nums = Arrays.asList(1, 2, 1, 3, 2, 4);
List<Integer> unique = nums.stream().distinct().collect(Collectors.toList());
// [1, 2, 3, 4]

// Performance: maintains a HashSet internally — O(n) time, O(n) space
// For parallel streams, adds synchronization overhead

// Distinct by field (custom distinctBy):
public static <T> Predicate<T> distinctByKey(Function<? super T, ?> keyExtractor) {
    Set<Object> seen = ConcurrentHashMap.newKeySet();
    return t -> seen.add(keyExtractor.apply(t));
}

// Usage:
List<Employee> uniqueByDept = employees.stream()
    .filter(distinctByKey(Employee::getDepartment))
    .collect(Collectors.toList());
```

---

## Q18: What is `Comparator.comparing()` and its chaining?

**Answer:**

```java
// Natural order:
Comparator<String> natural = Comparator.naturalOrder();
Comparator<String> reversed = Comparator.reverseOrder();

// By field:
Comparator<Employee> bySalary = Comparator.comparingDouble(Employee::getSalary);
Comparator<Employee> byName = Comparator.comparing(Employee::getName);

// Chaining:
Comparator<Employee> complex = Comparator
    .comparing(Employee::getDepartment)       // first: department
    .thenComparingDouble(Employee::getSalary) // then: salary
    .thenComparing(Employee::getName);         // then: name (tie-break)

// Reverse:
Comparator<Employee> bySalaryDesc = Comparator
    .comparingDouble(Employee::getSalary).reversed();

// Null-safe:
Comparator<Employee> nullSafe = Comparator.comparing(
    Employee::getMiddleName,
    Comparator.nullsFirst(Comparator.naturalOrder())
);

// Usage:
employees.sort(complex);
Optional<Employee> highest = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary));
```

---

## Q19: What is the difference between `findFirst()` and `findAny()`?

**Answer:**

- **`findFirst()`:** Returns the first element in encounter order. Deterministic. For parallel streams, it may still need to coordinate to ensure first-in-order is returned, reducing parallelism benefit.
- **`findAny()`:** Returns any element (implementation choice). In parallel streams, returns the first element that completes, giving better performance.

```java
// Sequential — both equivalent:
Optional<String> first = list.stream().findFirst();
Optional<String> any = list.stream().findAny();

// Parallel — findAny is faster (no ordering constraint):
Optional<String> anyParallel = list.parallelStream()
    .filter(s -> s.length() > 3)
    .findAny();  // returns whichever thread finds first
```

---

## Q20: Explain `Stream.peek()` and when to use it.

**Answer:**

`peek()` performs a side effect on each element without consuming the stream. Designed for debugging.

```java
// Debugging:
list.stream()
    .filter(s -> s.length() > 3)
    .peek(s -> System.out.println("After filter: " + s))  // debug
    .map(String::toUpperCase)
    .peek(s -> System.out.println("After map: " + s))     // debug
    .collect(Collectors.toList());

// Warning: peek() is NOT guaranteed to run without a terminal operation
// Intermediate operations are lazy!
stream.peek(System.out::println);  // Nothing printed — no terminal!
stream.peek(System.out::println).count();  // Printed — terminal triggered

// Antipattern — don't use peek for side effects like persistence:
stream.peek(entity -> repo.save(entity)).collect(Collectors.toList());  // Bad!
// Use forEach() or map() for meaningful side effects
```

---

## Q21: What is `Collectors.toMap()` and common pitfalls?

**Answer:**

```java
// Basic toMap:
Map<Long, User> usersById = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

Map<String, String> emailToName = users.stream()
    .collect(Collectors.toMap(User::getEmail, User::getName));

// Pitfall 1 — duplicate keys throws IllegalStateException:
// If two users have same email, toMap throws!
Map<String, String> safe = users.stream()
    .collect(Collectors.toMap(
        User::getEmail,
        User::getName,
        (existing, newValue) -> existing  // merge function — keep first
    ));

// Pitfall 2 — null values throw NullPointerException:
// Collectors.toMap doesn't allow null values
// Workaround:
Map<String, Optional<String>> withNulls = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        u -> Optional.ofNullable(u.getOptionalField())
    ));

// Specific map type:
Map<Long, User> linkedMap = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        Function.identity(),
        (a, b) -> a,
        LinkedHashMap::new  // preserve insertion order
    ));
```

---

## Q22: What is `Stream.ofNullable()` (Java 9+)?

**Answer:**

Creates a stream of zero or one elements, depending on whether the value is null.

```java
// Java 9+:
Stream.ofNullable(null).count();     // 0
Stream.ofNullable("hello").count();  // 1

// Useful in flatMap to handle nullable values:
List<String> results = list.stream()
    .flatMap(item -> Stream.ofNullable(item.getNullableField()))
    .collect(Collectors.toList());
// Replaces: .filter(item -> item.getNullableField() != null).map(...)
```

---

## Q23: What is `Optional.stream()` (Java 9+)?

**Answer:**

`Optional.stream()` converts an Optional to a Stream of 0 or 1 elements. Useful for integrating Optional into stream pipelines.

```java
// Flatten a stream of Optionals:
List<Optional<User>> optionalUsers = getOptionalUsers();
List<User> users = optionalUsers.stream()
    .flatMap(Optional::stream)  // flatMap + Optional.stream() = compact
    .collect(Collectors.toList());

// Alternative (pre Java 9):
List<User> users = optionalUsers.stream()
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

## Q24: How do you use streams with records (Java 16+)?

**Answer:**

Records work naturally with streams as they're immutable data carriers:

```java
record Product(String name, double price, String category) {}

List<Product> products = List.of(
    new Product("Laptop", 999.99, "Electronics"),
    new Product("Phone", 699.99, "Electronics"),
    new Product("Book", 19.99, "Education")
);

// Group by category with statistics:
Map<String, DoubleSummaryStatistics> statsByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::category,
        Collectors.summarizingDouble(Product::price)
    ));

// Find most expensive per category:
Map<String, Optional<Product>> priciest = products.stream()
    .collect(Collectors.groupingBy(
        Product::category,
        Collectors.maxBy(Comparator.comparingDouble(Product::price))
    ));

// Transform to different record:
record ProductSummary(String name, String formattedPrice) {}
List<ProductSummary> summaries = products.stream()
    .map(p -> new ProductSummary(p.name(), String.format("$%.2f", p.price())))
    .collect(Collectors.toList());
```

---

## Q25: What is the `Collector` interface and how to create a custom one?

**Answer:**

```java
// Custom collector to create a custom summary:
Collector<Transaction, ?, TransactionSummary> summarizer = Collector.of(
    () -> new TransactionSummary(),           // Supplier<A> — creates accumulator
    (summary, t) -> summary.accept(t),         // BiConsumer<A,T> — accumulates
    (s1, s2) -> s1.combine(s2),                // BinaryOperator<A> — combiner for parallel
    summary -> summary.build(),                // Function<A,R> — finisher
    Collector.Characteristics.UNORDERED        // characteristics
);

// Usage:
TransactionSummary summary = transactions.stream().collect(summarizer);

class TransactionSummary {
    double total = 0;
    long count = 0;
    void accept(Transaction t) { total += t.getAmount(); count++; }
    TransactionSummary combine(TransactionSummary other) {
        this.total += other.total; this.count += other.count; return this;
    }
    TransactionSummary build() { return this; }
}
```

---

## Q26: What is the `stream().sorted()` behavior?

**Answer:**

```java
// Natural ordering:
list.stream().sorted();

// Custom ordering:
list.stream().sorted(Comparator.reverseOrder());

// Stable sort — equal elements maintain original order
// TimSort algorithm — O(n log n)

// Warning: sorted() on parallel stream requires collecting all elements
// first (can't sort in parallel), then sorts — reduces parallel benefit

// For sorted input, mark with:
list.stream()
    .sorted()
    .collect(Collectors.toList());
```

---

## Q27: What is a spliterator?

**Answer:**

`Spliterator` is the core interface behind streams. It can both iterate and split a source for parallel processing.

```java
// Manual use (rare):
Spliterator<String> spliterator = list.spliterator();
spliterator.forEachRemaining(System.out::println);

// trySplit for parallel processing:
Spliterator<String> first = spliterator.trySplit();  // split in two
// first and spliterator now each cover half the data

// Characteristics:
// ORDERED — elements have encounter order
// DISTINCT — no duplicates
// SORTED — elements are sorted
// SIZED — exact size known upfront
// NONNULL — no null elements
// IMMUTABLE — source won't change during traversal
// CONCURRENT — source can be modified concurrently

// Custom spliterator for custom data source:
public class DatabaseSpliterator<T> implements Spliterator<T> { ... }
StreamSupport.stream(new DatabaseSpliterator<>(query), false)  // false = sequential
    .collect(Collectors.toList());
```

---

## Q28: How do you aggregate data with `reduce()`?

**Answer:**

```java
// Sum:
int sum = IntStream.rangeClosed(1, 100).reduce(0, Integer::sum);

// Product:
int product = IntStream.rangeClosed(1, 5).reduce(1, (a, b) -> a * b);  // 120

// Maximum without Optional:
int max = numbers.stream().reduce(Integer.MIN_VALUE, Integer::max);

// Concatenation:
String concat = Stream.of("a", "b", "c").reduce("", String::concat);

// Three-arg reduce for parallel (combiner needed):
// 0: identity, Integer::sum: accumulator, Integer::sum: combiner
int total = numbers.parallelStream()
    .reduce(0, Integer::sum, Integer::sum);

// Building a complex object:
OrderSummary summary = orders.stream()
    .reduce(OrderSummary.empty(),
            OrderSummary::addOrder,
            OrderSummary::merge);
```

---

## Q29: What are short-circuit operations in streams?

**Answer:**

Short-circuit operations can terminate without processing all elements.

**Short-circuit intermediate operations:**
- `limit(n)` — stops after n elements
- `takeWhile(predicate)` — stops when predicate returns false (Java 9+)
- `dropWhile(predicate)` — skips while predicate is true (Java 9+)

**Short-circuit terminal operations:**
- `findFirst()`, `findAny()` — returns as soon as one found
- `anyMatch()` — returns true as soon as one matches
- `noneMatch()` — returns false as soon as one matches
- `allMatch()` — returns false as soon as one doesn't match

```java
// takeWhile (Java 9+) — like filter but stops at first false:
Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(n -> n < 4)
    .collect(Collectors.toList());  // [1, 2, 3] — stops at 4

// dropWhile (Java 9+) — drops while true, keeps rest:
Stream.of(1, 2, 3, 4, 5, 1, 2)
    .dropWhile(n -> n < 4)
    .collect(Collectors.toList());  // [4, 5, 1, 2] — drops 1,2,3
```

---

## Q30: What are the performance best practices for Stream API?

**Answer:**

```java
// 1. Order of operations matters — filter before map (reduce elements early):
stream.filter(x -> expensive(x)).map(x -> transform(x));  // good — filter first
stream.map(x -> transform(x)).filter(x -> expensive(x));  // bad — transforms all

// 2. Use primitive streams for numerics:
IntStream.range(0, n).sum();  // better than Stream<Integer>.reduce(0, Integer::sum)

// 3. Avoid sorted() in parallel streams (negates parallelism):
stream.parallel().sorted()...  // sorted forces sequential collection

// 4. Don't use parallel for small collections or I/O:
smallList.parallelStream()...  // overhead > benefit

// 5. Use collect() over reduce() for mutable accumulation:
stream.collect(Collectors.toList());  // good
stream.reduce(new ArrayList<>(), ...);  // bad — creates many intermediate lists

// 6. Stream once — streams cannot be reused:
Stream<String> s = list.stream();
s.forEach(System.out::println);
s.forEach(System.out::println);  // IllegalStateException: stream already operated upon

// 7. Use findFirst() for ordered, findAny() for parallel:
stream.parallel().findAny();  // faster than findFirst() in parallel

// 8. Avoid stateful lambdas:
List<Integer> filtered = new ArrayList<>();
stream.filter(x -> { filtered.add(x); return x > 0; });  // stateful lambda = bad
```

---

## Q31: What is `Stream.concat()`?

**Answer:**

```java
// Concatenate two streams:
Stream<String> first = Stream.of("a", "b", "c");
Stream<String> second = Stream.of("d", "e", "f");
Stream<String> combined = Stream.concat(first, second);
// ["a", "b", "c", "d", "e", "f"]

// Multiple concatenation — use flatMap instead:
List<Stream<String>> streams = List.of(stream1, stream2, stream3);
Stream<String> all = streams.stream().flatMap(Function.identity());

// Concat with empty:
Stream<String> withEmpty = Stream.concat(Stream.of("x"), Stream.empty());
// ["x"]

// Note: concat() with many streams creates deeply nested structures
// Prefer flatMap for combining 3+ streams
```

---

## Q32: What is `Optional.or()` (Java 9+)?

**Answer:**

`Optional.or(Supplier<Optional<T>>)` returns the Optional if non-empty, otherwise evaluates the supplier.

```java
// Optional.or() — provides fallback Optional:
Optional<User> user = primaryRepo.findById(id)
    .or(() -> backupRepo.findById(id))      // try backup if primary has none
    .or(() -> guestUserFactory.create());   // create guest if both empty

// vs orElseGet which provides fallback value (not Optional):
User user2 = primaryRepo.findById(id)
    .orElseGet(() -> guestUserFactory.create().get());

// Optional chaining:
String email = getUserById(id)
    .map(User::getEmail)
    .filter(e -> e.contains("@"))
    .map(String::toLowerCase)
    .orElse("unknown@example.com");
```
