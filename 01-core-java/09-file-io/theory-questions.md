# File I/O — Theory Questions

> 18 theory questions on Java I/O streams, NIO.2, file operations, serialization, and performance patterns for Senior Java Backend Engineers.

---

## I/O Streams

**Q1. What is the difference between byte streams and character streams?**

**Byte streams** (`InputStream`/`OutputStream`) — work with raw bytes. Used for binary data: images, audio, serialized objects.

**Character streams** (`Reader`/`Writer`) — work with Unicode characters. Internally convert bytes using a `Charset`. Used for text files.

```java
// Byte streams
InputStream  → FileInputStream, BufferedInputStream, ByteArrayInputStream
OutputStream → FileOutputStream, BufferedOutputStream, ByteArrayOutputStream

// Character streams
Reader → FileReader, BufferedReader, InputStreamReader, StringReader
Writer → FileWriter, BufferedWriter, OutputStreamWriter, StringWriter

// Bridge classes — convert byte to char with explicit encoding
InputStreamReader reader = new InputStreamReader(fileInputStream, StandardCharsets.UTF_8);
OutputStreamWriter writer = new OutputStreamWriter(fileOutputStream, StandardCharsets.UTF_8);

// ALWAYS specify encoding — default varies by OS/locale
FileReader  // Uses Charset.defaultCharset() — AVOID, use InputStreamReader instead
FileWriter  // Same issue — AVOID
```

---

**Q2. How do decorator streams work in Java I/O?**

Java I/O uses the Decorator pattern. Streams can be wrapped to add functionality:

```
FileInputStream          (basic byte reading)
 └── BufferedInputStream  (adds buffering — fewer OS calls)
      └── DataInputStream (adds readInt(), readLong() etc.)

FileOutputStream
 └── BufferedOutputStream (adds buffering)
      └── GZIPOutputStream (adds gzip compression)

FileInputStream → GZIPInputStream → InputStreamReader → BufferedReader
(bytes) → (decompress) → (bytes-to-chars, UTF-8) → (line buffering)
```

```java
// Reading a gzipped UTF-8 text file, line by line
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new GZIPInputStream(
                new FileInputStream("data.gz")),
            StandardCharsets.UTF_8))) {

    reader.lines().forEach(System.out::println);
}
```

**Buffering is critical for performance:** `FileInputStream.read()` calls the OS for each byte. `BufferedInputStream` reads 8KB at a time, reducing system calls 8000-fold.

---

**Q3. What is Java NIO.2 (java.nio.file) and how does it improve on java.io.File?**

`java.io.File` problems:
- Methods like `delete()` return false without throwing — you can't tell WHY
- No symbolic link support
- No file metadata (permissions, creation time)
- No atomic operations
- Weak directory traversal

`java.nio.file` (Java 7+, NIO.2) improvements:
```java
// Path — immutable, handles separators, resolves relative paths
Path path = Path.of("/home/user/data", "report.txt");  // Java 11+
Path path = Paths.get("/home/user/data/report.txt");

// Files utility class — throws IOException with reason, not silent false
Files.delete(path);              // Throws NoSuchFileException, DirectoryNotEmptyException
Files.deleteIfExists(path);      // Returns boolean but still throws on real errors
Files.copy(src, dst, REPLACE_EXISTING, COPY_ATTRIBUTES);
Files.move(src, dst, ATOMIC_MOVE);  // Atomic rename when possible

// Metadata
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
attrs.creationTime();
attrs.lastModifiedTime();
attrs.size();
Files.isSymbolicLink(path);
Files.getOwner(path);

// Simple file reading (small files)
String content = Files.readString(path, StandardCharsets.UTF_8);  // Java 11+
byte[] bytes = Files.readAllBytes(path);
List<String> lines = Files.readAllLines(path, StandardCharsets.UTF_8);

// Simple file writing
Files.writeString(path, content, StandardCharsets.UTF_8);  // Java 11+
Files.write(path, bytes);
Files.write(path, lines, StandardCharsets.UTF_8);
```

---

**Q4. What is the `WatchService` and how is it used?**

`WatchService` monitors file system events (create, modify, delete, overflow) asynchronously:

