# Java Security — Theory Questions

## How to Use This File
Security questions at Senior/Staff level test both conceptual knowledge and implementation ability. Know OWASP Top 10, authentication/authorization patterns, cryptography basics, and secure coding practices.

---

## Question 1: What are the OWASP Top 10 vulnerabilities most relevant to Java backends?

| Rank | Vulnerability | Java/Spring Example | Prevention |
|---|---|---|---|
| A01 | Broken Access Control | Missing `@PreAuthorize`, IDOR | Spring Security method security, input validation |
| A02 | Cryptographic Failures | Storing passwords in MD5, HTTP transport | BCrypt, TLS everywhere |
| A03 | Injection | SQL injection via string concat | PreparedStatement, parameterized queries |
| A05 | Security Misconfiguration | Actuator endpoints public, default creds | Disable unused endpoints, externalise secrets |
| A07 | Identity/Auth Failures | Weak JWT, no session expiry | Strong algorithms, short-lived tokens |
| A08 | Software/Data Integrity | Deserializing untrusted data | Avoid Java serialization, validate input |
| A09 | Security Logging Failures | No audit log, logging PII | Structured audit logs, mask sensitive data |

---

## Question 2: Explain SQL Injection. How does PreparedStatement prevent it?

**Attack:** Attacker injects SQL syntax through user input to manipulate queries.

```java
// VULNERABLE — string concatenation
String sql = "SELECT * FROM users WHERE username = '" + username + "'";
// Input: ' OR '1'='1  → returns all users
// Input: '; DROP TABLE users; --  → destroys table

// SAFE — PreparedStatement with parameterized query
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?"
);
ps.setString(1, username);  // username is treated as DATA, never as SQL syntax
```

**How it works:** The database engine receives the query template and parameters separately. The parameter value is never interpreted as SQL syntax, regardless of content.

**Other injection types:**
- **HQL/JPQL injection** — use `@Query` with named parameters, never string concatenation
- **MongoDB injection** — use typed query builders, not string concatenation
- **LDAP injection** — escape LDAP special characters

```java
// Spring Data JPA — safe with parameter binding
@Query("SELECT u FROM User u WHERE u.email = :email AND u.status = :status")
Optional<User> findByEmailAndStatus(@Param("email") String email,
                                    @Param("status") UserStatus status);

// UNSAFE — never do this
@Query("SELECT u FROM User u WHERE u.email = '" + email + "'") // JPQL injection!
```

---

## Question 3: How does BCrypt work? Why is it the right choice for password hashing?

**Properties of a secure password hash:**
1. **One-way** — computationally infeasible to reverse
2. **Slow** — high cost per hash attempt (defeats brute force)
3. **Salted** — unique salt per password (defeats rainbow tables)
4. **Adaptive** — cost factor can be increased as hardware gets faster

```java
// BCrypt in Spring Security
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // cost factor: 2^12 = 4096 iterations
    // Cost 12 ≈ 250ms per hash on modern hardware — expensive for attacker, fine for user
}

// Registration
String encoded = passwordEncoder.encode(rawPassword);
user.setPasswordHash(encoded);  // store: $2a$12$saltBASE64encodedHASH

// Login
boolean valid = passwordEncoder.matches(submittedPassword, storedHash);

// BCrypt hash format:
// $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
//  ↑   ↑   ↑--- 22 chars salt ---↑---- 31 chars hash ----↑
// alg cost
```

**Why not MD5/SHA-256?**
- MD5/SHA can compute billions of hashes/second on modern GPUs
- A 10M password leak → cracked in minutes with rainbow tables or brute force
- BCrypt at cost 12 → ~250ms/hash → ~4 hashes/second → 10M attempts takes years

**BCrypt vs Argon2:**
- Argon2id (winner of Password Hashing Competition 2015) is more resistant to GPU attacks via memory-hardness
- Spring Security supports Argon2: `new Argon2PasswordEncoder()`
- Both are acceptable; BCrypt is more widely deployed

