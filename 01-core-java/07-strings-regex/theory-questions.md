# Strings & Regex — Theory Questions

> 20 theory questions on Java String internals, immutability, StringBuilder, string pool, and regular expressions for Senior Java Backend Engineers.

---

## String Internals

**Q1. Why is String immutable in Java? What are the benefits?**

`String` is implemented with a `private final char[]` (or `byte[]` in Java 9+) and no mutator methods.

**Benefits of immutability:**
1. **Thread safety**: multiple threads can share String references without synchronization
2. **String pool/interning**: JVM can safely share instances in the heap (two references to "hello" can point to the same object)
3. **Security**: passwords, hostnames, class names passed to APIs can't be changed mid-execution
4. **HashMap key safety**: String's hashCode is stable — cached on first call
5. **Caching hashCode**: `String.hashCode()` computed once and cached in the `hash` field

**How immutability is maintained:**
- `private final char[]` — array can't be reassigned and array contents shouldn't be exposed
- No setter methods
- Class is `final` — can't be subclassed to expose array

---

**Q2. What is the String pool (string intern pool)?**

The String pool (also called the intern pool or constant pool) is a special heap area where Java stores unique String literals.

```java
String a = "hello";          // Goes into pool
String b = "hello";          // Returns same reference from pool
String c = new String("hello"); // Creates new heap object (NOT in pool)

System.out.println(a == b);  // true (same reference in pool)
System.out.println(a == c);  // false (c is on heap, not in pool)
System.out.println(a.equals(c)); // true (same content)

// Manually intern a string
String d = c.intern();       // Returns pool reference
System.out.println(a == d);  // true
```

**String literals** are automatically interned by the compiler (added to the pool at class loading time).

**`intern()`**: searches pool for equal string; returns pool reference if found, otherwise adds to pool and returns that reference.

**Java 7+**: String pool moved from PermGen to the regular heap (garbage collectable).

---

**Q3. What is the difference between `==` and `.equals()` for Strings?**

- `==` compares **references** (memory addresses)
- `.equals()` compares **content** (character by character)

```java
String a = "hello";
String b = "hello";
String c = new String("hello");
String d = "hel" + "lo";  // Compile-time constant, interned!
String e = "hel";
String f = e + "lo";       // Runtime concatenation, new heap object

System.out.println(a == b);  // true (same pool reference)
System.out.println(a == c);  // false (c is new object)
System.out.println(a == d);  // true (compiler optimizes constant)
System.out.println(a == f);  // false (runtime concatenation)

System.out.println(a.equals(c)); // true
System.out.println(a.equals(f)); // true
```

**Rule: always use `.equals()` for String content comparison** in production code. Never use `==` for strings unless you explicitly need reference equality (which is rare).

---

**Q4. How does Java 9+ change String storage internally?**

Before Java 9: Strings stored as `char[]` — each char takes 2 bytes (UTF-16).

Java 9 introduced **Compact Strings** (JEP 254):
- Strings that contain only Latin-1 characters (0-255) are stored as `byte[]` with 1 byte per character
- Strings with non-Latin-1 characters use UTF-16 encoding with 2 bytes per character
- A `coder` flag (`LATIN1 = 0` or `UTF16 = 1`) indicates which encoding is used

**Impact:** 25-50% memory reduction for typical English text applications. Transparent to the developer — no API changes.

---

**Q5. What is the difference between String, StringBuilder, and StringBuffer?**

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|---------------|--------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread safety | Thread-safe (immutable) | Not thread-safe | Thread-safe (synchronized) |
| Performance | Slow for repeated concat | Fast | Slower than StringBuilder |
| Use when | Final values, keys | Single-threaded concat | Multi-threaded scenarios |

