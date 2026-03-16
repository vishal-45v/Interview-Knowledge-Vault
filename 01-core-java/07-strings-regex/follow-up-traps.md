# Strings & Regex — Follow-Up Traps & Tricky Interview Questions

---

## Trap 1: "Can you use == to compare Strings in Java?"

**What most people say:** No, always use `.equals()` for string comparison.

**Correct answer:** `==` compares object references (identity), not content. It gives correct results for string literals and compile-time constants (which are pooled), but wrong results for any runtime-constructed string. The subtle trap is that it sometimes appears to work.

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);      // true — both are pool references

String c = new String("hello");
System.out.println(a == c);      // false — c is a separate heap object

String d = "hel" + "lo";        // compile-time constant folding
System.out.println(a == d);      // true — compiler resolves to "hello" literal

String prefix = "hel";
String e = prefix + "lo";        // runtime concatenation
System.out.println(a == e);      // false — e is a new heap object

// The dangerous trap: works in unit tests but fails in production
// because unit test data is often literals, while production data
// comes from DB queries, HTTP requests, file reads — all heap objects
String fromDb = resultSet.getString("status");
String constant = "ACTIVE";
if (fromDb == constant) { // WRONG — always false in production!
    ...
}
// Always: if ("ACTIVE".equals(fromDb)) { ... }
// Using the literal as the receiver avoids NullPointerException too
```

---

## Trap 2: "Does new String(\"hello\") create a pool entry?"

**What most people say:** Yes, every string is pooled automatically.

**Correct answer:** `new String("hello")` explicitly creates a **new heap object** and bypasses the pool. It does NOT add to the pool. However, there is a subtlety: the literal `"hello"` inside the constructor call IS in the pool, but the resulting `String` object returned by `new String(...)` is a separate heap copy. Two calls to `new String("hello")` produce two different heap objects.

```java
String pooled = "hello";                    // goes into pool
String heap1  = new String("hello");        // heap object, not pooled
String heap2  = new String("hello");        // another heap object

pooled == heap1  // false
heap1  == heap2  // false
pooled.equals(heap1)  // true

// The "hello" literal used inside new String("hello") IS in the pool.
// The object returned is NOT.

// To move heap1 into the pool:
String fromPool = heap1.intern(); // returns pool reference
fromPool == pooled // true

// Practical implication: reading from ResultSet, HttpRequest, files
// all produce non-pooled strings — never use == on them
```

---

## Trap 3: "String immutability makes it more secure — is that always true?"

**What most people say:** Yes, immutability means strings are safe for passwords and credentials.

**Correct answer:** Immutability actually makes `String` **less** safe for sensitive data like passwords. Because strings are immutable, you cannot zero out the sensitive content after you are done. The string object stays in memory until GC runs, and even after GC the characters may linger in the heap. A heap dump or memory profiling tool can expose the password.

```java
// INSECURE: password as String
String password = "s3cr3tP@ss";
authenticate(password);
// Cannot zero it out — password sits in memory for unknown duration
// password = null just removes the reference; the "s3cr3tP@ss" object remains

// SECURE: password as char[]
char[] password = "s3cr3tP@ss".toCharArray();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0'); // zero out immediately after use
}
// The sensitive data is now gone from memory

// Java's own APIs follow this: javax.security.auth.Destroyable,
// KeyStore.getKey() returns char[] for passwords,
// JPasswordField.getPassword() returns char[] (not String)
// Console.readPassword() returns char[]
```

**Additional nuance:** Strings in the pool never get GC'd for the lifetime of the JVM, so an interned password would be especially dangerous. The standard recommendation is: never store passwords as `String`; use `char[]` and zero it out after use.

---

## Trap 4: "Is StringBuilder thread-safe if multiple threads only append to it?"

**What most people say:** StringBuilder is not thread-safe, so yes it is a problem.

**Correct answer:** Correct that `StringBuilder` is not thread-safe. Even concurrent `append()` calls can corrupt internal state because the operation `count` (length) increment and the char array write are not atomic. The result is undefined — you might get an `ArrayIndexOutOfBoundsException`, interleaved characters, or silently wrong output.

```java
// BROKEN: shared StringBuilder with concurrent appends
StringBuilder sb = new StringBuilder();
List<Runnable> tasks = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    int id = i;
    tasks.add(() -> sb.append("Item" + id + "\n"));
}
ExecutorService pool = Executors.newFixedThreadPool(10);
tasks.forEach(pool::submit);
pool.shutdown(); pool.awaitTermination(10, TimeUnit.SECONDS);
// Result: garbled, missing entries, possible exception

