# Chapter 09: File I/O — Follow-Up Traps

Tricky interview questions with explanations of the correct behavior.

---

## Trap 1: What Happens If You Don't Close a FileInputStream?

**Interviewer:** "Your code reads a file without closing the stream. What happens?"

**The Trap:** Candidates say "nothing, GC will clean it up."

**Answer:** The OS file descriptor is leaked. Each open file uses a file descriptor (a limited OS resource — typically 1024 per process on Linux). If you leak enough, you hit the limit and get `java.io.IOException: Too many open files`. GC may eventually finalize the stream and release it, but GC timing is non-deterministic — you cannot rely on it.

```java
// DANGEROUS — file descriptor leaked if exception occurs before close()
FileInputStream fis = new FileInputStream("data.txt");
byte[] data = fis.readAllBytes();
fis.close(); // what if readAllBytes() threw? close() never called!

// CORRECT — try-with-resources ensures close() is ALWAYS called
try (FileInputStream fis = new FileInputStream("data.txt")) {
    byte[] data = fis.readAllBytes();
} // fis.close() called here, even if exception thrown

// Nesting with try-with-resources
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("file.txt"),
            StandardCharsets.UTF_8))) {
    // br.close() closes BufferedReader, InputStreamReader, AND FileInputStream
    String line = br.readLine();
}

// Demonstrating file descriptor exhaustion
List<FileInputStream> leaked = new ArrayList<>();
try {
    for (int i = 0; i < 10_000; i++) {
        leaked.add(new FileInputStream("data.txt")); // leaking FDs
    }
} catch (IOException e) {
    System.out.println("Too many open files: " + e.getMessage());
    // Linux: "Too many open files" (EMFILE)
}
```

---

## Trap 2: FileOutputStream Truncates vs Appends by Default

**Interviewer:** "You open a FileOutputStream to write to an existing log file. What happens to the existing content?"

**The Trap:** Many candidates assume it appends. It does not.

**Answer:** By default, `new FileOutputStream("file.txt")` **truncates** (overwrites) the file. The existing content is destroyed. To append, you must pass `true` as the second argument.

```java
// First run — writes "Hello"
try (FileOutputStream fos = new FileOutputStream("log.txt")) {
    fos.write("Hello".getBytes());
}

// Second run — DEFAULT: truncates! "World" replaces "Hello"
try (FileOutputStream fos = new FileOutputStream("log.txt")) {
    fos.write("World".getBytes());
}
// log.txt now contains: "World" (not "HelloWorld")

// APPEND mode — second argument true
try (FileOutputStream fos = new FileOutputStream("log.txt", true)) {
    fos.write(" World".getBytes());
}
// log.txt now contains: "Hello World"

// Modern equivalent with StandardOpenOption
try (var out = Files.newOutputStream(Path.of("log.txt"),
        StandardOpenOption.APPEND,
        StandardOpenOption.CREATE)) {
    out.write("log entry\n".getBytes(StandardCharsets.UTF_8));
}

// TRAP: FileWriter has the same behavior
new FileWriter("file.txt")        // truncates
new FileWriter("file.txt", true)  // appends
```

---

## Trap 3: Serialization and serialVersionUID — What Happens Without It?

**Interviewer:** "What happens if you don't declare `serialVersionUID` in a Serializable class?"

**The Trap:** Candidates think it "just works" or that serialization will fail immediately.

**Answer:** Java auto-generates a `serialVersionUID` based on the class structure (field names, types, method signatures). If you change the class in any way (add a field, rename a method), the auto-generated UID changes. Attempting to deserialize old bytes with the new class throws `InvalidClassException`. This is a common production surprise during deployments.

```java
// Dangerous — no explicit serialVersionUID
public class UserProfile implements Serializable {
    // Java generates: serialVersionUID = -8948259234091234567L (example)
    private String name;
    private int age;
}

// After deployment, you add a field:
public class UserProfile implements Serializable {
    // Java now generates: serialVersionUID = 3421890234567890123L (different!)
    private String name;
    private int age;
    private String email; // added field
}

// Attempting to deserialize sessions stored before the deployment:
// java.io.InvalidClassException: UserProfile; local class incompatible:
//   stream classdesc serialVersionUID = -8948259234091234567,
//   local class serialVersionUID = 3421890234567890123

// CORRECT — always declare explicitly
public class UserProfile implements Serializable {
    private static final long serialVersionUID = 1L; // you control the version

    private String name;
    private int age;
    private String email; // adding this does NOT break old serialized data
    // email will be null when deserializing old data — acceptable
}
```