---

## Question 4: Explain JWT (JSON Web Token) structure, signing, and security concerns.

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9    <- Header (base64url)
.eyJzdWIiOiJ1c2VyMTIzIiwiZXhwIjoxNjAwMDAwMDAwfQ==  <- Payload (base64url)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c        <- Signature
```

**Header:** `{"alg": "RS256", "typ": "JWT"}`
**Payload:** `{"sub": "user123", "exp": 1700000000, "roles": ["USER"], "iat": 1699990000}`
**Signature:** `RS256(base64url(header) + "." + base64url(payload), privateKey)`

```java
// Generating JWT with JJWT library
@Service
public class JwtTokenService {
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;

    public String generateToken(UserDetails user) {
        return Jwts.builder()
            .subject(user.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 3600_000))  // 1 hour
            .claim("roles", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .signWith(privateKey)  // RS256 — asymmetric
            .compact();
    }

    public Claims validateAndParse(String token) {
        return Jwts.parser()
            .verifyWith(publicKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

**Security concerns:**

| Issue | Attack | Mitigation |
|---|---|---|
| `"alg": "none"` | Remove signature entirely | Always validate `alg` header, reject "none" |
| RS256 → HS256 downgrade | Use public key as HMAC secret | Pin algorithm in parser |
| Missing expiry | Token valid forever | Always set `exp`, short-lived (15min-1hr) |
| Sensitive data in payload | Payload is base64, NOT encrypted | Never put passwords, SSNs in JWT |
| Token theft | XSS steals token from localStorage | Use `HttpOnly` cookie for storage |
| Missing `aud` claim | Token used on wrong service | Validate `aud` (audience) claim |

---

## Question 5: What is OAuth 2.0? Explain the Authorization Code flow.

**OAuth 2.0** is an authorization framework, not authentication. It allows a resource owner to grant limited access to resources to a third party without sharing credentials.

**Roles:**
- **Resource Owner** — the user
- **Client** — your application
- **Authorization Server** — issues tokens (e.g., Google, Keycloak)
- **Resource Server** — your API that validates tokens

**Authorization Code Flow (most secure, for server-side apps):**

```
Browser                  Your App              Auth Server
   |                         |                      |
   |--1. Login with Google-->|                      |
   |                         |--2. Redirect to AS-->|
   |<--------3. Redirect to AS login page-----------|
   |--4. User consents------>Auth Server             |
   |<----5. Redirect with auth code to callback-----|
   |                         |<--5. auth code-----  |
   |                         |--6. code + client_secret --> Auth Server
   |                         |<--7. access_token + refresh_token---------|
   |                         |--8. use access_token to call Resource Server
```

```java
// Spring Security OAuth2 Resource Server configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwkSetUri("https://auth-server/.well-known/jwks.json")
                    .jwtAuthenticationConverter(jwtToAuthConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

**PKCE (Proof Key for Code Exchange):** Extension for public clients (SPAs, mobile apps) that don't have client secrets. Uses a code verifier/challenge to prevent auth code interception.

---

## Question 6: What is CSRF and how does Spring Security prevent it?

**CSRF (Cross-Site Request Forgery):** A malicious site tricks a logged-in user's browser into sending a request to your site. Browser automatically includes cookies (session/auth).

```
Victim's browser: logged into bank.com (session cookie)
Attacker's site: contains <img src="https://bank.com/transfer?to=attacker&amount=1000">
Browser: sends GET to bank.com with session cookie → transfers money!
```

**Prevention — Synchronizer Token Pattern:**

```java
// Spring Security CSRF — enabled by default for non-GET requests
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // Sets XSRF-TOKEN cookie; JS reads it and sends as X-XSRF-TOKEN header
        );
    return http.build();
}

// Thymeleaf auto-includes CSRF token in forms
// <input type="hidden" name="_csrf" value="${_csrf.token}">

// REST API with Angular — Angular reads XSRF-TOKEN cookie, sends X-XSRF-TOKEN header
```

**Why REST APIs often disable CSRF:**
- Stateless APIs use JWT in Authorization header (not cookies) — not CSRF-vulnerable
- CSRF attacks exploit cookie-based auth
- If using `HttpOnly` cookies for JWT, you need CSRF protection again

```java
// Safe to disable CSRF for stateless JWT auth
http.csrf(AbstractHttpConfigurer::disable)
    .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

---

## Question 7: Explain XSS (Cross-Site Scripting) and how to prevent it in Java backends.

**Types:**
- **Stored XSS** — malicious script saved to DB, served to all users
- **Reflected XSS** — script in URL parameter, reflected in response
- **DOM XSS** — script injected via DOM manipulation (frontend issue)

**Backend prevention:**

```java
// 1. Input validation — reject or sanitize on input
public String sanitizeHtml(String input) {
    // OWASP Java HTML Sanitizer library
    PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
    return policy.sanitize(input);
}

// 2. Output encoding — encode on output (primary defense)
// Thymeleaf auto-escapes by default: th:text="${userInput}" → HTML-encoded
// th:utext="${trustedHtml}" → unescaped (only for trusted content)

// 3. Content Security Policy header
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
    )
);

// 4. HttpOnly + Secure cookies — prevents JS from reading auth cookies
ResponseCookie cookie = ResponseCookie.from("session", token)
    .httpOnly(true)
    .secure(true)
    .sameSite("Strict")
    .path("/")
    .maxAge(Duration.ofHours(1))
    .build();
```

**Key principle:** Never trust user input. Validate on input, encode on output.

---

## Question 8: What is the difference between Authentication and Authorization in Spring Security?

**Authentication:** Who are you? (Identity verification)
**Authorization:** What can you do? (Permission check)

```
Request → Authentication Filter → SecurityContextHolder → Authorization Filter → Controller
              ↓                                                    ↓
         Who is this user?                            Can this user access this resource?
         (verify credentials)                         (check roles/permissions)
```

```java
// Authentication — UsernamePasswordAuthenticationFilter calls AuthenticationManager
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getEmail())
            .password(user.getPasswordHash())
            .authorities(user.getRoles().stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.name()))
                .toList())
            .build();
    }
}

// Authorization — method-level security
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserProfile getProfile(Long userId) { ... }

// Role hierarchy
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_ADMIN > ROLE_MANAGER
        ROLE_MANAGER > ROLE_USER
        """);
}
```

---

## Question 9: Explain symmetric vs asymmetric cryptography. When do you use each?

**Symmetric encryption (AES):**
- Same key for encryption and decryption
- Fast — suitable for bulk data
- Key distribution problem: how do you securely share the key?

```java
// AES-256-GCM — authenticated encryption (confidentiality + integrity + authenticity)
public byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    byte[] iv = new byte[12];  // 96-bit IV recommended for GCM
    new SecureRandom().nextBytes(iv);

    cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
    byte[] ciphertext = cipher.doFinal(plaintext);

    // Prepend IV to ciphertext (IV is not secret, needs to be stored with ciphertext)
    byte[] result = new byte[iv.length + ciphertext.length];
    System.arraycopy(iv, 0, result, 0, iv.length);
    System.arraycopy(ciphertext, 0, result, iv.length, ciphertext.length);
    return result;
}
```

**Asymmetric encryption (RSA, ECDSA):**
- Public key encrypts, private key decrypts (or private signs, public verifies)
- Slow — not suitable for bulk data
- Solves key distribution: publish public key freely

**TLS combines both:** Asymmetric for key exchange, symmetric for data transfer.

**Practical guide:**

| Use Case | Algorithm | Notes |
|---|---|---|
| Password hashing | BCrypt, Argon2 | NOT encryption — one-way |
| Data at rest | AES-256-GCM | Symmetric, fast |
| JWT signing | RS256 (RSA) or ES256 (ECDSA) | Asymmetric — public key for verification |
| TLS | ECDH key exchange + AES | Hybrid |
| Envelope encryption | RSA encrypts AES key, AES encrypts data | KMS pattern |

---

## Question 10: What is Secret Management? How do you avoid hardcoded secrets?

**Problem:** Secrets (DB passwords, API keys, JWT signing keys) must not be in source code, config files, or Docker images.

```java
// BAD — hardcoded secret
datasource.password=mysecretpassword123  // in application.properties, committed to git

// GOOD — inject from environment
datasource.password=${DB_PASSWORD}  // environment variable

// BETTER — use a secrets manager
@Configuration
public class SecretsConfig {

    @Bean
    public DataSource dataSource(SecretsManagerClient secretsClient) {
        GetSecretValueResponse response = secretsClient.getSecretValue(
            GetSecretValueRequest.builder()
                .secretId("prod/myapp/database")
                .build()
        );
        DbCredentials creds = objectMapper.readValue(
            response.secretString(), DbCredentials.class);

        return DataSourceBuilder.create()
            .url("jdbc:postgresql://db.internal:5432/myapp")
            .username(creds.username())
            .password(creds.password())
            .build();
    }
}
```

**Options ranked by security:**
1. **AWS Secrets Manager / HashiCorp Vault** — best for production
2. **Kubernetes Secrets** (base64, not encrypted by default — use sealed-secrets or external-secrets-operator)
3. **Environment variables** — better than files, but visible in `/proc/PID/environ`
4. **Encrypted config files** — Spring Cloud Config with encryption

**Spring Cloud Vault integration:**
```yaml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: AWS_IAM  # no static token needed
  config:
    import: vault://secret/myapp
```

---

## Question 11: What is Insecure Deserialization and why is Java particularly vulnerable?

**Attack:** Deserializing attacker-controlled data executes arbitrary code via gadget chains.

**Java's vulnerability:** `ObjectInputStream.readObject()` automatically calls `readObject()` / `readResolve()` on deserialized objects. If the classpath contains "gadget" classes (e.g., from Apache Commons Collections), an attacker can chain method calls to execute arbitrary code.

```java
// VULNERABLE — deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(untrustedInput);
Object obj = ois.readObject();  // RCE if gadget chains exist on classpath!

// SAFE alternatives
// 1. Use JSON/XML with known schema
ObjectMapper mapper = new ObjectMapper();
UserDto user = mapper.readValue(jsonInput, UserDto.class);  // no code execution

// 2. Validate class whitelist if you must use Java serialization
ObjectInputStream ois = new ValidatingObjectInputStream(input);
ois.accept(SafeClass.class, AnotherSafeClass.class);
// Throws InvalidClassException for any other class

// 3. Use SerialKiller or other deserialization filters
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.example.safe.*;java.lang.*;!*"  // whitelist
);
ois.setObjectInputFilter(filter);
```

**Best practice:** Never deserialize Java objects from untrusted sources. Use JSON, Protocol Buffers, or Avro instead.

---

## Question 12: How do you implement secure HTTP headers in Spring?

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.headers(headers -> headers
        // Prevent embedding in iframes (clickjacking)
        .frameOptions(frame -> frame.deny())

        // Force HTTPS — tell browser to only use HTTPS for 1 year
        .httpStrictTransportSecurity(hsts -> hsts
            .maxAgeInSeconds(31536000)
            .includeSubDomains(true)
            .preload(true)
        )

        // Prevent MIME type sniffing
        .contentTypeOptions(Customizer.withDefaults())

        // Referrer policy — don't leak URLs to third parties
        .referrerPolicy(referrer -> referrer
            .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
        )

        // Content Security Policy — prevent XSS
        .contentSecurityPolicy(csp -> csp
            .policyDirectives("""
                default-src 'self';
                script-src 'self' 'nonce-{nonce}';
                style-src 'self' 'unsafe-inline';
                img-src 'self' data: https:;
                connect-src 'self' https://api.example.com;
                frame-ancestors 'none'
                """)
        )

        // Permissions policy — restrict browser features
        .permissionsPolicy(pp -> pp
            .policy("camera=(), microphone=(), geolocation=()")
        )
    );
    return http.build();
}
```

**Headers checklist:**

| Header | Purpose | Value |
|---|---|---|
| `Strict-Transport-Security` | Force HTTPS | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | No MIME sniffing | `nosniff` |
| `X-Frame-Options` | Anti-clickjacking | `DENY` or `SAMEORIGIN` |
| `Content-Security-Policy` | XSS prevention | restrict sources |
| `Referrer-Policy` | Privacy | `strict-origin-when-cross-origin` |

---

## Question 13: What is the principle of least privilege? Apply it to Spring Security.

**Principle:** Grant only the minimum permissions necessary for a function to work.

```java
// Application-level: role granularity
// BAD: one role for everything
@PreAuthorize("hasRole('ADMIN')")
public void viewOrder(Long id) { ... }

