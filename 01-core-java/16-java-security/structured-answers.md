# Java Security — Structured Answers

---

## Q1: How Does BCrypt Password Hashing Work? Show Java Implementation

BCrypt is a password hashing function designed to be computationally expensive. It uses a cost factor (work factor) to control how many iterations of hashing are performed. As hardware gets faster, you increase the cost factor to keep brute force impractical.

**How BCrypt works:**
1. Generates a random 128-bit salt
2. Uses the Blowfish cipher as the core hashing algorithm
3. Performs `2^cost` iterations
4. Returns the hash string containing the algorithm version, cost factor, salt, and hash — all in one string

```java
// Maven dependency:
// org.springframework.security:spring-security-crypto
// or org.mindrot:jbcrypt

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

@Service
public class PasswordService {
    // cost factor 12 = ~400ms on modern hardware
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);

    public String hashPassword(String rawPassword) {
        // Each call produces a different hash due to random salt
        String hash = encoder.encode(rawPassword);
        // Example: $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
        return hash;
    }

    public boolean verifyPassword(String rawPassword, String storedHash) {
        // BCrypt extracts the salt from storedHash, re-hashes rawPassword,
        // and compares in constant time
        return encoder.matches(rawPassword, storedHash);
    }
}

// Spring Boot auto-configuration
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}

// User registration
@PostMapping("/register")
public ResponseEntity<Void> register(@RequestBody RegisterRequest req) {
    String hashedPassword = passwordEncoder.encode(req.getPassword());
    userRepository.save(new User(req.getUsername(), hashedPassword));
    return ResponseEntity.status(201).build();
}

// Login check
@PostMapping("/login")
public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest req) {
    User user = userRepository.findByUsername(req.getUsername())
        .orElseThrow(() -> new BadCredentialsException("Invalid credentials"));

    if (!passwordEncoder.matches(req.getPassword(), user.getPasswordHash())) {
        throw new BadCredentialsException("Invalid credentials");
    }
    // Generate and return JWT
    String token = jwtService.generateToken(user);
    return ResponseEntity.ok(new TokenResponse(token));
}
```

---

## Q2: How Do You Implement JWT Authentication in Spring Boot?

```java
// 1. JWT Utility class
@Component
public class JwtUtil {
    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiry-ms}")
    private long expiryMs;

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority).collect(Collectors.toList()));

        return Jwts.builder()
            .setClaims(claims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiryMs))
            .signWith(SignatureAlgorithm.HS256, secret.getBytes())
            .compact();
    }

    public String extractUsername(String token) {
        return Jwts.parser()
            .setSigningKey(secret.getBytes())
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    public boolean isTokenValid(String token) {
        try {
            Jwts.parser().setSigningKey(secret.getBytes()).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}

// 2. JWT Request Filter
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    @Autowired private JwtUtil jwtUtil;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = req.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            if (jwtUtil.isTokenValid(token)) {
                String username = jwtUtil.extractUsername(token);
                UserDetails user = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(req, res);
    }
}

// 3. Security configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired private JwtAuthFilter jwtAuthFilter;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
                .antMatchers("/auth/**").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

---

## Q3: What Is OAuth2 and How Does the Authorization Code Flow Work?

OAuth2 is an authorization framework that allows a third-party application to obtain limited access to a resource on behalf of a user, without the user sharing their credentials.

**Key roles:**
- **Resource Owner:** the user
- **Client:** your application
- **Authorization Server:** Google, GitHub, Okta (handles login and token issuance)
- **Resource Server:** the API with the user's data

**Spring Boot OAuth2 Authorization Code implementation:**

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2Config extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
            )
            .authorizeRequests()
                .antMatchers("/login", "/error").permitAll()
                .anyRequest().authenticated();
    }
}

@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    @Autowired private UserRepository userRepo;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest request) throws OAuth2AuthenticationException {
        OAuth2User oauthUser = super.loadUser(request);
        String email = oauthUser.getAttribute("email");

        // Provision user if first login
        userRepo.findByEmail(email).orElseGet(() ->
            userRepo.save(new AppUser(email, oauthUser.getAttribute("name")))
        );
        return oauthUser;
    }
}
```

---

## Q4: How Do You Prevent SQL Injection in Java? Show PreparedStatement Example

