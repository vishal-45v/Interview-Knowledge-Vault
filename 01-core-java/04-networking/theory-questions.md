# Networking — Theory Questions

> 25 theory questions covering TCP/UDP, HTTP, SSL/TLS, sockets, WebSockets, DNS, and load balancing for Senior Java Backend Engineers.

---

## TCP / UDP

**Q1. What are the differences between TCP and UDP? When would you choose each?**

TCP (Transmission Control Protocol):
- Connection-oriented — requires three-way handshake (SYN → SYN-ACK → ACK)
- Reliable delivery: acknowledgments, retransmission of lost packets
- Ordered delivery: packets reassembled in sequence
- Flow control and congestion control
- Higher overhead

UDP (User Datagram Protocol):
- Connectionless — fire and forget
- No guarantee of delivery, order, or duplicate prevention
- Very low overhead, minimal latency
- Supports broadcast and multicast

**Use TCP when:** reliability matters — REST APIs, database connections, file transfers, email.
**Use UDP when:** low latency matters more than reliability — live video streaming, DNS queries, online gaming, VoIP.

---

**Q2. Explain the TCP three-way handshake.**

```
Client                    Server
  │── SYN (seq=x) ────────► │   Client initiates connection
  │◄── SYN-ACK (seq=y, ack=x+1) ─│   Server acknowledges, sends its own seq
  │── ACK (ack=y+1) ────────► │   Client acknowledges server's seq
  │                           │
  │   [Connection Established] │
```

- **SYN**: Client picks a random initial sequence number x, sets SYN flag
- **SYN-ACK**: Server picks its own sequence number y, acknowledges x+1
- **ACK**: Client acknowledges y+1; connection is now established

Purpose: synchronize sequence numbers on both sides, ensuring ordered reliable delivery.

---

**Q3. What is the TCP four-way handshake (connection teardown)?**

```
Client                    Server
  │── FIN ─────────────────► │   Client done sending
  │◄── ACK ─────────────────│   Server acknowledges
  │◄── FIN ─────────────────│   Server done sending
  │── ACK ─────────────────► │   Client acknowledges
  │   [Connection Closed]    │
```

- After the final ACK, the client enters TIME_WAIT state (2×MSL, typically 60s) to handle duplicate delayed packets.
- Half-close: after client sends FIN, server can still send data (connection is half-closed).

---

**Q4. What is TCP flow control and congestion control?**

**Flow control** — prevents sender from overwhelming receiver:
- Receiver advertises a **window size** (receive buffer space available)
- Sender cannot send more unacknowledged data than the window size
- Window size changes dynamically as receiver processes data

**Congestion control** — prevents sender from overwhelming the network:
- **Slow start**: begin with 1 MSS, double cwnd each RTT until threshold
- **Congestion avoidance**: linear increase after threshold
- **Fast retransmit**: 3 duplicate ACKs → retransmit without waiting for timeout
- **Fast recovery**: halve cwnd on congestion, continue from there

---

**Q5. What is the TIME_WAIT state in TCP? Why is it important and when can it cause problems?**

After a connection is closed, the side that sent the final ACK stays in TIME_WAIT for 2×MSL (Maximum Segment Lifetime, usually 60-120 seconds).

