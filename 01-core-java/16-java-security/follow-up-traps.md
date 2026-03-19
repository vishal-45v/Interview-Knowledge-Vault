# Java Security — Follow-Up Traps

---

## Trap 1: JWT — Where to Store on Client: localStorage vs httpOnly Cookie?

**localStorage:**
- Accessible from JavaScript — vulnerable to XSS
- If any script on your page is compromised (even a third-party CDN), the attacker can steal the token with `localStorage.getItem('token')`
- Persists across tabs and browser restarts

**httpOnly Cookie:**
- NOT accessible from JavaScript — `document.cookie` cannot read it
- Browser automatically sends it with requests (CSRF risk)
- Mitigate CSRF with `SameSite=Strict` or `SameSite=Lax` and a CSRF token

**Recommendation:** store JWT in an `httpOnly`, `Secure`, `SameSite=Strict` cookie. This protects against XSS (script can't read the cookie) and, with SameSite, against CSRF.

```
Set-Cookie: jwt=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600
```

Never store a JWT in `localStorage` for sensitive applications.

---

## Trap 2: JWT "none" Algorithm Vulnerability

The JWT spec allows `"alg": "none"` — meaning no signature is applied. Some early JWT libraries would accept a token with no signature as valid if `alg` was `none`.

```
// Malicious token: header claims alg=none, no signature needed
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0   <- {"alg":"none","typ":"JWT"}
.
eyJzdWIiOiJhZG1pbiIsInJvbGVzIjpbIkFETUlOIl19  <- {"sub":"admin","roles":["ADMIN"]}
.
                                               <- no signature at all
```

An attacker modifies any JWT payload (claiming admin role) and sets alg to "none". Vulnerable servers accept it without signature verification.

**Prevention:**
```java
// Always explicitly specify allowed algorithms
JWTVerifier verifier = JWT.require(Algorithm.HMAC256(secret))
    .withIssuer("myapp")
    .build();
// This library will REJECT tokens with alg=none or any other algorithm
```

Verify that your JWT library explicitly rejects `alg: none` by default (all modern libraries do, but always check).

---

## Trap 3: BCrypt Cost Factor — What Is a Good Value?

The cost factor is `N` in `2^N` iterations. Higher = slower = more secure but more CPU.

| Cost | Approx. time (modern CPU) | Use case |
|---|---|---|
| 10 | ~100ms | Default for most apps |
| 12 | ~400ms | Higher security, tolerable for login |
| 14 | ~1.5s | Maximum practical for user login |
| 16 | ~6s | Too slow for interactive login |

**Rule of thumb:** pick the highest cost where login time stays under ~300ms on your production hardware.

```java
// Spring Security default is cost 10
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode("userPassword");
boolean matches = encoder.matches("userPassword", hash);
```

Increase cost over time as hardware gets faster — re-hash on next user login.

---

## Trap 4: SQL Injection Through ORDER BY Clause

`PreparedStatement` prevents injection in `WHERE` clauses, but `ORDER BY` column names **cannot be parameterized** — column names are structural, not data.

```java
// UNSAFE — attackers can inject SQL into ORDER BY
String userInput = "name; DROP TABLE users; --";
String query = "SELECT * FROM users ORDER BY " + userInput;  // DANGEROUS

// PreparedStatement does NOT help here:
// "ORDER BY ?" would be interpreted as ORDER BY the string literal "name",
// not the column — it doesn't work for column names
```

**Safe approach: allowlist validation:**
```java
private static final Set<String> ALLOWED_SORT_COLUMNS =
    Set.of("name", "email", "created_at", "age");

public List<User> getUsers(String sortColumn) {
    if (!ALLOWED_SORT_COLUMNS.contains(sortColumn)) {
        throw new IllegalArgumentException("Invalid sort column: " + sortColumn);
    }
    // Now safe to use in query
    return jdbcTemplate.query(
        "SELECT * FROM users ORDER BY " + sortColumn,
        userRowMapper
    );
}
```

The same issue applies to table names, schema names, and SQL keywords.

---

## Trap 5: Stored XSS vs Reflected XSS

**Stored (Persistent) XSS:**
- Malicious script is saved in the database (e.g., in a comment, bio, product description)
- Executes for every user who views that content
- More dangerous — affects all users, not just those who click a malicious link

**Reflected XSS:**
- Malicious script is in the URL or request parameter
- Server reflects the input back in the response
- Victim must be tricked into clicking a crafted link
- Easier to detect (URL looks suspicious)

**DOM-based XSS:**
- Script injected and executed entirely in the browser via JavaScript DOM manipulation
- Server never sees the malicious payload
- Hardest to detect with server-side scanning

Prevention is the same for all: encode all user-controlled output with context-appropriate encoding (HTML entities for HTML context, JS escaping for script context, URL encoding for URL context).

---

## Trap 6: CSRF — Do API Endpoints Need CSRF Protection?

**Traditional web apps (form submission, cookie-based sessions):** YES, need CSRF protection.

**Stateless REST APIs using JWT in Authorization header:** NO CSRF protection needed, because:
- The browser never automatically sends `Authorization: Bearer <token>` headers for cross-site requests
- CSRF exploits the browser's automatic cookie inclusion
- If you don't use cookies, CSRF is not possible

```java
// In Spring Security — if using JWT (stateless), disable CSRF
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .csrf().disable()  // safe because we use JWT in Authorization header
            .authorizeRequests()
            .antMatchers("/api/**").authenticated();
    }
}
```

However, if you store JWT in a cookie (even an httpOnly one), you need CSRF protection again — use `SameSite=Strict` on the cookie.

---

## Trap 7: OAuth2 — Access Token vs Refresh Token

| Aspect | Access Token | Refresh Token |
|---|---|---|
| Lifetime | Short (15 min – 1 hour) | Long (days to months) |
| Sent with | Every API request | Only to token endpoint |
| Stored | In memory or short-lived cookie | Secure storage |
| If stolen | Attacker has limited window | Full session compromise |
| Revocable | Usually not (stateless JWT) | Yes, on auth server |

**Why the short-lived access token design?**
JWTs are stateless — you can't revoke them before expiry. If stolen, the attacker has access until expiry. Short expiry limits the damage window.

The refresh token is long-lived but only exchanged with the authorization server over HTTPS. It never travels to the resource server.

```
Access token stolen: attacker has 15 minutes
Refresh token stolen: attacker can get new access tokens indefinitely
  -> Must revoke refresh token on auth server immediately
```

---

## Trap 8: HTTPS Doesn't Prevent All MITM — What Else Do You Need?

HTTPS encrypts transit and authenticates the server with a certificate. But:

1. **Certificate Authority (CA) compromise** — if a CA is compromised, attackers can get fraudulent certificates. Mitigation: HTTP Public Key Pinning (HPKP) or Certificate Transparency logs.

2. **SSL stripping attack** — attacker downgrades HTTPS to HTTP before it reaches the browser. Mitigation: HTTP Strict Transport Security (HSTS).
   ```
   Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
   ```

3. **User accepting invalid certs** — browsers warn, but users click "proceed anyway". Mitigation: HSTS preloading, user education.

4. **Endpoint compromise** — HTTPS protects data in transit, but if the server is hacked, the data is exposed at rest.

5. **Misbehaving client code** — mobile apps that disable cert validation (`trustAllCerts`) are vulnerable even with HTTPS.

---

## Trap 9: Password Reset Token Timing Attack

A naive password reset implementation:

```java
// VULNERABLE to timing attack
public boolean isValidResetToken(String token) {
    String stored = database.getResetToken(userId);
    return stored.equals(token);    // String.equals() short-circuits on first mismatch
}
```

`String.equals()` returns `false` immediately on the first mismatched character. This means:
- A token that matches the first 10 characters takes slightly longer than one that mismatches at character 1
- An attacker sends millions of requests with varying tokens and measures response times
- They can determine the correct token one character at a time

**Fix: constant-time comparison:**
```java
// Safe — always compares all bytes regardless of mismatch
public boolean isValidResetToken(String provided, String stored) {
    byte[] a = provided.getBytes(StandardCharsets.UTF_8);
    byte[] b = stored.getBytes(StandardCharsets.UTF_8);
    return MessageDigest.isEqual(a, b);  // constant time
}
```

Also: reset tokens should be single-use, expire quickly (15 minutes), and be generated with `SecureRandom`.

---

## Trap 10: Insecure Deserialization — How Is It an Attack Vector?

Java's native serialization (`ObjectInputStream.readObject()`) is dangerous with untrusted input:

```java
// DANGEROUS — deserializing user-supplied data
ObjectInputStream ois = new ObjectInputStream(request.getInputStream());
Object obj = ois.readObject();  // attacker controls the bytes!
```

During deserialization, Java calls the class's `readObject()`, constructors, and initializers. If the classpath contains certain library classes (Apache Commons Collections, Spring, etc.), an attacker can craft a serialized payload that, when deserialized, executes arbitrary code — a "gadget chain".

**Prevention:**
1. Never deserialize data from untrusted sources using Java native serialization.
2. If you must, use a `ObjectInputFilter` to allowlist permitted classes.
3. Prefer JSON/XML with strict schemas over Java serialization.
4. Use dedicated libraries for external data formats (Jackson, Gson).

```java
// Safer: ObjectInputFilter (Java 9+)
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(info -> {
    if (info.serialClass() == null) return ObjectInputFilter.Status.UNDECIDED;
    if (AllowedClasses.contains(info.serialClass())) return ObjectInputFilter.Status.ALLOWED;
    return ObjectInputFilter.Status.REJECTED;
});
```

Real-world CVEs: Apache Struts 2 (Equifax breach), WebLogic deserialization RCE, Jenkins RCE — all due to insecure deserialization.

---

## Trap 11: What Is the Difference Between Authentication Failure and Authorization Failure HTTP Status Codes?

- `401 Unauthorized` — actually means **unauthenticated**: "I don't know who you are, provide credentials."
- `403 Forbidden` — means **unauthorized**: "I know who you are, but you're not allowed to do this."

The naming is historically confusing (401 says "Unauthorized" but means unauthenticated). Correct usage matters in API design.

---

## Trap 12: Can You Enumerate Users via Login Error Messages?

```
VULNERABLE (reveals whether user exists):
  "Username not found"       <- user does NOT exist
  "Incorrect password"       <- user DOES exist, wrong password

SAFE (generic message):
  "Invalid username or password"  <- doesn't reveal which is wrong
```

Returning different error messages for non-existent users vs wrong passwords allows attackers to enumerate valid usernames. Always return the same generic error message for both cases. Also add rate limiting and account lockout to prevent brute force.
