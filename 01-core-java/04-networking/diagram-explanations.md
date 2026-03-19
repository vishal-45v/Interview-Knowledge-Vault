# Networking — ASCII Diagram Explanations

---

## 1. TCP Three-Way Handshake

Before any application data can flow, TCP establishes a connection using three messages: SYN, SYN-ACK, and ACK. Only after this dance is the connection considered established.

```
  Client                          Server
    │                               │
    │  ──── SYN (seq=x) ─────────►  │   Step 1: Client says "I want to connect,
    │                               │           my starting sequence is x"
    │                               │
    │  ◄─── SYN-ACK (seq=y,ack=x+1) │   Step 2: Server says "OK, I acknowledge x,
    │                               │           my starting sequence is y"
    │                               │
    │  ──── ACK (ack=y+1) ────────►  │   Step 3: Client acknowledges y.
    │                               │           Connection is now ESTABLISHED.
    │                               │
    │  ════ DATA TRANSFER ════════►  │
    │  ◄═══ DATA TRANSFER ════════   │
    │                               │
    │  ──── FIN ──────────────────►  │   Connection teardown (4-way)
    │  ◄─── FIN-ACK ──────────────   │
    │  ◄─── FIN ──────────────────   │
    │  ──── ACK ──────────────────►  │
```

**Key points:**
- SYN = synchronize (start of connection)
- SEQ numbers are random to prevent injection attacks
- After ACK the socket enters ESTABLISHED state on both sides
- A half-open connection (SYN sent, no SYN-ACK received) consumes server resources — this is the basis of a SYN flood attack

---

## 2. HTTP Request/Response Cycle

```
  Browser / Client                        Web Server
       │                                      │
       │  1. DNS lookup: example.com → IP     │
       │  2. TCP handshake (3-way)            │
       │                                      │
       │  ── HTTP Request ─────────────────►  │
       │  GET /index.html HTTP/1.1            │
       │  Host: example.com                   │
       │  Accept: text/html                   │
       │  Connection: keep-alive              │
       │                                      │
       │                                      │  3. Server processes request
       │                                      │     - parses path
       │                                      │     - reads file / queries DB
       │                                      │     - builds response
       │                                      │
       │  ◄─ HTTP Response ─────────────────  │
       │  HTTP/1.1 200 OK                     │
       │  Content-Type: text/html             │
       │  Content-Length: 4523                │
       │                                      │
       │  <html>...</html>                    │
       │                                      │
       │  4. Browser renders HTML             │
       │  5. Discovers sub-resources          │
       │     (CSS, JS, images)                │
       │  6. Sends more GET requests          │
       │     (can reuse keep-alive TCP conn)  │
```

**Status code categories:**
```
  1xx  Informational   (100 Continue)
  2xx  Success         (200 OK, 201 Created, 204 No Content)
  3xx  Redirection     (301 Moved Permanently, 302 Found, 304 Not Modified)
  4xx  Client Error    (400 Bad Request, 401 Unauthorized, 404 Not Found)
  5xx  Server Error    (500 Internal Server Error, 503 Service Unavailable)
```

---

## 3. HTTPS / TLS Handshake

```
  Client                                    Server
    │                                          │
    │  ── ClientHello ───────────────────────► │
    │     - TLS version supported              │
    │     - Cipher suites supported            │
    │     - Client random (28 bytes)           │
    │                                          │
    │  ◄─ ServerHello ───────────────────────  │
    │     - Chosen TLS version                 │
    │     - Chosen cipher suite                │
    │     - Server random (28 bytes)           │
    │                                          │
    │  ◄─ Certificate ───────────────────────  │
    │     - Server's public key                │
    │     - Signed by trusted CA               │
    │                                          │
    │  ◄─ ServerHelloDone ───────────────────  │
    │                                          │
    │  [Client verifies certificate]           │
    │   - CA signature valid?                  │
    │   - Domain matches?                      │
    │   - Not expired?                         │
    │   - Not revoked (OCSP)?                  │
    │                                          │
    │  ── ClientKeyExchange ─────────────────► │
    │     - Pre-master secret (encrypted       │
    │       with server's public key)          │
    │                                          │
    │  Both sides independently derive:        │
    │  Master Secret = PRF(pre-master,         │
    │                      client_random,      │
    │                      server_random)      │
    │                                          │
    │  ── ChangeCipherSpec ──────────────────► │
    │  ── Finished (encrypted) ──────────────► │
    │                                          │
    │  ◄─ ChangeCipherSpec ──────────────────  │
    │  ◄─ Finished (encrypted) ──────────────  │
    │                                          │
    │  ══════ Encrypted Application Data ════► │
    │  ◄═════ Encrypted Application Data ════  │

  Note: TLS 1.3 reduces this to 1 round-trip (0-RTT for resumption)
```