// GOOD: granular permissions
@PreAuthorize("hasAuthority('ORDER_READ')")
public void viewOrder(Long id) { ... }

@PreAuthorize("hasAuthority('ORDER_WRITE')")
public void createOrder(CreateOrderRequest req) { ... }

@PreAuthorize("hasAuthority('ORDER_DELETE') and hasRole('ADMIN')")
public void deleteOrder(Long id) { ... }

// Database level: application uses a DB user with minimal privileges
// CREATE USER app_user WITH PASSWORD '...';
// GRANT SELECT, INSERT, UPDATE ON orders, customers TO app_user;
// NOT GRANT CREATE TABLE, DROP, DELETE (if not needed)

// Microservice level: each service only accesses its own DB/tables
// Order service: access to orders schema only
// User service: access to users schema only
```

**Practical implications:**
- Service accounts should not have full admin rights
- DB connection pool user should not have DDL privileges (use separate migration user)
- AWS IAM roles: use specific resource ARNs, not `"Resource": "*"`

---

## Question 14: How do you prevent path traversal attacks in file serving?

**Attack:** `../../../etc/passwd` in a filename parameter reads files outside the intended directory.

```java
// VULNERABLE
@GetMapping("/files/{filename}")
public Resource getFile(@PathVariable String filename) {
    return new FileSystemResource("/uploads/" + filename);
    // Request: /files/../../etc/passwd  → reads /etc/passwd !
}