---

## Trap 4: transient Fields in Serialization — Are They Always Null After Deserialization?

**Interviewer:** "After deserializing an object, what value will a `transient` field have?"

**The Trap:** Candidates say "always null" — but that is only true for reference types.

**Answer:** After deserialization, `transient` fields get their **default values**: `null` for references, `0` for `int`/`long`, `false` for `boolean`, `0.0` for `double`. To restore a transient field to a computed value, implement `readObject()`.

```java
public class Connection implements Serializable {
    private static final long serialVersionUID = 1L;

    private String host;
    private int port;
    private transient Socket socket;    // null after deserialization
    private transient int retryCount;   // 0 after deserialization (not null!)
    private transient boolean connected; // false after deserialization

    // Restore transient fields after deserialization
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject(); // deserialize non-transient fields
        // Re-initialize transient fields
        this.socket = new Socket(host, port); // reconnect
        this.retryCount = 3;                  // restore default
        this.connected = (socket != null);
    }

    // Control serialization of specific fields
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject(); // serialize non-transient fields
        // Could write extra data here
    }
}
```

---

## Trap 5: Path.of() vs Paths.get() — Are They the Same?

**Interviewer:** "What is the difference between `Path.of("/tmp/file")` and `Paths.get("/tmp/file")`?"

**The Trap:** Candidates think there is a behavioral difference.

**Answer:** They are functionally identical — `Path.of()` (added in Java 11) is a static factory method on `Path` itself, and internally calls `FileSystems.getDefault().getPath(...)`. `Paths.get()` (Java 7+) does exactly the same thing. `Path.of()` was added for API consistency — the same pattern as `List.of()`, `Map.of()`, etc.

```java
// These produce identical results
Path p1 = Paths.get("/tmp/file.txt");              // Java 7+
Path p2 = Path.of("/tmp/file.txt");                // Java 11+ (preferred)
Path p3 = FileSystems.getDefault().getPath("/tmp/file.txt"); // verbose, same thing

System.out.println(p1.equals(p2)); // true
System.out.println(p1.equals(p3)); // true

// Both support varargs form
Path p4 = Paths.get("/tmp", "subdir", "file.txt");
Path p5 = Path.of("/tmp", "subdir", "file.txt");
// both → /tmp/subdir/file.txt

// Prefer Path.of() in Java 11+ for consistency with modern static factory conventions
// Use Paths.get() only for Java 7/8/9/10 compatibility
```

---

## Trap 6: Files.readAllBytes() for Large Files — Memory Risk

**Interviewer:** "Is `Files.readAllBytes()` safe for reading any file?"

**The Trap:** Candidates say yes because it is in the standard library.

**Answer:** No. `Files.readAllBytes()` loads the entire file into a single `byte[]` on the Java heap. For large files (hundreds of MB or GB), this causes `OutOfMemoryError` or severe GC pressure. For large files, use streaming, buffered reads, or memory-mapped files.

```java
// DANGEROUS for large files — loads everything into heap
byte[] all = Files.readAllBytes(Path.of("large-1GB-file.dat")); // OOM risk!
String text = Files.readString(Path.of("large-file.txt")); // same risk

// SAFE for large files — streaming line by line
try (Stream<String> lines = Files.lines(Path.of("large-file.txt"),
        StandardCharsets.UTF_8)) {
    lines.forEach(line -> processLine(line));
    // Only one line in memory at a time
}

// SAFE — buffered chunk reading
try (BufferedInputStream bis = new BufferedInputStream(
        new FileInputStream("large-file.dat"), 64 * 1024)) {
    byte[] chunk = new byte[64 * 1024];
    int bytesRead;
    while ((bytesRead = bis.read(chunk)) != -1) {
        process(chunk, bytesRead);
    }
}

// SAFE and fast — memory-mapped for random access
try (FileChannel channel = FileChannel.open(Path.of("large-file.dat"))) {
    // Map only a portion at a time if file > 2GB
    long chunkSize = 512 * 1024 * 1024L; // 512MB
    long position = 0;
    while (position < channel.size()) {
        long size = Math.min(chunkSize, channel.size() - position);
        MappedByteBuffer buf = channel.map(FileChannel.MapMode.READ_ONLY, position, size);
        // process buf...
        position += size;
    }
}
```

