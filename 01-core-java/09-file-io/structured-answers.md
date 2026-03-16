# Chapter 09: File I/O — Structured Answers

---

## Q1: What is the Difference Between Java IO and Java NIO?

**One-line answer:** Java IO is stream-based and blocking (one thread per connection); Java NIO is buffer-based and can be non-blocking (one thread handles many connections via a Selector).

**Key differences:**

| Feature | Java IO (java.io) | Java NIO (java.nio) |
|---|---|---|
| Data unit | Stream of bytes | Buffer of bytes |
| Blocking | Always blocking | Can be non-blocking |
| Thread model | One thread per connection | One thread per Selector |
| API year | Java 1.0 | Java 1.4 |
| Direction | Unidirectional (in or out) | Bidirectional (Channel reads and writes) |
| Scatter/gather | No | Yes |
| File attributes | Limited (File class) | Full (Files, BasicFileAttributes) |
| Memory mapping | No | Yes (MappedByteBuffer) |

**IO — stream model:**
```java
// Blocking: thread waits at read() until data is available
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(socket.getInputStream()))) {
    String line = reader.readLine(); // THREAD BLOCKED here
    // Cannot do anything else while waiting
}
```

**NIO — channel + selector model:**
```java
// Non-blocking: register interest, Selector tells you when ready
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // blocks until SOMETHING is ready (any channel)

    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) {
            // New connection arrived
            SocketChannel client = serverChannel.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // Data ready to read — no blocking
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            client.read(buffer);  // non-blocking, reads whatever is available
        }
    }
    selector.selectedKeys().clear();
}
```

**NIO.2 (Java 7) — the Files API:**
```java
// NIO.2 adds a cleaner API for file operations — not about non-blocking sockets,
// but about rich, portable file system access
Path path = Path.of("/tmp/data.txt");
Files.copy(Path.of("src.txt"), path, StandardCopyOption.REPLACE_EXISTING);
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
System.out.println(attrs.lastModifiedTime());
```

---

## Q2: How Does try-with-resources Help with File I/O?

**One-line answer:** `try-with-resources` (Java 7) guarantees that any `AutoCloseable` resource is closed when the block exits — even if an exception is thrown — eliminating resource leak bugs that plagued manual `finally` blocks.

**The problem it solves:**
```java
// PRE-JAVA-7: correct resource cleanup is painful
FileInputStream fis = null;
FileOutputStream fos = null;
try {
    fis = new FileInputStream("input.txt");
    fos = new FileOutputStream("output.txt");
    // ... process ...
} catch (IOException e) {
    e.printStackTrace();
} finally {
    // Must close in finally, must handle exceptions in finally,
    // must null-check, must close in reverse order
    if (fos != null) {
        try { fos.close(); } catch (IOException e) { e.printStackTrace(); }
    }
    if (fis != null) {
        try { fis.close(); } catch (IOException e) { e.printStackTrace(); }
    }
}
```

**With try-with-resources:**
```java
// JAVA-7+: clean, correct, and readable
try (FileInputStream fis = new FileInputStream("input.txt");
     FileOutputStream fos = new FileOutputStream("output.txt")) {
    // ... process ...
    byte[] buffer = new byte[8192];
    int n;
    while ((n = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, n);
    }
} // Both fis and fos are ALWAYS closed here, even if exception thrown
  // Closed in REVERSE order: fos first, then fis
```

**Suppressed exceptions:**
```java
// What if both the body AND close() throw exceptions?
// try-with-resources handles this: body exception is primary,
// close() exceptions are "suppressed" and attached to it

try (BadResource r = new BadResource()) {
    throw new RuntimeException("primary");
} catch (RuntimeException e) {
    System.out.println("Primary: " + e.getMessage()); // "primary"
    for (Throwable suppressed : e.getSuppressed()) {
        System.out.println("Suppressed: " + suppressed.getMessage()); // from close()
    }
}
```