// SAFE — normalize and validate
@GetMapping("/files/{filename}")
public ResponseEntity<Resource> getFile(@PathVariable String filename) {
    // 1. Validate filename format
    if (!filename.matches("[a-zA-Z0-9._-]+")) {
        throw new BadRequestException("Invalid filename");
    }

    // 2. Resolve and check canonical path stays within allowed directory
    Path baseDir = Paths.get("/uploads").toAbsolutePath().normalize();
    Path filePath = baseDir.resolve(filename).normalize();

    if (!filePath.startsWith(baseDir)) {
        throw new SecurityException("Path traversal attempt detected");
    }

    if (!Files.exists(filePath) || !Files.isRegularFile(filePath)) {
        throw new ResourceNotFoundException("File not found: " + filename);
    }

    Resource resource = new FileSystemResource(filePath);
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION,
            "attachment; filename=\"" + filePath.getFileName() + "\"")
        .body(resource);
}
```

---

## Question 15: What is rate limiting and how do you implement it to prevent abuse?

**Why:** Prevent brute force (password guessing), DoS, credential stuffing, resource exhaustion.

```java
// Different rate limits for different endpoints
@Configuration
public class RateLimitConfig {

    // Login: 5 attempts per IP per minute (brute force prevention)
    @Bean
    public Bucket loginBucket() {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(5, Refill.greedy(5, Duration.ofMinutes(1))))
            .build();
    }

    // API: 1000 requests per user per hour (usage control)
    @Bean
    public BucketConfiguration apiRateLimitConfig() {
        return BucketConfiguration.builder()
            .addLimit(Bandwidth.classic(1000, Refill.intervally(1000, Duration.ofHours(1))))
            .build();
    }
}