---

## Trap 7: Encoding Trap — Reading File Without Specifying Charset

**Interviewer:** "Your code uses `new FileReader("file.txt")` and works fine on your Mac but garbles text on the production Linux server. Why?"

**The Trap:** Candidates blame the file or the Linux system.

**Answer:** `FileReader` uses `Charset.defaultCharset()`, which is determined by the OS locale/environment. Mac defaults to UTF-8, but production Linux servers configured for legacy systems may use ISO-8859-1 or Cp1252. Always specify charset explicitly.

```java
// BROKEN — uses Charset.defaultCharset(), varies by environment
FileReader fr = new FileReader("file.txt");        // Mac: UTF-8, Server: ISO-8859-1?
BufferedReader br = new BufferedReader(fr);

// CORRECT — always explicit
BufferedReader safeBr = new BufferedReader(
    new InputStreamReader(
        new FileInputStream("file.txt"),
        StandardCharsets.UTF_8           // explicit, same everywhere
    )
);

// Modern API (Java 11+) — charset is always specified
String content = Files.readString(Path.of("file.txt"), StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("file.txt"), StandardCharsets.UTF_8);

// Writing — same trap
PrintWriter pw = new PrintWriter("out.txt");              // uses default charset
PrintWriter safePw = new PrintWriter(
    new OutputStreamWriter(
        new FileOutputStream("out.txt"),
        StandardCharsets.UTF_8
    )
);
Files.writeString(Path.of("out.txt"), content, StandardCharsets.UTF_8); // best
```

---

## Trap 8: ObjectInputStream and ClassNotFoundException

**Interviewer:** "You serialize an object from service A and send it to service B. What can go wrong when deserializing?"

**The Trap:** Candidates only think about serialVersionUID mismatch.

**Answer:** If service B does not have the class on its classpath, deserialization throws `ClassNotFoundException` — even if the bytes are valid. This is a common issue in microservices that share serialized objects over queues or caches.

```java
// Service A serializes com.company.events.OrderCreatedEvent
// Service B tries to deserialize — does it have that class?

try (ObjectInputStream ois = new ObjectInputStream(
        new ByteArrayInputStream(bytes))) {
    Object obj = ois.readObject(); // THROWS ClassNotFoundException
    // if OrderCreatedEvent.class is not on Service B's classpath
} catch (ClassNotFoundException e) {
    // "com.company.events.OrderCreatedEvent"
    System.err.println("Missing class: " + e.getMessage());
} catch (InvalidClassException e) {
    System.err.println("Incompatible version: " + e.getMessage());
}

// Better solution: don't use Java serialization across service boundaries
// Use JSON (Jackson), Protocol Buffers, Avro, or XML instead
// These are language-neutral and do not require shared class files

// If you must use Java serialization across JVMs, ensure:
// 1. Both JVMs have the same version of the class on classpath
// 2. serialVersionUID is explicitly declared and matches
// 3. The class is in a shared library (jar) used by both services
```

---

## Trap 9: WatchService and Missed Events

**Interviewer:** "Can `WatchService` miss events? Under what circumstances?"

**The Trap:** Candidates assume it captures every single event.