**Custom AutoCloseable:**
```java
// Any class implementing AutoCloseable works with try-with-resources
public class DatabaseConnection implements AutoCloseable {
    private final Connection conn;

    public DatabaseConnection(String url) throws SQLException {
        this.conn = DriverManager.getConnection(url);
    }

    @Override
    public void close() throws SQLException {
        conn.close();
        System.out.println("Connection closed");
    }
}

try (DatabaseConnection db = new DatabaseConnection("jdbc:...")) {
    // use db.conn
} // conn.close() guaranteed
```

---

## Q3: How Do You Efficiently Read a Large File Without Loading It Into Memory?

**One-line answer:** Use `Files.lines()` with try-with-resources for line-by-line streaming, or use buffered chunk reads for binary data — never `Files.readAllBytes()` or `Files.readString()` for large files.

**Option 1 — Stream lines (best for text files):**
```java
// Memory usage: one line at a time
Path logFile = Path.of("/var/log/app/application.log");

try (Stream<String> lines = Files.lines(logFile, StandardCharsets.UTF_8)) {
    long errorCount = lines
        .filter(line -> line.contains("ERROR"))
        .peek(System.out::println)
        .count();
    System.out.println("Total errors: " + errorCount);
} // stream closed, file handle released
```

**Option 2 — BufferedReader.readLine() for processing:**
```java
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("/var/log/huge.log"),
            StandardCharsets.UTF_8),
        64 * 1024)) { // 64KB buffer

    String line;
    while ((line = reader.readLine()) != null) {
        if (line.startsWith("ERROR")) {
            writeToErrorLog(line);
        }
    }
}
```

**Option 3 — Chunked binary reading:**
```java
Path binaryFile = Path.of("large-data.bin");

try (FileChannel channel = FileChannel.open(binaryFile, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(256 * 1024); // 256KB direct buffer

    while (channel.read(buffer) > 0) {
        buffer.flip();
        processChunk(buffer); // process this chunk
        buffer.clear();       // reset for next chunk
    }
}
```

**Option 4 — Memory-mapped file for random access:**
```java
try (FileChannel channel = FileChannel.open(Path.of("index.db"))) {
    // Map 512MB at a time if file > 2GB
    long chunkSize = 512L * 1024 * 1024;
    long fileSize = channel.size();
    long pos = 0;

    while (pos < fileSize) {
        long remaining = Math.min(chunkSize, fileSize - pos);
        MappedByteBuffer mapped = channel.map(
            FileChannel.MapMode.READ_ONLY, pos, remaining);

        // Random access within this chunk — no IO overhead
        while (mapped.hasRemaining()) {
            processRecord(mapped);
        }
        pos += remaining;
    }
}
```

**What to avoid:**
```java
// BAD — loads entire file into heap
byte[] all = Files.readAllBytes(path);          // OOM for 1GB+ files
String content = Files.readString(path);        // same problem
List<String> lines = Files.readAllLines(path);  // all lines in a List!
```

---

## Q4: What is Serialization? What Problems Can Arise With It?

**One-line answer:** Serialization converts a Java object graph into a byte stream for storage or transmission; deserialization reconstructs it. Problems include security vulnerabilities, versioning fragility, performance overhead, and tight coupling.

**Basic serialization:**
```java
public class Order implements Serializable {
    private static final long serialVersionUID = 1L;

    private long id;
    private String customerId;
    private List<OrderItem> items;  // OrderItem must also implement Serializable
    private BigDecimal total;
    private transient LocalDateTime cachedDeliveryDate; // not serialized
}

// Serialize to file
try (ObjectOutputStream oos = new ObjectOutputStream(
        new BufferedOutputStream(new FileOutputStream("order.ser")))) {
    oos.writeObject(order);
}

// Deserialize from file
try (ObjectInputStream ois = new ObjectInputStream(
        new BufferedInputStream(new FileInputStream("order.ser")))) {
    Order restored = (Order) ois.readObject();
}
```

**Problems:**

**1. Security — Deserialization Gadget Chains:**
```java
// Attacker sends crafted bytes that exploit deserialization
// Famous CVEs: Apache Commons Collections, Spring Framework, JBoss
// The exploit: crafted byte stream triggers code execution during readObject()

// DEFENSE: filter incoming streams (Java 9+)
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(info -> {
    if (info.serialClass() != null) {
        // Only allow known safe classes
        if (!WHITELIST.contains(info.serialClass().getName())) {
            return ObjectInputFilter.Status.REJECTED;
        }
    }
    return ObjectInputFilter.Status.ALLOWED;
});
```

