# Networking — Structured Interview Answers

---

## Q1: How does Java's HttpClient work? (Java 11+)

**Answer:** Java 11 introduced `java.net.http.HttpClient` as a modern replacement for the legacy `HttpURLConnection`. It is built on non-blocking I/O, supports HTTP/1.1 and HTTP/2, and offers both synchronous and asynchronous APIs.

**Core design:**
- `HttpClient` is immutable and thread-safe — create once and reuse across threads.
- `HttpRequest` describes a single request (URI, method, headers, body).
- `HttpResponse` carries the status, headers, and body processed through a `BodyHandler`.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

// Build the client once (expensive — reuse it)
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)          // prefer HTTP/2, fall back to 1.1
        .connectTimeout(Duration.ofSeconds(5))
        .followRedirects(HttpClient.Redirect.NORMAL) // follow 3xx except HTTPS→HTTP
        .executor(Executors.newFixedThreadPool(10))  // custom thread pool (optional)
        .build();

// Synchronous GET
HttpRequest getRequest = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users/42"))
        .header("Accept", "application/json")
        .timeout(Duration.ofSeconds(10))
        .GET()
        .build();

HttpResponse<String> response = client.send(
        getRequest, HttpResponse.BodyHandlers.ofString());

System.out.println(response.statusCode());  // 200
System.out.println(response.body());        // JSON string

// Asynchronous POST with JSON body
String json = """
        {"name": "Alice", "email": "alice@example.com"}
        """;

HttpRequest postRequest = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(json))
        .build();

CompletableFuture<HttpResponse<String>> future = client.sendAsync(
        postRequest, HttpResponse.BodyHandlers.ofString());

future.thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .exceptionally(ex -> { ex.printStackTrace(); return null; });
```

**Key `BodyHandlers`:**
- `ofString()` — decodes the body as a UTF-8 string
- `ofInputStream()` — streaming; good for large responses
- `ofFile(Path)` — writes directly to a file
- `ofByteArray()` — raw bytes
- `discarding()` — ignores the body (useful when you only care about status)

**HTTP/2 behaviour:** The `HttpClient` will attempt HTTP/2 via ALPN during the TLS handshake. If the server supports it, subsequent requests to the same origin reuse one multiplexed connection automatically.

---

## Q2: How do you configure connection timeout vs read timeout in Spring RestTemplate?

**Answer:** `RestTemplate` delegates connection management to an underlying `ClientHttpRequestFactory`. The default `SimpleClientHttpRequestFactory` wraps `HttpURLConnection`; for production use, the Apache `HttpComponentsClientHttpRequestFactory` offers more control.

```java
import org.apache.hc.client5.http.classic.HttpClient;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManager;
import org.apache.hc.core5.util.Timeout;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(Timeout.ofSeconds(3))      // TCP handshake limit
                .setResponseTimeout(Timeout.ofSeconds(10))    // time until first byte
                .setConnectionRequestTimeout(Timeout.ofSeconds(2)) // wait for pool slot
                .build();

        PoolingHttpClientConnectionManager cm =
                new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(200);          // max connections across all routes
        cm.setDefaultMaxPerRoute(50); // max connections per host

        HttpClient httpClient = HttpClients.custom()
                .setConnectionManager(cm)
                .setDefaultRequestConfig(requestConfig)
                .evictExpiredConnections()
                .build();

        HttpComponentsClientHttpRequestFactory factory =
                new HttpComponentsClientHttpRequestFactory(httpClient);

        return new RestTemplate(factory);
    }
}
```

**Three distinct timeouts explained:**

| Timeout | What it controls | Typical value |
|---------|-----------------|--------------|
| `connectTimeout` | Time to complete TCP (+ TLS) handshake | 1–5 seconds |
| `responseTimeout` (read timeout) | Time to wait for first response byte after request sent | 5–30 seconds depending on SLA |
| `connectionRequestTimeout` | Time to wait for a free connection from the pool | 1–3 seconds |

Missing the `connectionRequestTimeout` is a common oversight: under heavy load all 50 connections per route may be in use, and without this timeout a thread can block indefinitely waiting for one to be freed.

**Note:** `RestTemplate` is in maintenance mode. Prefer `WebClient` (from Spring WebFlux) for new code:
```java
WebClient client = WebClient.builder()
        .baseUrl("https://api.example.com")
        .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                        .responseTimeout(Duration.ofSeconds(10))))
        .build();