---

## 4. DNS Resolution Chain

```
  User types: www.example.com
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         Client Machine                              │
  │  1. Check local hosts file  (/etc/hosts)  ──► found? return IP      │
  │  2. Check OS DNS cache      ──────────────► found? return IP        │
  │  3. Query local DNS Resolver (ISP / 8.8.8.8)                        │
  └──────────────────────────────┬──────────────────────────────────────┘
                                 │ cache miss
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │               Recursive DNS Resolver (e.g. 8.8.8.8)             │
  │  - Acts as intermediary, does the heavy lifting                  │
  │  - Has its own cache; returns from cache if TTL not expired      │
  └────────┬────────────────────────────────────────────────────────┘
           │ cache miss — asks root
           ▼
  ┌──────────────────────────┐
  │  Root Nameserver (13     │   "I don't know www.example.com,
  │  clusters worldwide)     │    but .com TLD server is at 192.5.6.30"
  └──────────────────────────┘
           │
           ▼
  ┌──────────────────────────┐
  │  .com TLD Nameserver     │   "I don't know that host, but
  │  (Verisign)              │    example.com is handled by ns1.example.com"
  └──────────────────────────┘
           │
           ▼
  ┌──────────────────────────┐
  │  Authoritative           │   "www.example.com = 93.184.216.34
  │  Nameserver for          │    TTL = 3600 seconds"
  │  example.com             │
  └──────────────────────────┘
           │
           └──────────────────────────────────────────────────────►
                          IP returned to Resolver
                          Resolver caches it and returns to Client
                          Client connects to 93.184.216.34
```

---

## 5. WebSocket Upgrade and Bidirectional Communication

```
  Client                                     Server
    │                                           │
    │  ── HTTP GET /chat HTTP/1.1 ───────────►  │
    │     Upgrade: websocket                    │
    │     Connection: Upgrade                   │
    │     Sec-WebSocket-Key: dGhlIHNh...        │
    │     Sec-WebSocket-Version: 13             │
    │                                           │
    │  ◄─ HTTP 101 Switching Protocols ───────  │
    │     Upgrade: websocket                    │
    │     Connection: Upgrade                   │
    │     Sec-WebSocket-Accept: s3pPL...        │
    │                                           │
    │  ══════════ WebSocket Frames ═══════════  │
    │                                           │
    │  ── Frame: "Hello server!" ────────────►  │  Client sends anytime
    │  ◄─ Frame: "Hello client!" ────────────  │  Server sends anytime
    │  ◄─ Frame: "New message from Alice" ───  │  Server PUSHES unprompted
    │  ── Frame: "I'm typing..." ────────────►  │
    │  ◄─ Frame: "Alice is typing..." ───────  │
    │  ── Ping frame ────────────────────────►  │  Keepalive
    │  ◄─ Pong frame ────────────────────────  │
    │                                           │
    │  ── Close frame (code 1000) ───────────►  │  Graceful close
    │  ◄─ Close frame (code 1000) ───────────  │

  vs HTTP polling:
  Client ──GET /messages──► Server  (every 2 seconds, wasteful)
  Client ──GET /messages──► Server
  Client ──GET /messages──► Server  (most responses: "nothing new")
```

---

## 6. HTTP/1.1 vs HTTP/2 Multiplexing

