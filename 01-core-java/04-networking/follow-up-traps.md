# Networking — Follow-Up Traps & Tricky Interview Questions

---

## Trap 1: "TCP is always better than UDP — why would you ever use UDP?"

**What most people say:** UDP is unreliable so you should always use TCP for safety.

**Correct answer:** UDP is the right choice when low latency matters more than perfect delivery, or when the application itself handles ordering and reliability at a higher level. DNS queries use UDP because a lost packet just triggers a fast retry and the overhead of a TCP handshake would triple the lookup time. Video conferencing (WebRTC), online games, and live video streaming use UDP because a slightly stale frame is better than waiting for a retransmit and freezing the stream. QUIC (the protocol under HTTP/3) is built on UDP precisely to avoid TCP's head-of-line blocking while adding its own reliability mechanism. The choice depends entirely on the application's tolerance for latency vs data loss.

---

## Trap 2: "Is HTTP GET idempotent? What about POST?"

**What most people say:** GET is safe, POST creates things, PUT updates things — standard REST.

**Correct answer:** Idempotency and safety are distinct properties. A method is **safe** if it does not modify server state. A method is **idempotent** if calling it N times produces the same result as calling it once.

| Method  | Safe | Idempotent |
|---------|------|-----------|
| GET     | Yes  | Yes        |
| HEAD    | Yes  | Yes        |
| OPTIONS | Yes  | Yes        |
| PUT     | No   | Yes        |
| DELETE  | No   | Yes        |
| POST    | No   | No         |
| PATCH   | No   | Not guaranteed |

The trap is **DELETE**: deleting a resource twice is idempotent (the resource is gone after the first call; the second returns 404 but the server state is the same). The trap is also **PATCH**: PATCH is not guaranteed idempotent because `PATCH /user/balance {"increment": 10}` has a different effect each call. Interviewers love asking about DELETE and PATCH because candidates often get them wrong.

---

## Trap 3: "WebSocket vs Server-Sent Events (SSE) — when do you choose one over the other?"

**What most people say:** WebSocket is for real-time communication, SSE is outdated.

**Correct answer:** SSE (Server-Sent Events) is not outdated — it is the better choice when communication is **unidirectional from server to client**. SSE uses plain HTTP (no protocol upgrade), works through HTTP/2 naturally, supports automatic reconnect built into the browser, and is simpler to implement. Use cases: live news feed, server log streaming, progress notifications.

WebSocket is right when you need **bidirectional** communication: chat, collaborative editing, multiplayer games where the client also sends data frequently.

The hidden trap: SSE does not work well with HTTP/1.1 load balancers that buffer responses. WebSocket requires sticky sessions (IP hash) at the load balancer or a pub/sub broker (Redis, Kafka) to broadcast to clients on different servers. Neither is "better" — they serve different traffic patterns.

---

## Trap 4: "What is the difference between Connection: keep-alive in HTTP/1.1 and HTTP/2 multiplexing?"

**What most people say:** They are the same thing — both reuse TCP connections.

**Correct answer:** They are fundamentally different. HTTP/1.1 keep-alive reuses a single TCP connection for **sequential** requests — request 2 cannot start until the response to request 1 is fully received (head-of-line blocking). The browser works around this by opening up to 6 parallel TCP connections per origin. HTTP/2 multiplexing sends **many requests concurrently** as independent streams over a **single** TCP connection — there is no head-of-line blocking at the HTTP layer (though TCP-level HOL blocking still exists, which HTTP/3/QUIC addresses). HTTP/2 also adds header compression (HPACK) and server push. With HTTP/2 you want **fewer** connections, not more; connection pooling strategies for HTTP/2 differ significantly from HTTP/1.1.

---

## Trap 5: "Does a CORS preflight request happen for every cross-origin request?"

**What most people say:** Yes, every cross-origin request triggers a preflight OPTIONS request.

**Correct answer:** No. Browsers only send a preflight for **non-simple** requests. A request is "simple" if it uses GET, HEAD, or POST with a `Content-Type` of `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`, and has no custom headers. A simple cross-origin GET or form POST does **not** trigger a preflight. The preflight is triggered when you use PUT, DELETE, PATCH, or POST with `Content-Type: application/json`, or add custom headers like `Authorization`. This matters for performance: an API that uses GET for reads avoids preflights entirely, but a POST with JSON body pays an extra round trip.