```java
// UNSAFE — never do this
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = '" + username + "'";
    Statement stmt = connection.createStatement();
    ResultSet rs = stmt.executeQuery(sql);  // SQL injection possible
    // ...
}

// SAFE — PreparedStatement
public User findUser(String username) throws SQLException {
    String sql = "SELECT id, username, email FROM users WHERE username = ?";
    try (PreparedStatement ps = connection.prepareStatement(sql)) {
        ps.setString(1, username);           // parameter bound as data
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return new User(rs.getLong("id"), rs.getString("username"), rs.getString("email"));
            }
            return null;
        }
    }
}

// With Spring JdbcTemplate (preferred in Spring apps)
@Repository
public class UserRepository {
    @Autowired private JdbcTemplate jdbc;

    public Optional<User> findByUsername(String username) {
        String sql = "SELECT id, username, email FROM users WHERE username = ?";
        return jdbc.query(sql, userRowMapper(), username).stream().findFirst();
    }

    public List<User> findByAgeRange(int minAge, int maxAge) {
        return jdbc.query(
            "SELECT * FROM users WHERE age BETWEEN ? AND ?",
            userRowMapper(),
            minAge, maxAge
        );
    }

    private RowMapper<User> userRowMapper() {
        return (rs, rowNum) -> new User(
            rs.getLong("id"),
            rs.getString("username"),
            rs.getString("email")
        );
    }
}

// With JPA (also parameterized)
@Repository
public interface UserJpaRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.username = :username")
    Optional<User> findByUsername(@Param("username") String username);
    // Spring Data automatically uses parameterized query
}
```

**Also prevent second-order injection:** sanitize data when reading FROM the database if it will be used in subsequent queries, not just on initial insertion.

---

## Q5: What Is XSS and How Do You Prevent It in a Spring Boot App?

XSS (Cross-Site Scripting) allows attackers to inject malicious scripts into web pages viewed by other users.

```java
// PREVENTION IN SPRING BOOT

// 1. Thymeleaf auto-escapes by default
// th:text escapes HTML — safe
// th:utext does NOT escape — only use for trusted HTML
<p th:text="${userComment}"><!-- safe: &lt;script&gt; becomes visible text --></p>
<p th:utext="${trustedHtml}"><!-- DANGEROUS if userComment is user input --></p>

// 2. OWASP Java Encoder library for explicit encoding
import org.owasp.encoder.Encode;

@GetMapping("/search")
public String search(@RequestParam String query, Model model) {
    model.addAttribute("query", Encode.forHtml(query));  // explicit HTML encoding
    return "search";
}

// 3. Content Security Policy header
@Configuration
public class SecurityHeadersConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.headers()
            .contentSecurityPolicy(
                "default-src 'self'; " +
                "script-src 'self' 'nonce-{nonce}'; " +
                "style-src 'self'; " +
                "img-src 'self' data:; " +
                "frame-ancestors 'none'"
            )
            .and()
            .xssProtection().block(true)
            .and()
            .frameOptions().deny();
    }
}

// 4. Input validation on server side
@PostMapping("/comment")
public ResponseEntity<Void> postComment(@RequestBody @Valid CommentRequest req) {
    // @Valid triggers validation
    commentService.save(req.getContent());
    return ResponseEntity.ok().build();
}

public class CommentRequest {
    @NotBlank
    @Size(max = 1000)
    // Do NOT use @Pattern to try to block XSS — blacklists can be bypassed
    // Instead, encode on output (see above)
    private String content;
}
```

**Key principle:** encode on output, not on input. Encoding on input is fragile (you might need raw data later, double-encoding issues). Always encode at the point of rendering.

---

## Q6: How Do You Prevent CSRF Attacks? When Can You Disable CSRF?

```java
// Spring Security CSRF protection is ENABLED by default for stateful apps

@Configuration
public class CsrfConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // Stores CSRF token in a cookie readable by JS (so your JS can include it)
                // But the actual validation checks the X-XSRF-TOKEN header
            );
    }
}

// In your forms (Thymeleaf automatically includes CSRF):
<form th:action="@{/transfer}" method="post">
    <!-- Thymeleaf adds: <input type="hidden" name="_csrf" value="token"> -->
    <input type="text" name="amount">
    <button type="submit">Transfer</button>
</form>

// REST API with Angular/React: read token from cookie, send in header
// Angular HttpClient does this automatically (XSRF-TOKEN cookie -> X-XSRF-TOKEN header)

// When it's SAFE to disable CSRF:
@Configuration
public class RestApiSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .csrf().disable()  // SAFE if using JWT in Authorization header (not cookies)
            // Reason: CSRF exploits cookie auto-inclusion
            // If there are no session cookies, CSRF is not applicable
            .authorizeRequests()
                .antMatchers("/api/**").authenticated();
    }
}
```

