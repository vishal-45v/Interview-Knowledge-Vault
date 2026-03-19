# Spring Security — Diagram Explanations

---

## Diagram 1: JWT Authentication Flow

```
  Client sends: GET /api/orders, Authorization: Bearer eyJhbGci...

  SecurityFilterChain:
         │
         ▼
  JwtAuthenticationFilter.doFilterInternal()
    1. Extract Bearer token from header
    2. JwtTokenProvider.validateToken(token)
       ├── Parse JWT signature ✓
       ├── Check expiration ✓
       └── Return true
    3. getUsernameFromToken → "alice@example.com"
    4. UserDetailsService.loadUserByUsername("alice@example.com")
       → UserDetails{username="alice", roles=[ROLE_USER]}
    5. SecurityContextHolder.setAuthentication(auth)
         │
         ▼
  Request continues to Controller
  @PreAuthorize checks SecurityContextHolder
  Controller executes ✓
```

---

## Diagram 2: Spring Security Filter Chain

```
  HTTP Request
       │
       ▼
  ┌──────────────────────────────────────────────────────────┐
  │  SecurityFilterChain                                      │
  │                                                           │
  │  1. CorsFilter → CORS headers                            │
  │  2. CsrfFilter → CSRF token check (disabled for JWT)     │
  │  3. LogoutFilter → handle /logout                        │
  │  4. JwtAuthFilter → parse JWT, set SecurityContext       │
  │  5. ExceptionTranslationFilter → catch auth exceptions   │
  │  6. AuthorizationFilter → check roles/permissions        │
  │                                                           │
  └──────────────────────────────────────────────────────────┘
       │
       ▼
  DispatcherServlet → Controller
```

---

## Diagram 3: Refresh Token Pattern

```
  Login:
  Client ──POST /auth/login {credentials}──► Server
                                              ├── Validate credentials
                                              ├── Generate access_token (15 min TTL)
                                              ├── Generate refresh_token (7 days, stored in DB)
                                              └──► Response: {access_token, refresh_token}

  Using API:
  Client ──GET /api/data, Authorization: Bearer access_token──► Server ──► 200 OK

  Access token expired:
  Client ──GET /api/data──► Server ──► 401 Unauthorized

  Refresh:
  Client ──POST /auth/refresh {refresh_token}──► Server
                                                  ├── Validate refresh_token in DB
                                                  ├── Generate new access_token
                                                  ├── Rotate refresh_token (new value)
                                                  └──► Response: {new_access_token, new_refresh_token}

  Logout:
  Client ──POST /auth/logout {refresh_token}──► Server
                                                 ├── Delete refresh_token from DB
                                                 └──► 200 OK
  (access_token still valid until expiry — keep TTL short)
```