In Spring Boot, you configure CORS like this:

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("Authorization", "Content-Type")
                .allowCredentials(true)
                .maxAge(3600); // Cache preflight for 1 hour
    }
}
```

The `maxAge` is critical — it tells the browser to cache the preflight response, eliminating repeated OPTIONS calls.

---

## Trap 6: "How do you enable gzip compression when using Java 11's HttpClient?"

**What most people say:** Just add `Accept-Encoding: gzip` to the request headers and Java handles the rest.

**Correct answer:** Java's built-in `HttpClient` does **not** automatically decompress gzip responses. You must handle decompression yourself. Setting `Accept-Encoding: gzip` tells the server to send compressed bytes, but Java will hand you those raw compressed bytes — you will get garbled output if you try to read them as a String directly.

```java
import java.net.http.*;
import java.net.URI;
import java.util.zip.GZIPInputStream;
import java.io.*;
import java.nio.charset.StandardCharsets;

HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .header("Accept-Encoding", "gzip")
        .GET()
        .build();

HttpResponse<InputStream> response = client.send(
        request, HttpResponse.BodyHandlers.ofInputStream());

String encoding = response.headers()
        .firstValue("Content-Encoding").orElse("");

String body;
if ("gzip".equalsIgnoreCase(encoding)) {
    try (GZIPInputStream gzipStream = new GZIPInputStream(response.body());
         InputStreamReader reader = new InputStreamReader(gzipStream, StandardCharsets.UTF_8)) {
        body = new String(reader.readAllBytes(), StandardCharsets.UTF_8);
    }
} else {
    body = new String(response.body().readAllBytes(), StandardCharsets.UTF_8);
}
```

Compare this with OkHttp or Apache HttpClient, which handle decompression transparently.

---

## Trap 7: "What is the difference between connection timeout and read timeout? Which one is more dangerous to omit?"

**What most people say:** They are both just timeouts, one is for connecting and one is for reading.

**Correct answer:** They are distinct and both critical, but the **read timeout** is more dangerous to omit in practice. A connection timeout fires if the TCP handshake does not complete within the configured time (the server is unreachable or overloaded). This typically fails fast. A read timeout fires if no data arrives from the server within the window after the connection is established. If a downstream service accepts your connection but then hangs internally (deadlock, GC pause, slow query), the read timeout is the only protection you have. Without it, your thread blocks indefinitely holding a connection pool slot, and under load all threads block, causing your entire service to hang — a cascading failure.

```java
// Spring RestTemplate example — ALWAYS configure both
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(3_000);   // 3s to establish TCP connection
    factory.setReadTimeout(10_000);     // 10s to receive response data
    return new RestTemplate(factory);
}

// Java 11 HttpClient example
HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))
        .build();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/slow"))
        .timeout(Duration.ofSeconds(10)) // This is the overall request timeout
        .GET()
        .build();