**When to keep CSRF enabled:**
- Traditional MVC apps with cookie-based sessions and HTML forms
- Any app that uses cookies for authentication

**When it's safe to disable:**
- Purely stateless REST APIs using JWT in Authorization header
- APIs consumed only by mobile apps (no browser, no cookie)

---

## Q7: What Is the Difference Between Symmetric and Asymmetric Encryption?

```java
// SYMMETRIC ENCRYPTION (AES) — same key to encrypt and decrypt
import javax.crypto.*;
import javax.crypto.spec.*;

public class AesExample {
    public static byte[] encrypt(String plaintext, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
        byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
        // Prepend IV to ciphertext for use during decryption
        byte[] result = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, result, 0, iv.length);
        System.arraycopy(ciphertext, 0, result, iv.length, ciphertext.length);
        return result;
    }

    public static SecretKey generateKey() throws Exception {
        KeyGenerator kg = KeyGenerator.getInstance("AES");
        kg.init(256);
        return kg.generateKey();
    }
}

// ASYMMETRIC ENCRYPTION (RSA) — public key encrypts, private key decrypts
public class RsaExample {
    public static KeyPair generateKeyPair() throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
        kpg.initialize(2048);
        return kpg.generateKeyPair();
    }

    public static byte[] encrypt(String plaintext, PublicKey publicKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
    }

    public static String decrypt(byte[] ciphertext, PrivateKey privateKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return new String(cipher.doFinal(ciphertext), StandardCharsets.UTF_8);
    }
}
```

| Aspect | Symmetric (AES) | Asymmetric (RSA) |
|---|---|---|
| Keys | One shared key | Public + private key pair |
| Speed | Very fast | Much slower (~1000x) |
| Key distribution | Hard (how to share the key securely?) | Easy (public key can be shared freely) |
| Use case | Bulk data encryption | Key exchange, digital signatures, TLS handshake |
| Key size | 128 or 256 bits | 2048 or 4096 bits |

TLS uses **both**: asymmetric encryption to establish a shared secret, then symmetric encryption for the data stream.

---

## Q8: How Do You Store Secrets Securely?

**Never do:**
```java
// NEVER hardcode secrets
private static final String DB_PASSWORD = "mypassword123";
private static final String API_KEY = "sk-1234567890abcdef";
```

**Spring Boot — environment variables:**
```yaml
# application.yml — use placeholders
spring:
  datasource:
    password: ${DB_PASSWORD}    # read from env var at runtime

jwt:
  secret: ${JWT_SECRET}
```

**Spring Cloud Config + Vault:**
```yaml
# bootstrap.yml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        default-context: myapp
```

```java
@Value("${secret.db-password}")   // loaded from Vault at startup
private String dbPassword;
```

**AWS Secrets Manager integration:**
```java
@Configuration
public class AwsSecretsConfig {
    @Bean
    public DataSource dataSource() {
        AWSSecretsManager client = AWSSecretsManagerClientBuilder.standard()
            .withRegion("us-east-1")
            .build();

        GetSecretValueResult result = client.getSecretValue(
            new GetSecretValueRequest().withSecretId("prod/myapp/db")
        );

        // Parse JSON secret
        ObjectMapper mapper = new ObjectMapper();
        Map<String, String> secret = mapper.readValue(result.getSecretString(), Map.class);

        return DataSourceBuilder.create()
            .url(secret.get("host"))
            .username(secret.get("username"))
            .password(secret.get("password"))
            .build();
    }
}
```

**Rules for secrets:**
1. Never in source code or version control
2. Never in `application.properties` that gets committed (use `application-local.properties` in `.gitignore`)
3. Different secrets per environment (dev/staging/prod)
4. Rotate secrets regularly
5. Use IAM roles for AWS service-to-service authentication (no static keys at all)

---

## Q9: What Are the OWASP Top 10 Vulnerabilities Relevant to Java Backends?

1. **Broken Access Control** — users accessing resources they shouldn't. Fix: always authorize server-side, never trust client claims.
   ```java
   @PreAuthorize("@securityService.canAccess(authentication, #orderId)")
   @GetMapping("/orders/{orderId}")
   public Order getOrder(@PathVariable Long orderId) { ... }
   ```

2. **Cryptographic Failures** — MD5/SHA1 for passwords, HTTP instead of HTTPS, secrets in logs.
   Fix: BCrypt for passwords, TLS everywhere, never log sensitive data.

