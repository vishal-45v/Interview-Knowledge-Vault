# Spring Security — Architecture Notes

---

## Security Filter Chain Architecture

```
HTTP Request
     │
     ▼
DelegatingFilterProxy (bridges Spring Security to Servlet container)
     │
     ▼
FilterChainProxy (orchestrates multiple SecurityFilterChains)
     │
     ▼
SecurityFilterChain (your configured chain)
  [Filters execute in order:]
  1.  SecurityContextHolderFilter
  2.  WebAsyncManagerIntegrationFilter
  3.  HeaderWriterFilter (X-Content-Type, X-Frame-Options, etc.)
  4.  CorsFilter (if CORS configured)
  5.  CsrfFilter (if CSRF enabled)
  6.  LogoutFilter
  7.  [JwtAuthenticationFilter — custom, added here]
  8.  UsernamePasswordAuthenticationFilter
  9.  BasicAuthenticationFilter (if configured)
  10. ExceptionTranslationFilter ← catches AccessDeniedException
  11. AuthorizationFilter (was FilterSecurityInterceptor)
     │
     ▼
DispatcherServlet → Controller
```

---

## Authentication vs Authorization Flow

```
Authentication (Who are you?):
  JwtFilter parses JWT
    → JwtTokenProvider.validateToken()
    → JwtTokenProvider.getUsernameFromToken()
    → UserDetailsService.loadUserByUsername()
    → Create UsernamePasswordAuthenticationToken
    → SecurityContextHolder.getContext().setAuthentication(auth)

Authorization (What can you do?):
  AuthorizationFilter checks secured endpoint
    → hasRole('ADMIN')? → check authentication.getAuthorities()
    → @PreAuthorize SpEL evaluation
    → If denied → AccessDeniedException
    → ExceptionTranslationFilter catches → AccessDeniedHandler
```

---

## JWT Claims Structure

```json
Header:
{
  "alg": "HS512",
  "typ": "JWT"
}

Payload (claims):
{
  "sub": "alice@example.com",     ← subject
  "iat": 1705320000,              ← issued at
  "exp": 1705323600,              ← expiration (1 hour later)
  "jti": "abc-123-xyz",           ← JWT ID (for revocation)
  "iss": "https://myapp.com",     ← issuer
  "roles": ["ROLE_USER", "ROLE_PREMIUM"],  ← custom claim
  "userId": 42                    ← custom claim
}

Signature:
HMAC-SHA512(base64(header) + "." + base64(payload), secretKey)
```
