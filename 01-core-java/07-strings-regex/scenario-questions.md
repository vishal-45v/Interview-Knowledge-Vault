# Strings & Regex — Scenario Questions

> 18 real-world scenarios on String manipulation, regex patterns, and text processing in Java backend applications.

---

## Scenario 1: Masking Sensitive Data in Logs

**Situation:** Your application logs request bodies that may contain credit card numbers, SSNs, and passwords. Implement a log sanitizer.

```java
@Component
public class LogSanitizer {

    // Pre-compile all patterns for performance
    private static final Pattern CREDIT_CARD = Pattern.compile(
        "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})\\b");

    private static final Pattern SSN = Pattern.compile(
        "\\b\\d{3}-\\d{2}-\\d{4}\\b");

    private static final Pattern PASSWORD_FIELD = Pattern.compile(
        "\"password\"\\s*:\\s*\"[^\"]+\"",
        Pattern.CASE_INSENSITIVE);

    private static final Pattern EMAIL = Pattern.compile(
        "\\b([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\\.[a-zA-Z]{2,})\\b");

    public String sanitize(String input) {
        if (input == null) return null;

        String result = input;
        result = CREDIT_CARD.matcher(result).replaceAll("****-****-****-####");
        result = SSN.matcher(result).replaceAll("***-**-####");
        result = PASSWORD_FIELD.matcher(result).replaceAll("\"password\":\"[REDACTED]\"");

        // Partial email masking: alice@example.com → a***@example.com
        result = EMAIL.matcher(result).replaceAll(m -> {
            String user = m.group(1);
            String domain = m.group(2);
            return user.charAt(0) + "***@" + domain;
        });

        return result;
    }
}

// Use in logging interceptor
@Aspect
@Component
public class RequestLoggingAspect {

    @Autowired private LogSanitizer sanitizer;

    @Around("execution(* com.example.api.*Controller.*(..))")
    public Object logRequest(ProceedingJoinPoint pjp) throws Throwable {
        String args = Arrays.stream(pjp.getArgs())
            .map(arg -> sanitizer.sanitize(String.valueOf(arg)))
            .collect(Collectors.joining(", "));

        log.info("Calling {} with args: {}", pjp.getSignature().getName(), args);
        // ...
    }
}
```

---

## Scenario 2: Parsing Log Files with Regex

**Situation:** Parse Apache/Nginx access log entries to extract structured data for analytics.

```java
@Component
public class AccessLogParser {

    // Apache Combined Log Format:
    // 192.168.1.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326
    private static final Pattern LOG_PATTERN = Pattern.compile(
        "^(\\S+)" +                    // IP address
        "\\s+\\S+\\s+" +               // ident and userid
        "(\\S+)" +                     // authenticated user
        "\\s+\\[([^\\]]+)\\]" +        // timestamp
        "\\s+\"(\\S+)\\s+(\\S+)\\s+(\\S+)\"" +  // method, path, protocol
        "\\s+(\\d{3})" +               // status code
        "\\s+(\\d+|-)$"                // bytes transferred
    );

    private static final DateTimeFormatter LOG_DATE_FORMAT =
        DateTimeFormatter.ofPattern("dd/MMM/yyyy:HH:mm:ss Z", Locale.ENGLISH);

    public Optional<AccessLogEntry> parse(String line) {
        Matcher m = LOG_PATTERN.matcher(line.trim());
        if (!m.matches()) {
            log.debug("Could not parse log line: {}", line);
            return Optional.empty();
        }

        return Optional.of(new AccessLogEntry(
            m.group(1),                                      // IP
            m.group(2).equals("-") ? null : m.group(2),     // User (null if -)
            ZonedDateTime.parse(m.group(3), LOG_DATE_FORMAT), // Timestamp
            m.group(4),                                      // Method
            m.group(5),                                      // Path
            Integer.parseInt(m.group(7)),                    // Status
            m.group(8).equals("-") ? 0L : Long.parseLong(m.group(8)) // Bytes
        ));
    }

    // Process file as stream
    public Stream<AccessLogEntry> parseFile(Path logFile) throws IOException {
        return Files.lines(logFile, StandardCharsets.UTF_8)
            .filter(line -> !line.isBlank())
            .map(this::parse)
            .flatMap(Optional::stream);
    }
}
```