```java
WatchService watcher = FileSystems.getDefault().newWatchService();

Path dir = Path.of("/config");
dir.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

// Polling loop (typically in a background thread)
while (true) {
    WatchKey key = watcher.take();  // Blocks until event

    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();

        if (kind == StandardWatchEventKinds.OVERFLOW) continue;  // Too many events

        @SuppressWarnings("unchecked")
        WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
        Path changed = dir.resolve(pathEvent.context());

        System.out.println(kind + ": " + changed);
    }

    boolean valid = key.reset();  // Must reset to receive more events
    if (!valid) break;  // Directory deleted — stop watching
}
```

**Use cases:** Configuration file hot-reload, build tools watching source files, deployment artifact monitoring.

---

**Q5. What is Java serialization and what are its pitfalls?**

Java serialization converts object graphs to byte streams (`ObjectOutputStream`) and back (`ObjectInputStream`). Classes must implement `Serializable`.

```java
// Serialization
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("order.ser"))) {
    oos.writeObject(order);
}

// Deserialization
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("order.ser"))) {
    Order order = (Order) ois.readObject();
}
```

**Pitfalls:**
1. **Security vulnerabilities**: deserializing untrusted data can execute arbitrary code (RCE gadget chains — Apache Commons, Spring attack vectors)
2. **Tight coupling**: serialized form becomes part of the public API — hard to change class structure
3. **Performance**: slow, produces large byte streams
4. **`serialVersionUID`**: omitting this causes `InvalidClassException` on any class change

```java
// Always declare serialVersionUID explicitly
private static final long serialVersionUID = 1L;

// Mark fields that shouldn't be serialized
private transient String cachedValue;  // Not serialized
private transient Logger log;          // Never serialize loggers!
```

**Modern alternative:** Use JSON (Jackson), Protobuf, or Avro instead of Java serialization.

---

**Q6. What is `transient` keyword in Java?**

`transient` marks a field to be excluded from serialization.

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private String username;
    private String email;
    private transient String password;   // Never serialize passwords!
    private transient Logger logger;     // Non-serializable field
    private transient Cache<String, Object> localCache;  // Transient state

    // After deserialization, transient fields are null/0/false
    // Use readObject() to restore transient fields if needed
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();  // Deserialize non-transient fields
        this.logger = LoggerFactory.getLogger(User.class);  // Restore transient
    }
}
```

---

## NIO Files and Paths

**Q7. How do you efficiently walk a directory tree in Java?**

```java
// Walk all files in a directory tree
try (Stream<Path> walk = Files.walk(Path.of("/data"))) {
    List<Path> javaFiles = walk
        .filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .collect(Collectors.toList());
}

// Files.find() — with attribute-based filtering (more efficient)
try (Stream<Path> found = Files.find(
        Path.of("/data"),
        Integer.MAX_VALUE,  // Max depth
        (path, attrs) -> attrs.isRegularFile()
            && attrs.size() > 1024 * 1024  // Files > 1MB
            && path.toString().endsWith(".log"))) {
    found.forEach(System.out::println);
}

// List only immediate children (not recursive)
try (Stream<Path> children = Files.list(Path.of("/data"))) {
    children.filter(Files::isDirectory).forEach(System.out::println);
}

// FileVisitor pattern for more control (continue/skip/terminate)
Files.walkFileTree(Path.of("/data"), new SimpleFileVisitor<Path>() {
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        System.out.println(file);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException e) {
        log.warn("Cannot access: {}", file, e);
        return FileVisitResult.CONTINUE;  // Skip inaccessible files
    }

    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
        if (dir.getFileName().toString().startsWith(".")) {
            return FileVisitResult.SKIP_SUBTREE;  // Skip hidden directories
        }
        return FileVisitResult.CONTINUE;
    }
});
```

---

**Q8. What is a `Path` and how is it different from `File`?**

```java
// File — represents a file or directory
File file = new File("/home/user/data.txt");
file.delete();    // Returns boolean (false on failure — no exception)
file.exists();
file.toPath();    // Convert to Path

// Path — more modern, immutable
Path path = Path.of("/home/user/data.txt");

// Path operations — path manipulation only, no I/O
path.getFileName();       // data.txt
path.getParent();         // /home/user
path.getRoot();           // /
path.isAbsolute();        // true
path.toAbsolutePath();    // Resolve relative to CWD