// CORRECT options:
// Option 1: StringBuffer (synchronized — slow)
StringBuffer safe = new StringBuffer();

// Option 2: separate StringBuilders per thread, combine at end
String result = IntStream.range(0, 100)
        .parallel()
        .mapToObj(i -> "Item" + i + "\n")
        .collect(Collectors.joining()); // Collectors.joining is thread-safe

// Option 3: ConcurrentLinkedQueue, then join
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
tasks.forEach(id -> queue.add("Item" + id));
String result2 = String.join("\n", queue);
```

---

## Trap 5: "What is catastrophic backtracking and can it crash your Java app?"

**What most people say:** It is a regex performance issue but nothing that would crash production.

**Correct answer:** Catastrophic backtracking (also called ReDoS — Regular Expression Denial of Service) can hang a thread indefinitely, effectively causing a denial of service. It occurs when a regex with nested quantifiers on patterns that can match the same characters in multiple ways encounters input that nearly matches but ultimately fails.

```java
// DANGEROUS regex: (a+)+ with a string of 'a's followed by something that won't match
Pattern p = Pattern.compile("(a+)+");
Matcher m = p.matcher("aaaaaaaaaaaaaaaaaaaaX"); // 20 a's + X
// This can take minutes to years depending on input length
// Thread blocks; your Spring thread pool exhausts; service hangs

// Why: the outer + and inner + create exponential backtracking paths
// For n 'a's: 2^n paths explored before giving up

// SAFE alternatives:
// 1. Possessive quantifiers — no backtracking at all
Pattern safe1 = Pattern.compile("(a++)++"); // possessive inner +
// If the possessive match fails, no backtracking into the inner group

// 2. Atomic groups
Pattern safe2 = Pattern.compile("(?>(a+))+"); // atomic outer group

// 3. Rewrite to eliminate ambiguity
Pattern safe3 = Pattern.compile("a+"); // matches the same strings without exponential paths

// 4. Timeout using Java 9+ re-interrupt trick (hack, not idiomatic)
// There is no built-in regex timeout in Java's java.util.regex
// For untrusted input, use a library like RE2/J (linear time, no backtracking)
// or wrap in a Future with timeout:
ExecutorService exec = Executors.newSingleThreadExecutor();
Future<Boolean> future = exec.submit(
        () -> Pattern.compile("(a+)+").matcher(input).matches());
try {
    boolean result = future.get(1, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true); // interrupt the regex thread
    throw new IllegalArgumentException("Regex timeout on input: " + input);
}
```

---

## Trap 6: "What does String.valueOf(null) return?"

**What most people say:** It throws a NullPointerException.

**Correct answer:** `String.valueOf(null)` is ambiguous and compiles differently depending on context. If the argument is typed as `Object`, it returns the string `"null"`. If the argument is a `char[]` typed as `null`, it throws `NullPointerException`.

```java
// Case 1: null as Object argument
Object obj = null;
String s1 = String.valueOf(obj);  // returns "null" (the 4-char string)
System.out.println(s1);           // prints: null
System.out.println(s1 == null);   // false — it is the string "null"

// Case 2: null literal — compiler resolves to char[] overload!
String s2 = String.valueOf(null); // COMPILE WARNING: chooses char[] overload
// At runtime: throws NullPointerException
// Because String.valueOf(char[]) calls new String(char[]) which fails on null

// Why: The compiler resolves overloads at compile time.
// String.valueOf(char[]) is more specific than String.valueOf(Object),
// so null literal picks the char[] overload.

// Safe pattern: always cast or check
String safe = String.valueOf((Object) null); // explicitly picks Object overload → "null"

// Common in logging:
log.info("Value: {}", someObject); // SLF4J handles null safely → prints "null"
// vs
log.info("Value: " + someObject);  // also safe — String + null = "null"
// but:
System.out.printf("Value: %s%n", String.valueOf(null)); // NPE if char[] overload chosen
```

---

## Trap 7: "Can you use charAt() to iterate over a string that contains emoji?"

**What most people say:** Yes, charAt() returns the character at a given index.

**Correct answer:** `charAt()` returns a `char` (a 16-bit UTF-16 code unit). Most emoji and many non-BMP Unicode characters require two `char` values (a surrogate pair) to represent a single logical character (code point). Using `charAt()` on such strings breaks them into meaningless surrogate halves.

```java
String emoji = "Hello 😀 World";
System.out.println(emoji.length()); // 14 — NOT 13! 😀 counts as 2 chars

