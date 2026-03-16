# Java Security — Scenario Questions

---

## Scenario 1: Secure User Registration and Login Flow

**Setup:** Design a secure user registration and login system. Cover password storage, account enumeration prevention, and login attempt tracking.

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @PostMapping("/register")
    public ResponseEntity<Void> register(@Valid @RequestBody RegisterRequest request) {
        // Check for existing user — but don't reveal whether email exists
        if (userService.existsByEmail(request.email())) {
            // Return 200 anyway to prevent account enumeration
            // Send "if this email is new, check your inbox" email
            emailService.sendRegistrationEmail(request.email());
            return ResponseEntity.ok().build();
        }

        User user = userService.createUser(request.email(),
            passwordEncoder.encode(request.password()));
        emailService.sendVerificationEmail(user);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request,
                                               HttpServletRequest httpRequest) {
        String ip = extractClientIp(httpRequest);

        // Check if IP is blocked
        if (loginAttemptService.isBlocked(ip)) {
            // Don't reveal if it's blocked — same response as wrong credentials
            return ResponseEntity.status(401)
                .body(TokenResponse.error("Invalid credentials"));
        }

        try {
            Authentication auth = authManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.email(), request.password())
            );
            loginAttemptService.recordSuccess(ip);

            String accessToken = tokenService.generateAccessToken(auth);
            String refreshToken = tokenService.generateRefreshToken(auth);

            // Store refresh token as HttpOnly cookie
            ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken)
                .httpOnly(true)
                .secure(true)
                .sameSite("Strict")
                .path("/api/auth/refresh")
                .maxAge(Duration.ofDays(30))
                .build();

            return ResponseEntity.ok()
                .header(HttpHeaders.SET_COOKIE, cookie.toString())
                .body(TokenResponse.success(accessToken, 3600));

        } catch (BadCredentialsException e) {
            loginAttemptService.recordFailure(ip);
            return ResponseEntity.status(401)
                .body(TokenResponse.error("Invalid credentials"));
        }
    }
}
```

**Account enumeration prevention:** All responses (success, user-not-found, wrong-password) return identical HTTP status and response body. Attackers cannot distinguish between "email doesn't exist" and "wrong password."

---

## Scenario 2: JWT Token Refresh with Rotation

**Setup:** Implement refresh token rotation — when a client uses a refresh token, invalidate it and issue a new one. Detect token reuse (sign of theft).

```java
@Service
public class RefreshTokenService {

    private final RefreshTokenRepository tokenRepository;
    private final JwtTokenService jwtService;

    @Transactional
    public TokenPair refresh(String refreshToken) {
        RefreshToken stored = tokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new InvalidTokenException("Refresh token not found"));

        // Detect token reuse — indicates token theft
        if (stored.isRevoked()) {
            // A revoked token was used → the token family was compromised
            // Revoke ALL tokens for this user (security response)
            tokenRepository.revokeAllForUser(stored.getUserId());
            auditService.log("REFRESH_TOKEN_REUSE_DETECTED", stored.getUserId());
            throw new SecurityException("Token reuse detected — all sessions invalidated");
        }

        if (stored.getExpiresAt().isBefore(Instant.now())) {
            throw new InvalidTokenException("Refresh token expired");
        }

        // Rotate: revoke old, issue new
        stored.revoke();
        tokenRepository.save(stored);

        String newAccessToken = jwtService.generateAccessToken(stored.getUserId());
        String newRefreshToken = generateSecureToken();

        // Store new refresh token linked to same family
        tokenRepository.save(RefreshToken.builder()
            .token(newRefreshToken)
            .userId(stored.getUserId())
            .familyId(stored.getFamilyId())  // track lineage for reuse detection
            .expiresAt(Instant.now().plus(30, ChronoUnit.DAYS))
            .build());

        return new TokenPair(newAccessToken, newRefreshToken);
    }

    private String generateSecureToken() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

---

## Scenario 3: Row-Level Security for Multi-Tenant Data

**Setup:** Your SaaS platform must ensure Tenant A can never access Tenant B's data, even if there's a bug in query logic.