// Progressive delays for failed login attempts (NIST 800-63B recommendation)
@Service
public class LoginAttemptService {
    private final Cache<String, Integer> attemptCache =
        Caffeine.newBuilder().expireAfterWrite(15, TimeUnit.MINUTES).build();

    public void recordFailure(String identifier) {
        int attempts = getAttempts(identifier) + 1;
        attemptCache.put(identifier, attempts);
    }

    public boolean isBlocked(String identifier) {
        return getAttempts(identifier) >= 5;
    }

    public Duration getDelay(String identifier) {
        int attempts = getAttempts(identifier);
        return switch (attempts) {
            case 0, 1, 2 -> Duration.ZERO;
            case 3 -> Duration.ofSeconds(5);
            case 4 -> Duration.ofSeconds(30);
            default -> Duration.ofMinutes(15);
        };
    }
}
```

---

## Question 16: Explain secure random number generation in Java.

```java
// WRONG — Math.random() or new Random() are NOT cryptographically secure
String token = String.valueOf(Math.random());  // predictable!
UUID uuid = UUID.randomUUID();  // Version 4 UUID uses SecureRandom — OK for tokens

// CORRECT — SecureRandom for security-sensitive values
SecureRandom secureRandom = new SecureRandom();  // uses OS entropy pool