// Path resolution
Path base = Path.of("/home/user");
Path resolved = base.resolve("data/report.txt");  // /home/user/data/report.txt
Path relative = base.relativize(resolved);         // data/report.txt

// Normalization (remove . and ..)
Path.of("/home/user/../user/data.txt").normalize();  // /home/user/data.txt

// Comparing paths
path.startsWith("/home");  // Check prefix
path.endsWith("txt");      // Check suffix

// Convert to URI, File
path.toUri();
path.toFile();
```

---

**Q9. How do you copy files efficiently in Java?**

```java
// Files.copy — simple
Files.copy(src, dst);  // Throws FileAlreadyExistsException if dst exists
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.COPY_ATTRIBUTES);

// Copy from InputStream to Path
try (InputStream is = url.openStream()) {
    Files.copy(is, targetPath, StandardCopyOption.REPLACE_EXISTING);
}

// Copy large files efficiently — NIO channel transfer
try (FileChannel srcChannel = FileChannel.open(src, StandardOpenOption.READ);
     FileChannel dstChannel = FileChannel.open(dst,
         StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    srcChannel.transferTo(0, srcChannel.size(), dstChannel);
    // Uses zero-copy (sendfile syscall) when possible — bypasses JVM heap
}

// Atomic rename (move to same filesystem) — zero content risk
Path temp = Files.createTempFile(dst.getParent(), "temp", ".tmp");
try {
    // Write to temp file
    Files.writeString(temp, content);
    // Atomic replacement
    Files.move(temp, dst, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
} catch (Exception e) {
    Files.deleteIfExists(temp);  // Cleanup on failure
    throw e;
}
```

---

## Performance and Memory

**Q10. What is memory-mapped files and when should you use them?**

Memory-mapped files (`MappedByteBuffer`) map file content directly to virtual memory. The OS handles paging — no explicit read/write calls needed.

```java
try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    MappedByteBuffer buffer = channel.map(
        FileChannel.MapMode.READ_ONLY,
        0,                    // Start offset
        channel.size()        // Length to map
    );

    // Access file contents directly from memory
    while (buffer.hasRemaining()) {
        byte b = buffer.get();  // OS handles I/O transparently
    }

    // Random access without seeking
    buffer.position(1000);
    int value = buffer.getInt();

    // Search within large file efficiently
    for (int i = 0; i < buffer.limit() - 4; i++) {
        if (buffer.getInt(i) == TARGET_MAGIC) {
            // Found at position i
        }
    }
}
```

**When to use:**
- Reading/writing large files (> 100MB) — especially for random access
- Inter-process shared memory
- Performance-critical log parsing, binary file processing

**When NOT to use:**
- Small files — setup overhead not worth it
- Streaming (sequential processing) — regular buffered streams are fine
- When you need to handle file size changes dynamically

---

**Q11. How does `BufferedReader.lines()` work and how does it compare to `Files.lines()`?**

```java
// BufferedReader.lines() — from any Reader
try (BufferedReader reader = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
    reader.lines()            // Stream<String>, lazy
          .filter(line -> !line.isBlank())
          .map(String::trim)
          .forEach(System.out::println);
    // BufferedReader is closed, Stream is lazily evaluated
}

// Files.lines() — convenience wrapper (Java 8+)
// MUST close the stream to close the underlying file handle!
try (Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8)) {
    lines.filter(line -> line.startsWith("ERROR"))
         .collect(Collectors.toList());
}
// WRONG:
// Stream<String> lines = Files.lines(path);  // File handle stays open if stream not closed!

// Files.readAllLines() — reads entire file into memory
// OK for small files, bad for large files
List<String> allLines = Files.readAllLines(path);  // All in memory at once
```

**Key difference:** `Files.lines()` is lazy (reads one line at a time, good for large files). `Files.readAllLines()` loads everything into memory at once (simple but memory-intensive for large files).

---

## Concurrent File Access

**Q12. How do you safely write to files from multiple threads?**

```java
// Option 1: Synchronized writes
public class SafeFileWriter {
    private final Path path;
    private final Object lock = new Object();

    public void append(String line) throws IOException {
        synchronized (lock) {
            Files.writeString(path, line + "\n",
                StandardCharsets.UTF_8,
                StandardOpenOption.APPEND,
                StandardOpenOption.CREATE);
        }
    }
}