**2. Versioning:**
```java
// Add a field to a class after deployment
public class UserProfile implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private String email; // added in v2

    // readObject can handle missing fields from older serialized forms
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        if (email == null) email = "unknown@example.com"; // default for old data
    }
}
```

**3. Performance:**
```java
// Java serialization is slow — benchmark vs JSON/Protobuf
// Java serialization: ~200MB/s throughput
// Jackson JSON:       ~400MB/s throughput
// Protocol Buffers:  ~1000MB/s throughput

// For high-performance use cases, prefer JSON, Protobuf, Avro, or Kryo
ObjectMapper mapper = new ObjectMapper(); // Jackson JSON
byte[] json = mapper.writeValueAsBytes(order);   // fast, human-readable
Order fromJson = mapper.readValue(json, Order.class); // no class sharing needed
```

**4. Tight coupling:**
```java
// Java serialization requires the SAME class on both ends
// API changes (rename field, move class, change type) break serialization
// JSON/Protobuf are more tolerant of schema evolution

// General recommendation: do NOT use Java serialization for:
// - Inter-service communication (use REST/gRPC/message queues)
// - Caching (use JSON or a dedicated serialization library)
// - Long-term persistence (use a database or structured file format)
//
// Acceptable use cases:
// - HTTP session replication (within same app, same classpath)
// - Java-to-Java RMI (rare today)
// - Short-lived in-process caching
```

---

## Q5: How Do You Watch a Directory for File Changes Using WatchService?

**One-line answer:** Register a `Path` with a `WatchService` for `ENTRY_CREATE`, `ENTRY_MODIFY`, or `ENTRY_DELETE` events, then loop calling `watcher.take()` (or `poll()`) to receive events.

```java
public class DirectoryWatcher implements Runnable, Closeable {

    private final Path directory;
    private final WatchService watchService;
    private volatile boolean running = true;

    public DirectoryWatcher(Path directory) throws IOException {
        this.directory = directory;
        this.watchService = FileSystems.getDefault().newWatchService();

        directory.register(watchService,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY,
            StandardWatchEventKinds.ENTRY_DELETE);
    }

    @Override
    public void run() {
        while (running) {
            WatchKey key;
            try {
                key = watchService.poll(1, TimeUnit.SECONDS); // timeout allows checking 'running'
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            if (key == null) continue; // timeout, no events

            for (WatchEvent<?> event : key.pollEvents()) {
                WatchEvent.Kind<?> kind = event.kind();

                if (kind == StandardWatchEventKinds.OVERFLOW) {
                    System.err.println("Events lost! Doing full rescan.");
                    rescanDirectory();
                    continue;
                }

                @SuppressWarnings("unchecked")
                WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
                Path changedFile = directory.resolve(pathEvent.context());

                if (kind == StandardWatchEventKinds.ENTRY_CREATE) {
                    onFileCreated(changedFile);
                } else if (kind == StandardWatchEventKinds.ENTRY_MODIFY) {
                    onFileModified(changedFile);
                } else if (kind == StandardWatchEventKinds.ENTRY_DELETE) {
                    onFileDeleted(changedFile);
                }
            }

            // CRITICAL: must reset — if not reset, no more events from this key
            boolean valid = key.reset();
            if (!valid) {
                System.out.println("Watch key invalid — directory deleted or inaccessible");
                break;
            }
        }
    }

    private void onFileCreated(Path file) {
        System.out.println("CREATED: " + file);
    }

    private void onFileModified(Path file) {
        System.out.println("MODIFIED: " + file);
        if (file.getFileName().toString().equals("config.properties")) {
            reloadConfig();
        }
    }

    private void onFileDeleted(Path file) {
        System.out.println("DELETED: " + file);
    }

    private void rescanDirectory() { /* full scan */ }
    private void reloadConfig() { /* reload logic */ }

    @Override
    public void close() throws IOException {
        running = false;
        watchService.close();
    }
}

// Usage
try (DirectoryWatcher watcher = new DirectoryWatcher(Path.of("/etc/myapp"))) {
    Thread watchThread = new Thread(watcher, "directory-watcher");
    watchThread.setDaemon(true);
    watchThread.start();
    // ... application runs ...
}
```