```java
// String concatenation in loop — creates N intermediate objects!
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // Creates new String each time
}
// BAD — O(n²) copies

// StringBuilder — single buffer, amortized O(1) append
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // Internal char[] grows as needed
}
String result = sb.toString();
// GOOD — O(n) total

// Java compiler optimizes simple concatenation with +
String s = "Hello" + " " + "World";  // Compiled to: new StringBuilder().append(...)
// But NOT inside a loop — Java can't know when loop ends
```

---

**Q6. What does `String.format()` vs `String.formatted()` vs text blocks provide?**

```java
// String.format() — classic
String s1 = String.format("User %s has %d orders", username, count);

// String.formatted() — Java 15+, instance method
String s2 = "User %s has %d orders".formatted(username, count);

// Text blocks — Java 15+ (multi-line string literals)
String json = """
        {
            "name": "%s",
            "age": %d
        }
        """.formatted(name, age);

// Text blocks preserve relative indentation, strip leading whitespace
// The """ on last line determines baseline indentation
```

**Text block features:**
- Automatic removal of common leading whitespace
- `\` at end of line = line continuation (no newline)
- `\s` = trailing space preservation
- Very useful for SQL, JSON, HTML in tests

---

**Q7. What is `String.valueOf()` and how does it differ from `toString()`?**

```java
String.valueOf(null)    // Returns "null" (safe!)
null.toString()         // NullPointerException!

String.valueOf(42)      // "42"
String.valueOf(true)    // "true"
String.valueOf(3.14)    // "3.14"
String.valueOf(new char[]{'a','b'}) // "ab"

// In practice:
Object obj = null;
String s1 = String.valueOf(obj);   // "null" — safe
String s2 = "" + obj;             // "null" — also safe (compiler uses valueOf)
String s3 = obj.toString();        // NullPointerException!
```

---

## String Methods

**Q8. Describe key String methods and their time complexity.**

```java
String s = "Hello World";

// Length and access
s.length();          // O(1) — field
s.charAt(0);         // O(1) — array access
s.isEmpty();         // O(1)
s.isBlank();         // O(n) — must check each char for whitespace

// Searching
s.indexOf('o');      // O(n)
s.lastIndexOf('o');  // O(n)
s.contains("World"); // O(n) — uses indexOf internally
s.startsWith("Hel"); // O(n), but fast in practice
s.endsWith("rld");   // O(n)

// Transforming
s.toUpperCase();     // O(n) — new String
s.toLowerCase();     // O(n)
s.trim();            // O(n) — strips ASCII whitespace
s.strip();           // O(n) — strips Unicode whitespace (Java 11+, prefer this)
s.stripLeading();    // O(n)
s.stripTrailing();   // O(n)

// Splitting and substrings
s.substring(0, 5);   // O(1) in modern Java — shares backing array? Actually O(n) since Java 7u6 (new array copy for safety)
s.split(" ");        // O(n) — regex-based
s.split(" ", 2);     // O(n) — limit limits result count

// Java 11+ methods
s.repeat(3);         // O(n*k)
s.lines();           // Stream<String>
"  hi  ".strip();    // "hi"
```

---

**Q9. How does `String.split()` work and what are its gotchas?**

`String.split(regex)` uses regex, which means some characters need escaping:

```java
// Common gotchas
"a.b.c".split(".")    // WRONG! "." in regex = any char → ["", "", "", "", ""]
"a.b.c".split("\\.")  // Correct — escaped dot → ["a", "b", "c"]

"a|b|c".split("|")    // WRONG! "|" in regex = alternation → each char
"a|b|c".split("\\|")  // Correct → ["a", "b", "c"]

// Trailing empty strings are removed by default
"a,,b,,".split(",")    // ["a", "", "b"] — trailing empty strings removed!
"a,,b,,".split(",", -1) // ["a", "", "b", "", ""] — -1 limit: keep all

// For simple single-char splitting (no regex), use:
"a,b,c".split(",")  // OK for non-special chars
// Or for performance: use indexOf loop or String.valueOf() approaches
```

---

**Q10. What is `String.intern()` and when would you use it in production?**

`intern()` ensures you have the canonical (pooled) reference for a string's content.

**Performance use case:** When you have many duplicate strings from an external source (CSV rows, database results), interning can reduce memory:

```java
// Large dataset with repeated values (e.g., status fields)
record Order(String status, String customerId) {}

