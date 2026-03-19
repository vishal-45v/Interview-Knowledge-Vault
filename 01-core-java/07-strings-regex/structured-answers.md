# Strings & Regex — Structured Interview Answers

---

## Q1: Why is String immutable in Java?

**Answer:** String immutability is a deliberate design decision with four primary motivations:

**1. Security:** Strings are used everywhere as security-sensitive data: file paths, class names for reflection, network hostnames, database credentials. If strings were mutable, code passed a filename for permission checking could have that filename changed by another thread before the actual file open. Immutability closes this race window.

```java
// If String were mutable, this would be a TOCTOU vulnerability:
void openFile(String path) {
    checkPermission(path);    // validates "/safe/file.txt"
    // malicious code mutates path to "/etc/shadow" between checks
    Files.readAllBytes(Path.of(path)); // now reads the wrong file
}
// With immutable String, this attack is impossible
```

**2. String pool / memory efficiency:** The JVM can safely cache strings in the pool because no code can alter a pooled string. If strings were mutable, two variables sharing the same pool reference would interfere with each other.

**3. Thread safety:** Immutable objects are inherently thread-safe. Any thread can read a `String` without synchronisation because the content can never change. This makes `String` a zero-cost shared-state object in concurrent programs.

**4. Hash code caching:** `String.hashCode()` is used constantly in `HashMap`, `HashSet`, and `Hashtable` as a key. `String` caches its hash code after the first computation. If strings were mutable, the hash code would have to be recomputed on every call, and keys stored in hash maps could "disappear" if the string was mutated after being used as a key.

```java
// hash code caching in action (simplified String source):
public int hashCode() {
    int h = hash; // cached field, 0 means not yet computed
    if (h == 0 && value.length > 0) {
        for (char c : value) {
            h = 31 * h + c;
        }
        hash = h; // cache it — safe because String never changes
    }
    return h;
}
```

**How immutability is enforced:**
- The internal `byte[]` (Java 9+ compact strings) or `char[]` (older) field is `private final`.
- No setter methods exist.
- The class is declared `final` — it cannot be subclassed (a subclass could add mutation).
- The internal array is never returned by reference; `getBytes()` returns a defensive copy.

---

## Q2: How does the String pool work? When is intern() useful?

**Answer:** The JVM maintains a string intern pool — a hash table of unique string objects, stored in the main heap (Java 7+; previously in PermGen). String literals and compile-time string constants are placed in the pool automatically by the classloader. At runtime, `intern()` explicitly queries the pool.

```java
// Literals and compile-time constants → automatic pool entry
String a = "Java";          // pool entry created (or reused)
String b = "Java";          // returns the same pool object as `a`
String c = "Ja" + "va";    // compile-time constant → same pool object
assert a == b;              // true
assert a == c;              // true

// Runtime strings → NOT pooled automatically
String d = new String("Java");        // new heap object
String e = "Ja".toUpperCase() + "va"; // "JAva" — wrong example; means runtime concat
String f = System.getProperty("lang"); // from external source

// Manual interning:
String g = d.intern(); // checks pool for "Java", returns existing pool reference
assert a == g;         // true — g is now the pool object
assert d != g;         // true — d is still the old heap object
```

**When `intern()` is genuinely useful:**

1. **Memory optimisation for repeated values from external sources:**
```java
// Reading 10M rows from DB where "country" has 200 distinct values
while (rs.next()) {
    // Without intern: 10M String objects for country field
    // With intern: ~200 String objects, 10M references pointing to pool
    String country = rs.getString("country").intern();
    records.add(new Record(country, rs.getString("city")));
}
```

2. **Canonical string comparison where == is required by a third-party API:** (rare, legacy code).

**When NOT to use `intern()`:**
- Dynamic strings (user inputs, UUIDs, session tokens) — they accumulate in the pool permanently (no GC), causing a memory leak.
- Modern JVMs offer `-XX:+UseStringDeduplication` (G1 GC) which does this automatically without code changes.

---

## Q3: Why use StringBuilder instead of String concatenation in a loop?

**Answer:** String objects are immutable. Every `+` concatenation creates a new `String` object, copying all existing characters. In a loop of N iterations building a string of final length L, the total characters copied is approximately L²/2 — O(n²) complexity. `StringBuilder` maintains a mutable internal buffer, growing by doubling when needed, giving O(n) total work.

