# Java 8 Functional — Scenario Questions

> 30 scenario-based questions about applying functional programming in real Java applications.

---

## Scenario 1: Replacing Null Checks with Optional

**Context:**
```java
public String getUserCity(Long userId) {
    User user = userRepo.findById(userId);
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            City city = address.getCity();
            if (city != null) {
                return city.getName();
            }
        }
    }
    return "Unknown";
}
```
**Question:** Rewrite with Optional.

**Answer:**
```java
public String getUserCity(Long userId) {
    return userRepo.findById(userId)   // returns Optional<User>
        .map(User::getAddress)          // Optional<Address>
        .map(Address::getCity)          // Optional<City>
        .map(City::getName)             // Optional<String>
        .orElse("Unknown");
}
```

---

## Scenario 2: Parallel Stream Safety

**Context:**
```java
List<User> validUsers = new ArrayList<>();
users.parallelStream().forEach(user -> {
    if (validate(user)) validUsers.add(user);  // NOT thread-safe!
});
```
**Question:** Fix this.

**Answer:**
```java
// Fix 1 — use collect (thread-safe):
List<User> validUsers = users.parallelStream()
    .filter(this::validate)
    .collect(Collectors.toList());

// Fix 2 — synchronize manually (worse performance):
List<User> validUsers = Collections.synchronizedList(new ArrayList<>());
users.parallelStream().forEach(user -> {
    if (validate(user)) validUsers.add(user);
});

// Fix 1 is correct — never use non-thread-safe collections with parallelStream.forEach()
```

---

## Scenario 3: Complex Data Transformation Pipeline

**Context:** Transform a list of sales records into a report: filter last 30 days, group by region, sum revenue per region, sort by revenue descending.

**Answer:**
```java
LocalDate thirtyDaysAgo = LocalDate.now().minusDays(30);

Map<String, Double> revenueByRegion = salesRecords.stream()
    .filter(s -> s.getDate().isAfter(thirtyDaysAgo))
    .collect(Collectors.groupingBy(
        SaleRecord::getRegion,
        Collectors.summingDouble(SaleRecord::getRevenue)
    ));

List<Map.Entry<String, Double>> sortedRegions = revenueByRegion.entrySet().stream()
    .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
    .collect(Collectors.toList());
```

---

## Scenario 4: Function Composition for Validation Pipeline

**Context:** Create a composable validation pipeline for user input.

**Answer:**
```java
@FunctionalInterface
interface Validator<T> extends Function<T, Optional<String>> {
    static <T> Validator<T> of(Predicate<T> check, String errorMsg) {
        return t -> check.test(t) ? Optional.empty() : Optional.of(errorMsg);
    }

    default Validator<T> and(Validator<T> other) {
        return t -> {
            Optional<String> error = this.apply(t);
            return error.isPresent() ? error : other.apply(t);
        };
    }
}

Validator<String> notEmpty = Validator.of(s -> !s.isBlank(), "Cannot be empty");
Validator<String> validEmail = Validator.of(s -> s.contains("@"), "Invalid email");
Validator<String> correctLength = Validator.of(s -> s.length() <= 100, "Too long");

Validator<String> emailValidator = notEmpty.and(validEmail).and(correctLength);
Optional<String> error = emailValidator.apply(userInput);
```

---

## Scenario 5: Lazy Evaluation with Supplier

**Context:** An expensive default value should only be computed if needed.

**Answer:**
```java
// Bad — always computes expensive default:
String result = cache.get(key);
if (result == null) result = expensiveComputation(key);

// Good — Optional.orElseGet() with Supplier (lazy):
String result = cache.getOptional(key)
    .orElseGet(() -> expensiveComputation(key));  // only called if empty

// In log statements:
// Bad:
logger.debug("User: " + userService.buildDetailedReport(userId));  // always built

// Good — Supplier-based logging:
logger.debug("User: {}", () -> userService.buildDetailedReport(userId));  // lazy
```