```java
// Approach 1: Database-level row security (PostgreSQL RLS)
// SQL executed at startup per-schema:
// CREATE POLICY tenant_isolation ON orders
//     USING (tenant_id = current_setting('app.tenant_id')::bigint);
// ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

// Spring sets the tenant ID for each connection
@Component
public class TenantContextFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String tenantId = extractTenantId(request);
        TenantContext.setCurrentTenant(tenantId);
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}

// Set tenant ID on each JDBC connection before use
@Component
public class TenantAwareDataSource extends AbstractDataSource {
    private final DataSource delegate;

    @Override
    public Connection getConnection() throws SQLException {
        Connection conn = delegate.getConnection();
        try (Statement stmt = conn.createStatement()) {
            String tenantId = TenantContext.getCurrentTenant();
            // PostgreSQL session variable
            stmt.execute("SET app.tenant_id = " + Long.parseLong(tenantId));
        }
        return conn;
    }
}

// Approach 2: Application-level — always include tenant_id in queries
@Repository
public class OrderRepository {

    public List<Order> findAll() {
        Long tenantId = TenantContext.getCurrentTenantId();
        // tenantId is ALWAYS added, even if developer forgets it in the WHERE clause
        return jdbcTemplate.query(
            "SELECT * FROM orders WHERE tenant_id = ?",
            orderRowMapper,
            tenantId
        );
    }
}
```

**Defense in depth:** Combine application-level filtering (most cases) with database RLS (safety net). Even if a developer writes `SELECT * FROM orders` without a tenant filter, RLS blocks cross-tenant access.

---

## Scenario 4: Secure File Upload with Content Validation

**Setup:** Build a file upload endpoint that validates file type using magic bytes (not just content-type header), scans for malware patterns, and stores securely.

```java
@Service
public class SecureFileUploadService {

    // Magic bytes for allowed types
    private static final Map<byte[], String> MAGIC_BYTES = Map.of(
        new byte[]{(byte)0xFF, (byte)0xD8, (byte)0xFF}, "image/jpeg",
        new byte[]{(byte)0x89, 0x50, 0x4E, 0x47}, "image/png",
        new byte[]{0x25, 0x50, 0x44, 0x46}, "application/pdf"
    );

    public FileUploadResult upload(MultipartFile file, Long userId) throws IOException {
        byte[] content = file.getBytes();

        // 1. Validate file size
        if (content.length > 10 * 1024 * 1024) {
            throw new FileTooLargeException("File exceeds 10MB limit");
        }

        // 2. Validate magic bytes — cannot be spoofed like Content-Type header
        String detectedType = detectMimeType(content);
        if (detectedType == null) {
            throw new UnsupportedFileTypeException("File type not allowed");
        }

        // 3. Sanitize filename — prevent path traversal and special characters
        String originalFilename = file.getOriginalFilename();
        String safeFilename = sanitizeFilename(originalFilename);

        // 4. Generate a new random filename — don't use user-supplied name for storage
        String storedFilename = UUID.randomUUID() + getExtension(detectedType);

        // 5. Store in location NOT accessible via web server directly
        Path uploadDir = Paths.get("/var/data/uploads").toAbsolutePath();
        Path targetPath = uploadDir.resolve(storedFilename);

        // 6. Set restrictive permissions
        Files.write(targetPath, content);
        Files.setPosixFilePermissions(targetPath,
            Set.of(PosixFilePermission.OWNER_READ, PosixFilePermission.OWNER_WRITE));

        // 7. Async virus scan
        virusScanQueue.submit(new ScanJob(storedFilename, userId));

        return new FileUploadResult(storedFilename, safeFilename, detectedType);
    }

    private String detectMimeType(byte[] content) {
        for (Map.Entry<byte[], String> entry : MAGIC_BYTES.entrySet()) {
            byte[] magic = entry.getKey();
            if (content.length >= magic.length) {
                boolean match = true;
                for (int i = 0; i < magic.length; i++) {
                    if (content[i] != magic[i]) { match = false; break; }
                }
                if (match) return entry.getValue();
            }
        }
        return null;
    }

    private String sanitizeFilename(String filename) {
        if (filename == null) return "upload";
        // Remove path components, null bytes, and dangerous characters
        return Paths.get(filename).getFileName().toString()
            .replaceAll("[^a-zA-Z0-9._-]", "_")
            .replaceAll("^\\.+", "_");  // prevent hidden files starting with dot
    }
}
```

---