```

---

## Q3: What is the difference between HTTP/1.1 keep-alive and HTTP/2 multiplexing?

**Answer:** Both reuse TCP connections but they do so in fundamentally different ways.

**HTTP/1.1 keep-alive** (persistent connection):
- The `Connection: keep-alive` header tells the server not to close the TCP socket after responding.
- Requests are still **sequential**: the client sends request 2 only after receiving the full response to request 1 (head-of-line blocking within a connection).
- To achieve parallelism, browsers open up to 6 TCP connections per origin, multiplying the TLS handshake overhead.

**HTTP/2 multiplexing:**
- A single TCP connection carries many **streams** simultaneously as independent binary frames, each tagged with a stream ID.
- No head-of-line blocking at the HTTP layer (though TCP-level HOL blocking remains — addressed by HTTP/3/QUIC).
- Headers are compressed with HPACK, dramatically reducing per-request overhead.
- Stream priorities allow the client to indicate which resources are most urgent.

```
HTTP/1.1 with keep-alive (pipelined, which browsers rarely use):
  TCP conn 1:  [Request1]─────[Response1][Request2]────[Response2]
  TCP conn 2:  [Request3]─────────[Response3]
  TCP conn 3:  [Request4]──[Response4]

HTTP/2 (one connection):
  Stream 1:  [HDR][DATA]...[DATA]
  Stream 3:  [HDR][DATA][DATA]
  Stream 5:  [HDR][DATA]...[DATA]...[DATA]
  → interleaved on wire: [S1:HDR][S3:HDR][S5:HDR][S1:DATA][S3:DATA]...
```

**Practical implication:** For HTTP/2, you should configure your connection pool with a **small** number of connections (often 1) per host, because one connection already carries all streams. Creating many connections to the same HTTP/2 server defeats the purpose.

---

## Q4: How does SSL/TLS handshake work in Java?

**Answer:** Java's `javax.net.ssl` package handles TLS transparently. When you connect to an HTTPS URL, the JVM negotiates TLS using the `SSLContext`, `KeyManager` (for client certificates), and `TrustManager` (for verifying server certificates).

```java
import javax.net.ssl.*;
import java.security.*;
import java.security.cert.X509Certificate;

// 1. Default usage (trusts JVM's default CA store — cacerts)
HttpsURLConnection conn = (HttpsURLConnection)
        new URL("https://api.example.com").openConnection();
// TLS handshake happens automatically on connect

// 2. Custom TrustManager — load a self-signed CA (e.g. for internal services)
KeyStore trustStore = KeyStore.getInstance("JKS");
try (InputStream is = new FileInputStream("truststore.jks")) {
    trustStore.load(is, "changeit".toCharArray());
}

TrustManagerFactory tmf = TrustManagerFactory.getInstance(
        TrustManagerFactory.getDefaultAlgorithm());
tmf.init(trustStore);

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, tmf.getTrustManagers(), new SecureRandom());

HttpClient client = HttpClient.newBuilder()
        .sslContext(sslContext)
        .build();

// 3. Mutual TLS (mTLS) — client also presents a certificate
KeyStore keyStore = KeyStore.getInstance("PKCS12");
try (InputStream is = new FileInputStream("client.p12")) {
    keyStore.load(is, "password".toCharArray());
}

KeyManagerFactory kmf = KeyManagerFactory.getInstance(
        KeyManagerFactory.getDefaultAlgorithm());
kmf.init(keyStore, "password".toCharArray());

SSLContext mtlsContext = SSLContext.getInstance("TLS");
mtlsContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);
```

**TLS handshake steps (TLS 1.2):**
1. Client sends `ClientHello` with supported cipher suites and a random nonce.
2. Server responds with `ServerHello` (chosen cipher), its certificate, and `ServerHelloDone`.
3. Client verifies the certificate chain against the `TrustManager`'s trust anchors.
4. Client sends a pre-master secret encrypted with the server's public key.
5. Both sides derive the symmetric session key from the pre-master secret + nonces.
6. Both send `ChangeCipherSpec` + `Finished` (an encrypted MAC of the handshake).
7. Application data flows encrypted with AES-GCM (or ChaCha20-Poly1305).

**TLS 1.3 improvement:** The handshake is reduced to one round-trip (down from two). Session resumption can achieve 0-RTT (though this has replay-attack considerations).

---

## Q5: What is TCP backlog and why does it matter?

**Answer:** When a server's listening socket receives a `SYN` packet, the OS places the half-open connection in the **SYN queue** (incomplete connections). Once the three-way handshake completes, the connection moves to the **accept queue** (complete connections ready to be accepted by the application). The `backlog` parameter controls the size of the accept queue.

```java
import java.net.*;
import java.io.*;