---

## Scenario 3: Building Dynamic Query Strings Safely

**Situation:** Construct SQL-like query strings for Elasticsearch from user search input. Must prevent injection.

```java
@Service
public class SearchQueryBuilder {

    // Escape special Elasticsearch query string characters
    private static final Pattern SPECIAL_CHARS = Pattern.compile(
        "[+\\-=&|!(){}\\[\\]^\"~*?:\\\\/]");

    public String buildQuery(String userInput) {
        if (userInput == null || userInput.isBlank()) {
            return "*";
        }

        // 1. Trim and normalize whitespace
        String normalized = userInput.strip()
            .replaceAll("\\s+", " ");

        // 2. Escape special characters
        String escaped = SPECIAL_CHARS.matcher(normalized)
            .replaceAll("\\\\$0");  // Prepend \ before each special char

        // 3. Truncate to prevent extremely long queries
        if (escaped.length() > 500) {
            escaped = escaped.substring(0, 500);
        }

        return escaped;
    }

    // Template-based query with validated field names
    private static final Set<String> ALLOWED_FIELDS = Set.of(
        "title", "description", "category", "status");

    private static final String QUERY_TEMPLATE = """
            {
              "query": {
                "multi_match": {
                  "query": "%s",
                  "fields": [%s]
                }
              }
            }
            """;

    public String buildMultiFieldQuery(String query, List<String> fields) {
        // Validate fields against allowlist — never trust user-supplied field names
        String validatedFields = fields.stream()
            .filter(ALLOWED_FIELDS::contains)
            .map(f -> "\"" + f + "\"")
            .collect(Collectors.joining(", "));

        return QUERY_TEMPLATE.formatted(
            buildQuery(query),
            validatedFields
        );
    }
}
```

---

## Scenario 4: Extracting and Validating Data from Text

**Situation:** Parse a product description that includes structured metadata in a specific format to extract SKUs, prices, and attributes.

```java
@Service
public class ProductDescriptionParser {

    // Format: "SKU: ABC-123, Price: $99.99, Attr: color=red;size=L;material=cotton"
    private static final Pattern SKU_PATTERN =
        Pattern.compile("SKU:\\s*([A-Z]+-\\d+)");
    private static final Pattern PRICE_PATTERN =
        Pattern.compile("Price:\\s*\\$([\\d,]+\\.\\d{2})");
    private static final Pattern ATTR_PATTERN =
        Pattern.compile("Attr:\\s*([\\w=;]+)");

    public ProductMetadata parse(String description) {
        String sku = extractGroup(SKU_PATTERN, description, 1)
            .orElseThrow(() -> new ParseException("Missing SKU in: " + description));

        BigDecimal price = extractGroup(PRICE_PATTERN, description, 1)
            .map(s -> s.replace(",", ""))  // Remove thousands separator
            .map(BigDecimal::new)
            .orElse(null);

        Map<String, String> attributes = extractGroup(ATTR_PATTERN, description, 1)
            .map(attrStr -> Arrays.stream(attrStr.split(";"))
                .map(pair -> pair.split("=", 2))
                .filter(parts -> parts.length == 2)
                .collect(Collectors.toMap(
                    parts -> parts[0].toLowerCase(),
                    parts -> parts[1]
                )))
            .orElse(Map.of());

        return new ProductMetadata(sku, price, attributes);
    }

    private Optional<String> extractGroup(Pattern pattern, String text, int group) {
        Matcher m = pattern.matcher(text);
        return m.find() ? Optional.of(m.group(group)) : Optional.empty();
    }
}
```

---

## Scenario 5: Efficient String Processing for High-Throughput Pipeline

**Situation:** Your event processor receives 50,000 strings per second. It must check if each string matches any of 200 different patterns. Naive loop is too slow.