**Answer:** Yes. If events pile up faster than your consumer processes them, the OS event queue overflows and `WatchService` returns `StandardWatchEventKinds.OVERFLOW` instead of a specific event. This means some file changes are lost. Also, event delivery is OS-specific — on macOS, `WatchService` uses polling internally (slower, higher latency than on Linux's inotify).

```java
WatchService watcher = FileSystems.getDefault().newWatchService();
Path dir = Path.of("/var/log/app");
dir.register(watcher, StandardWatchEventKinds.ENTRY_MODIFY);

while (true) {
    WatchKey key = watcher.take();

    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();

        // ALWAYS handle OVERFLOW — events may have been lost
        if (kind == StandardWatchEventKinds.OVERFLOW) {
            System.out.println("WARNING: File events lost due to overflow!");
            // Do a full directory scan to catch up
            rescanDirectory(dir);
            continue;
        }

        Path filename = (Path) event.context();
        System.out.println(kind + ": " + filename);
    }

    // CRITICAL: must reset, or no more events are delivered
    boolean valid = key.reset();
    if (!valid) {
        System.out.println("Directory no longer accessible");
        break;
    }
}

// Common mistake: forgetting key.reset() — events stop being delivered after first batch
```

---

## Trap 10: RandomAccessFile vs FileChannel

**Interviewer:** "What is the difference between `RandomAccessFile` and `FileChannel` for random access?"

**The Trap:** Candidates say "they do the same thing."

**Answer:** Both allow seeking to arbitrary positions in a file, but `FileChannel` (NIO) is far more capable: it supports memory mapping, file locking, direct transfer between channels (`transferTo`/`transferFrom`), and works with NIO Selectors. `RandomAccessFile` is older, simpler, and lacks these features. For new code, always prefer `FileChannel`.

```java
// RandomAccessFile — simple but limited
RandomAccessFile raf = new RandomAccessFile("data.bin", "rw");
raf.seek(1024);                     // jump to byte 1024
int value = raf.readInt();          // read 4 bytes as int
raf.seek(2048);
raf.writeUTF("hello");             // write with length prefix
raf.close();

// FileChannel — full-featured NIO
try (FileChannel fc = FileChannel.open(Path.of("data.bin"),
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {

    fc.position(1024);              // seek to byte 1024
    ByteBuffer buf = ByteBuffer.allocate(4);
    fc.read(buf);                   // read 4 bytes
    buf.flip();
    int value = buf.getInt();

    // Zero-copy transfer between channels (no heap copy)
    try (FileChannel dest = FileChannel.open(Path.of("backup.bin"),
            StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
        fc.transferTo(0, fc.size(), dest); // kernel-level copy
    }

    // File locking — prevents concurrent access
    FileLock lock = fc.lock(0, Long.MAX_VALUE, false); // exclusive lock
    try {
        // safe to modify file exclusively
    } finally {
        lock.release();
    }
}
```

---

## Trap 11: Files.copy() — Does It Preserve Attributes?

**Interviewer:** "You use `Files.copy(src, dest)` to back up a file. The copy exists but its modification date is today, not the original's date. Why?"

**The Trap:** Candidates assume `Files.copy()` is a true bit-for-bit clone including metadata.

**Answer:** `Files.copy()` without options copies content only. To copy file attributes (timestamps, permissions), you must pass `StandardCopyOption.COPY_ATTRIBUTES`.

```java
Path src  = Path.of("original.txt");
Path dest = Path.of("backup.txt");

// Content only — dest gets current timestamp
Files.copy(src, dest);

// Content + attributes — dest gets same timestamps, permissions
Files.copy(src, dest,
    StandardCopyOption.COPY_ATTRIBUTES,  // preserve modification time etc.
    StandardCopyOption.REPLACE_EXISTING  // overwrite if dest exists
);

// Atomic move — rename within same filesystem (instant, no data copy)
Files.move(src, dest, StandardCopyOption.ATOMIC_MOVE);

// The difference: ATOMIC_MOVE either fully succeeds or fully fails
// Regular copy/move can be interrupted and leave partial file
```

---

## Trap 12: Closing a Stream from Files.lines() — Resource Leak

**Interviewer:** "Is this code safe: `Files.lines(path).filter(...).findFirst()`?"

**The Trap:** Candidates focus on the filter logic and miss the resource leak.

**Answer:** No. `Files.lines()` returns a `Stream<String>` that holds an open file handle. If you do not close it, the file descriptor leaks. `Stream` implements `AutoCloseable`, so you must use try-with-resources.

```java
// RESOURCE LEAK — stream (and underlying file) never closed
Optional<String> result = Files.lines(Path.of("data.txt"))
    .filter(line -> line.contains("ERROR"))
    .findFirst();
// The Stream is not closed — file descriptor leaked!

// CORRECT — try-with-resources closes the stream
Optional<String> safeResult;
try (Stream<String> lines = Files.lines(Path.of("data.txt"),
        StandardCharsets.UTF_8)) {
    safeResult = lines
        .filter(line -> line.contains("ERROR"))
        .findFirst();
} // stream.close() called — file handle released

// Alternatively, read all at once (for small files)
List<String> errors = Files.readAllLines(Path.of("data.txt"), StandardCharsets.UTF_8)
    .stream()
    .filter(line -> line.contains("ERROR"))
    .collect(Collectors.toList());
// readAllLines() returns a plain List — no open resource
```