// Option 2: Single-writer thread with queue
public class AsyncFileWriter implements Closeable {
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>(10000);
    private final Path path;
    private final ExecutorService writer = Executors.newSingleThreadExecutor();

    public AsyncFileWriter(Path path) throws IOException {
        this.path = path;
        writer.submit(this::writingLoop);
    }

    public void write(String line) {
        queue.offer(line);  // Non-blocking; caller continues immediately
    }

    private void writingLoop() {
        try (BufferedWriter bw = Files.newBufferedWriter(path,
                StandardCharsets.UTF_8, StandardOpenOption.APPEND, StandardOpenOption.CREATE)) {
            while (!Thread.currentThread().isInterrupted()) {
                String line = queue.poll(1, TimeUnit.SECONDS);
                if (line != null) {
                    bw.write(line);
                    bw.newLine();
                    bw.flush();
                }
            }
        } catch (Exception e) {
            log.error("File writer error", e);
        }
    }

    @Override
    public void close() {
        writer.shutdownNow();
    }
}

// Option 3: File locking for cross-process coordination
try (FileChannel channel = FileChannel.open(path,
        StandardOpenOption.WRITE, StandardOpenOption.APPEND, StandardOpenOption.CREATE);
     FileLock lock = channel.lock()) {  // Exclusive lock
    // Write while holding lock
    channel.write(ByteBuffer.wrap(data));
}
// Lock released when FileLock is closed
```

---

## Modern File API

**Q13. How do you create temporary files and directories?**

```java
// Create temp file — deleted when JVM exits (if deleteOnExit is set)
Path tempFile = Files.createTempFile("prefix-", ".suffix");
// Creates: /tmp/prefix-1234567890.suffix
tempFile.toFile().deleteOnExit();

// Create temp file in specific directory
Path tempInDir = Files.createTempFile(Path.of("/data/temp"), "upload-", ".json");

// Create temp directory
Path tempDir = Files.createTempDirectory("processing-");

// Proper cleanup with try-finally
Path tempFile = Files.createTempFile("report-", ".csv");
try {
    // Use tempFile for processing
    Files.writeString(tempFile, csvData);
    processAndUpload(tempFile);
} finally {
    Files.deleteIfExists(tempFile);  // Always clean up
}
```

---

**Q14. What are `StandardOpenOption` values and when do you use each?**

```java
// Open options for Files.newOutputStream(), FileChannel.open(), etc.
Files.newOutputStream(path,
    StandardOpenOption.CREATE,       // Create if not exists (no-op if exists)
    StandardOpenOption.CREATE_NEW,   // Create; fail if already exists
    StandardOpenOption.APPEND,       // Seek to end before each write
    StandardOpenOption.TRUNCATE_EXISTING, // Truncate file to 0 on open
    StandardOpenOption.WRITE,        // Open for writing
    StandardOpenOption.READ,         // Open for reading
    StandardOpenOption.SYNC,         // Sync content+metadata after each write
    StandardOpenOption.DSYNC,        // Sync content only (faster, still durable)
    StandardOpenOption.DELETE_ON_CLOSE  // Delete when stream/channel closed
);

// Common combinations:
// Overwrite file:   WRITE, CREATE, TRUNCATE_EXISTING (default for write)
// Append to file:   WRITE, CREATE, APPEND
// New file only:    WRITE, CREATE_NEW
// Atomic new file:  WRITE, CREATE_NEW (fails if file exists = safe)
// Temp file:        WRITE, CREATE_NEW, DELETE_ON_CLOSE
```

---

**Q15. How do you handle large file processing without loading everything into memory?**

```java
// Process CSV with 10M rows — streaming approach
@Service
public class LargeFileProcessor {

    public ProcessingResult processLargeCSV(Path filePath) throws IOException {
        long lineCount = 0;
        long errorCount = 0;

        // Stream line by line — constant memory usage regardless of file size
        try (Stream<String> lines = Files.lines(filePath, StandardCharsets.UTF_8)) {
            Iterator<String> iter = lines.iterator();

            // Skip header
            if (iter.hasNext()) iter.next();

            while (iter.hasNext()) {
                String line = iter.next();
                try {
                    processLine(line);
                    lineCount++;
                } catch (Exception e) {
                    errorCount++;
                    log.warn("Error on line {}: {}", lineCount + errorCount, e.getMessage());
                }
            }
        }

        return new ProcessingResult(lineCount, errorCount);
    }