```

Note: Java 11's `HttpClient` has no separate read timeout — `.timeout()` on the request is the total request timeout (connect + read combined).

---

## Trap 8: "HTTP/2 Server Push is great for performance — should you use it aggressively?"

**What most people say:** Yes, push CSS and JS before the browser asks for them to save round trips.

**Correct answer:** Server Push is largely considered a failed feature in practice. The core problem is that the server cannot know what is already in the browser's cache. If the browser already has the CSS file cached, the server pushes it anyway, wasting bandwidth. The browser can send a `RST_STREAM` to cancel an unwanted push, but the server has already started sending by then. Studies (including Chrome's own data) showed that Server Push often made performance worse rather than better. Chrome removed Server Push support in Chrome 106. The correct modern approach is to use `<link rel="preload">` or the `103 Early Hints` status code, which lets the browser decide whether to fetch based on its cache. HTTP/3 (QUIC) does not support Server Push at all.

---

## Trap 9: "What is TLS certificate pinning and what are its risks?"

**What most people say:** Certificate pinning makes HTTPS more secure by hardcoding the expected certificate.

**Correct answer:** Certificate pinning (or public key pinning) makes a client refuse to connect unless the server presents a specific certificate or public key, preventing MITM attacks even from rogue trusted CAs. However, the risks are severe and often outweigh the benefits for most applications:

1. **Operational risk**: If the server's certificate is renewed (which happens at least annually), the old pin no longer matches and all clients that have not updated will be broken — you get an outage.
2. **Update dependency**: Mobile apps with pinning must ship an update before certificate rotation, creating a tight coupling between certificate management and release cycles.
3. **Debugging difficulty**: Corporate proxies and developer MITM proxies (like Charles) are blocked, making troubleshooting production issues harder.

Pinning is appropriate for: banking apps, government services, apps that cannot tolerate any MITM. For most REST APIs, relying on a well-maintained certificate chain with certificate transparency logs is sufficient.

In Java (OkHttp):
```java
CertificatePinner pinner = new CertificatePinner.Builder()
        .add("api.example.com",
             "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
        .build();

OkHttpClient client = new OkHttpClient.Builder()
        .certificatePinner(pinner)
        .build();
```

---

## Trap 10: "Is REST stateless? What does stateless actually mean?"

**What most people say:** REST is stateless means the server doesn't store sessions — use JWTs instead of server-side sessions.

**Correct answer:** Statelessness in REST means **each request must contain all the information necessary to understand and process it** — the server must not rely on context from a previous request stored server-side. This is a constraint on the server, not the client. A JWT in an `Authorization` header makes REST stateless because the token carries the identity and claims inline. A session cookie referencing a server-side session object violates REST's stateless constraint because the server must look up state to understand the request.

The trap: statelessness has a **tradeoff**. Truly stateless APIs send credentials/tokens on every request, which means: (a) tokens must be validated on every request (CPU cost or network cost for remote validation), (b) you cannot instantly revoke a JWT short of maintaining a blacklist (which re-introduces state). Short-lived tokens + refresh tokens are the pragmatic compromise. Saying "JWT is always better than sessions" is wrong — session stores with sticky load balancing or distributed caches (Redis) work perfectly well at scale and support instant revocation.

---

## Trap 11: "Can TCP guarantee message boundaries (framing)?"

**What most people say:** TCP is reliable so if I send a 100-byte message, the receiver will read exactly 100 bytes.

**Correct answer:** TCP is a **byte stream** protocol with no concept of message boundaries. If you send 100 bytes, the receiver might read them as one 100-byte chunk, two 50-byte chunks, or 100 one-byte reads — it depends on buffering, Nagle's algorithm, and network conditions. This is a classic gotcha when writing raw TCP servers in Java: you cannot assume one `socket.getInputStream().read()` call returns one complete message.

You must implement your own framing protocol. Common approaches:
```java
// Approach 1: Length-prefix framing
// Send: [4-byte length][payload bytes]
DataOutputStream out = new DataOutputStream(socket.getOutputStream());
byte[] payload = "Hello".getBytes(StandardCharsets.UTF_8);
out.writeInt(payload.length);   // 4-byte big-endian length
out.write(payload);

DataInputStream in = new DataInputStream(socket.getInputStream());
int length = in.readInt();          // Always reads exactly 4 bytes
byte[] data = in.readNBytes(length); // Always reads exactly `length` bytes

// Approach 2: Delimiter-based (e.g. newline for text protocols)
BufferedReader reader = new BufferedReader(
        new InputStreamReader(socket.getInputStream()));
String line = reader.readLine(); // reads until \n
```

Netty, gRPC, and most protocol frameworks handle framing automatically.

---

## Trap 12: "What happens to in-flight HTTP requests during a server restart / rolling deploy?"

**What most people say:** Requests fail during the restart — you need to handle retries on the client.

**Correct answer:** With a proper **graceful shutdown** the server stops accepting new connections but completes all in-flight requests before shutting down. In Spring Boot:

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

This tells Spring to stop accepting new requests immediately but give existing requests up to 30 seconds to complete. The load balancer should also stop routing new traffic to the instance being shut down (via health check going unhealthy) before the JVM process exits. The full sequence for zero-downtime deploys:
1. Deploy sends SIGTERM to the old instance
2. Spring sets health endpoint to DOWN
3. Load balancer stops routing to that instance (after health check interval)
4. Graceful shutdown completes in-flight requests
5. JVM exits

The trap is that GET requests to idempotent endpoints can safely be retried automatically (HTTP spec allows this), but POST/PATCH/DELETE should not be auto-retried without idempotency keys, or you risk double-processing.