## Scenario 5: API Key Authentication with Scope-Based Authorization

**Setup:** Implement API key authentication for third-party integrations. Keys must be hashed in the database, support scopes, and have expiry.

```java
// API Key entity — store hash, not raw key
@Entity
public class ApiKey {
    @Id @GeneratedValue private Long id;
    @Column(unique = true) private String keyHash;  // BCrypt hash
    private String name;
    private Long ownerId;
    @ElementCollection
    private Set<String> scopes;  // e.g., "orders:read", "orders:write"
    private LocalDateTime expiresAt;
    private boolean revoked;
    private Instant lastUsedAt;
}

// Generation — return raw key once, never again
@Service
public class ApiKeyService {

    public ApiKeyCreationResult generateKey(Long userId, String name, Set<String> scopes,
                                            Duration ttl) {
        // Generate a cryptographically secure key
        byte[] rawBytes = new byte[32];
        new SecureRandom().nextBytes(rawBytes);
        String rawKey = "sk_" + Base64.getUrlEncoder().withoutPadding()
            .encodeToString(rawBytes);

        // Store hash
        ApiKey apiKey = new ApiKey();
        apiKey.setKeyHash(passwordEncoder.encode(rawKey));
        apiKey.setName(name);
        apiKey.setOwnerId(userId);
        apiKey.setScopes(scopes);
        apiKey.setExpiresAt(LocalDateTime.now().plus(ttl));
        apiKeyRepository.save(apiKey);

        // Return raw key to user — this is the ONLY time it's shown
        return new ApiKeyCreationResult(apiKey.getId(), rawKey);
    }

    public Optional<ApiKey> validateKey(String rawKey) {
        // Iterate all non-expired, non-revoked keys and check hash
        // This is O(n) — in production, use the first few chars as lookup index
        return apiKeyRepository.findAllActive().stream()
            .filter(k -> passwordEncoder.matches(rawKey, k.getKeyHash()))
            .peek(k -> k.setLastUsedAt(Instant.now()))
            .findFirst();
    }
}

// Authentication filter
@Component
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey == null) {
            chain.doFilter(request, response);
            return;
        }

        apiKeyService.validateKey(apiKey).ifPresentOrElse(
            key -> {
                List<GrantedAuthority> authorities = key.getScopes().stream()
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toList());
                SecurityContextHolder.getContext().setAuthentication(
                    new PreAuthenticatedAuthenticationToken(key.getOwnerId(), null, authorities)
                );
                chain.doFilter(request, response);
            },
            () -> {
                response.setStatus(401);
                response.getWriter().write("{\"error\":\"Invalid or expired API key\"}");
            }
        );
    }
}
```

---

## Scenario 6: OWASP Mass Assignment Prevention

**Setup:** A user update endpoint receives a JSON body. An attacker sends additional fields like `role`, `isAdmin`, `balance` that should not be user-settable.

```java
// VULNERABLE — binding all request fields to entity
@PutMapping("/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody User user) {
    // Attacker sends: {"name": "Alice", "role": "ADMIN", "balance": 999999}
    // All fields get set on the entity!
    return userRepository.save(user);
}

// SAFE approach 1: Use separate request DTO with only allowed fields
public record UpdateUserRequest(
    @NotBlank @Size(max = 100) String firstName,
    @NotBlank @Size(max = 100) String lastName,
    @Email String email
    // No 'role', 'isAdmin', 'balance' — they simply can't be set via this DTO
) {}

@PutMapping("/users/{id}")
public UserResponse updateUser(@PathVariable Long id,
                               @Valid @RequestBody UpdateUserRequest request,
                               @AuthenticatedUser UserProfile currentUser) {
    if (!currentUser.getId().equals(id)) {
        throw new ForbiddenException("Cannot update another user's profile");
    }
    User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User", id));

    // Only map the allowed fields
    user.setFirstName(request.firstName());
    user.setLastName(request.lastName());
    user.setEmail(request.email());

    return mapper.toResponse(userRepository.save(user));
}

// SAFE approach 2: @JsonIgnoreProperties on entity for serialization
@Entity
@JsonIgnoreProperties({"role", "passwordHash", "isAdmin"})
public class User { ... }
```

---

## Scenario 7: Secure Logging — Mask PII and Sensitive Data