// Without interning: each row's "PENDING" string is a separate object
List<Order> orders = csvRows.stream()
    .map(row -> new Order(row[0], row[1]))  // 1M "PENDING" objects!
    .collect(toList());

// With interning: all "PENDING" strings share one reference
List<Order> orders = csvRows.stream()
    .map(row -> new Order(row[0].intern(), row[1]))  // 1 "PENDING" object
    .collect(toList());
```

**Downsides:**
- Intern pool is hash table — has lookup cost
- In modern Java, prefer using enums or dedicated type instead of String constants
- GC management is complex (Java 7+ made the pool heap-allocated, which helps)

**In practice:** Rarely needed explicitly. Use enums for known finite sets of values.

---

## Regular Expressions

**Q11. What are the key regex constructs to know for Java backend interviews?**

```java
// Character classes
[abc]      // a, b, or c
[^abc]     // NOT a, b, or c
[a-z]      // lowercase a to z
[0-9]      // digits (same as \d)
.          // any character except newline

// Predefined classes
\d         // digit [0-9]
\D         // non-digit [^0-9]
\w         // word char [a-zA-Z0-9_]
\W         // non-word
\s         // whitespace [ \t\n\r\f]
\S         // non-whitespace

// Quantifiers
a*         // 0 or more
a+         // 1 or more
a?         // 0 or 1
a{3}       // exactly 3
a{2,5}     // 2 to 5
a{2,}      // 2 or more
a*?        // 0 or more, LAZY (minimal match)
a+?        // 1 or more, LAZY

// Anchors
^          // start of line (or string in default mode)
$          // end of line
\b         // word boundary
\A         // start of input
\Z         // end of input

// Groups
(abc)      // capturing group
(?:abc)    // non-capturing group
(?<name>abc) // named group
abc|def    // alternation
```

---

**Q12. How do you compile and reuse regex patterns efficiently in Java?**

`Pattern.compile()` is expensive (builds state machine). Always reuse compiled patterns:

```java
// WRONG — compiles pattern on every call
public boolean isValidEmail(String email) {
    return email.matches("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    // matches() calls Pattern.compile() internally each time
}

// CORRECT — compile once, reuse
public class EmailValidator {
    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
    );

    public boolean isValidEmail(String email) {
        return EMAIL_PATTERN.matcher(email).matches();
    }
}