**Why it exists:**
1. Ensures the final ACK reaches the other side (if it's lost, the server will retransmit FIN and the client can re-ACK)
2. Prevents "ghost" packets from a previous connection with the same 4-tuple from being mistaken for new connection data

**When it causes problems:**
- High-throughput servers creating many short-lived connections can accumulate thousands of TIME_WAIT sockets
- Each TIME_WAIT uses a port from the ephemeral range (typically 32768-60999)
- Solution: `SO_REUSEADDR` socket option, connection keep-alive, reverse proxy, or reduce TIME_WAIT with `tcp_tw_reuse` (Linux)

---

## HTTP

**Q6. What are the differences between HTTP/1.1, HTTP/2, and HTTP/3?**

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|---------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No (HOL blocking) | Yes (streams) | Yes (independent streams) |
| Header compression | No | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Connection | One request/connection or keep-alive with queuing | Single connection, multiple streams | Single connection, faster recovery |
| TLS | Optional | Required (de facto) | Required |

HTTP/2 introduced **multiplexing** — multiple requests/responses over a single TCP connection without head-of-line blocking at the HTTP level. However, TCP-level HOL blocking still exists.

HTTP/3 uses **QUIC** over UDP, eliminating TCP HOL blocking entirely. Lost packet only blocks the stream it belongs to, not all streams.

---

**Q7. Explain HTTP keep-alive (persistent connections). Why does it matter for performance?**

In HTTP/1.0, each request opened a new TCP connection (expensive: 3-way handshake + TLS handshake).

HTTP/1.1 introduced persistent connections by default (`Connection: keep-alive`):
- Single TCP connection reused for multiple requests
- Reduces latency (no repeated handshake overhead)
- `Keep-Alive: timeout=60, max=100` — server hints at timeout and max requests

In Java, `HttpURLConnection` and `OkHttpClient` use connection pools (keep-alive by default).

---

**Q8. What are HTTP methods and their idempotency/safety properties?**

| Method | Safe | Idempotent | Description |
|--------|------|------------|-------------|
| GET | Yes | Yes | Retrieve resource |
| HEAD | Yes | Yes | Like GET, headers only |
| OPTIONS | Yes | Yes | Describe communication options |
| PUT | No | Yes | Replace resource entirely |
| DELETE | No | Yes | Remove resource |
| POST | No | No | Create resource / trigger action |
| PATCH | No | No | Partial update (by convention) |

**Safe** = does not modify server state.
**Idempotent** = calling N times has same effect as calling once.

DELETE is idempotent: deleting a resource that's already gone should return 404 but the server state is the same.

---

**Q9. What is HTTPS and how does TLS work at a high level?**

HTTPS = HTTP over TLS (Transport Layer Security). TLS provides:
- **Confidentiality**: encrypted traffic (symmetric encryption after handshake)
- **Integrity**: MAC/AEAD prevents tampering
- **Authentication**: server identity verified via certificate chain

**TLS 1.3 handshake (simplified):**
```
Client                         Server
  │── ClientHello ─────────────► │  (cipher suites, key shares)
  │◄── ServerHello + Certificate │  (selected cipher, key share, cert)
  │◄── {EncryptedExtensions}   │
  │◄── {Finished} ─────────────│
  │── {Finished} ──────────────► │
  │   [Application Data]         │
```

TLS 1.3 eliminates the RSA key exchange (only forward-secret ECDHE), requires only 1 RTT (0-RTT possible for resumed sessions).

---

**Q10. What are HTTP status codes? Give examples of common ones and what they mean.**

**2xx Success:**
- 200 OK — standard success
- 201 Created — resource created (include Location header)
- 202 Accepted — async processing started
- 204 No Content — success, no body (DELETE, PUT)

**3xx Redirection:**
- 301 Moved Permanently — cached by browser
- 302 Found — temporary redirect
- 304 Not Modified — ETag/Last-Modified matched, use cache

**4xx Client Error:**
- 400 Bad Request — malformed request, validation failure
- 401 Unauthorized — authentication required
- 403 Forbidden — authenticated but no permission
- 404 Not Found
- 409 Conflict — state conflict (optimistic lock, duplicate)
- 422 Unprocessable Entity — syntactically valid but semantically invalid
- 429 Too Many Requests — rate limited

**5xx Server Error:**
- 500 Internal Server Error
- 502 Bad Gateway — upstream service failed
- 503 Service Unavailable — overloaded or maintenance
- 504 Gateway Timeout — upstream timeout

---

## WebSockets

**Q11. How does WebSocket differ from HTTP? When would you use it?**

WebSocket is a **full-duplex, persistent bidirectional** communication protocol over a single TCP connection.

**Upgrade handshake:**
```
Client → GET /ws HTTP/1.1
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

Server → HTTP/1.1 101 Switching Protocols
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After upgrade, both sides can send frames at any time without request-response overhead.

**Use WebSocket when:**
- Real-time updates: chat, collaborative editing, live dashboards
- Low-latency bidirectional communication: trading platforms, gaming
- Frequent small messages: tracking, notifications

**Use HTTP when:** request-response pattern, infrequent updates, stateless operations.

**Alternative: Server-Sent Events (SSE)** — one-directional (server to client), uses HTTP, simpler, auto-reconnect built-in. Good for live feeds, notifications.

---

**Q12. What is HTTP long polling and how does it compare to WebSockets?**

**Long polling:**
1. Client sends request
2. Server holds the request open until there's data (or timeout)
3. Server responds with data
4. Client immediately sends another request

Works over standard HTTP, simpler to implement. Higher latency than WebSocket (new TCP connection per cycle or keep-alive with queuing). Good for low-frequency updates.

**Short polling:** client polls repeatedly at fixed intervals. Simple but wastes bandwidth when no data is available.

**Comparison:**
```
Short Polling: C→S, S→C (empty), C→S, S→C (empty), ... (many empty responses)
Long Polling:  C→S (wait...), S→C (data ready!), C→S (wait...), ...
WebSocket:     C↔S (persistent, instant both ways)
SSE:           C→S (open), S→C→C→C (server pushes continuously)
```

---

## DNS

**Q13. How does DNS resolution work? What is the full lookup chain?**

```
Browser                    OS                   Recursive Resolver          Root NS       TLD NS      Authoritative NS
  │── query example.com ──► │                         │                        │              │                │
  │   check local cache      │                         │                        │              │                │
  │◄── not found ───────────│                         │                        │              │                │
  │── check /etc/hosts ─────► │                         │                        │              │                │
  │◄── not found ───────────│                         │                        │              │                │
  │── UDP query ────────────────────────────────────► │                        │              │                │
  │                                                    │── who handles .com? ──► │              │                │
  │                                                    │◄── TLD servers ─────────│              │                │
  │                                                    │── who handles example? ──────────────► │                │
  │                                                    │◄── authoritative servers ───────────────│                │
  │                                                    │── what is example.com? ─────────────────────────────► │
  │                                                    │◄── 93.184.216.34 ────────────────────────────────────│
  │◄── 93.184.216.34 ──────────────────────────────────│
```

**DNS record types:**
- **A** — hostname to IPv4
- **AAAA** — hostname to IPv6
- **CNAME** — alias to another hostname
- **MX** — mail server
- **TXT** — verification, SPF records
- **NS** — authoritative nameserver for domain
- **SRV** — service location (port + hostname)

**TTL**: cached at each level; lower TTL = faster propagation of changes, more queries.

---

**Q14. What is a CNAME record and what are its limitations?**

A CNAME (Canonical Name) creates an alias from one hostname to another:
```
shop.example.com CNAME myapp.cloudprovider.com
```

The resolver follows the chain until it reaches an A or AAAA record.

**Limitations:**
1. Cannot coexist with other records at the same name (RFC 1034) — most importantly, cannot use CNAME on the zone apex (root domain, `example.com`) since it must have an SOA and NS record
2. Extra lookup in resolution chain adds latency
3. Solution for apex: **ALIAS** (Route 53) or **ANAME** records — proprietary extensions that resolve at the DNS server level

---

## SSL/TLS

**Q15. What is a TLS certificate and certificate chain?**

A TLS certificate contains:
- Subject (domain name, organization)
- Public key
- Issuer (Certificate Authority name)
- Validity period (not before / not after)
- Digital signature by the issuer's private key
- Subject Alternative Names (SANs)

**Certificate chain:**
```
Leaf Certificate (your domain)
  │ signed by
Certificate Authority (Intermediate)
  │ signed by
Root CA Certificate (self-signed, trusted by OS/browser)
```

The browser validates the chain:
1. Signature from intermediate matches leaf
2. Signature from root matches intermediate
3. Root is in the trusted root store
4. Certificate is not expired and not revoked (OCSP/CRL)

---

**Q16. What is mutual TLS (mTLS) and when would you use it?**

In standard TLS, only the server presents a certificate. In mTLS, **both sides authenticate**:

```
Client                         Server
  │── ClientHello ─────────────► │
  │◄── ServerHello + Certificate │
  │◄── CertificateRequest ───────│   ← Server asks client for cert
  │── Client Certificate ────────► │
  │── CertificateVerify ─────────► │
  │── Finished ──────────────────► │
  │◄── Finished ─────────────────│
```

**Use cases:**
- Service-to-service authentication in microservices (zero-trust networking)
- API client authentication (replacing API keys)
- Kubernetes service mesh (Istio, Linkerd)
- IoT device authentication

In Java, configure both `truststore` (CA certs to trust) and `keystore` (your own cert+private key).

---

## Sockets and Java Networking

**Q17. How do you create a TCP server socket in Java? What are the key socket options?**

```java
// Basic TCP server
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    serverSocket.setReuseAddress(true);   // Allow reuse after TIME_WAIT
    serverSocket.setSoTimeout(0);          // Accept blocks indefinitely

    while (true) {
        Socket clientSocket = serverSocket.accept();  // Blocks until connection
        // Handle in thread pool
        executor.submit(() -> handleClient(clientSocket));
    }
}