---

## Scenario 6: Stream vs For Loop Decision

**Context:** A developer uses streams for everything. When should you prefer a for loop?

**Answer:**
```java
// Prefer stream when: transformation/aggregation, clear intent
List<String> emails = users.stream()
    .filter(User::isActive)
    .map(User::getEmail)
    .collect(Collectors.toList());

// Prefer for loop when:
// 1. Need to break/continue:
for (User user : users) {
    if (user.isAdmin()) break;  // can't break from forEach
    process(user);
}

// 2. Need index:
for (int i = 0; i < list.size(); i++) {
    list.set(i, transform(list.get(i)));  // can't do this with stream
}

// 3. Checked exceptions:
for (String line : lines) {
    try { process(line); } catch (IOException e) { handle(e); }
}

// 4. Accumulating state between iterations:
int runningTotal = 0;
for (int amount : amounts) {
    runningTotal += amount;
    if (runningTotal > budget) { alert(); break; }
}
```

---

## Scenario 7: Partitioning Batch Jobs

**Context:** Divide a large work list into batches for processing.

**Answer:**
```java
// Using IntStream with subList:
public <T> List<List<T>> partition(List<T> list, int batchSize) {
    return IntStream.range(0, (list.size() + batchSize - 1) / batchSize)
        .mapToObj(i -> list.subList(
            i * batchSize,
            Math.min((i + 1) * batchSize, list.size())
        ))
        .collect(Collectors.toList());
}

// Process each batch:
partition(users, 100).stream()
    .map(batch -> processBatch(batch))
    .collect(Collectors.toList());

// Parallel batches:
partition(users, 100).parallelStream()
    .forEach(batch -> processBatch(batch));
```

---

## Scenario 8: Collecting to Custom Result

**Context:** Build a transaction summary from a stream.

**Answer:**
```java
record TransactionSummary(BigDecimal total, long count, BigDecimal average, BigDecimal max) {}

TransactionSummary summary = transactions.stream()
    .map(Transaction::getAmount)
    .collect(Collectors.collectingAndThen(
        Collectors.summarizingDouble(BigDecimal::doubleValue),
        stats -> new TransactionSummary(
            BigDecimal.valueOf(stats.getSum()),
            stats.getCount(),
            BigDecimal.valueOf(stats.getAverage()),
            BigDecimal.valueOf(stats.getMax())
        )
    ));
```

---

## Scenario 9: Using flatMap to Flatten Errors

**Context:** Parse a list of strings as integers, collecting parse errors separately.

**Answer:**
```java
record ParseResult(Optional<Integer> value, Optional<String> error) {
    static ParseResult success(int value) { return new ParseResult(Optional.of(value), Optional.empty()); }
    static ParseResult failure(String error) { return new ParseResult(Optional.empty(), Optional.of(error)); }
}

List<String> inputs = List.of("1", "abc", "3", "xyz", "5");

List<ParseResult> results = inputs.stream()
    .map(s -> {
        try { return ParseResult.success(Integer.parseInt(s)); }
        catch (NumberFormatException e) { return ParseResult.failure("Invalid: " + s); }
    })
    .collect(Collectors.toList());

List<Integer> valid = results.stream()
    .flatMap(r -> r.value().stream())
    .collect(Collectors.toList());  // [1, 3, 5]

List<String> errors = results.stream()
    .flatMap(r -> r.error().stream())
    .collect(Collectors.toList());  // ["Invalid: abc", "Invalid: xyz"]
```

---

## Scenario 10: Method References for Cleaner Code

**Context:** Replace verbose lambdas with method references.

