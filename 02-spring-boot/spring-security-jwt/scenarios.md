# Spring Security & JWT — Scenarios

> 20+ real-world security scenarios covering filter chain, JWT, OAuth2, and authorization.

---

## Scenario 1: JWT Authentication Filter

**Problem:** Implement JWT-based stateless authentication for a REST API.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            String username = jwtTokenProvider.getUsernameFromToken(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request));

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

---

## Scenario 2: JwtTokenProvider

```java
@Component
public class JwtTokenProvider {

    @Value("${app.jwt.secret}")
    private String jwtSecret;

    @Value("${app.jwt.expiration-ms}")
    private long jwtExpirationMs;

    public String generateToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs);

        return Jwts.builder()
            .setSubject(userPrincipal.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .claim("roles", userPrincipal.getAuthorities()
                .stream().map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()))
            .signWith(Keys.hmacShaKeyFor(jwtSecret.getBytes()), SignatureAlgorithm.HS512)
            .compact();
    }

    public String getUsernameFromToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(jwtSecret.getBytes()))
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (MalformedJwtException ex) {
            log.warn("Invalid JWT token: {}", ex.getMessage());
        } catch (ExpiredJwtException ex) {
            log.warn("JWT token is expired: {}", ex.getMessage());
        } catch (UnsupportedJwtException ex) {
            log.warn("JWT token is unsupported: {}", ex.getMessage());
        } catch (IllegalArgumentException ex) {
            log.warn("JWT claims string is empty: {}", ex.getMessage());
        }
        return false;
    }
}
```

---

## Scenario 3: Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationFilter jwtAuthFilter;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // Disable CSRF for stateless API
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(exc -> exc
                .authenticationEntryPoint(jwtAuthEntryPoint())
                .accessDeniedHandler(jwtAccessDeniedHandler()));

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // Cost factor 12
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public AuthenticationEntryPoint jwtAuthEntryPoint() {
        return (request, response, authException) -> {
            response.setContentType("application/json");
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Unauthorized\",\"message\":\""
                + authException.getMessage() + "\"}");
        };
    }

    @Bean
    public AccessDeniedHandler jwtAccessDeniedHandler() {
        return (request, response, accessDeniedException) -> {
            response.setContentType("application/json");
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"Access denied\"}");
        };
    }
}
```

---

## Scenario 4: @PreAuthorize with SpEL

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // Only users with ADMIN role can access
    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public List<OrderResponse> getAllOrders() { ... }

    // User can only access their own orders (or admin can access any)
    @GetMapping("/{id}")
    @PreAuthorize("@orderSecurityService.canAccessOrder(#id, authentication)")
    public OrderResponse getOrder(@PathVariable Long id) { ... }

    // Can check method parameters
    @PostMapping
    @PreAuthorize("hasRole('USER') and !hasRole('BANNED')")
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) { ... }
}

@Service("orderSecurityService")
public class OrderSecurityService {

    @Autowired
    private OrderRepository orderRepository;

    public boolean canAccessOrder(Long orderId, Authentication authentication) {
        if (authentication.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;  // Admins can access any order
        }

        String currentUser = authentication.getName();
        return orderRepository.findById(orderId)
            .map(order -> order.getCustomer().getEmail().equals(currentUser))
            .orElse(false);
    }
}
```

---

## Scenario 5: Refresh Token Pattern

**Problem:** JWT access tokens expire in 15 minutes. Users get logged out too frequently.

```java
@PostMapping("/auth/refresh")
public ResponseEntity<AuthResponse> refreshToken(@RequestBody RefreshTokenRequest request) {

    String refreshToken = request.getRefreshToken();

    // Validate refresh token (stored in DB, not JWT)
    RefreshToken storedToken = refreshTokenRepository.findByToken(refreshToken)
        .orElseThrow(() -> new InvalidTokenException("Refresh token not found"));

    if (storedToken.isExpired()) {
        refreshTokenRepository.delete(storedToken);
        throw new InvalidTokenException("Refresh token expired. Please login again.");
    }

    User user = storedToken.getUser();

    // Issue new access token
    String newAccessToken = jwtTokenProvider.generateAccessToken(user);

    // Rotate refresh token (security best practice)
    storedToken.setToken(UUID.randomUUID().toString());
    storedToken.setExpiryDate(Instant.now().plus(7, ChronoUnit.DAYS));
    refreshTokenRepository.save(storedToken);

    return ResponseEntity.ok(new AuthResponse(newAccessToken, storedToken.getToken()));
}
```

---

## Scenario 6: CORS Configuration

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://myapp.com", "https://admin.myapp.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);  // Pre-flight cache duration
    }
}

// Or configure in SecurityFilterChain (preferred with Spring Security):
http.cors(cors -> cors.configurationSource(request -> {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    return config;
}));
```

---

## Scenario 7: Method Security with @PostAuthorize

```java
@Service
public class DocumentService {

    // @PostAuthorize: Method executes, then checks result
    // Useful when you need to check the returned object's ownership
    @PostAuthorize("returnObject.ownerId == authentication.name")
    public Document getDocument(Long id) {
        return documentRepository.findById(id).orElseThrow();
        // Method runs, then Spring checks if the caller owns the returned document
        // If not, throws AccessDeniedException
    }

    // @PreFilter and @PostFilter for collections
    @PostFilter("filterObject.ownerId == authentication.name")
    public List<Document> getAllDocuments() {
        return documentRepository.findAll();
        // Filters the returned list to only include documents owned by current user
    }
}
```

---

## Scenario 8: OAuth2 Resource Server

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated());

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");  // Custom claim name

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}

// application.yml:
// spring:
//   security:
//     oauth2:
//       resourceserver:
//         jwt:
//           issuer-uri: https://auth.mycompany.com
//           jwk-set-uri: https://auth.mycompany.com/.well-known/jwks.json
```

---

## Scenario 9: Securing WebSocket Connections

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketSecurityConfig extends AbstractSecurityWebSocketMessageBrokerConfigurer {

    @Override
    protected void configureInbound(MessageSecurityMetadataSourceRegistry messages) {
        messages
            .nullDestMatcher().authenticated()
            .simpSubscribeDestMatchers("/user/**").authenticated()
            .simpMessageDestMatchers("/app/**").authenticated()
            .anyMessage().denyAll();
    }

    @Override
    protected boolean sameOriginDisabled() {
        return true;  // Disable same-origin policy for WebSockets
    }
}

// In the WebSocket handshake, validate JWT token:
@Component
public class WebSocketAuthInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler,
                                   Map<String, Object> attributes) throws Exception {
        String token = extractTokenFromRequest(request);
        if (token != null && jwtTokenProvider.validateToken(token)) {
            String username = jwtTokenProvider.getUsernameFromToken(token);
            attributes.put("username", username);
            return true;
        }
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        return false;
    }
}
```

---

## Scenario 10: Rate Limiting with Spring Security

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final RateLimiter rateLimiter = RateLimiter.create(100.0);  // 100 req/sec

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // Apply rate limiting only to auth endpoints (prevent brute force)
        if (request.getRequestURI().startsWith("/api/auth/")) {
            String clientIp = getClientIp(request);
            if (!ipRateLimiter.tryAcquire(clientIp)) {
                response.setStatus(429);
                response.setContentType("application/json");
                response.getWriter().write(
                    "{\"error\":\"Too Many Requests\",\"retryAfter\":60}");
                return;
            }
        }

        filterChain.doFilter(request, response);
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```