---

## Q6: What is a Memory-Mapped File and When Would You Use It?

**One-line answer:** A memory-mapped file maps a file (or region of a file) directly into the process's virtual address space, allowing the OS to page file data in/out on demand — providing fast random access without copying data to the Java heap.

```java
// Creating a memory-mapped file
Path path = Path.of("data.bin");

try (FileChannel channel = FileChannel.open(path,
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {

    // Map the entire file into memory
    MappedByteBuffer buffer = channel.map(
        FileChannel.MapMode.READ_WRITE,  // or READ_ONLY, PRIVATE
        0,                               // starting file position
        channel.size()                   // number of bytes to map
    );

    // Read — as fast as reading from an array
    buffer.position(1000);
    int value = buffer.getInt();         // reads 4 bytes from offset 1000

    // Write — changes go to OS page cache, flushed lazily to disk
    buffer.position(2000);
    buffer.putLong(System.currentTimeMillis());

    // Force writes to disk immediately (like fsync)
    buffer.force();
}
```

**High-performance log reader using memory map:**
```java
public class FastLogSearcher {

    public List<Long> findLineOffsets(Path logFile, String searchTerm)
            throws IOException {
        List<Long> offsets = new ArrayList<>();
        byte[] searchBytes = searchTerm.getBytes(StandardCharsets.UTF_8);

        try (FileChannel channel = FileChannel.open(logFile, StandardOpenOption.READ)) {
            long fileSize = channel.size();
            // Map 256MB at a time (MappedByteBuffer uses int for positions)
            long chunkSize = 256 * 1024 * 1024L;
            long position = 0;

            while (position < fileSize) {
                long size = Math.min(chunkSize, fileSize - position);
                MappedByteBuffer mapped = channel.map(
                    FileChannel.MapMode.READ_ONLY, position, size);

                // Scan the mapped region
                for (int i = 0; i < mapped.limit() - searchBytes.length; i++) {
                    if (matches(mapped, i, searchBytes)) {
                        offsets.add(position + i);
                    }
                }
                position += size;
            }
        }
        return offsets;
    }

    private boolean matches(MappedByteBuffer buf, int pos, byte[] target) {
        for (int i = 0; i < target.length; i++) {
            if (buf.get(pos + i) != target[i]) return false;
        }
        return true;
    }
}
```

**When to use memory-mapped files:**
- Large file random access (searching log files, database B-tree pages)
- Shared memory between processes on the same machine
- Zero-copy reads: bypass Java heap entirely
- Files that are read multiple times (OS caches pages in RAM)

**When NOT to use:**
- Small files (overhead not justified, just use `Files.readAllBytes()`)
- Files smaller than one OS page (4KB)
- Heavy write workloads on Windows (file stays locked while mapped)
- Very large files (>2GB) require multiple map regions since `MappedByteBuffer` uses `int` positions

---

## Q7: How Do You Copy a File Efficiently in Java?

**One-line answer:** Use `Files.copy()` for simple cases; for maximum performance use `FileChannel.transferTo()` which can use OS-level zero-copy.