**Answer:**
```java
// Before:
list.stream()
    .map(s -> s.toUpperCase())
    .filter(s -> s.startsWith("A"))
    .sorted((a, b) -> a.compareTo(b))
    .forEach(s -> System.out.println(s));

// After:
list.stream()
    .map(String::toUpperCase)
    .filter(s -> s.startsWith("A"))  // lambda OK — has literal "A"
    .sorted(String::compareTo)
    .forEach(System.out::println);

// Constructor references:
List<StringBuilder> builders = strings.stream()
    .map(StringBuilder::new)  // StringBuilder(String) constructor
    .collect(Collectors.toList());

// With custom classes:
List<User> users = names.stream()
    .map(name -> new User(name))  // lambda
    .collect(Collectors.toList());
// If User has User(String name) constructor:
List<User> users2 = names.stream()
    .map(User::new)  // constructor reference
    .collect(Collectors.toList());
```

---

## Scenario 11: Memoization with Function

**Context:** Cache expensive function results.

**Answer:**
```java
public static <T, R> Function<T, R> memoize(Function<T, R> fn) {
    Map<T, R> cache = new ConcurrentHashMap<>();
    return key -> cache.computeIfAbsent(key, fn);
}

// Usage:
Function<String, UserDetails> lookup = memoize(userId -> userService.fetchDetails(userId));
UserDetails details1 = lookup.apply("user123");  // fetches from service
UserDetails details2 = lookup.apply("user123");  // returns from cache

// With expiration (using Caffeine):
LoadingCache<String, UserDetails> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(userId -> userService.fetchDetails(userId));
```

---

## Scenario 12: Pipeline with Multiple Transformations

**Context:** Process customer orders for a report: filter premium customers, get top 3 orders per customer by value, flatten into a single list.

**Answer:**
```java
List<Order> topOrdersFromPremium = customers.stream()
    .filter(Customer::isPremium)
    .flatMap(customer -> customer.getOrders().stream()
        .sorted(Comparator.comparingDouble(Order::getValue).reversed())
        .limit(3)
    )
    .sorted(Comparator.comparingDouble(Order::getValue).reversed())
    .collect(Collectors.toList());
```

---

## Scenario 13: Grouping with Collecting to Summary

**Context:** Create a category report showing count, average price, and total revenue.

**Answer:**
```java
record CategoryReport(String category, long count, double avgPrice, double totalRevenue) {}

Map<String, CategoryReport> report = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.collectingAndThen(
            Collectors.toList(),
            list -> {
                DoubleSummaryStatistics stats = list.stream()
                    .collect(Collectors.summarizingDouble(Product::getPrice));
                String category = list.get(0).getCategory();
                return new CategoryReport(
                    category,
                    stats.getCount(),
                    stats.getAverage(),
                    stats.getSum()
                );
            }
        )
    ));
```

---

## Scenario 14: Using Comparator.comparing() in Sort

**Context:** Sort employees by multiple criteria.

**Answer:**
```java
employees.sort(Comparator
    .comparing(Employee::getDepartment)
    .thenComparing(Employee::getLevel, Comparator.reverseOrder())  // senior first
    .thenComparingDouble(Employee::getSalary).reversed()           // highest salary first
    .thenComparing(Employee::getLastName)                          // alphabetical tie-break
);

// With null-safe:
employees.sort(Comparator.comparing(
    Employee::getMiddleName,
    Comparator.nullsLast(Comparator.naturalOrder())
));
```

---

## Scenario 15: Converting Imperative to Functional

**Context:**
```java
List<String> result = new ArrayList<>();
for (String word : words) {
    String upper = word.toUpperCase();
    if (upper.length() > 3) {
        result.add(upper);
    }
}
Collections.sort(result);
return result;
```
**Question:** Rewrite functionally.

**Answer:**
```java
return words.stream()
    .map(String::toUpperCase)
    .filter(w -> w.length() > 3)
    .sorted()
    .collect(Collectors.toList());
```

---

## Scenario 16: Handling Optional Chains

**Context:**
```java
Optional<String> optionalCity = user.getAddress()  // returns Optional<Address>
    .map(Address::getCity)                          // Optional<City>
    .map(City::getName);                            // Optional<String>

// What if Address::getCity returns Optional<City> (not City)?
```