// backlog = 50: up to 50 fully-established connections can queue
// before the OS starts dropping new incoming connections (or sending RST)
ServerSocket serverSocket = new ServerSocket(8080, 50);

while (true) {
    Socket client = serverSocket.accept(); // picks from the accept queue
    new Thread(() -> handleClient(client)).start();
}
```

**Why it matters:**
- If the application is slow to call `accept()` (e.g. blocked on slow initialisation), the accept queue fills up and the OS starts rejecting new connections with `ECONNREFUSED` or silently drops SYN packets.
- Under a traffic spike or GC pause, a properly sized backlog acts as a buffer preventing connection rejections.
- In Spring Boot / Tomcat, the backlog is configured via `server.tomcat.accept-count` (default: 100). The complete queue depth is `max-connections + accept-count`.

```properties
# application.properties
server.tomcat.max-connections=8192
server.tomcat.accept-count=200
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
```

**SYN flood mitigation:** Linux's `tcp_syncookies` sysctl encodes connection state in the SYN-ACK's sequence number, eliminating the SYN queue and making the backlog irrelevant for incomplete connections.

---

## Q6: How would you implement a simple TCP server in Java using Sockets?

**Answer:** A basic multi-threaded TCP echo server demonstrating socket lifecycle, framing, and proper resource management:

```java
import java.net.*;
import java.io.*;
import java.util.concurrent.*;

public class TcpEchoServer {

    private final int port;
    private final ExecutorService threadPool;

    public TcpEchoServer(int port, int threadPoolSize) {
        this.port = port;
        this.threadPool = Executors.newFixedThreadPool(threadPoolSize);
    }

    public void start() throws IOException {
        // SO_REUSEADDR lets us restart quickly without "Address already in use"
        try (ServerSocket serverSocket = new ServerSocket()) {
            serverSocket.setReuseAddress(true);
            serverSocket.setSoTimeout(0); // block forever on accept
            serverSocket.bind(new InetSocketAddress(port), 100); // backlog=100

            System.out.println("Server listening on port " + port);

            while (!Thread.currentThread().isInterrupted()) {
                Socket client = serverSocket.accept();
                threadPool.submit(() -> handleClient(client));
            }
        }
    }

    private void handleClient(Socket socket) {
        String clientAddr = socket.getRemoteSocketAddress().toString();
        System.out.println("Connected: " + clientAddr);

        try (socket;
             DataInputStream in = new DataInputStream(socket.getInputStream());
             DataOutputStream out = new DataOutputStream(socket.getOutputStream())) {

            socket.setSoTimeout(30_000); // 30s read timeout per message

            while (true) {
                // Length-prefix framing: read 4-byte length, then that many bytes
                int length;
                try {
                    length = in.readInt();
                } catch (EOFException e) {
                    break; // client disconnected cleanly
                }

                if (length <= 0 || length > 1_048_576) { // sanity check: max 1MB
                    System.out.println("Invalid frame length: " + length);
                    break;
                }

                byte[] data = in.readNBytes(length);
                String message = new String(data, java.nio.charset.StandardCharsets.UTF_8);
                System.out.println("Received from " + clientAddr + ": " + message);

                // Echo back
                byte[] response = ("Echo: " + message)
                        .getBytes(java.nio.charset.StandardCharsets.UTF_8);
                out.writeInt(response.length);
                out.write(response);
                out.flush();
            }
        } catch (SocketTimeoutException e) {
            System.out.println("Client timed out: " + clientAddr);
        } catch (IOException e) {
            System.out.println("Error with " + clientAddr + ": " + e.getMessage());
        } finally {
            System.out.println("Disconnected: " + clientAddr);
        }
    }