```java
@Component
public class EventClassifier {

    // SLOW approach — iterates all patterns for each event
    private final List<Pattern> patterns = loadPatterns();

    public EventType classify_slow(String event) {
        for (Pattern p : patterns) {           // 200 pattern checks per event
            if (p.matcher(event).find()) {     // Potentially expensive
                return getEventType(p);
            }
        }
        return EventType.UNKNOWN;
    }

    // FASTER — combine all patterns into one alternation
    private final Pattern combinedPattern;
    private final Map<String, EventType> groupNameToType;

    @PostConstruct
    public void buildCombinedPattern() {
        // Build: (?<type1>pattern1)|(?<type2>pattern2)|...
        StringBuilder sb = new StringBuilder();
        groupNameToType = new HashMap<>();

        for (int i = 0; i < eventDefinitions.size(); i++) {
            EventDefinition def = eventDefinitions.get(i);
            String groupName = "type" + i;
            groupNameToType.put(groupName, def.getEventType());

            if (sb.length() > 0) sb.append("|");
            sb.append("(?<").append(groupName).append(">").append(def.getPattern()).append(")");
        }
        combinedPattern = Pattern.compile(sb.toString());
    }

    public EventType classify_fast(String event) {
        Matcher m = combinedPattern.matcher(event);
        if (!m.find()) return EventType.UNKNOWN;

        // Find which named group matched
        for (Map.Entry<String, EventType> entry : groupNameToType.entrySet()) {
            if (m.group(entry.getKey()) != null) {
                return entry.getValue();
            }
        }
        return EventType.UNKNOWN;
    }
}
```

---

## Scenario 6: Internationalizing String Formatting

**Situation:** Your application serves users in 20 countries. Format numbers, dates, and currencies according to user's locale.

```java
@Service
public class LocalizedFormatterService {

    public String formatAmount(BigDecimal amount, String currencyCode, Locale locale) {
        NumberFormat formatter = NumberFormat.getCurrencyInstance(locale);
        formatter.setCurrency(Currency.getInstance(currencyCode));
        return formatter.format(amount);
        // US: $1,234.56
        // DE: 1.234,56 €
        // JP: ¥1,235
    }

    public String formatDate(LocalDate date, Locale locale) {
        return DateTimeFormatter
            .ofLocalizedDate(FormatStyle.LONG)
            .withLocale(locale)
            .format(date);
        // US: March 15, 2024
        // DE: 15. März 2024
        // JP: 2024年3月15日
    }

    public String formatMessage(String key, Locale locale, Object... args) {
        // Use MessageFormat with ResourceBundle for i18n
        ResourceBundle bundle = ResourceBundle.getBundle("messages", locale);
        String pattern = bundle.getString(key);
        return MessageFormat.format(pattern, args);
        // messages_en.properties: order.created=Order {0} created for {1}
        // messages_de.properties: order.created=Auftrag {0} für {1} erstellt
    }

    // Comparing strings across locales
    public List<String> sortForLocale(List<String> strings, Locale locale) {
        Collator collator = Collator.getInstance(locale);
        collator.setStrength(Collator.PRIMARY);  // Ignore case and accents
        return strings.stream()
            .sorted(collator)
            .collect(Collectors.toList());
        // Sorts ü, ö, ä correctly for German locale
    }
}
```

---

## Scenario 7: URL and Path Parsing

**Situation:** Parse and manipulate URLs from various formats in a web crawler.