```java
// SLOW: O(n²) — creates N intermediate String objects
String result = "";
for (int i = 0; i < N; i++) {
    result += words[i] + " "; // new String object every iteration
}

// FAST: O(n) — one StringBuilder, one final String
StringBuilder sb = new StringBuilder(N * 8); // pre-estimate capacity
for (int i = 0; i < N; i++) {
    sb.append(words[i]).append(' ');
}
String result = sb.toString();

// Benchmark (approximate, N=10,000 words of 5 chars each):
// String + in loop:   ~500ms
// StringBuilder:      ~1ms
// Ratio:              ~500x faster

// Alternative for simple joins: String.join() or Collectors.joining()
String joined = String.join(" ", words);                    // clean, readable
String joined2 = Arrays.stream(words)
        .collect(Collectors.joining(" ", "[", "]"));        // with prefix/suffix
```

**When the compiler optimises for you:**
```java
// Compiler does hoist StringBuilder for single-line multi-part concatenations:
String s = firstName + " " + lastName; // → new StringBuilder().append(firstName)
                                         //                      .append(" ")
                                         //                      .append(lastName)
                                         //                      .toString()

// But NOT for loop bodies — a new StringBuilder is created per iteration:
for (String word : words) {
    result += word; // each iteration: new StringBuilder(result).append(word).toString()
}
```

---

## Q4: What is catastrophic backtracking in regex and how do you prevent it?

**Answer:** Catastrophic backtracking (ReDoS) occurs when a regex with nested or ambiguous quantifiers explores an exponential number of matching paths before concluding no match exists. It can hang a thread indefinitely, causing a denial of service.

```java
// VULNERABLE patterns:
"(a+)+"             // outer + and inner + can match the same 'a' chars multiple ways
"(a|aa)+"           // 'a' and 'aa' overlap
"([a-zA-Z]+)*"      // any letter can match the group in multiple ways
"(.*a){20}"         // on long strings without 20 'a's: catastrophic

// Triggering input: near-match that forces maximum backtracking
Pattern p = Pattern.compile("(a+)+");
p.matcher("aaaaaaaaaaaaaaaaaaaaX").matches();
// With 20 'a's + non-matching X: tries 2^20 = 1,048,576 paths
// With 30 'a's: 2^30 = 1 billion paths → minutes/hours to complete

// PREVENTION STRATEGIES:

// 1. Possessive quantifiers — commit to match, no backtracking
Pattern safe1 = Pattern.compile("(a++)++"); // inner ++ is possessive
Pattern safe2 = Pattern.compile("[a-zA-Z]++"); // possessive, no backtrack

// 2. Atomic groups — once group matches, engine cannot try alternatives
Pattern safe3 = Pattern.compile("(?>(a+))+");

// 3. Rewrite to eliminate overlap
// Instead of: (a|ab)+  use: a(b)?|ab which has no overlap
// Instead of: (a+)+    use: a+  (matches same strings, no nesting needed)

// 4. Use RE2/J for untrusted input (linear time, no backtracking)
// Available as: com.google.re2j:re2j
import com.google.re2j.Pattern;
Pattern re2Pattern = Pattern.compile("(a+)+"); // RE2/J: always O(n)
re2Pattern.matcher("aaaaaaaaX").matches();     // completes immediately

// 5. Enforce timeout using Future (stopgap, not ideal)
ExecutorService exec = Executors.newSingleThreadExecutor();
Future<Boolean> f = exec.submit(
        () -> java.util.regex.Pattern.compile("(a+)+")
                .matcher(untrustedInput).matches());
try {
    return f.get(500, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
    f.cancel(true);
    throw new IllegalArgumentException("Regex timed out");
} finally {
    exec.shutdownNow();
}

// Testing for ReDoS vulnerability: use redos-detector tools or
// test with: "a" * N + "X" for increasing N — if time grows exponentially, it's vulnerable
```

---

## Q5: How do you efficiently split a large string?

**Answer:** The choice between splitting strategies depends on whether you need all parts upfront, can process them lazily, or need a high-performance tokenizer.