    // Better: process in batches for bulk inserts
    public void processBatched(Path filePath, int batchSize) throws IOException {
        try (Stream<String> lines = Files.lines(filePath, StandardCharsets.UTF_8)) {
            List<String> batch = new ArrayList<>(batchSize);

            lines.skip(1)  // Skip header
                 .forEach(line -> {
                     batch.add(line);
                     if (batch.size() >= batchSize) {
                         processBatch(batch);
                         batch.clear();
                     }
                 });

            if (!batch.isEmpty()) processBatch(batch);  // Remaining items
        }
    }
}
```

---

## Serialization Alternatives

**Q16. What is Externalizable and how does it differ from Serializable?**

`Externalizable` gives complete control over serialization format by requiring you to implement `writeExternal()` and `readExternal()`:

```java
public class Order implements Externalizable {

    private Long id;
    private String status;
    private BigDecimal total;

    // Required: public no-arg constructor for deserialization
    public Order() {}

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeLong(id);
        out.writeUTF(status);
        out.writeUTF(total.toPlainString());  // Custom serialization of BigDecimal
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.id = in.readLong();
        this.status = in.readUTF();
        this.total = new BigDecimal(in.readUTF());
    }
}
```

| Feature | Serializable | Externalizable |
|---------|-------------|----------------|
| Control | JVM controls format | You control format |
| Performance | Slower (reflection-based) | Faster (manual) |
| No-arg constructor | Optional | Required |
| Effort | Minimal (just marker) | Must implement methods |
| Use when | Quick serialization | Performance-critical, compact format |

**In practice:** Use JSON (Jackson), Protobuf, or MessagePack — not Java serialization.

---

**Q17. How do you implement object serialization to JSON with Jackson?**

```java
// Jackson configuration for a Spring Boot application
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())    // LocalDate, Instant, etc.
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .enable(MapperFeature.DEFAULT_VIEW_INCLUSION)
            .build();
    }
}

// Usage
@Service
public class JsonFileService {

    @Autowired private ObjectMapper objectMapper;

    public void saveOrder(Order order, Path file) throws IOException {
        // Serialize to file
        objectMapper.writerWithDefaultPrettyPrinter().writeValue(file.toFile(), order);

        // Or to String
        String json = objectMapper.writeValueAsString(order);

        // Or to byte array (for compression)
        byte[] bytes = objectMapper.writeValueAsBytes(order);
    }

    public Order loadOrder(Path file) throws IOException {
        return objectMapper.readValue(file.toFile(), Order.class);
    }

    // For collections with generics
    public List<Order> loadOrders(Path file) throws IOException {
        return objectMapper.readValue(file.toFile(),
            objectMapper.getTypeFactory().constructCollectionType(List.class, Order.class));
    }
}
```

---

**Q18. What are the key best practices for file I/O in production Java applications?**

**1. Always specify character encoding:**
```java
Files.newBufferedReader(path, StandardCharsets.UTF_8);   // Explicit
new FileReader(path);                                     // Never (uses default)
```

**2. Always use try-with-resources:**
```java
try (BufferedReader r = Files.newBufferedReader(path)) { ... }  // Auto-closes
```

**3. Buffer your I/O:**
```java
new BufferedInputStream(new FileInputStream(path))  // Not raw FileInputStream
```

**4. Handle large files with streaming:**
```java
Files.lines(path)   // Stream<String> — lazy
Files.readAllLines  // List<String> — all in memory — only for small files
```

**5. Use atomic writes for critical data:**
```java
// Write to temp, then atomically rename
Path tmp = Files.createTempFile(target.getParent(), ".tmp", null);
Files.writeString(tmp, content);
Files.move(tmp, target, ATOMIC_MOVE, REPLACE_EXISTING);
```

**6. Handle file system race conditions:**
```java
// Check-then-act is a race condition
if (!Files.exists(path)) { Files.createFile(path); }  // TOCTOU race!
// Better:
try { Files.createFile(path); }
catch (FileAlreadyExistsException e) { /* Handle */ }
```

**7. Clean up temporary files:**
```java
try { ... }
finally { Files.deleteIfExists(tempFile); }
```