// Generate secure token (API key, password reset token, session ID)
byte[] tokenBytes = new byte[32];  // 256 bits
secureRandom.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
// Result: URL-safe base64, 43 characters, 256-bit entropy

// WRONG: predictable ID generation
long id = System.currentTimeMillis();  // predictable, sequential

// Time-based one-time passwords (TOTP for MFA)
// Use well-audited library: Google Authenticator, OATH Toolkit
```

**SecureRandom vs Random:**
- `Random` uses linear congruential generator — mathematically predictable from any observed value
- `SecureRandom` uses OS entropy (`/dev/urandom` on Linux) — cryptographically unpredictable
- **Never use `Random` for:** session IDs, tokens, nonces, salts, OTPs

---

## Question 17: What is input validation and output encoding? Why are both needed?

**They solve different problems:**
- **Input validation:** Reject malformed/unexpected input at the boundary
- **Output encoding:** Ensure data is interpreted as data, not code

```java
// Input validation — at REST controller/service boundary
@PostMapping("/comments")
public Comment addComment(@Valid @RequestBody AddCommentRequest request) { ... }

public record AddCommentRequest(
    @NotBlank @Size(max = 2000) String text,
    @NotNull Long postId
) {}

// Input sanitization — strip dangerous content (for rich text fields)
@Service
public class CommentService {
    private static final PolicyFactory SAFE_HTML = Sanitizers.FORMATTING
        .and(Sanitizers.LINKS)
        .and(Sanitizers.BLOCKS);

    public Comment addComment(AddCommentRequest request, Long userId) {
        String safeText = SAFE_HTML.sanitize(request.text());  // strip <script>, onclick, etc.
        return commentRepository.save(new Comment(userId, request.postId(), safeText));
    }
}

// Output encoding — Thymeleaf example
// th:text="${comment.text}"  → HTML-encoded (safe, recommended)
// th:utext="${comment.text}" → raw HTML (ONLY for sanitized trusted content)

// JSON API — Jackson auto-escapes: < → \u003c, > → \u003e by default
// This prevents JSON injection
```

**Key principle:** Validate inputs where they enter your system. Encode outputs based on the context (HTML, JSON, SQL, shell command each has different encoding rules).

---

## Question 18: How do you audit security-sensitive operations in a Java application?

```java
// Audit log entry
public record AuditEvent(
    String eventType,         // "USER_LOGIN", "ORDER_DELETED", "PERMISSION_CHANGE"
    String actorId,           // who did it
    String actorIp,           // from where
    String resourceType,      // what type of resource
    String resourceId,        // which resource
    Map<String, Object> before,  // state before (for mutations)
    Map<String, Object> after,   // state after (for mutations)
    Instant occurredAt,
    boolean success,
    String failureReason
) {}

