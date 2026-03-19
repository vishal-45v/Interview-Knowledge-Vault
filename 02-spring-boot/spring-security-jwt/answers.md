# Spring Security & JWT — Structured Answers

---

## Q1: How Does Spring Security Filter Chain Work?

Spring Security is implemented as a chain of servlet filters. The SecurityFilterChain processes every HTTP request through a series of security filters in order:

```
Request → SecurityContextPersistenceFilter
       → LogoutFilter
       → UsernamePasswordAuthenticationFilter (form login)
       → [JwtAuthenticationFilter (custom)]
       → ExceptionTranslationFilter
       → FilterSecurityInterceptor (authorization)
       → Controller
```

Each filter can modify the SecurityContext or stop the chain. After authentication, the SecurityContextHolder holds the Authentication object for the rest of the request.

---

## Q2: JWT Structure and Validation

JWT has three parts separated by dots: header.payload.signature

Header: Algorithm and token type (Base64 encoded)
Payload (claims): Subject, issuer, expiration, custom data (Base64 encoded)
Signature: HMAC(header + payload, secret) or RSA signature

Validation:
1. Decode header and payload (public — anyone can read)
2. Recalculate signature using secret key
3. Compare with provided signature
4. Check exp (expiration), nbf (not before), iss (issuer) claims

Note: JWT payload is NOT encrypted by default — only signed. Never put sensitive data in JWT.

---

## Q3: JWT vs Session-Based Authentication

| Aspect | JWT | Session |
|--------|-----|---------|
| State | Stateless | Stateful (session in server memory/DB) |
| Scalability | Easy (no shared state) | Requires sticky sessions or shared session store |
| Revocation | Difficult (must use blacklist) | Easy (delete session) |
| Size | Larger (token in every request) | Small cookie (session ID) |
| CSRF | Not needed (no cookie by default) | Required |
| Use case | Microservices, mobile, stateless APIs | Traditional web apps, when revocation needed |

---

## Q4: How to Revoke a JWT?

JWTs cannot be invalidated before expiry since they're stateless. Solutions:

1. Short expiry (5-15 min) + refresh token: Reduces window of token misuse
2. Token blacklist in Redis: Store revoked JTI (JWT ID) until expiry time. Check on each request.
3. Token versioning: Store user's token version in DB. Include in JWT claim. On logout, increment version.

---

## Q5: CSRF — When Is It Needed?

CSRF (Cross-Site Request Forgery) protection is needed when:
- Using session-based authentication (cookies)
- Browser automatically sends cookies with cross-origin requests

NOT needed when:
- Using Bearer token authentication (JWT in Authorization header)
- Browser doesn't automatically send Authorization header cross-origin

For REST APIs using JWT in Authorization header: disable CSRF.
For web apps using session cookies: enable CSRF.

---

## Q6: @PreAuthorize vs @Secured vs @RolesAllowed

@PreAuthorize: Most powerful — supports SpEL expressions. Can check method arguments, return values, and complex conditions.

@Secured: Simple role-based access. Cannot use SpEL.

@RolesAllowed: JSR-250 standard annotation. Similar to @Secured.

All require @EnableMethodSecurity (or @EnableGlobalMethodSecurity for older versions).