    public static void main(String[] args) throws IOException {
        new TcpEchoServer(8080, 50).start();
    }
}
```

**Key points in this implementation:**
- `setReuseAddress(true)` — prevents "Address already in use" on restart before TIME_WAIT expires.
- `setSoTimeout(30_000)` — per-socket read timeout; throws `SocketTimeoutException` on idle connections.
- Length-prefix framing — solves TCP's byte-stream nature (no message boundaries).
- `try-with-resources` on `Socket` — ensures the socket is always closed even on exception.
- Dedicated `ExecutorService` — prevents thread exhaustion; a new connection gets a thread from the pool.

---

## Q7: What are the risks of not closing HTTP connections?

**Answer:** Failing to close HTTP connections causes resource leaks at multiple layers:

**1. Connection pool exhaustion:** If connections are never returned to the pool, it fills up. Subsequent requests block waiting for a free slot until `connectionRequestTimeout` expires — your service hangs.

**2. File descriptor (FD) exhaustion:** Each TCP socket consumes an OS file descriptor. The default FD limit on Linux is 1024 per process. Leaking connections drains this, eventually causing `java.net.SocketException: Too many open files`, which kills all further I/O including log file writes.

**3. Server-side resource waste:** The remote server keeps its end of the TCP connection open, wasting memory and FDs on the server too.

```java
// WRONG — HttpURLConnection not closed, response stream not consumed
URL url = new URL("https://api.example.com/data");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
String body = new BufferedReader(new InputStreamReader(conn.getInputStream()))
        .lines().collect(Collectors.joining());
// Missing: conn.disconnect() and stream close — leaks!

// CORRECT — always use try-with-resources
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .build();

// HttpClient manages connections internally; just don't leak the client itself
HttpResponse<String> response = httpClient.send(
        request, HttpResponse.BodyHandlers.ofString());
// Body is fully consumed by ofString() — connection returned to pool automatically

// When using InputStream body handler, YOU must consume and close the stream
HttpResponse<InputStream> streamResponse = httpClient.send(
        request, HttpResponse.BodyHandlers.ofInputStream());
try (InputStream body = streamResponse.body()) {
    // process body...
} // stream closed → connection returned to pool
```

**4. TIME_WAIT accumulation:** If you close many connections on the client side (instead of reusing them), the OS enters TIME_WAIT state for each closed TCP connection (lasting ~2 minutes). Under high throughput this can exhaust the ephemeral port range (typically 32,768–60,999 ports), causing `java.net.BindException: Address already in use`.

The solution: use a connection pool, always consume response bodies, and use try-with-resources for streams.

---

## Q8: How does CORS work and how do you configure it in Spring Boot?

**Answer:** CORS (Cross-Origin Resource Sharing) is a browser security mechanism. A browser executing JavaScript from `https://frontend.com` will block requests to `https://api.backend.com` unless `api.backend.com` explicitly permits cross-origin access via response headers.

**Simple requests** (GET, POST with non-JSON content type) receive just:
```
Access-Control-Allow-Origin: https://frontend.com
```

**Preflight** (non-simple methods, custom headers): the browser sends `OPTIONS` first:
```
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```
Server responds:
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
Access-Control-Allow-Credentials: true
```

**Spring Boot configuration:**

```java
// Option 1: Global CORS via WebMvcConfigurer
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.com", "https://staging.frontend.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)   // needed if using cookies/Authorization header
                .maxAge(3600);
    }
}

// Option 2: Per-controller annotation
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "https://frontend.com",
             allowedHeaders = {"Content-Type", "Authorization"},
             maxAge = 3600)
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) { ... }
}

// Option 3: Spring Security CORS (use this when Spring Security is present —
// the WebMvcConfigurer config is bypassed by Security filter chain)
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://frontend.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

**Common mistake:** Using `allowedOrigins("*")` with `allowCredentials(true)` — browsers reject this combination because wildcard origin with credentials is a security hole. You must list specific origins when allowing credentials.

---

## Q9: Difference between synchronous and asynchronous HTTP clients in Java

**Answer:** The distinction is about thread utilisation while waiting for I/O.

**Synchronous (blocking):** The calling thread blocks until the full response is received. Simple, easy to reason about, but each concurrent request needs its own thread.

```java
// Synchronous — thread blocks here until response arrives
HttpResponse<String> response = httpClient.send(
        request, HttpResponse.BodyHandlers.ofString());
// thread continues here
processResponse(response.body());
```

**Asynchronous (non-blocking):** The calling thread submits the request and continues. A callback or `CompletableFuture` fires when the response is ready.