private void handleClient(Socket socket) throws IOException {
    socket.setSoTimeout(30_000);    // 30s read timeout — CRITICAL for production
    socket.setTcpNoDelay(true);     // Disable Nagle's algorithm (send immediately)
    socket.setKeepAlive(true);      // Enable TCP keepalive

    try (InputStream in = socket.getInputStream();
         OutputStream out = socket.getOutputStream()) {
        // Read/write data
    }
}
```

**Key socket options:**
- `SO_TIMEOUT`: read timeout (prevents indefinite blocking)
- `SO_REUSEADDR`: reuse port in TIME_WAIT state
- `TCP_NODELAY`: disable Nagle's algorithm (for low-latency small messages)
- `SO_KEEPALIVE`: send TCP keepalive probes to detect dead connections
- `SO_RCVBUF` / `SO_SNDBUF`: socket buffer sizes

---

**Q18. What is Java NIO and how does it differ from blocking I/O (java.net)?**

**Blocking I/O (java.io / java.net):**
- One thread per connection
- Thread blocks on read/write
- Simple to understand
- Doesn't scale beyond a few thousand connections

**Non-blocking I/O (java.nio):**
- Single thread can handle many connections via `Selector`
- `Channels` are non-blocking — read returns immediately (0 bytes if no data)
- `Selector.select()` blocks until at least one channel has events (readable, writable, connectable, acceptable)

```java
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();   // blocks until event
    Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
    while (keys.hasNext()) {
        SelectionKey key = keys.next();
        keys.remove();
        if (key.isAcceptable()) { /* accept connection */ }
        if (key.isReadable())   { /* read data */ }
    }
}
```

**In practice:** Use Netty or Vert.x rather than raw NIO. Spring WebFlux uses Project Reactor on Netty.

---

**Q19. What is Netty and why would you choose it over a Servlet container?**

Netty is an asynchronous event-driven network framework built on Java NIO.

**Key concepts:**
- **EventLoop**: single thread handling many channels (no thread-per-connection)
- **Channel**: represents a connection (NIO channel + lifecycle)
- **ChannelPipeline**: chain of `ChannelHandler`s (encoder/decoder/business logic)
- **ByteBuf**: Netty's efficient buffer (pooled, off-heap option, reference counted)

**Choose Netty over Servlet container when:**
- Need very high connection count (10K+ concurrent connections)
- Custom protocol (binary, framing, streaming)
- Ultra-low latency requirements
- WebSocket or other protocol upgrades

**Choose Servlet container (Tomcat/Jetty) when:**
- Standard HTTP REST APIs
- Team familiar with Spring MVC
- Synchronous request-response pattern

Spring WebFlux can use both Netty (default, reactive) and Tomcat (servlet 3.1 async).

---

## Load Balancing

**Q20. What are the common load balancing algorithms? Compare them.**

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Distribute requests cyclically | Homogeneous servers, similar request cost |
| Weighted Round Robin | Like round robin but proportional to capacity | Mixed server capacities |
| Least Connections | Route to server with fewest active connections | Long-lived connections, varying request duration |
| Least Response Time | Route to fastest-responding server | Latency-sensitive applications |
| IP Hash | Hash client IP to pick server | Session affinity (sticky sessions) |
| Random | Random server selection | Similar to round robin, simpler |
| Resource-Based | Choose server with most available resources | CPU/memory-intensive workloads |

**Sticky sessions (session affinity):** Route all requests from a client to the same server. Needed when application state is stored locally. Better solution: externalize session state (Redis, database) to allow any server to handle any request.

---

**Q21. What is the difference between L4 and L7 load balancing?**

**L4 (Transport Layer):**
- Routes based on IP address and TCP/UDP port
- Does not inspect packet content
- Very fast, minimal processing
- Examples: AWS Network Load Balancer, HAProxy (TCP mode)
- Cannot make routing decisions based on URL, headers, or cookies

**L7 (Application Layer):**
- Routes based on HTTP headers, URL, cookies, body content
- Can terminate SSL, add/remove headers, rewrite URLs
- Richer routing rules: `api/users/*` → users service, `api/orders/*` → orders service
- Examples: AWS Application Load Balancer, Nginx, Envoy, Traefik
- Higher processing overhead than L4

For microservices routing, L7 is standard. For raw throughput where every microsecond counts, L4.

---

## Connection Pooling and Timeouts

**Q22. Why is connection pooling essential for database connections? What happens without it?**

Opening a database connection is expensive:
1. TCP three-way handshake (~1ms)
2. TLS handshake (~2-5ms)
3. Database authentication and session setup (~2-5ms)
4. Total: ~5-10ms per connection

Without a pool: each request pays 5-10ms just to get a connection, plus connection teardown. A 100 RPS service would create 100 connections/second — typical database limit is 100-500 total connections.

With HikariCP (default in Spring Boot):
- Connections created once, reused across requests
- Pool checks connection health before returning to caller
- Configurable timeouts prevent waiting forever
- Monitors pool metrics (active, idle, waiting)

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10       # Max connections in pool
  minimum-idle: 5             # Always keep 5 warm connections
  connection-timeout: 5000    # Max wait to get connection from pool
  idle-timeout: 600000        # Remove idle connections after 10 min
  max-lifetime: 1800000       # Recycle connections every 30 min
```

---

**Q23. What are the types of timeouts you need to configure for HTTP clients?**

When making HTTP calls from a Java backend (RestTemplate, WebClient, OkHttpClient):

```java
// OkHttpClient example
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(3, TimeUnit.SECONDS)   // Time to establish TCP connection
    .readTimeout(10, TimeUnit.SECONDS)     // Time to read response body
    .writeTimeout(5, TimeUnit.SECONDS)     // Time to write request body
    .callTimeout(15, TimeUnit.SECONDS)     // End-to-end timeout for entire call
    .build();
```

**Types:**
- **Connect timeout**: time to complete TCP handshake + TLS handshake. Set to 2-5 seconds. If the remote host is unreachable, fail fast.
- **Read timeout**: time between receiving bytes from the server. Set based on expected response time (P99 + buffer).
- **Write timeout**: time to write request body. Relevant for large uploads.
- **Call/request timeout**: maximum end-to-end time. Safety net for all phases combined.

**Never omit timeouts** — a slow upstream service will hold your threads indefinitely, causing thread pool exhaustion and cascading failures.

---

**Q24. What is HTTP content negotiation? How does it work?**

Content negotiation allows clients to request the representation format they prefer.

**Headers:**
- `Accept: application/json, application/xml;q=0.8` — client prefers JSON, accepts XML at 80% preference
- `Accept-Language: en-US, en;q=0.9`
- `Accept-Encoding: gzip, deflate, br`

**Spring MVC content negotiation order (default):**
1. Path extension: `/users/1.json` (deprecated)
2. Request parameter: `/users/1?format=json`
3. `Accept` header (most standard)

```java
// Spring Boot: register XML support alongside JSON
@Bean
public WebMvcConfigurer contentNegotiationConfigurer() {
    return new WebMvcConfigurer() {
        @Override
        public void configureContentNegotiation(ContentNegotiationConfigurer config) {
            config.favorParameter(false)
                  .ignoreAcceptHeader(false)
                  .defaultContentType(MediaType.APPLICATION_JSON);
        }
    };
}
```

Server responds with `Content-Type: application/json` indicating the actual format sent.

---

**Q25. What is CORS and how does it work? How do you configure it in Spring Boot?**

CORS (Cross-Origin Resource Sharing) is a browser security mechanism. The **Same-Origin Policy** blocks browser JavaScript from making requests to a different origin (scheme + host + port).

CORS allows servers to explicitly permit cross-origin requests via response headers.

**Simple request** (GET, POST with limited headers): browser adds `Origin` header, server responds with `Access-Control-Allow-Origin`.

**Preflight request** (PUT, DELETE, custom headers): browser sends `OPTIONS` first:
```
OPTIONS /api/orders
Origin: https://frontend.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization, Content-Type

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 3600   ← cache preflight for 1 hour
```

**Spring Boot CORS configuration:**
```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

Never use `allowedOrigins("*")` with `allowCredentials(true)` — browsers reject this.