```java
// 1. String.split() — simple, returns String[] (all in memory)
String csv = "Alice,Bob,Charlie,Dave";
String[] parts = csv.split(",");
// Compiles the regex on every call — cache if called frequently
// Limit parameter prevents trailing empty strings trap:
"a,b,,c,".split(",")     // ["a", "b", "", "c"] — trailing empty stripped
"a,b,,c,".split(",", -1) // ["a", "b", "", "c", ""] — keep all, including trailing

// 2. StringTokenizer — legacy, fast, but inflexible (no regex, no empty tokens)
StringTokenizer st = new StringTokenizer(csv, ",");
while (st.hasMoreTokens()) {
    process(st.nextToken()); // streaming — doesn't allocate all parts upfront
}

// 3. Pattern.split() — compile once, reuse for high-frequency splitting
private static final Pattern COMMA = Pattern.compile(",");
String[] parts = COMMA.split(input); // no recompilation

// 4. Scanner — streaming, useful for large strings/files
Scanner scanner = new Scanner(largeString).useDelimiter(",");
while (scanner.hasNext()) {
    process(scanner.next()); // lazy — good for huge strings
}

// 5. indexOf loop — fastest for simple delimiters (no regex overhead)
int start = 0;
int comma;
while ((comma = csv.indexOf(',', start)) != -1) {
    process(csv.substring(start, comma));
    start = comma + 1;
}
process(csv.substring(start)); // last segment

// 6. Streams (Java 8+) — functional, lazy with splitAsStream
Pattern.compile(",")
       .splitAsStream(csv)              // lazy Stream<String>
       .map(String::trim)
       .filter(s -> !s.isEmpty())
       .forEach(this::process);

// Performance benchmark (1M splits of 100-char CSV):
// indexOf loop:           ~50ms   (fastest — no regex engine)
// Pattern.split (cached): ~120ms
// String.split():         ~400ms  (recompiles pattern each call)
// StringTokenizer:        ~80ms
```

---

## Q6: What is the difference between String.matches() and Pattern.matcher()?

**Answer:** Three key differences: anchoring behaviour, compilation caching, and API flexibility.

```java
// ANCHORING: String.matches() requires FULL string match; Pattern.find() searches within
String s = "2024-01-15 and more text";
String dateRegex = "\\d{4}-\\d{2}-\\d{2}";

s.matches(dateRegex);                             // false — doesn't span full string
s.matches(".*" + dateRegex + ".*");               // true — but wasteful

Pattern p = Pattern.compile(dateRegex);
p.matcher(s).find();                              // true — found within
p.matcher(s).matches();                           // false — same as String.matches()

// COMPILATION: String.matches() recompiles the pattern EVERY call
// For N calls with same pattern: N compilations
for (String line : millionLines) {
    line.matches("\\d+"); // compiles pattern 1,000,000 times — very slow
}
// Pattern compiled once:
Pattern digits = Pattern.compile("\\d+");
for (String line : millionLines) {
    digits.matcher(line).matches(); // compiled once, reused
}

// API FLEXIBILITY: Pattern.matcher() offers much more
Matcher m = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})").matcher(s);
if (m.find()) {
    String year  = m.group(1); // "2024"
    String month = m.group(2); // "01"
    String day   = m.group(3); // "15"
}

// Named groups (Java 7+):
Matcher m2 = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})")
                    .matcher(s);
if (m2.find()) {
    String year = m2.group("year");   // "2024"
}

// Find all matches in a string:
Matcher m3 = Pattern.compile("\\d+").matcher("abc 123 def 456");
List<String> numbers = new ArrayList<>();
while (m3.find()) {
    numbers.add(m3.group()); // ["123", "456"]
}

// Replace with computation:
String result = Pattern.compile("\\d+")
        .matcher("abc 5 def 10")
        .replaceAll(mr -> String.valueOf(Integer.parseInt(mr.group()) * 2));
// "abc 10 def 20"
```

---

## Q7: How do you handle Unicode and emoji in Java strings?

**Answer:** Java strings use UTF-16 internally. Most characters (the Basic Multilingual Plane, code points U+0000 to U+FFFF) fit in one `char`. Supplementary characters (code points U+10000 to U+10FFFF) — including most emoji — require two `char` values called a surrogate pair. The `codePointAt()` / `codePoints()` API handles this correctly; `charAt()` does not.

```java
String s = "Hello 😀!"; // 😀 is U+1F600, a supplementary character

// WRONG — charAt() breaks surrogate pairs
System.out.println(s.length());      // 9, not 8 — 😀 counts as 2 chars
System.out.println(s.charAt(6));     // \uD83D (first surrogate — garbage)
System.out.println(s.charAt(7));     // \uDE00 (second surrogate — garbage)

// CORRECT — codePointAt() returns the full Unicode code point
System.out.println(s.codePointAt(6)); // 128512 = 0x1F600 = 😀

// CORRECT iteration over code points
s.codePoints().forEach(cp -> {
    String character = new String(Character.toChars(cp));
    System.out.println(character + " U+" + Integer.toHexString(cp).toUpperCase());
});

// CORRECT character count
long charCount = s.codePoints().count(); // 8, not 9

// CORRECT string reversal
String reversed = new StringBuilder(s).reverse().toString();
// StringBuilder.reverse() handles surrogate pairs correctly since Java 1.5

// Normalization — critical for comparing strings with accented characters
// "é" can be represented as:
// - U+00E9 (single code point: LATIN SMALL LETTER E WITH ACUTE)
// - U+0065 U+0301 (two code points: 'e' + COMBINING ACUTE ACCENT)
// These look identical but are NOT equal with .equals()

import java.text.Normalizer;
String nfc1 = Normalizer.normalize("é", Normalizer.Form.NFC); // single code point
String nfc2 = Normalizer.normalize("é", Normalizer.Form.NFC); // same
nfc1.equals(nfc2); // true after normalization

// Regex with Unicode
Pattern.compile("\\p{L}+")          // matches Unicode letters (not just ASCII)
       .matcher("Héllo Wörld")
       .matches();                   // true

Pattern.compile("[\\u0000-\\u007F]+") // ASCII only
       .matcher("hello")
       .matches();                   // true

// String.format with Unicode
System.out.printf("%c%n", 0x1F600); // prints 😀 if terminal supports it
```