```java
@Component
public class UrlProcessor {

    private static final Pattern URL_PATTERN = Pattern.compile(
        "https?://([^/\\s:]+)(?::(\\d+))?(/[^\\s?#]*)?(?:\\?([^\\s#]*))?(?:#([^\\s]*))?");

    public UrlComponents parse(String url) {
        Matcher m = URL_PATTERN.matcher(url);
        if (!m.matches()) {
            throw new IllegalArgumentException("Invalid URL: " + url);
        }

        return new UrlComponents(
            m.group(1),                                   // host
            m.group(2) != null ? Integer.parseInt(m.group(2)) : -1, // port
            m.group(3) != null ? m.group(3) : "/",       // path
            parseQueryParams(m.group(4)),                 // query params
            m.group(5)                                    // fragment
        );
    }

    private Map<String, String> parseQueryParams(String queryString) {
        if (queryString == null || queryString.isEmpty()) return Map.of();

        return Arrays.stream(queryString.split("&"))
            .map(param -> param.split("=", 2))
            .filter(parts -> parts.length == 2)
            .collect(Collectors.toMap(
                parts -> URLDecoder.decode(parts[0], StandardCharsets.UTF_8),
                parts -> URLDecoder.decode(parts[1], StandardCharsets.UTF_8),
                (existing, replacement) -> replacement  // Last one wins on duplicate keys
            ));
    }

    // Normalize URL — remove trailing slash, sort params, lowercase host
    public String normalize(String url) {
        try {
            URI uri = new URI(url);
            String path = uri.getPath();
            if (path.endsWith("/") && path.length() > 1) {
                path = path.substring(0, path.length() - 1);
            }

            // Sort query parameters for canonical form
            String query = uri.getQuery();
            String normalizedQuery = null;
            if (query != null) {
                normalizedQuery = Arrays.stream(query.split("&"))
                    .sorted()
                    .collect(Collectors.joining("&"));
            }

            return new URI(
                uri.getScheme(),
                uri.getAuthority(),
                path,
                normalizedQuery,
                null  // Strip fragment
            ).toString().toLowerCase();

        } catch (URISyntaxException e) {
            throw new IllegalArgumentException("Invalid URL: " + url, e);
        }
    }
}
```

---

## Scenario 8: Text Diff and Comparison

**Situation:** Implement a service that shows what changed between two versions of a product description for an audit trail.

```java
@Service
public class TextComparisonService {

    // Find all differences between two strings
    public List<TextChange> findChanges(String original, String revised) {
        List<String> originalWords = tokenize(original);
        List<String> revisedWords = tokenize(revised);

        // Simple word-level diff using LCS
        List<TextChange> changes = new ArrayList<>();
        int[][] dp = computeLCS(originalWords, revisedWords);
        backtrack(dp, originalWords, revisedWords,
                  originalWords.size(), revisedWords.size(), changes);

        return changes;
    }

    // Check if strings are similar enough (fuzzy match)
    public double similarity(String s1, String s2) {
        if (s1 == null || s2 == null) return 0;
        if (s1.equals(s2)) return 1.0;

        int maxLen = Math.max(s1.length(), s2.length());
        if (maxLen == 0) return 1.0;

        int editDistance = levenshteinDistance(s1, s2);
        return 1.0 - (double) editDistance / maxLen;
    }

    // Levenshtein distance — dynamic programming
    private int levenshteinDistance(String s1, String s2) {
        int m = s1.length(), n = s2.length();
        int[] prev = new int[n + 1];
        int[] curr = new int[n + 1];

        for (int j = 0; j <= n; j++) prev[j] = j;

        for (int i = 1; i <= m; i++) {
            curr[0] = i;
            for (int j = 1; j <= n; j++) {
                if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                    curr[j] = prev[j - 1];
                } else {
                    curr[j] = 1 + Math.min(prev[j - 1],
                               Math.min(prev[j], curr[j - 1]));
                }
            }
            int[] temp = prev; prev = curr; curr = temp;
        }
        return prev[n];
    }

    private List<String> tokenize(String text) {
        return Arrays.asList(text.trim().split("\\s+"));
    }
}
```

---

## Scenario 9: Template Engine Implementation

**Situation:** Implement a simple template engine that replaces `{{variable}}` placeholders with values.