// charAt() — WRONG for emoji/non-BMP
for (int i = 0; i < emoji.length(); i++) {
    char c = emoji.charAt(i);
    // 😀 is represented as two chars: '\uD83D' '\uDE00'
    // charAt() gives you each surrogate individually — not a meaningful character
}

// codePointAt() / codePoints() — CORRECT
emoji.codePoints().forEach(cp -> {
    String s = new String(Character.toChars(cp));
    System.out.println(s); // prints each logical character including 😀
});

// Correct string reversal (naive reversal breaks surrogates):
String broken = new StringBuilder(emoji).reverse().toString(); // may break emoji

String correct = emoji.codePoints()
        .mapToObj(cp -> new String(Character.toChars(cp)))
        .collect(StringBuilder::new, (sb, s) -> sb.insert(0, s), StringBuilder::append)
        .toString();

// Counting actual characters (code points):
int actualCharCount = (int) emoji.codePoints().count(); // 13, not 14
```

---

## Trap 8: "What bytecode does Java generate for String concatenation with + in a loop?"

**What most people say:** The compiler optimises it to use StringBuilder, so it is fine.

**Correct answer:** The compiler DOES use `StringBuilder` for simple one-liner concatenation, but inside a loop it creates a **new** `StringBuilder` on **every iteration**, defeating the purpose. The automatic optimisation does not hoist the `StringBuilder` out of the loop.

```java
// What you write:
String result = "";
for (int i = 0; i < 1000; i++) {
    result = result + i + ", ";
}

// What the compiler generates (approximately):
String result = "";
for (int i = 0; i < 1000; i++) {
    result = new StringBuilder()   // NEW StringBuilder every iteration!
                .append(result)    // copies entire accumulated string
                .append(i)
                .append(", ")
                .toString();       // creates new String object
}
// Iterations: 1000
// New StringBuilder objects created: 1000
// Total chars copied: 0 + 3 + 6 + 9 + ... ≈ O(n²)

// CORRECT — manual StringBuilder outside loop:
StringBuilder sb = new StringBuilder(6000); // pre-size
for (int i = 0; i < 1000; i++) {
    sb.append(i).append(", ");
}
String result = sb.toString();
// ONE StringBuilder, ONE final String — O(n)

// NOTE: Java 9+ uses invokedynamic + StringConcatFactory which is smarter
// for single-expression concatenation, but still creates a new StringBuilder
// per loop iteration in loop scenarios — the advice remains: use StringBuilder manually
```

---

## Trap 9: "Is String.intern() still useful in modern Java (Java 8+)?"

**What most people say:** intern() is obsolete because PermGen is gone in Java 8.

**Correct answer:** `intern()` still has legitimate uses but the tradeoffs changed significantly. In Java 8+, the string pool is in the main heap (GC-managed), eliminating PermGen overflow risk. `intern()` is still useful when you have many duplicate strings from external data sources (DB query results, JSON parsing) and you want to reduce heap pressure.

```java
// Still useful: large dataset with many repeated string values
// e.g. parsing a CSV with 1M rows and "country" column with ~200 distinct values

List<Record> records = new ArrayList<>();
while (csvReader.hasNextRow()) {
    String country = csvReader.getField("country").intern(); // dedup
    records.add(new Record(country, ...));
}
// Without intern(): 1M String objects for country
// With intern(): ~200 String objects for country, 1M references to them

// Performance nuance: intern() itself uses a native hash table
// Calling it in a tight loop has overhead — measure before adopting

// Modern alternative: explicit deduplication map (more controlled)
Map<String, String> dedup = new HashMap<>();
String country = dedup.computeIfAbsent(raw, k -> k); // cheaper than intern()

// Java 8u20+ JVM flag: -XX:+UseStringDeduplication
// Works with G1 GC; the JVM itself deduplicates identical strings
// during GC — no code changes required. Better than manual intern()
// for most applications.

