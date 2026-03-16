# Spring Security — Trap Questions

---

## Trap 1: permitAll() Still Runs Security Filters

**Question:** You configured .requestMatchers("/public/**").permitAll(). Are security filters skipped?

**Answer:** No. permitAll() means all users (authenticated or not) are allowed — but ALL security filters still run. If your filter throws an exception for unauthenticated requests, it will affect /public endpoints too.

To completely bypass security filters for certain paths, configure ignoring:
```java
@Bean
public WebSecurityCustomizer webSecurityCustomizer() {
    return web -> web.ignoring().requestMatchers("/static/**", "/favicon.ico");
}
```

---

## Trap 2: @EnableMethodSecurity Not Enabled

**Question:** @PreAuthorize annotation has no effect. Why?

**Answer:** Method security must be explicitly enabled with @EnableMethodSecurity on a @Configuration class. Without it, @PreAuthorize/@PostAuthorize annotations are completely ignored.

---

## Trap 3: SecurityContext and @Async

**Question:** SecurityContextHolder.getContext().getAuthentication() returns null inside an @Async method.

**Answer:** Spring Security stores Authentication in a ThreadLocal. @Async runs in a different thread that doesn't inherit the SecurityContext.

Fix: Configure Security to propagate context to async threads:
```java
@Bean
public Executor taskExecutor() {
    return new DelegatingSecurityContextAsyncTaskExecutor(
        new ThreadPoolTaskExecutor()
    );
}
```

---

## Trap 4: BCrypt Cost Factor and Login Latency

**Question:** BCrypt with cost factor 12 makes login feel slow. Should you reduce it?

**Answer:** BCrypt slowing down login is intentional — it prevents brute force attacks. Cost factor 12 means 2^12 = 4096 iterations. The recommended minimum is 10.

If login is too slow:
- Check if you're hashing more than once (double-hashing bug)
- Ensure you're not using a test password with 100 characters
- Consider Argon2 which allows more fine-grained tuning

Never reduce below 10 for production. Login should take ~100-300ms for security.

---

## Trap 5: JWT Secret Rotation

**Question:** You need to rotate your JWT signing secret. What happens to existing tokens?

**Answer:** Existing tokens signed with the old secret become invalid immediately — users are forcibly logged out. To rotate without forced logout:

1. Keep both old and new secrets during rotation period
2. Validate against both (try new key first, fallback to old)
3. After expiration of all old tokens, remove old key support