```java
@Component
public class SimpleTemplateEngine {

    private static final Pattern PLACEHOLDER = Pattern.compile("\\{\\{\\s*(\\w+)\\s*\\}\\}");

    public String render(String template, Map<String, Object> context) {
        StringBuffer result = new StringBuffer();
        Matcher m = PLACEHOLDER.matcher(template);

        while (m.find()) {
            String variableName = m.group(1);
            Object value = context.get(variableName);

            String replacement;
            if (value == null) {
                replacement = "";  // Or: throw exception for missing required variables
                log.warn("Template variable '{}' not found in context", variableName);
            } else {
                replacement = Matcher.quoteReplacement(String.valueOf(value));
                // quoteReplacement escapes $ and \ which have special meaning in replacements
            }

            m.appendReplacement(result, replacement);
        }
        m.appendTail(result);

        return result.toString();
    }

    // Type-safe template context builder
    public static class TemplateContext {
        private final Map<String, Object> values = new LinkedHashMap<>();

        public TemplateContext with(String key, Object value) {
            values.put(key, value);
            return this;
        }

        public Map<String, Object> build() { return Collections.unmodifiableMap(values); }
    }
}

// Usage:
String email = templateEngine.render(
    "Dear {{firstName}},\n\nYour order {{orderId}} for {{total}} has been confirmed.",
    new TemplateEngine.TemplateContext()
        .with("firstName", "Alice")
        .with("orderId", "ORD-12345")
        .with("total", "$99.99")
        .build()
);
```

---

## Scenario 10: CSV Parsing Without External Library

**Situation:** Parse CSV files that may contain quoted fields, embedded commas, and escaped quotes.

```java
@Component
public class CsvParser {

    // RFC 4180 compliant CSV parser
    private static final Pattern CSV_FIELD = Pattern.compile(
        "\"((?:[^\"]*(?:\"\"[^\"]*)*))\"" + // Quoted field (""  = escaped quote)
        "|" +
        "([^,\r\n]*)");                       // Unquoted field

    public List<String> parseLine(String line) {
        List<String> fields = new ArrayList<>();
        Matcher m = CSV_FIELD.matcher(line);

        int lastEnd = 0;
        while (m.find()) {
            if (m.start() != lastEnd && lastEnd < line.length()) {
                // Empty field (consecutive commas)
                fields.add("");
            }

            if (m.group(1) != null) {
                // Quoted field — unescape doubled quotes
                fields.add(m.group(1).replace("\"\"", "\""));
            } else {
                fields.add(m.group(2));
            }

            lastEnd = m.end();
            // Skip the comma separator
            if (lastEnd < line.length() && line.charAt(lastEnd) == ',') {
                lastEnd++;
                m.region(lastEnd, line.length());
            }
        }

        return fields;
    }

    public List<Map<String, String>> parseWithHeaders(InputStream input)
            throws IOException {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(input, StandardCharsets.UTF_8))) {

            // First line is headers
            String headerLine = reader.readLine();
            if (headerLine == null) return List.of();
            List<String> headers = parseLine(headerLine);

            return reader.lines()
                .filter(line -> !line.isBlank())
                .map(this::parseLine)
                .map(fields -> {
                    Map<String, String> row = new LinkedHashMap<>();
                    for (int i = 0; i < headers.size(); i++) {
                        row.put(headers.get(i), i < fields.size() ? fields.get(i) : "");
                    }
                    return row;
                })
                .collect(Collectors.toList());
        }
    }
}
```

---

## Scenario 11: String Interning for Memory Optimization

**Situation:** Your application loads 10M records from a database where `status`, `category`, and `region` fields have high cardinality duplication (only 5-20 distinct values but millions of rows).

```java
@Service
public class RecordLoader {

    // Using String.intern() — leverages JVM string pool
    public List<Record> loadRecordsOptimized() {
        return jdbcTemplate.query("SELECT * FROM records", (rs, rowNum) -> {
            return new Record(
                rs.getLong("id"),
                rs.getString("status").intern(),    // "ACTIVE", "INACTIVE", etc.
                rs.getString("category").intern(),  // "ELECTRONICS", "CLOTHING", etc.
                rs.getString("region").intern(),    // "US-EAST", "EU-WEST", etc.
                rs.getString("name")               // Unique per row — don't intern
            );
        });
    }

    // Better alternative: use enums or a dedicated intern cache
    // Enums are interned by definition and provide type safety
    public enum Status { ACTIVE, INACTIVE, PENDING, SUSPENDED }

    // Custom intern cache — size-bounded, predictable
    public static class StringCache {
        private final Map<String, String> cache = new ConcurrentHashMap<>(64);

        public String intern(String s) {
            return cache.computeIfAbsent(s, Function.identity());
        }
    }

    // Usage with cache
    private final StringCache statusCache = new StringCache();
    private final StringCache categoryCache = new StringCache();

    public Record loadRecord(ResultSet rs) throws SQLException {
        return new Record(
            rs.getLong("id"),
            statusCache.intern(rs.getString("status")),
            categoryCache.intern(rs.getString("category")),
            rs.getString("name")
        );
    }
}
```