// Risk: interned strings are never GC'd (pool holds strong refs)
// If you intern dynamically generated strings (user IDs, session tokens)
// you effectively have a memory leak
String userId = request.getHeader("X-User-Id");
userId.intern(); // DANGER: this user ID is now permanently in the pool!
```

---

## Trap 10: "Does String.matches() use the full input or search within it?"

**What most people say:** Both String.matches() and Pattern.matcher() do the same thing — find a match.

**Correct answer:** `String.matches(regex)` anchors the pattern to the **entire string** — it is equivalent to `Pattern.matches("^(?:regex)$", input)`. It returns true only if the entire string matches. `Pattern.matcher(input).find()` searches for the pattern **anywhere** within the string.

```java
String s = "hello world";

// String.matches() — requires FULL string to match
s.matches("hello");       // false — "hello" doesn't span the whole string
s.matches(".*hello.*");   // true — .* matches the rest
s.matches("hello world"); // true — exact full match

// Pattern.matcher().find() — searches within
Pattern.compile("hello").matcher(s).find();    // true — found at start
Pattern.compile("world").matcher(s).find();    // true — found at end
Pattern.compile("^hello$").matcher(s).find();  // false — anchors prevent it

// Pattern.matcher().matches() — same as String.matches() (full string)
Pattern.compile("hello").matcher(s).matches(); // false

// Performance trap: String.matches() compiles the pattern every call!
for (String line : lines) {
    line.matches("\\d{4}-\\d{2}-\\d{2}"); // recompiles pattern 1000x times!
}

// CORRECT: compile once, reuse
Pattern datePattern = Pattern.compile("\\d{4}-\\d{2}-\\d{2}");
for (String line : lines) {
    datePattern.matcher(line).matches(); // compiled once
}
```

---

## Trap 11: "Can String concatenation with + operator cause issues with null values?"

**What most people say:** It throws NullPointerException when you concatenate null.

**Correct answer:** Java's `+` operator for strings handles `null` gracefully — it converts null to the string `"null"`. This is sometimes helpful but often a silent bug.

```java
String name = null;
String greeting = "Hello, " + name; // does NOT throw NPE
System.out.println(greeting); // "Hello, null"

// The trap: silently writes "null" to databases, APIs, files
userRecord.setName("Hello, " + firstName); // if firstName is null → stores "Hello, null"

// String.format also converts null to "null":
String msg = String.format("User: %s", null); // "User: null"

// But some methods DO throw NPE with null:
"Hello".concat(null); // NullPointerException
null + "world";       // NullPointerException (left operand is null, not a String op)
// Wait — this is actually: (String)null + "world"
// In practice: null + "world" compiles to StringBuilder append of null → "nullworld"
// But: calling methods on null throws NPE:
// null.concat("world") → NullPointerException

// Safe null-handling patterns:
String safe = Objects.toString(name, "Unknown"); // returns "Unknown" if null
String safe2 = name != null ? name : "Unknown";
```

---

## Trap 12: "Are String operations safe for multithreaded access?"

**What most people say:** Yes, strings are immutable so they are inherently thread-safe.

**Correct answer:** The `String` objects themselves are thread-safe for reading because they are immutable. However, there are subtle traps around **references** to strings and lazy initialisation.

```java
// The String object itself: perfectly thread-safe to read concurrently
String shared = "Hello";
// Any number of threads can call shared.toUpperCase(), shared.charAt() etc.
// without synchronisation — they get new objects, never modify shared

// The TRAP: lazy-initialised String field without proper visibility
class Config {
    private String cachedValue = null; // not volatile

    public String getValue() {
        if (cachedValue == null) {      // thread A reads null
            cachedValue = compute();    // thread B may also compute
        }
        return cachedValue;
        // Two threads can both see null and both compute
        // This is a race condition on the REFERENCE, not the String
    }
}

// CORRECT: volatile guarantees visibility of the reference update
class Config {
    private volatile String cachedValue = null;

    public String getValue() {
        if (cachedValue == null) {
            synchronized (this) {
                if (cachedValue == null) { // double-checked locking
                    cachedValue = compute();
                }
            }
        }
        return cachedValue;
    }
}

// Another trap: StringBuilder shared between threads (different issue)
// StringBuilder is NOT thread-safe — see Trap 4
// String is immutable and thread-safe
// StringBuffer is thread-safe but rarely needed
```