**Setup:** Application logs must not contain passwords, credit card numbers, SSNs, or other PII. Implement a log sanitization mechanism.

```java
// Custom Logback converter that masks sensitive fields
public class SensitiveDataMaskingConverter extends ClassicConverter {

    private static final Pattern CREDIT_CARD = Pattern.compile(
        "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})\\b"
    );
    private static final Pattern SSN = Pattern.compile(
        "\\b\\d{3}-\\d{2}-\\d{4}\\b"
    );
    private static final Pattern PASSWORD_FIELD = Pattern.compile(
        "(?i)(\"password\"\\s*:\\s*\")([^\"]+)(\")",
        Pattern.CASE_INSENSITIVE
    );
    private static final Pattern BEARER_TOKEN = Pattern.compile(
        "(?i)(bearer\\s+)([A-Za-z0-9._-]{10,})"
    );

    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        message = CREDIT_CARD.matcher(message).replaceAll("****-****-****-$0".substring(20));
        message = SSN.matcher(message).replaceAll("***-**-****");
        message = PASSWORD_FIELD.matcher(message).replaceAll("$1[REDACTED]$3");
        message = BEARER_TOKEN.matcher(message).replaceAll("$1[TOKEN]");
        return message;
    }
}

// logback-spring.xml
// <conversionRule conversionWord="maskedMsg" converterClass="...SensitiveDataMaskingConverter"/>
// <pattern>%d{ISO8601} %-5level [%X{correlationId}] %logger{36} - %maskedMsg%n</pattern>

// Structured logging with explicit field filtering
@Service
public class AuditLogger {

    private static final Logger log = LoggerFactory.getLogger(AuditLogger.class);

    public void logUserAction(String action, Long userId, Map<String, Object> details) {
        // Remove sensitive fields before logging
        Map<String, Object> safeDetails = new HashMap<>(details);
        safeDetails.remove("password");
        safeDetails.remove("cardNumber");
        safeDetails.remove("ssn");

        // Mask remaining PII
        if (safeDetails.containsKey("email")) {
            safeDetails.put("email", maskEmail((String) safeDetails.get("email")));
        }

        log.info("ACTION={} userId={} details={}", action, userId, safeDetails);
    }

    private String maskEmail(String email) {
        int atIdx = email.indexOf('@');
        if (atIdx <= 1) return "***@" + email.substring(atIdx + 1);
        return email.charAt(0) + "***" + email.charAt(atIdx - 1) + email.substring(atIdx);
    }
}
```

---

## Scenario 8: SQL Injection in Dynamic Search with JPA Criteria API

**Setup:** A search endpoint builds a query dynamically based on user-provided filters. Prevent SQL injection while supporting flexible filtering.

```java
// VULNERABLE — string concatenation in JPQL
@Query("SELECT o FROM Order o WHERE o.status = '" + status + "'")  // INJECT!

// SAFE — Criteria API with type-safe, parameterized queries
@Service
public class OrderSearchService {

    @PersistenceContext
    private EntityManager em;

    public Page<Order> search(OrderSearchRequest request, Pageable pageable) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> query = cb.createQuery(Order.class);
        Root<Order> root = query.from(Order.class);

        List<Predicate> predicates = new ArrayList<>();

        // All inputs are bound as parameters — never interpolated into SQL
        if (request.getStatus() != null) {
            predicates.add(cb.equal(root.get("status"),
                OrderStatus.valueOf(request.getStatus())));  // enum validation — invalid values rejected
        }

        if (request.getMinAmount() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                root.get("totalAmount"), request.getMinAmount()));
        }

        if (request.getKeyword() != null && !request.getKeyword().isBlank()) {
            String pattern = "%" + escapeLikePattern(request.getKeyword()) + "%";
            predicates.add(cb.like(
                cb.lower(root.get("description")),
                pattern.toLowerCase()
            ));
        }

        if (request.getFromDate() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                root.get("createdAt"), request.getFromDate().atStartOfDay()));
        }

        query.where(cb.and(predicates.toArray(new Predicate[0])));

        // Add sorting — validate sort field is in allowed list
        if (pageable.getSort().isSorted()) {
            List<javax.persistence.criteria.Order> orders = pageable.getSort().stream()
                .filter(sort -> ALLOWED_SORT_FIELDS.contains(sort.getProperty()))
                .map(sort -> sort.isAscending()
                    ? cb.asc(root.get(sort.getProperty()))
                    : cb.desc(root.get(sort.getProperty())))
                .toList();
            query.orderBy(orders);
        }

        List<Order> results = em.createQuery(query)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();

        return new PageImpl<>(results, pageable, countResults(predicates));
    }

    private static final Set<String> ALLOWED_SORT_FIELDS =
        Set.of("createdAt", "totalAmount", "status", "id");

    // Escape LIKE wildcards to prevent pattern injection
    private String escapeLikePattern(String input) {
        return input.replace("\\", "\\\\")
                    .replace("%", "\\%")
                    .replace("_", "\\_");
    }
}
```