---

## Scenario 12: Phone Number Normalization

**Situation:** Users enter phone numbers in many formats. Normalize them to E.164 format for SMS sending.

```java
@Component
public class PhoneNumberNormalizer {

    // Remove all non-digit characters
    private static final Pattern NON_DIGIT = Pattern.compile("[^+\\d]");
    // E.164 format: +[1-3 digit country code][up to 14 digits]
    private static final Pattern E164_PATTERN = Pattern.compile("^\\+[1-9]\\d{6,14}$");

    public String normalize(String input, String defaultCountryCode) {
        if (input == null || input.isBlank()) {
            throw new IllegalArgumentException("Phone number cannot be empty");
        }

        // Step 1: Remove formatting characters (spaces, dashes, parentheses, dots)
        String cleaned = NON_DIGIT.matcher(input.trim()).replaceAll("");

        // Step 2: Handle country codes
        if (cleaned.startsWith("+")) {
            // Already has country code
        } else if (cleaned.startsWith("00")) {
            // International format without +
            cleaned = "+" + cleaned.substring(2);
        } else if (cleaned.startsWith("1") && cleaned.length() == 11) {
            // US number with country code but no +
            cleaned = "+" + cleaned;
        } else if (cleaned.length() == 10 && defaultCountryCode.equals("US")) {
            // US 10-digit — add +1
            cleaned = "+1" + cleaned;
        } else if (cleaned.length() > 0) {
            cleaned = "+" + defaultCountryCode + cleaned;
        }

        // Step 3: Validate
        if (!E164_PATTERN.matcher(cleaned).matches()) {
            throw new IllegalArgumentException(
                "Cannot normalize to E.164 format: " + input);
        }

        return cleaned;
    }

    // Test cases:
    // "(415) 555-1234" → "+14155551234"
    // "+44 20 7946 0958" → "+442079460958"
    // "0044 20 7946 0958" → "+442079460958"
}
```

---

## Scenario 13: Tokenizing and Parsing DSL Expressions

**Situation:** Implement a parser for a simple filter expression DSL used in a search API: `age > 18 AND (status = "active" OR status = "trial")`.

```java
@Component
public class FilterExpressionParser {

    // Tokenizer pattern — match tokens in order of specificity
    private static final Pattern TOKEN = Pattern.compile(
        "(?i)(AND|OR|NOT)" +            // Logical operators (case-insensitive)
        "|([()])" +                      // Parentheses
        "|([a-zA-Z_][a-zA-Z0-9_.]*)" + // Identifiers (field names)
        "|(>=|<=|!=|[><=])" +           // Comparison operators
        "|\"([^\"]*)\"" +               // Quoted strings
        "|(\\d+(?:\\.\\d+)?)" +         // Numbers
        "|\\s+"                          // Whitespace (skip)
    );

    public List<Token> tokenize(String expression) {
        List<Token> tokens = new ArrayList<>();
        Matcher m = TOKEN.matcher(expression);

        while (m.find()) {
            if (m.group(1) != null) {
                tokens.add(new Token(TokenType.LOGICAL_OP, m.group(1).toUpperCase()));
            } else if (m.group(2) != null) {
                tokens.add(new Token(TokenType.PAREN, m.group(2)));
            } else if (m.group(3) != null) {
                tokens.add(new Token(TokenType.IDENTIFIER, m.group(3)));
            } else if (m.group(4) != null) {
                tokens.add(new Token(TokenType.COMPARISON_OP, m.group(4)));
            } else if (m.group(5) != null) {
                tokens.add(new Token(TokenType.STRING_LITERAL, m.group(5)));
            } else if (m.group(6) != null) {
                tokens.add(new Token(TokenType.NUMBER_LITERAL, m.group(6)));
            }
            // group 7 is whitespace — skip
        }

        return tokens;
    }

    public record Token(TokenType type, String value) {}

    public enum TokenType {
        LOGICAL_OP, PAREN, IDENTIFIER, COMPARISON_OP, STRING_LITERAL, NUMBER_LITERAL
    }
}
```