---

## Q8: What does String.intern() do and what are its risks in modern Java?

**Answer:** `intern()` returns the canonical representation of the string from the JVM's string pool. If the pool already contains a string equal to this string (by `equals()`), it returns the pool's string. Otherwise, it adds this string to the pool and returns it.

```java
// Basic mechanism
String a = new String("hello"); // heap object, not pooled
String b = a.intern();          // b is the pool reference
String c = "hello";             // literal — same pool reference as b

System.out.println(b == c);    // true
System.out.println(a == b);    // false — a is still the original heap object
System.out.println(a.equals(b)); // true

// Memory savings (when beneficial)
// Scenario: loading a large dataset where 1% of unique strings repeat 1000x
List<Employee> employees = new ArrayList<>(1_000_000);
try (CSVReader reader = new CSVReader(new FileReader("employees.csv"))) {
    String[] row;
    while ((row = reader.readNext()) != null) {
        // department has ~50 unique values but appears 1M times total
        String dept = row[2].intern(); // ~50 objects instead of 1M
        employees.add(new Employee(row[0], row[1], dept));
    }
}

// Risks in modern Java:
// 1. PERMANENT RETENTION — pool objects are never GC'd (strong references from pool)
//    Interning dynamic strings (UUIDs, tokens, user inputs) = permanent memory leak
String sessionToken = generateToken();
sessionToken.intern(); // DANGER: this token lives forever in the pool!

// 2. CONTENTION — the native intern pool has a lock; heavy use from many
//    threads causes contention and throughput degradation
IntStream.range(0, 1_000_000).parallel()
         .mapToObj(i -> ("key" + i).intern()) // high contention on pool lock
         .count();

// 3. UNPREDICTABLE POOL SIZE — JVM has internal limits on pool capacity
//    Tunable via -XX:StringTableSize=N (default ~60,013 in Java 8)
//    Larger table reduces collision chains, but uses more native memory

// Modern alternatives:
// Option 1: JVM flag (G1 GC only, automatic, no code changes)
// -XX:+UseStringDeduplication

// Option 2: Explicit dedup map in application code (more control)
private final Map<String, String> internCache = new ConcurrentHashMap<>();
String intern(String s) {
    return internCache.computeIfAbsent(s, k -> k);
}
// This cache CAN be cleared when no longer needed — unlike JVM pool
```

---

## Q9: How would you implement a simple template engine using regex?

**Answer:** A template engine replaces placeholders like `{{name}}` or `${variable}` in a template string with values from a map. `Matcher.appendReplacement()` and `appendTail()` enable this efficiently without splitting or multiple passes.