**Answer:**
```java
// If getCity() returns Optional<City>:
Optional<String> optionalCity = user.getAddress()   // Optional<Address>
    .flatMap(Address::getCity)                        // Optional<City> — flatMap to avoid Optional<Optional<City>>
    .map(City::getName);                              // Optional<String>

// Rule: use map() when function returns a plain T
//       use flatMap() when function returns Optional<T>
```

---

## Scenario 17: Infinite Stream for ID Generation

**Context:** Generate sequential unique IDs.

**Answer:**
```java
public class IdGenerator {
    private final AtomicLong counter = new AtomicLong(0);

    public Stream<Long> generate() {
        return Stream.generate(counter::getAndIncrement);
    }

    public Long next() {
        return counter.getAndIncrement();
    }
}

// Take 5 IDs:
List<Long> ids = new IdGenerator().generate()
    .limit(5)
    .collect(Collectors.toList());

// With prefix:
Stream.iterate("ID-" + 1, id -> "ID-" + (Long.parseLong(id.substring(3)) + 1))
    .limit(10)
    .forEach(System.out::println);
```

---

## Scenario 18: Reduce for Building Domain Object

**Context:** Build an order total from line items with discounts.

**Answer:**
```java
BigDecimal orderTotal = lineItems.stream()
    .reduce(BigDecimal.ZERO,
            (total, item) -> total.add(item.getPrice().multiply(
                BigDecimal.ONE.subtract(item.getDiscountRate())
            ).multiply(BigDecimal.valueOf(item.getQuantity()))),
            BigDecimal::add  // combiner for parallel
    );

// More readable with intermediate stream:
BigDecimal orderTotal2 = lineItems.stream()
    .map(item -> item.getPrice()
        .multiply(BigDecimal.ONE.subtract(item.getDiscountRate()))
        .multiply(BigDecimal.valueOf(item.getQuantity())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

---

## Scenario 19: Stream with Checked Exception Wrapper

**Context:** Parse dates from strings in a stream, handling parse failures.

**Answer:**
```java
static <T, R> Function<T, Optional<R>> safely(Function<T, R> fn) {
    return t -> {
        try { return Optional.of(fn.apply(t)); }
        catch (Exception e) { return Optional.empty(); }
    };
}

List<LocalDate> dates = strings.stream()
    .map(safely(s -> LocalDate.parse(s, DateTimeFormatter.ISO_DATE)))
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

## Scenario 20: Functional Configuration Builder

**Context:** Build a service configuration using functional style.

**Answer:**
```java
@Builder
@Getter
public class ServiceConfig {
    private final String host;
    private final int port;
    private final Duration timeout;
    private final int maxConnections;
    private final boolean ssl;

    // Functional update (creates new instance):
    public ServiceConfig with(UnaryOperator<ServiceConfig.ServiceConfigBuilder> modifier) {
        return modifier.apply(toBuilder()).build();
    }
}

// Usage:
ServiceConfig base = ServiceConfig.builder()
    .host("localhost").port(8080).timeout(Duration.ofSeconds(5))
    .maxConnections(10).ssl(false).build();

// Create variants:
ServiceConfig prod = base.with(b -> b.host("prod.api.example.com").port(443).ssl(true));
ServiceConfig test = base.with(b -> b.maxConnections(2).timeout(Duration.ofSeconds(1)));
```

---

## Scenario 21: Collectors.joining() for Formatted Output

**Context:** Build a CSV line from a list of fields.

**Answer:**
```java
List<String> headers = List.of("id", "name", "email", "status");
String csvHeader = headers.stream()
    .collect(Collectors.joining(",", "", "\n"));

List<User> users = getUsers();
String csv = users.stream()
    .map(u -> Stream.of(u.getId().toString(), u.getName(), u.getEmail(), u.getStatus().name())
        .map(field -> "\"" + field.replace("\"", "\"\"") + "\"")  // CSV escape
        .collect(Collectors.joining(",")))
    .collect(Collectors.joining("\n", csvHeader, ""));
```