```java
Path src  = Path.of("source.txt");
Path dest = Path.of("backup.txt");

// Option 1: Files.copy() — simple and correct (Java 7+)
Files.copy(src, dest,
    StandardCopyOption.REPLACE_EXISTING,  // overwrite if dest exists
    StandardCopyOption.COPY_ATTRIBUTES);  // preserve timestamps, permissions

// Option 2: FileChannel.transferTo() — zero-copy, fastest for large files
try (FileChannel in  = FileChannel.open(src,  StandardOpenOption.READ);
     FileChannel out = FileChannel.open(dest,
         StandardOpenOption.WRITE,
         StandardOpenOption.CREATE,
         StandardOpenOption.TRUNCATE_EXISTING)) {

    long position = 0;
    long size = in.size();
    while (position < size) {
        position += in.transferTo(position, size - position, out);
        // transferTo() uses sendfile(2) on Linux — data never copied to user space
    }
}

// Option 3: Buffered streams — readable but slower than transferTo
try (InputStream  is = new BufferedInputStream(new FileInputStream(src.toFile()),  64*1024);
     OutputStream os = new BufferedOutputStream(new FileOutputStream(dest.toFile()), 64*1024)) {
    byte[] buf = new byte[64 * 1024];
    int n;
    while ((n = is.read(buf)) != -1) {
        os.write(buf, 0, n);
    }
}

// Performance comparison (approximate, 1GB file, SSD):
// Files.copy():          ~800 MB/s
// FileChannel.transferTo(): ~1000 MB/s (OS zero-copy)
// Buffered streams (64KB): ~600 MB/s
// Buffered streams (4KB):  ~200 MB/s
```

---

## Q8: What is the Difference Between FileWriter and OutputStreamWriter?

**One-line answer:** `FileWriter` is a convenience subclass of `OutputStreamWriter` with a hardcoded `FileOutputStream` — but crucially, it uses the platform's default charset, making it dangerous for portable code. `OutputStreamWriter` lets you specify the charset explicitly.

```java
// FileWriter — convenience, but uses platform default charset
FileWriter fw = new FileWriter("output.txt");   // UTF-8 on Mac, maybe Cp1252 on Windows
fw.write("Hello, 世界");                         // may corrupt on wrong platform
fw.close();

// OutputStreamWriter — explicit charset, always safe
OutputStreamWriter osw = new OutputStreamWriter(
    new FileOutputStream("output.txt"),
    StandardCharsets.UTF_8);              // always UTF-8, everywhere
osw.write("Hello, 世界");                // correct on all platforms
osw.close();

// With buffering (always use buffering for performance)
try (Writer writer = new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream("output.txt"),
            StandardCharsets.UTF_8))) {
    writer.write("Line 1\n");
    writer.write("Line 2\n");
} // flush + close

// Modern one-liner (Java 11+)
Files.writeString(Path.of("output.txt"), "Hello, 世界\n",
    StandardCharsets.UTF_8,
    StandardOpenOption.CREATE,
    StandardOpenOption.APPEND);
```

**Under the hood:** `FileWriter` source code is approximately:
```java
public class FileWriter extends OutputStreamWriter {
    public FileWriter(String fileName) throws IOException {
        super(new FileOutputStream(fileName)); // charset = Charset.defaultCharset()
    }
    public FileWriter(String fileName, boolean append) throws IOException {
        super(new FileOutputStream(fileName, append));
    }
    // Java 11 added charset constructors to FileWriter, finally fixing this issue
    public FileWriter(String fileName, Charset charset) throws IOException {
        super(new FileOutputStream(fileName), charset);
    }
}
```

---

## Q9: How Do You Handle Different Character Encodings When Reading Files?

**One-line answer:** Always specify the charset explicitly when creating `InputStreamReader` or using the `Files` API; never rely on the platform default charset.