```java
import java.util.*;
import java.util.regex.*;

public class SimpleTemplateEngine {

    // Pattern matches {{variableName}} — name is letters, digits, dots, underscores
    private static final Pattern PLACEHOLDER =
            Pattern.compile("\\{\\{([a-zA-Z0-9_.]+)\\}\\}");

    /**
     * Replaces {{variable}} placeholders with values from the context map.
     * Supports nested keys via dot notation: {{user.name}}
     */
    public String render(String template, Map<String, Object> context) {
        Matcher matcher = PLACEHOLDER.matcher(template);
        StringBuilder result = new StringBuilder(template.length() + 64);

        while (matcher.find()) {
            String key = matcher.group(1);          // e.g. "user.name"
            Object value = resolveKey(key, context); // supports "user.name" → map lookup
            String replacement = value != null ? value.toString() : "{{" + key + "}}";
            matcher.appendReplacement(result,
                    Matcher.quoteReplacement(replacement)); // escape $ and \
        }
        matcher.appendTail(result); // append remaining text after last match

        return result.toString();
    }

    // Resolve dot-notation keys: "user.name" → context.get("user").get("name")
    @SuppressWarnings("unchecked")
    private Object resolveKey(String key, Map<String, Object> context) {
        String[] parts = key.split("\\.");
        Object current = context;
        for (String part : parts) {
            if (current instanceof Map) {
                current = ((Map<String, Object>) current).get(part);
            } else {
                return null; // key path doesn't exist
            }
        }
        return current;
    }

    public static void main(String[] args) {
        SimpleTemplateEngine engine = new SimpleTemplateEngine();

        Map<String, Object> user = new HashMap<>();
        user.put("name", "Alice");
        user.put("email", "alice@example.com");

        Map<String, Object> context = new HashMap<>();
        context.put("user", user);
        context.put("appName", "MyApp");
        context.put("count", 42);

        String template = """
                Welcome to {{appName}}, {{user.name}}!
                Your email: {{user.email}}
                You have {{count}} unread messages.
                Unknown key: {{missing}} (left as-is)
                """;

        System.out.println(engine.render(template, context));
        // Welcome to MyApp, Alice!
        // Your email: alice@example.com
        // You have 42 unread messages.
        // Unknown key: {{missing}} (left as-is)
    }
}

// Extension: conditional blocks using regex for {{#if condition}}...{{/if}}
// This requires a proper recursive parser — regex alone cannot handle nesting.
// For production use: FreeMarker, Thymeleaf, Mustache (jmustache), or Pebble.
```

**Performance note:** `Matcher.appendReplacement()` is much faster than calling `replaceAll()` or `replaceFirst()` in a loop because it processes the string in a single pass. Using `Matcher.quoteReplacement()` on the replacement string is critical — without it, dollar signs and backslashes in replacement values are interpreted as group references and escape sequences.

---

## Q10: What is the difference between greedy, reluctant, and possessive quantifiers in regex?

**Answer:** Quantifiers control how much input a regex element tries to consume when there are multiple ways to match.

**Greedy** (default) — matches as much as possible, then backtracks:
```java
Pattern greedy = Pattern.compile("<.+>");
Matcher m = greedy.matcher("<a>hello</a>");
m.find();
System.out.println(m.group()); // "<a>hello</a>" — consumed as much as possible
```

**Reluctant / Lazy** (add `?`) — matches as little as possible, then expands:
```java
Pattern lazy = Pattern.compile("<.+?>");
Matcher m = lazy.matcher("<a>hello</a>");
m.find();
System.out.println(m.group()); // "<a>" — stopped at first >

// Find all tags:
Matcher allTags = Pattern.compile("<.+?>").matcher("<a>hello</a><b>world</b>");
while (allTags.find()) {
    System.out.println(allTags.group()); // "<a>", "</a>", "<b>", "</b>"
}
```

**Possessive** (add `+`) — matches as much as possible, NO backtracking at all:
```java
Pattern possessive = Pattern.compile("<.++>");
Matcher m = possessive.matcher("<a>hello</a>");
m.find();
System.out.println(m.find()); // false! .++ consumed the final > too, then couldn't match >

// The key difference from greedy:
// Greedy:     tries max match, backtracks if overall match fails
// Possessive: tries max match, does NOT backtrack — either succeeds immediately or fails fast

// Performance comparison on long non-matching input:
String input = "a".repeat(30) + "X";

// Greedy version of catastrophic backtracking pattern:
long t1 = System.nanoTime();
Pattern.compile("(a+)+").matcher(input).matches(); // exponential time
long t2 = System.nanoTime();

// Possessive version: fails immediately
long t3 = System.nanoTime();
Pattern.compile("(a++)++").matcher(input).matches(); // O(n) time
long t4 = System.nanoTime();

System.out.println("Greedy: " + (t2-t1)/1_000_000 + "ms");
System.out.println("Possessive: " + (t4-t3)/1_000_000 + "ms");
// Greedy: potentially thousands of ms
// Possessive: 0ms

// Summary table:
// Quantifier  Symbol  Example  Behavior
// ──────────────────────────────────────────────────────────────
// Greedy      (none)  a+       max match, allows backtracking
// Reluctant   ?       a+?      min match, expands if needed
// Possessive  +       a++      max match, NO backtracking
//
// Greedy:     a+ on "aaab" → tries "aaa", engine backtracks if needed
// Reluctant:  a+? on "aaab" → tries "a", expands to "aa", "aaa" only if needed
// Possessive: a++ on "aaab" → grabs "aaa", refuses to give any back

// Practical rule of thumb:
// - Default to greedy
// - Use lazy (.+?) for HTML tag parsing or quoted string extraction
// - Use possessive or atomic groups when performance matters
//   and you know backtracking will never produce a valid match
```