---

## Scenario 9: Implementing CORS Correctly

**Setup:** Configure CORS for a REST API that is consumed by a React SPA on a different domain. Be restrictive — not `*`.

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(allowedOrigins.toArray(String[]::new))
            // NEVER use allowedOrigins("*") when allowCredentials(true)
            // Browsers reject this combination
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization", "X-Correlation-ID",
                "X-Requested-With")
            .exposedHeaders("X-Correlation-ID", "X-RateLimit-Remaining")
            .allowCredentials(true)  // needed for cookies (HttpOnly refresh token)
            .maxAge(3600);  // preflight cache: 1 hour
    }
}

// For fine-grained control — Spring Security CORS filter (runs before authentication)
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(allowedOrigins);
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// Configure in Spring Security chain
http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

**CORS security notes:**
- `allowedOrigins("*")` + `allowCredentials(true)` = browser error (browsers reject this)
- Wildcard origin means any site can make credentialed requests → session riding attack
- Always specify exact origins in production
- OPTIONS preflight requests must return 200 (not 401) — place CORS filter before security filter

---

## Scenario 10: Implement a Security Audit Trail

**Setup:** PCI-DSS requires auditing all access to cardholder data. Log who accessed what, when, from where, and with what outcome.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuditAccess {
    String resource();
    String action();
    boolean sensitiveData() default false;
}

@Aspect
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 1)  // run last, after auth
public class AuditAspect {

    private static final Logger auditLog =
        LoggerFactory.getLogger("AUDIT_LOG");  // separate appender in logback

    @Around("@annotation(audit)")
    public Object auditAccess(ProceedingJoinPoint pjp, AuditAccess audit) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String userId = auth != null ? auth.getName() : "anonymous";
        String ip = RequestContextHolder.getRequestAttributes() instanceof
            ServletRequestAttributes sra
            ? sra.getRequest().getRemoteAddr() : "unknown";

        String resourceId = extractResourceId(pjp.getArgs());
        Instant start = Instant.now();

        try {
            Object result = pjp.proceed();

            auditLog.info("""
                AUDIT event={} resource={} resourceId={} userId={} ip={} outcome=SUCCESS \
                durationMs={}""",
                audit.action(), audit.resource(), resourceId, userId, ip,
                Duration.between(start, Instant.now()).toMillis()
            );

            return result;
        } catch (AccessDeniedException e) {
            auditLog.warn("""
                AUDIT event={} resource={} resourceId={} userId={} ip={} outcome=ACCESS_DENIED""",
                audit.action(), audit.resource(), resourceId, userId, ip
            );
            throw e;
        } catch (Exception e) {
            auditLog.error("""
                AUDIT event={} resource={} resourceId={} userId={} ip={} outcome=ERROR error={}""",
                audit.action(), audit.resource(), resourceId, userId, ip, e.getMessage()
            );
            throw e;
        }
    }
}

// Usage
@GetMapping("/payments/{id}")
@PreAuthorize("hasAuthority('PAYMENT_READ')")
@AuditAccess(resource = "Payment", action = "READ", sensitiveData = true)
public PaymentResponse getPayment(@PathVariable Long id) {
    return paymentService.findById(id);
}
```

---

## Scenario 11: Protect Against Open Redirect

**Setup:** Your OAuth2 login returns users to a `redirect_uri` parameter. Prevent open redirect attacks.

```java
// VULNERABLE
@GetMapping("/login/callback")
public RedirectView callback(@RequestParam String code,
                              @RequestParam String redirect_uri) {
    // Process OAuth2 callback...
    return new RedirectView(redirect_uri);  // OPEN REDIRECT! Any URL accepted
}