```java
// Reading UTF-8 (most common)
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("utf8file.txt"),
            StandardCharsets.UTF_8))) {
    reader.lines().forEach(System.out::println);
}

// Reading Windows-1252 (common in legacy European data files)
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("windows-file.txt"),
            Charset.forName("Windows-1252")))) {
    // ...
}

// Reading ISO-8859-1 (Latin-1)
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("latin1.txt"),
            StandardCharsets.ISO_8859_1))) {
    // ...
}

// Converting between encodings (read one encoding, write another)
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("legacy.txt"),
            Charset.forName("Windows-1252")));
     BufferedWriter writer = new BufferedWriter(
        new OutputStreamWriter(
            new FileOutputStream("modern.txt"),
            StandardCharsets.UTF_8))) {

    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line);
        writer.newLine();
    }
}

// Detecting BOM (Byte Order Mark)
public BufferedReader openWithBomDetection(Path file) throws IOException {
    InputStream raw = new FileInputStream(file.toFile());
    byte[] bom = raw.readNBytes(3);
    Charset charset;
    int skip = 0;

    if (bom.length >= 3 && bom[0] == (byte)0xEF
            && bom[1] == (byte)0xBB && bom[2] == (byte)0xBF) {
        charset = StandardCharsets.UTF_8;
        skip = 3; // skip BOM bytes
    } else if (bom.length >= 2 && bom[0] == (byte)0xFF && bom[1] == (byte)0xFE) {
        charset = StandardCharsets.UTF_16LE;
        skip = 2;
    } else if (bom.length >= 2 && bom[0] == (byte)0xFE && bom[1] == (byte)0xFF) {
        charset = StandardCharsets.UTF_16BE;
        skip = 2;
    } else {
        charset = StandardCharsets.UTF_8; // assume UTF-8
        skip = 0;
    }

    // Re-open from beginning if no BOM, or skip BOM bytes
    InputStream stream = new FileInputStream(file.toFile());
    stream.skipNBytes(skip);
    return new BufferedReader(new InputStreamReader(stream, charset));
}
```

---

## Q10: What Are the Security Risks of Java Serialization?

**One-line answer:** Java deserialization is a remote code execution (RCE) vector. Specially crafted byte streams can trigger `readObject()` methods in library classes that form "gadget chains" — executing arbitrary code during what appears to be a simple object read.

**The attack mechanism:**
```java
// Normal deserialization — appears harmless
Order order = (Order) objectInputStream.readObject();

// What actually happens during readObject():
// 1. JVM loads class from stream descriptor
// 2. Calls readObject() (or readResolve()) on every object in the graph
// 3. Many library classes (Commons Collections, Spring, etc.) have readObject()
//    methods that perform operations — file writes, JNDI lookups, reflection...
// 4. Attacker crafts byte stream where these operations perform: exec("rm -rf /")
```

**Real CVEs involving Java deserialization:**
- CVE-2015-4852: JBoss/WildFly — RCE via Commons Collections gadget chain
- CVE-2016-3427: Oracle JDK — JMX deserialization
- CVE-2017-9805: Apache Struts 2 — REST plugin
- CVE-2019-14379: Jackson with default typing enabled

**Defenses:**

```java
// Defense 1: Deserialization filter (Java 9+, backported to 8u121)
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filterInfo -> {
    Class<?> cls = filterInfo.serialClass();
    if (cls == null) return ObjectInputFilter.Status.ALLOWED; // array size checks etc.

    // Whitelist only known-safe classes
    Set<String> whitelist = Set.of(
        "com.myapp.Order",
        "com.myapp.OrderItem",
        "java.util.ArrayList",
        "java.math.BigDecimal"
    );
    if (whitelist.contains(cls.getName())) {
        return ObjectInputFilter.Status.ALLOWED;
    }
    System.err.println("BLOCKED deserialization of: " + cls.getName());
    return ObjectInputFilter.Status.REJECTED;
});

// Defense 2: Global JVM filter (Java 9+)
// jdk.serialFilter system property or security.properties:
// jdk.serialFilter=com.myapp.*;java.util.*;java.math.*;!*

// Defense 3: Don't use Java serialization at all for external data
// Use JSON (Jackson), Protocol Buffers, or a safe alternative
// Jackson — no deserialization gadget risk (by default)
ObjectMapper mapper = new ObjectMapper();
mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
// NEVER enable: mapper.enableDefaultTyping() — this re-introduces the risk!

Order order = mapper.readValue(json, Order.class); // safe

// Defense 4: If you must use serialization, use a library with built-in filtering
// Kryo, FST, MessagePack — designed with security in mind
```

**Summary of recommendations:**
1. Never deserialize untrusted data using `ObjectInputStream` without filters
2. Apply an allowlist filter using `ObjectInputFilter`
3. Prefer JSON/Protobuf/Avro for inter-service or external-facing serialization
4. Use Jackson with default typing DISABLED (`DeserializationFeature.USE_BIG_DECIMAL_FOR_FLOATS` does not help — avoid `enableDefaultTyping()`)
5. Monitor for deserialization attacks using security tools (e.g., SerialKiller library)