---

## Scenario 22: Stream Reuse Error

**Context:**
```java
Stream<String> stream = list.stream().filter(s -> s.length() > 3);
long count = stream.count();
List<String> filtered = stream.collect(Collectors.toList());  // Error!
```

**Question:** Why does this fail and how to fix it?

**Answer:**
Streams are single-use. Once a terminal operation (count()) is called, the stream is consumed and cannot be reused. Calling collect() on an already-consumed stream throws `IllegalStateException`.

**Fix:**
```java
// Option 1 — use a Supplier to create fresh streams:
Supplier<Stream<String>> streamSupplier = () -> list.stream().filter(s -> s.length() > 3);
long count = streamSupplier.get().count();
List<String> filtered = streamSupplier.get().collect(Collectors.toList());

// Option 2 — do both in one stream:
List<String> filtered = list.stream().filter(s -> s.length() > 3).collect(Collectors.toList());
long count = filtered.size();
```

---

## Scenario 23: GroupingBy with Custom Grouping Key

**Context:** Group transactions by week number.

**Answer:**
```java
Map<Integer, List<Transaction>> byWeek = transactions.stream()
    .collect(Collectors.groupingBy(
        t -> t.getDate().get(WeekFields.ISO.weekOfYear())
    ));

// Group by quarter:
Map<Integer, DoubleSummaryStatistics> statsByQuarter = transactions.stream()
    .collect(Collectors.groupingBy(
        t -> t.getDate().getMonthValue() / 4 + 1,  // rough quarter: months 1-3 = Q1, 4-6 = Q2...
        Collectors.summarizingDouble(Transaction::getAmount)
    ));
```

---

## Scenario 24: Using peek() for Debugging Without Side Effects

**Context:** Debug a stream pipeline in production without changing behavior.

**Answer:**
```java
@Slf4j
public List<ProcessedOrder> processOrders(List<Order> orders) {
    return orders.stream()
        .peek(o -> log.debug("Processing order: {}", o.getId()))
        .filter(this::isEligible)
        .peek(o -> log.debug("Eligible order: {}", o.getId()))
        .map(this::process)
        .peek(po -> log.debug("Processed: {}", po.getStatus()))
        .collect(Collectors.toList());
}
// peek() is perfect for debug logging — no side effects on the pipeline
// Can be compiled out in production by setting log level to INFO
```

---

## Scenario 25: Functional Error Handling with Either

**Context:** Return either a result or an error from stream operations.

**Answer:**
```java
// Simple Either type:
sealed interface Either<L, R> permits Either.Left, Either.Right {
    record Left<L, R>(L value) implements Either<L, R> {}
    record Right<L, R>(R value) implements Either<L, R> {}

    static <L, R> Either<L, R> left(L value) { return new Left<>(value); }
    static <L, R> Either<L, R> right(R value) { return new Right<>(value); }

    default boolean isRight() { return this instanceof Right; }
    default <U> Either<L, U> map(Function<R, U> fn) {
        return this instanceof Right<L, R> r ? right(fn.apply(r.value())) : (Either<L, U>) this;
    }
}

// Usage in streams:
List<Either<String, Integer>> results = strings.stream()
    .map(s -> {
        try { return Either.<String, Integer>right(Integer.parseInt(s)); }
        catch (NumberFormatException e) { return Either.<String, Integer>left("Invalid: " + s); }
    })
    .collect(Collectors.toList());

List<Integer> valid = results.stream()
    .filter(Either::isRight).map(e -> ((Either.Right<String, Integer>) e).value()).toList();
List<String> errors = results.stream()
    .filter(e -> !e.isRight()).map(e -> ((Either.Left<String, Integer>) e).value()).toList();
```

---

## Scenario 26: Stream with Database Cursor (Lazy Loading)

**Context:** Process database records lazily to avoid loading all into memory.