```java
// Asynchronous — calling thread returns immediately
CompletableFuture<String> future = httpClient
        .sendAsync(request, HttpResponse.BodyHandlers.ofString())
        .thenApply(HttpResponse::body)
        .thenApply(body -> {
            // This runs on HttpClient's internal executor, not calling thread
            return processBody(body);
        });

// Chain multiple async calls without blocking threads
CompletableFuture<UserProfile> profile = fetchUser(userId)
        .thenCompose(user -> fetchOrders(user.getId()))
        .thenCompose(orders -> fetchRecommendations(orders))
        .thenApply(UserProfile::build);
```

**Reactive (WebClient — Spring WebFlux):**
```java
// Fully reactive — no threads blocked at any point
Mono<String> response = webClient.get()
        .uri("/api/users/{id}", userId)
        .retrieve()
        .bodyToMono(String.class);

// Compose without threads
Flux<Order> orders = webClient.get()
        .uri("/api/orders")
        .retrieve()
        .bodyToFlux(Order.class)
        .filter(o -> o.getStatus() == Status.PENDING);
```

**When to use which:**

| Scenario | Choice |
|----------|--------|
| Simple batch script, CLI tool | Synchronous (`HttpClient.send()`) |
| Traditional Spring MVC service with moderate load | Synchronous with connection pool |
| High-concurrency service (thousands of concurrent requests) | Async (`CompletableFuture`) or Reactive (`WebClient`) |
| Microservice calling multiple downstream APIs in parallel | Async / Reactive — zip/combine futures |
| Streaming large responses | `BodyHandlers.ofInputStream()` or reactive |

The key insight: blocking I/O is fine when you have few concurrent requests per instance. Async becomes necessary when you have many concurrent requests and cannot afford one thread per request (thread pools top out at ~200-500 threads; async I/O can handle tens of thousands of concurrent connections with the same thread count).

---

## Q10: What is a socket timeout and what exception does Java throw?

**Answer:** A socket timeout (`SO_TIMEOUT`) defines the maximum time a read operation will block waiting for data. It is set at the socket level and applies to each individual `read()` call, not the total transfer time.

```java
import java.net.*;
import java.io.*;

Socket socket = new Socket();
socket.connect(new InetSocketAddress("api.example.com", 80), 3000); // connect timeout: 3s
socket.setSoTimeout(10_000); // read timeout: 10s per read() call

try {
    InputStream in = socket.getInputStream();
    byte[] buffer = new byte[4096];
    int bytesRead = in.read(buffer); // blocks up to 10 seconds
    // if nothing arrives in 10s → throws SocketTimeoutException
} catch (SocketTimeoutException e) {
    // Specific subclass of IOException — connection is still valid,
    // you can retry the read or close the socket
    System.out.println("Read timed out after 10 seconds: " + e.getMessage());
    socket.close();
} catch (IOException e) {
    // Other I/O errors (connection reset, broken pipe etc.)
}

// With HttpURLConnection
URL url = new URL("https://api.example.com/slow-endpoint");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setConnectTimeout(3_000);  // TCP handshake: 3s
conn.setReadTimeout(10_000);    // Response data: 10s

try {
    int status = conn.getResponseCode(); // triggers connection + first read
} catch (SocketTimeoutException e) {
    System.out.println("Timed out: " + e.getMessage());
    // SocketTimeoutException extends InterruptedIOException extends IOException
}
```

**Exception hierarchy:**
```
java.io.IOException
  └── java.io.InterruptedIOException
        └── java.net.SocketTimeoutException
              message: "Read timed out" (for SO_TIMEOUT)
              message: "connect timed out" (for connect timeout)
```

**Important nuance:** `SocketTimeoutException` does **not** mean the connection is broken. After a timeout on a read, the socket is still valid and you could retry. However, in most application code you should close and discard the socket/connection because the server's state is now unknown — it may have processed part of your request and be waiting for the rest, or may be mid-response.

**Timeout types summary:**
```
Connection Timeout: time to complete TCP handshake
  throws: SocketTimeoutException ("connect timed out")

Read/Socket Timeout (SO_TIMEOUT): time between data chunks during read
  throws: SocketTimeoutException ("Read timed out")

Connection Request Timeout: time waiting for a connection pool slot
  throws: ConnectionPoolTimeoutException (Apache HttpClient)
          or TimeoutException (HikariCP)

Response Timeout (Java 11 HttpClient): total wall-clock time for entire request
  throws: HttpTimeoutException
```