// AOP-based audit logging
@Aspect
@Component
public class AuditAspect {

    @AfterReturning(
        pointcut = "@annotation(Audited)",
        returning = "result"
    )
    public void auditSuccess(JoinPoint jp, Audited audited, Object result) {
        SecurityContext ctx = SecurityContextHolder.getContext();
        auditService.record(AuditEvent.success(
            audited.eventType(),
            ctx.getAuthentication().getName(),
            getCurrentIp(),
            audited.resourceType(),
            extractResourceId(jp.getArgs())
        ));
    }

    @AfterThrowing(
        pointcut = "@annotation(Audited)",
        throwing = "ex"
    )
    public void auditFailure(JoinPoint jp, Audited audited, Exception ex) {
        auditService.record(AuditEvent.failure(
            audited.eventType(),
            getCurrentPrincipal(),
            ex.getMessage()
        ));
    }
}

// Usage
@DeleteMapping("/users/{id}")
@PreAuthorize("hasRole('ADMIN')")
@Audited(eventType = "USER_DELETED", resourceType = "User")
public void deleteUser(@PathVariable Long id) { ... }
```

**Audit log requirements:**
- Write-once (append-only, no updates/deletes)
- Tamper-evident (hash chain or write to separate write-only DB account)
- Retained per compliance (SOC2: 1 year, PCI-DSS: 1 year minimum)
- Never log sensitive data: passwords, full card numbers, SSNs

---

## Question 19: What is mTLS (mutual TLS) and when would you use it?

**Standard TLS:** Client verifies server's certificate. Server doesn't verify client.

**mTLS:** Both sides present and verify certificates.

```
Standard TLS:             mTLS:
Client ←cert— Server     Client ←cert→ Server
        verified                both verified
```

**Use cases:**
- Service-to-service authentication in microservices (zero-trust networking)
- Replacing API keys for machine-to-machine authentication
- PCI-DSS and HIPAA requirements for internal service communication

```java
// Spring Boot — configure mTLS on the server
# application.yml
server:
  ssl:
    key-store: classpath:server-keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    trust-store: classpath:client-truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    client-auth: need  # require client certificate (vs "want" for optional)

// Extracting client certificate identity
@Component
public class MtlsAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        X509Certificate[] certs = (X509Certificate[])
            request.getAttribute("javax.servlet.request.X509Certificate");
        if (certs != null && certs.length > 0) {
            String cn = extractCN(certs[0].getSubjectX500Principal().getName());
            // Set as authentication principal
        }
    }
}
```

---

## Question 20: What are the security implications of Spring Boot Actuator?

**Risk:** Actuator exposes application internals — `/env` (environment variables including secrets), `/heapdump` (heap dump with all in-memory data), `/shutdown`, `/beans`, `/mappings`.

```java
// Secure Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus  # whitelist only needed endpoints
        # NEVER expose: env, heapdump, threaddump, shutdown in production
  endpoint:
    health:
      show-details: when_authorized  # full details only for authenticated users
      show-components: when_authorized
    shutdown:
      enabled: false  # explicit disable
  server:
    port: 8081  # separate port, only accessible internally (not via load balancer)

// Require authentication for all actuator endpoints
@Bean
public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
    http
        .securityMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(EndpointRequest.to(HealthEndpoint.class)).permitAll()
            .anyRequest().hasRole("ACTUATOR_ADMIN")
        )
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

**Production checklist:**
- Run actuator on internal port only (not exposed via load balancer)
- Authenticate all non-health endpoints
- Never expose `/env`, `/heapdump`, `/threaddump` publicly
- `/shutdown` should be disabled or only accessible from localhost