---

## Scenario 14: String Splitting and Joining for Config Parsing

**Situation:** Parse configuration values from environment variables that use various delimiter formats.

```java
@Component
public class ConfigValueParser {

    // Parse comma/semicolon/pipe-delimited list with trimming
    public List<String> parseList(String value) {
        if (value == null || value.isBlank()) return List.of();

        return Arrays.stream(value.split("[,;|]"))
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .distinct()
            .collect(Collectors.toList());
    }

    // Parse key=value pairs from string
    public Map<String, String> parseProperties(String value) {
        if (value == null || value.isBlank()) return Map.of();

        return Arrays.stream(value.split("[,;]"))
            .map(String::trim)
            .filter(s -> s.contains("="))
            .map(pair -> pair.split("=", 2))
            .collect(Collectors.toMap(
                parts -> parts[0].trim(),
                parts -> parts[1].trim(),
                (v1, v2) -> v2  // Last value wins for duplicates
            ));
    }

    // Parse ranges like "1-100,200-300,400"
    public List<Integer> parseRange(String value) {
        List<Integer> result = new ArrayList<>();
        for (String part : value.split(",")) {
            part = part.trim();
            if (part.contains("-")) {
                String[] bounds = part.split("-", 2);
                int start = Integer.parseInt(bounds[0].trim());
                int end = Integer.parseInt(bounds[1].trim());
                for (int i = start; i <= end; i++) result.add(i);
            } else {
                result.add(Integer.parseInt(part));
            }
        }
        return result;
    }

    // Safe toString conversion
    public String toListString(Collection<?> items) {
        return items.stream()
            .map(String::valueOf)
            .collect(Collectors.joining(","));
    }
}
```

---

## Scenario 15: Building a String Trie for Prefix Search

**Situation:** Implement autocomplete that suggests product names for a given prefix. Must be fast (under 5ms for 100K products).

```java
public class ProductTrie {

    private final TrieNode root = new TrieNode();

    private static class TrieNode {
        final Map<Character, TrieNode> children = new HashMap<>();
        final List<String> completions = new ArrayList<>();
        boolean isWord = false;
    }

    public void insert(String product) {
        TrieNode node = root;
        String lower = product.toLowerCase();
        for (char c : lower.toCharArray()) {
            node = node.children.computeIfAbsent(c, k -> new TrieNode());
            // Store up to 10 completions at each node
            if (node.completions.size() < 10) {
                node.completions.add(product);
            }
        }
        node.isWord = true;
    }

    public List<String> autocomplete(String prefix) {
        TrieNode node = root;
        for (char c : prefix.toLowerCase().toCharArray()) {
            node = node.children.get(c);
            if (node == null) return List.of();
        }
        return List.copyOf(node.completions);
    }

    // Build from product list
    public static ProductTrie build(List<String> products) {
        ProductTrie trie = new ProductTrie();
        products.stream()
            .sorted()  // Sort so completions are alphabetical
            .forEach(trie::insert);
        return trie;
    }
}
```

---

## Scenario 16: Secure Password Hashing and Comparison

**Situation:** Implement secure password storage — never store plaintext passwords.

```java
@Service
public class PasswordService {

    private final PasswordEncoder encoder = new BCryptPasswordEncoder(12); // Cost factor 12

    public String hash(String plaintext) {
        if (plaintext == null || plaintext.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }
        return encoder.encode(plaintext);
        // Output: $2a$12$[22-char salt][31-char hash]
    }

    public boolean verify(String plaintext, String hash) {
        // BCrypt verify — timing-safe comparison built in
        return encoder.matches(plaintext, hash);
    }

    // For API keys or tokens — constant-time comparison to prevent timing attacks
    public boolean safeEquals(String a, String b) {
        if (a == null || b == null) return false;
        byte[] aBytes = a.getBytes(StandardCharsets.UTF_8);
        byte[] bBytes = b.getBytes(StandardCharsets.UTF_8);
        return MessageDigest.isEqual(aBytes, bBytes);
        // MessageDigest.isEqual is timing-safe (constant time regardless of mismatch position)
    }

    // For tokens: generate a secure random token
    public String generateToken(int lengthBytes) {
        byte[] bytes = new byte[lengthBytes];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
        // 32 bytes → 43-char URL-safe base64 token
    }
}
```