// Using groups
Pattern datePattern = Pattern.compile(
    "(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
Matcher m = datePattern.matcher("2024-03-15");
if (m.matches()) {
    String year = m.group("year");   // "2024"
    String month = m.group("month"); // "03"
    String day = m.group("day");     // "15"
}

// Replacing
String cleaned = Pattern.compile("\\s+").matcher(input).replaceAll(" ");
// Equivalent: input.replaceAll("\\s+", " ");  — but creates Pattern each time!
```

---

**Q13. What is catastrophic backtracking in regex and how do you prevent it?**

Catastrophic (exponential) backtracking occurs with certain regex patterns on specific inputs, causing the engine to try exponentially many paths.

**Classic example:**
```java
// Pattern: (a+)+ on input "aaaaaaaaaaab" (many a's, no match)
Pattern p = Pattern.compile("(a+)+b");  // Catastrophic!
p.matcher("aaaaaaaaaaaaaaab").matches();  // Exponential time!

// Why: (a+)+ allows engine to try every way to group the 'a' characters
// Then when 'b' doesn't match, backtracks and tries another grouping
// With n 'a' characters: 2^n possibilities to explore
```

**How to prevent:**
```java
// 1. Use atomic groups (?>...) — no backtracking inside
Pattern.compile("(?>(a+))+b");  // Atomic group — commits to first match

// 2. Possessive quantifiers a++ — greedy, no backtracking
Pattern.compile("(a++)++b");

// 3. Rewrite to eliminate ambiguity
Pattern.compile("a+b");  // The original pattern can be simplified!

// 4. Set timeout with Thread interruption
Thread thread = Thread.currentThread();
Timer timer = new Timer();
timer.schedule(new TimerTask() {
    public void run() { thread.interrupt(); }
}, 1000);  // 1 second limit
try {
    boolean match = pattern.matcher(input).matches();
} catch (InterruptedException e) {
    throw new RegexTimeoutException("Regex took too long");
}
```

**Input validation**: always validate input length before applying regex to user input.

---

**Q14. How do you use regex for string replacement and extraction in Java?**

```java
// Replace all occurrences
String result = "Hello   World".replaceAll("\\s+", " ");  // "Hello World"

// Replace with back-reference (reference group in replacement)
String masked = "4111-1111-1111-1234".replaceAll(
    "(\\d{4})-(\\d{4})-(\\d{4})-(\\d{4})",
    "****-****-****-$4");  // "****-****-****-1234"

// Extract all matches
Pattern emailPattern = Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
String text = "Contact alice@example.com or bob@test.org for help";
Matcher m = emailPattern.matcher(text);

List<String> emails = new ArrayList<>();
while (m.find()) {
    emails.add(m.group());  // Finds next match
}
// ["alice@example.com", "bob@test.org"]

// Java 9+: results() method
List<String> emails = emailPattern.matcher(text)
    .results()                         // Stream<MatchResult>
    .map(MatchResult::group)
    .collect(Collectors.toList());
```

---

**Q15. What is the difference between `matches()`, `find()`, and `lookingAt()` on a Matcher?**

```java
Pattern p = Pattern.compile("\\d+");
Matcher m1 = p.matcher("123abc");
Matcher m2 = p.matcher("abc123");
Matcher m3 = p.matcher("123");

m1.matches();    // false — pattern must match ENTIRE string
m1.find();       // true — finds pattern anywhere in string
m1.lookingAt();  // true — pattern must match at START of string

m2.matches();    // false
m2.find();       // true
m2.lookingAt();  // false — doesn't start with digits

m3.matches();    // true — entire string is digits
m3.find();       // true
m3.lookingAt();  // true
```

**Remember:**
- `matches()` → whole string
- `find()` → anywhere in string
- `lookingAt()` → from start of string

---

## String Performance

**Q16. How does Java optimize string concatenation with `+`?**

```java
// Simple constant concatenation — inlined by compiler:
String s = "Hello" + " " + "World";
// Becomes: String s = "Hello World"; (compile-time constant)

// Non-constant concatenation — Java compiler rewrites with StringBuilder:
String name = "World";
String s = "Hello " + name + "!";
// Becomes:
// new StringBuilder().append("Hello ").append(name).append("!").toString()

// BUT: in a loop, compiler creates new StringBuilder each iteration:
String result = "";
for (String item : list) {
    result = result + item;  // New StringBuilder per iteration — O(n²)
}
// Manually use StringBuilder outside the loop for O(n)
```

**Java 9+ optimization (JEP 280):** String concatenation now uses `invokedynamic` instead of StringBuilder — the JIT can optimize concatenation chains more aggressively.

---

**Q17. What are `String.join()`, `String.format()`, and `Collectors.joining()` — when to use each?**

```java
// String.join() — simple delimiter joining
String s = String.join(", ", "alice", "bob", "carol");  // "alice, bob, carol"
String s = String.join(", ", List.of("a", "b", "c"));   // "a, b, c"

// String.format() — template with positional/named slots (uses printf format)
String s = String.format("Order %s: %.2f USD", orderId, total);

// Collectors.joining() — for streams
String s = userIds.stream()
    .map(id -> id.toString())
    .collect(Collectors.joining(", ", "[", "]"));  // "[1, 2, 3]"
// Parameters: delimiter, prefix, suffix

// StringJoiner — the backing class for both join() and joining()
StringJoiner sj = new StringJoiner(", ", "{", "}");
sj.add("a").add("b").add("c");
sj.toString();  // "{a, b, c}"
sj.setEmptyValue("{}");  // Return this if nothing added
```

---

**Q18. How does `hashCode()` work for String? Why is it important?**

```java
// String.hashCode() implementation:
// s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
// Uses Horner's method: h = 31*h + charAt(i)

// Cached in the hash field (0 = not computed, which is fine since 0 is unlikely)
public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) {
        h = isLatin1() ? StringLatin1.hashCode(value)
                       : StringUTF16.hashCode(value);
        if (h == 0) {
            hashIsZero = true;
        } else {
            hash = h;
        }
    }
    return h;
}
```

**Why 31?** Powers of 2 as multiplier leads to bit loss. 31 is prime, and `31*i = (i << 5) - i` can be optimized by JIT to a shift+subtract instead of multiplication.

**Why it matters for HashMap:** String keys' hashCode must be consistent with equals() (which it is — same content = same hash). The cached hash makes HashMap lookups fast for String keys.

---

**Q19. What are the common performance pitfalls with String operations?**

```java
// 1. String concatenation in loops (already covered above)