// SAFE — whitelist-based redirect validation
@Component
public class RedirectUriValidator {

    @Value("${app.allowed-redirect-uris}")
    private List<String> allowedUris;

    public String validate(String redirectUri) {
        if (redirectUri == null || redirectUri.isBlank()) {
            return "/dashboard";  // default
        }

        // Option 1: Exact match against whitelist
        if (allowedUris.contains(redirectUri)) {
            return redirectUri;
        }

        // Option 2: Path-only redirects (no scheme/host = safe for same-origin)
        try {
            URI uri = new URI(redirectUri);
            if (uri.getScheme() == null && uri.getHost() == null) {
                // Relative URL — safe for same-origin redirect
                String path = uri.getPath();
                if (path.startsWith("/") && !path.startsWith("//")) {
                    return redirectUri;
                }
            }
        } catch (URISyntaxException e) {
            // Invalid URI — use default
        }

        log.warn("Invalid redirect URI rejected: {}", redirectUri);
        return "/dashboard";  // safe default
    }
}
```

---

## Scenario 12: Secrets Rotation Without Downtime

**Setup:** Rotate database credentials without application restart or downtime.

```java
// Multi-version credential support during rotation
@Component
public class RotatingCredentialDataSource extends AbstractDataSource {

    private volatile DataSource primary;
    private volatile DataSource secondary;  // old credentials, kept briefly during rotation

    public void rotateCredentials(DatabaseCredentials newCreds) {
        DataSource newDataSource = buildDataSource(newCreds);

        // 1. Validate new credentials before switching
        try (Connection test = newDataSource.getConnection()) {
            test.isValid(5);
        } catch (SQLException e) {
            throw new CredentialRotationException("New credentials failed validation", e);
        }

        // 2. Demote current primary to secondary (for in-flight connections)
        this.secondary = this.primary;

        // 3. Promote new credentials
        this.primary = newDataSource;

        // 4. Schedule secondary shutdown after connection drain time
        scheduler.schedule(() -> {
            if (secondary instanceof HikariDataSource hds) {
                hds.close();  // drain existing connections
            }
            secondary = null;
        }, 30, TimeUnit.SECONDS);

        log.info("Database credentials rotated successfully");
    }

    @Override
    public Connection getConnection() throws SQLException {
        try {
            return primary.getConnection();
        } catch (SQLException e) {
            if (secondary != null) {
                log.warn("Primary credentials failed, trying secondary");
                return secondary.getConnection();
            }
            throw e;
        }
    }
}

// AWS Secrets Manager rotation with Lambda + Spring Refresh Scope
@RefreshScope  // Spring Cloud Context — bean recreated on /actuator/refresh
@Bean
public DataSource dataSource(SecretsManagerClient secrets) {
    String secretJson = secrets.getSecretValue(
        GetSecretValueRequest.builder().secretId("prod/db/credentials").build()
    ).secretString();
    Credentials creds = objectMapper.readValue(secretJson, Credentials.class);
    return buildDataSource(creds.username(), creds.password());
}
```

---

## Follow-Up Security Questions

1. **HTTPS only enforcement:** How do you force all traffic to use HTTPS?
   Spring Security HSTS + `server.ssl.enabled=true` + redirect HTTP → HTTPS at load balancer. Add HSTS header with preload directive.

2. **Dependency security:** How do you manage vulnerable dependencies?
   OWASP Dependency-Check plugin in Maven/Gradle build. Snyk or Dependabot for automated PR updates. Never ignore HIGH/CRITICAL CVEs.

3. **Secrets in CI/CD:** How do you handle secrets in GitHub Actions?
   GitHub Secrets for CI values. OIDC + AWS IAM roles for cloud access (no long-lived credentials). Never commit `.env` files.

4. **Security testing:** What types of security tests do you write?
   Unit tests for authorization logic. Integration tests for CORS, CSRF, and authentication flows. OWASP ZAP or Burp Suite for DAST. Regular penetration testing for production.

5. **Logging PII compliance:** Under GDPR, can you log IP addresses?
   IP addresses can be PII under GDPR. Log only what's necessary. Pseudonymize (hash + salt) if needed for fraud detection. Document retention policies.