**Answer:**
```java
// Spring Data JPA with @QueryHints for streaming:
@Query("SELECT u FROM User u WHERE u.status = :status")
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "" + Integer.MIN_VALUE))
Stream<User> streamByStatus(@Param("status") UserStatus status);

// Usage — must close the stream after use:
@Transactional(readOnly = true)
public void processAllActiveUsers() {
    try (Stream<User> users = userRepository.streamByStatus(UserStatus.ACTIVE)) {
        users.forEach(user -> {
            processUser(user);
            entityManager.detach(user);  // prevent memory leak!
        });
    }
}
```

---

## Scenario 27: Comparing Java 8 Stream with Traditional Approach

**Context:** Find the 3 most common words (case-insensitive) in a text.

**Answer:**
```java
String text = "the quick brown fox jumps over the lazy dog the fox";

// Traditional approach:
Map<String, Integer> freq = new HashMap<>();
for (String word : text.toLowerCase().split("\\s+")) {
    freq.merge(word, 1, Integer::sum);
}
List<String> top3 = new ArrayList<>(freq.entrySet()).stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .limit(3)
    .map(Map.Entry::getKey)
    .collect(Collectors.toList());

// Stream approach (more concise):
List<String> top3Stream = Arrays.stream(text.toLowerCase().split("\\s+"))
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .limit(3)
    .map(Map.Entry::getKey)
    .collect(Collectors.toList());
```

---

## Scenario 28: Stateful vs Stateless Operations

**Context:** A developer uses a shared counter in a parallel stream.

```java
AtomicInteger counter = new AtomicInteger(0);
list.parallelStream().forEach(item -> {
    if (someCondition(item)) counter.incrementAndGet();
});
```

**Question:** Is this safe? Is it a good pattern?

**Answer:**
Using `AtomicInteger` in a parallel stream forEach is thread-safe but is a code smell. The correct functional approach is:

```java
// Count matching elements:
long count = list.parallelStream()
    .filter(this::someCondition)
    .count();  // terminal operation designed for counting

// Or with a side-effect counter:
long count = list.parallelStream()
    .mapToInt(item -> someCondition(item) ? 1 : 0)
    .sum();  // uses LongAdder-like optimization internally

// The original AtomicInteger approach works but is imperative thinking
// forced into functional style — use collect/reduce/count instead
```

---

## Scenario 29: Building a Retry Mechanism Functionally

**Context:** Retry a function up to N times on failure.

**Answer:**
```java
public static <T> T withRetry(Supplier<T> fn, int maxAttempts) {
    return IntStream.range(0, maxAttempts)
        .mapToObj(attempt -> {
            try { return Optional.of(fn.get()); }
            catch (Exception e) {
                if (attempt == maxAttempts - 1) throw new RuntimeException("Max retries exceeded", e);
                return Optional.<T>empty();
            }
        })
        .filter(Optional::isPresent)
        .findFirst()
        .flatMap(Function.identity())
        .orElseThrow(() -> new RuntimeException("All retries failed"));
}

// Usage:
String result = withRetry(() -> unstableService.call(), 3);
```

---

## Scenario 30: Functional Pipeline with Multiple Outputs

**Context:** Process orders — count, sum, and collect by status in a single pass.

**Answer:**
```java
// Single-pass aggregation to avoid multiple stream traversals:
record OrderStats(long count, BigDecimal total, Map<OrderStatus, List<Order>> byStatus) {}

OrderStats stats = orders.stream()
    .collect(Collectors.teeing(
        Collectors.counting(),                          // collector 1
        Collectors.groupingBy(Order::getStatus),        // collector 2
        (count, byStatus) -> {
            BigDecimal total = byStatus.values().stream()
                .flatMap(Collection::stream)
                .map(Order::getTotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            return new OrderStats(count, total, byStatus);
        }
    ));
// Collectors.teeing (Java 12+) — run two collectors in a single pass
```