---

## Scenario 17: Text Truncation for Display

**Situation:** Truncate product descriptions for display cards while respecting word boundaries and Unicode.

```java
@Component
public class TextTruncator {

    // Truncate at word boundary, not in the middle of a word
    public String truncate(String text, int maxLength, String ellipsis) {
        if (text == null) return "";
        if (text.length() <= maxLength) return text;

        int cutoff = maxLength - ellipsis.length();
        if (cutoff <= 0) return ellipsis.substring(0, maxLength);

        // Find last word boundary before cutoff
        int lastSpace = text.lastIndexOf(' ', cutoff);
        if (lastSpace <= cutoff / 2) {
            // No good word boundary — hard cut
            // But be careful of Unicode surrogate pairs!
            cutoff = adjustForSurrogates(text, cutoff);
            return text.substring(0, cutoff) + ellipsis;
        }

        return text.substring(0, lastSpace) + ellipsis;
    }

    // Ensure we don't cut in the middle of a surrogate pair (emoji, etc.)
    private int adjustForSurrogates(String text, int index) {
        if (index > 0 && Character.isLowSurrogate(text.charAt(index))
                && Character.isHighSurrogate(text.charAt(index - 1))) {
            return index - 1;  // Back up to include the complete character
        }
        return index;
    }

    // Truncate HTML while preserving tag integrity
    public String truncateHtml(String html, int maxTextLength) {
        // Strip all HTML tags first (simple approach)
        String textOnly = html.replaceAll("<[^>]+>", "");
        textOnly = textOnly.replaceAll("&[a-z]+;", " ");  // Replace HTML entities
        return truncate(textOnly, maxTextLength, "...");
    }
}
```

---

## Scenario 18: Named Entity Extraction with Regex

**Situation:** Extract mentions of amounts, dates, and order IDs from customer support messages for auto-routing.

```java
@Component
public class EntityExtractor {

    private static final Pattern ORDER_ID = Pattern.compile(
        "\\b(?:order|ord|#)\\s*(?:id|no\\.?)?\\s*:?\\s*([A-Z0-9-]{6,20})\\b",
        Pattern.CASE_INSENSITIVE);

    private static final Pattern AMOUNT = Pattern.compile(
        "\\$\\s*(\\d{1,3}(?:,\\d{3})*(?:\\.\\d{1,2})?)" +
        "|" +
        "(\\d{1,3}(?:,\\d{3})*(?:\\.\\d{1,2})?)\\s*(?:USD|dollars?)");

    private static final Pattern DATE = Pattern.compile(
        "(?:(\\d{1,2})[/\\-](\\d{1,2})[/\\-](\\d{2,4}))" +   // MM/DD/YYYY
        "|" +
        "(?:(January|February|March|April|May|June|July|August|" +
           "September|October|November|December)\\s+(\\d{1,2})(?:st|nd|rd|th)?,?\\s+(\\d{4}))",
        Pattern.CASE_INSENSITIVE);

    public ExtractionResult extract(String message) {
        List<String> orderIds = new ArrayList<>();
        List<BigDecimal> amounts = new ArrayList<>();
        List<String> dates = new ArrayList<>();

        Matcher m;

        m = ORDER_ID.matcher(message);
        while (m.find()) orderIds.add(m.group(1).toUpperCase());

        m = AMOUNT.matcher(message);
        while (m.find()) {
            String raw = (m.group(1) != null ? m.group(1) : m.group(2))
                .replace(",", "");
            amounts.add(new BigDecimal(raw));
        }

        m = DATE.matcher(message);
        while (m.find()) dates.add(m.group());

        return new ExtractionResult(
            orderIds.stream().distinct().collect(Collectors.toList()),
            amounts,
            dates
        );
    }
}
```