```
  HTTP/1.1 — one request at a time per connection (head-of-line blocking)
  ─────────────────────────────────────────────────────────────────────
  Connection 1:  [REQ1─────────────][REQ2────────][REQ3──────────────]
  Connection 2:  [REQ4────][REQ5──────────────────]
  Connection 3:  [REQ6───────────────────────────────]
  (Browser opens 6 parallel TCP connections to work around this)

  HTTP/2 — all requests multiplexed on ONE TCP connection
  ─────────────────────────────────────────────────────────────────────
  Single TCP connection:
  ├── Stream 1:  [HDR][DATA][DATA][DATA─────────────────]
  ├── Stream 3:  [HDR][DATA][DATA]
  ├── Stream 5:  [HDR][DATA────────────────────────────]
  ├── Stream 7:  [HDR][DATA][DATA][DATA]
  └── Stream 9:  [HDR][DATA]
         ↓ all interleaved as binary frames on the wire ↓
  TCP:  [S1:HDR][S3:HDR][S5:HDR][S1:DATA][S7:HDR][S3:DATA]...

  HTTP/2 additional features:
  ┌──────────────────────────────────────────────────────────┐
  │  Header Compression (HPACK)                              │
  │  Server Push (server sends resources before client asks) │
  │  Stream Prioritization                                   │
  │  Binary Framing Layer (not text like HTTP/1.1)           │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Load Balancer — Round-Robin Routing

```
  Clients                   Load Balancer                  Backend Servers
                           ┌─────────────┐
  Client A ──Request──────►│             │──────────────►  Server 1 (healthy)
                           │  Algorithm: │
  Client B ──Request──────►│  Round      │──────────────►  Server 2 (healthy)
                           │  Robin      │
  Client C ──Request──────►│             │──────────────►  Server 3 (healthy)
                           │             │
  Client D ──Request──────►│             │──────────────►  Server 1 (wraps)
                           │             │
  Client E ──Request──────►│             │──X (skipped)─   Server 4 (DOWN)
                           │             │──────────────►  Server 2 (next healthy)
                           └─────────────┘

  Health Check Loop:
  ┌──────────────────────────────────────────────────────────┐
  │  Load Balancer ──── GET /health ────► Server 1: 200 OK   │
  │  Load Balancer ──── GET /health ────► Server 2: 200 OK   │
  │  Load Balancer ──── GET /health ────► Server 3: timeout  │
  │                                                          │
  │  → Server 3 removed from rotation                        │
  │  → Alert sent to ops team                                │
  │  → When Server 3 returns 200 OK, re-added automatically  │
  └──────────────────────────────────────────────────────────┘

  Other algorithms:
  ┌──────────────────────────────────────────────────────────┐
  │  Least Connections  → route to server with fewest active │
  │  IP Hash           → same client always hits same server │
  │  Weighted RR       → Server 1 gets 3x traffic of Srv 2   │
  │  Resource Based    → route based on CPU/memory metrics   │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Connection Pool Lifecycle

```
  Application Start
         │
         ▼
  ┌─────────────────────────────────────────────────────────┐
  │              Connection Pool (e.g. HikariCP)            │
  │                                                         │
  │  Min pool size = 5, Max pool size = 20                  │
  │                                                         │
  │  Initial state: create 5 TCP+TLS connections            │
  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                    │
  │  │Con1│ │Con2│ │Con3│ │Con4│ │Con5│  ← IDLE             │
  │  └────┘ └────┘ └────┘ └────┘ └────┘                    │
  └─────────────────────────────────────────────────────────┘
         │
         ▼  Thread requests a connection
  ┌─────────────────────────────────────────────────────────┐
  │  Con1 BORROWED by Thread-A  (state: IN-USE)             │
  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐                           │
  │  │Con2│ │Con3│ │Con4│ │Con5│  ← still IDLE              │
  │  └────┘ └────┘ └────┘ └────┘                           │
  └─────────────────────────────────────────────────────────┘
         │
         ▼  Thread-A finishes, returns connection
  ┌─────────────────────────────────────────────────────────┐
  │  Con1 RETURNED to pool  (state: IDLE again)             │
  │  Pool validates connection (test-on-return query)       │
  │  If connection is stale/broken → discard, create new    │
  └─────────────────────────────────────────────────────────┘
         │
         ▼  Traffic spike: 25 simultaneous requests
  ┌─────────────────────────────────────────────────────────┐
  │  Pool grows to max (20 connections)                     │
  │  Requests 21-25 wait in queue for connectionTimeout ms  │
  │  If no connection freed in time → throw TimeoutException│
  └─────────────────────────────────────────────────────────┘
         │
         ▼  Idle connections exceed maxIdleTime
  ┌─────────────────────────────────────────────────────────┐
  │  Idle connections pruned back to minPoolSize            │
  │  Prevents holding connections the DB side has closed    │
  └─────────────────────────────────────────────────────────┘
```