// 2. Using String.split() for high-frequency hot path
// split() compiles a new Pattern each time if pattern length > 1 or has special chars
// For comma/colon/single-char splits, it's OK
// For patterns: cache the Pattern

// 3. Unnecessary substring creation
// Better: use indexOf + manual parsing instead of creating substrings for large strings

// 4. String.format() in hot path — creates new objects
// In tight loops, use StringBuilder directly
// Or use structured logging (SLF4J) which delays toString until needed:
log.debug("Order {}: {}", orderId, order);  // No toString if DEBUG disabled

// 5. toUpperCase/toLowerCase with default locale — avoid in sort/comparison
// Default locale can cause unexpected results (Turkish 'i' → 'İ')
s.toUpperCase(Locale.ROOT)    // Safe for comparisons
s.toLowerCase(Locale.ENGLISH) // Safe for ASCII

// 6. Creating strings for equality checks with constants
// WRONG: "PENDING".equals(status.toUpperCase())  -- creates intermediate String
// CORRECT: "PENDING".equalsIgnoreCase(status)
```

---

**Q20. How do you handle Unicode and character encoding issues in Java?**

```java
// Java Strings are internally Unicode (UTF-16)
// When reading/writing to external systems, specify encoding explicitly

// Reading file — always specify charset
List<String> lines = Files.readAllLines(Path.of("file.txt"), StandardCharsets.UTF_8);

// Bad — uses platform default encoding (varies by OS/locale)
new FileReader("file.txt")  // WRONG — uses Charset.defaultCharset()

// Good — specify encoding
new InputStreamReader(new FileInputStream("file.txt"), StandardCharsets.UTF_8)

// Convert between charsets
byte[] utf8Bytes = "Hello".getBytes(StandardCharsets.UTF_8);
String restored = new String(utf8Bytes, StandardCharsets.UTF_8);

// Check string length vs Unicode code points
String s = "Hello";
s.length();         // 5 — char count (UTF-16 code units)
s.codePointCount(0, s.length()); // 5 — Unicode code points

// Emoji/supplementary characters take 2 chars (surrogate pair)
String emoji = "😀";
emoji.length();     // 2 (two char code units in UTF-16!)
emoji.codePointCount(0, emoji.length()); // 1 (one Unicode code point)

// Iterate Unicode code points safely
emoji.codePoints()  // IntStream of code points
     .forEach(cp -> System.out.println(Character.toChars(cp)));

// String comparison ignoring case — use Locale-aware method for user-facing data
Collator collator = Collator.getInstance(Locale.GERMAN);
int result = collator.compare("ü", "u");  // Locale-aware comparison
```