3. **Injection** — SQL, LDAP, OS command injection.
   Fix: PreparedStatement, parameterized queries, input validation.

4. **Insecure Design** — no rate limiting, no account lockout, predictable tokens.
   Fix: threat model during design, not just coding.

5. **Security Misconfiguration** — default credentials, directory listing, verbose error messages.
   Fix: custom error handlers, disable unnecessary endpoints.
   ```java
   @ControllerAdvice
   public class GlobalExceptionHandler {
       @ExceptionHandler(Exception.class)
       public ResponseEntity<ErrorResponse> handleAll(Exception e) {
           log.error("Unhandled exception", e);
           // Return generic message — don't expose stack traces
           return ResponseEntity.status(500).body(new ErrorResponse("Internal server error"));
       }
   }
   ```

6. **Vulnerable and Outdated Components** — old Spring Boot, old Jackson, Log4Shell (Log4j CVE-2021-44228).
   Fix: use dependency scanning (OWASP Dependency Check, Snyk, Dependabot).

7. **Identification and Authentication Failures** — weak passwords, no MFA, predictable session IDs.
   Fix: BCrypt, JWT, Spring Security's strong defaults.

8. **Software and Data Integrity Failures** — insecure deserialization, unsigned updates.
   Fix: avoid Java native serialization with untrusted data; verify signatures on updates.

9. **Security Logging and Monitoring Failures** — no audit logs, no alerting.
   Fix: log authentication events, access denials, and data modifications.
   ```java
   @EventListener
   public void onAuthenticationFailure(AuthenticationFailureBadCredentialsEvent event) {
       log.warn("Failed login attempt for user: {}", event.getAuthentication().getName());
       // Optionally: increment failed-attempt counter, trigger lockout
   }
   ```

10. **Server-Side Request Forgery (SSRF)** — server makes requests to URLs controlled by user, potentially reaching internal services.
    Fix: validate and allowlist URLs your server fetches.

---

## Q10: How Do You Implement Rate Limiting in Spring Boot?

**Approach 1: Using Bucket4j (token bucket algorithm)**
```java
// Maven: com.github.vladimir-bukhtoyarov:bucket4j-core

@Component
public class RateLimitFilter extends OncePerRequestFilter {
    // 100 requests per minute per IP
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket getBucketForIp(String ip) {
        return buckets.computeIfAbsent(ip, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
                .build()
        );
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String ip = req.getRemoteAddr();
        Bucket bucket = getBucketForIp(ip);

        if (bucket.tryConsume(1)) {
            chain.doFilter(req, res);
        } else {
            res.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            res.setHeader("X-Rate-Limit-Retry-After-Seconds",
                String.valueOf(bucket.estimateAbilityToConsume(1).getNanosToWaitForRefill() / 1_000_000_000));
            res.getWriter().write("Too many requests");
        }
    }
}
```

**Approach 2: Spring Boot Starter Rate Limiting with Redis (distributed)**
```java
@Configuration
public class RateLimitConfig {
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        // 5 requests per second replenish rate, burst capacity of 10
        return new RedisRateLimiter(5, 10, 1);
    }
}

// In Spring Cloud Gateway (for API gateway rate limiting):
@Bean
public RouteLocator routes(RouteLocatorBuilder builder,
                           RedisRateLimiter rateLimiter) {
    return builder.routes()
        .route("api", r -> r.path("/api/**")
            .filters(f -> f.requestRateLimiter(c -> c
                .setRateLimiter(rateLimiter)
                .setKeyResolver(exchange -> Mono.just(
                    exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                ))
            ))
            .uri("lb://API-SERVICE"))
        .build();
}
```

**Approach 3: Simple in-memory with Resilience4j**
```java
@RateLimiter(name = "default", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/data")
public ResponseEntity<Data> getData() {
    return ResponseEntity.ok(dataService.fetchData());
}

public ResponseEntity<Data> rateLimitFallback(RequestNotPermitted e) {
    return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).build();
}
```

```yaml
# application.yml
resilience4j:
  ratelimiter:
    instances:
      default:
        limitForPeriod: 50          # 50 calls per period
        limitRefreshPeriod: 1s      # period = 1 second
        timeoutDuration: 0          # don't wait — fail immediately
```

**For production distributed rate limiting**, use Redis-backed Bucket4j or Spring Cloud Gateway's rate limiter. In-memory rate limiters don't work correctly across multiple pod replicas.
