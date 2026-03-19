# Chapter 09: File I/O — Analogy Explanations

---

## InputStream vs OutputStream

Think of a garden hose. An `InputStream` is like a hose connected to a tap — water (data) flows INTO you. You hold the end and drink (read bytes). You cannot push water back into the tap.

An `OutputStream` is like a hose connected to a bucket — you pour water (write bytes) into the bucket. You are the source, the bucket is the destination.

The direction is always relative to your program: **Input = into your program**, **Output = out of your program**.

```java
// InputStream — data flows INTO your program (you are reading)
InputStream in = new FileInputStream("data.bin");
int byteValue = in.read(); // reads ONE byte, returns 0-255 (or -1 for EOF)
byte[] buffer = new byte[1024];
int bytesRead = in.read(buffer); // reads up to 1024 bytes into buffer

// OutputStream — data flows OUT of your program (you are writing)
OutputStream out = new FileOutputStream("output.bin");
out.write(65);            // writes byte with value 65 (ASCII 'A')
out.write(buffer, 0, bytesRead); // writes bytesRead bytes from buffer
```

---

## Buffered I/O (BufferedReader / BufferedWriter)

Imagine going to a warehouse to pick up boxes. Without buffering, every time you need one box, you make a full trip to the warehouse and back. 1000 boxes = 1000 trips. Exhausting.

With buffering, you bring a cart. One trip to the warehouse, you load the cart with 100 boxes, drive back. Next 100 requests come from the cart in the garage — no warehouse trip needed. You only go back when the cart is empty.

`BufferedReader` does exactly this — instead of making a system call for every single byte, it reads a large chunk (8 KB by default) from the file into memory, and subsequent `read()` calls are served from that in-memory buffer — much faster.

```java
// Without buffering — system call for every character (slow)
FileReader raw = new FileReader("file.txt");
int c;
while ((c = raw.read()) != -1) {
    // Each raw.read() may trigger a system call
}

// With buffering — reads 8192 characters at once, then serves from memory
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
String line;
while ((line = br.readLine()) != null) {
    // readLine() reads from the in-memory buffer — fast
    System.out.println(line);
}

// BufferedWriter — accumulates writes in memory, flushes in chunks
BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"));
bw.write("Hello");
bw.write(" World");
bw.flush(); // now data actually goes to disk
```

---

## File vs Path/Files (Old vs New API)

Think of the difference between a paper filing system from the 1980s and a modern computer filing system.

The old `java.io.File` class (Java 1.0) is like a filing cabinet with sticky labels. It works, but it is hard to navigate complex paths, error handling is poor (returns `false` instead of telling you WHY it failed), and it has no support for symbolic links, file attributes, or watching for changes.

The new `java.nio.file.Path` and `java.nio.file.Files` API (Java 7) is like a modern document management system. It gives you proper exceptions, metadata access, symlink awareness, bulk operations, and a clean streaming API.

```java
// Old File API — error handling is guesswork
File file = new File("/tmp/data.txt");
boolean deleted = file.delete(); // returns false — but WHY? No exception.
boolean created = file.mkdir();  // same problem

// New Path/Files API — proper exceptions, rich information
Path path = Path.of("/tmp/data.txt");       // Java 11+
// Path path = Paths.get("/tmp/data.txt");  // Java 7-10

try {
    Files.delete(path);             // throws NoSuchFileException, IOException etc.
    Files.createDirectory(path);    // throws FileAlreadyExistsException
} catch (NoSuchFileException e) {
    System.out.println("File not found: " + e.getFile());
}

// Rich metadata
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
System.out.println(attrs.creationTime());
System.out.println(attrs.size());
System.out.println(attrs.isSymbolicLink());
```

---

## Serialization

Think of packing for a move. You have furniture in your house (live Java objects in memory). Serialization is packing everything into labeled boxes (bytes). Each box has the item type, its contents, and any sub-items. When you arrive at the new house (another JVM, or restarting the app), you unpack the boxes (deserialization) and reassemble everything exactly as it was.

The key rule: every item you pack must be "packable" (`Serializable`). If a piece of furniture contains a part that is not packable (a `transient` field, or a non-Serializable object), you either skip it or throw an error.

```java
// Marking a class as "packable"
public class UserSession implements Serializable {
    private static final long serialVersionUID = 1L; // version label on the box

    private String username;
    private List<String> roles;
    private transient String password;     // SKIP this — don't pack sensitive data
    private transient Connection dbConn;   // SKIP this — can't pack a live connection

    // After deserialization, password and dbConn will be null
}

// Packing (serialization)
UserSession session = new UserSession("alice", List.of("ADMIN"), "secret123");
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("session.ser"))) {
    oos.writeObject(session);
}

// Unpacking (deserialization)
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("session.ser"))) {
    UserSession restored = (UserSession) ois.readObject();
    System.out.println(restored.getUsername()); // alice
    System.out.println(restored.getPassword()); // null — transient was skipped
}
```

---

## NIO vs IO (Blocking vs Non-Blocking)

Think of a restaurant and its waiting system.

The old IO (blocking) is like a restaurant where a waiter is assigned exclusively to one table. The waiter stands by your table and waits while you read the menu, waits while you eat, waits for you to pay. That waiter cannot serve anyone else. For 1000 tables, you need 1000 waiters.

The new NIO (non-blocking) is like a restaurant with a small team and a buzzer system. One waiter checks on all tables periodically. If table 7 needs something, the buzzer rings — the waiter goes. Otherwise, the waiter handles other tables. Three waiters can serve 1000 tables efficiently.

In Java terms: IO blocks the thread until data is ready. NIO uses a `Selector` that monitors many channels and only notifies you when data is actually available.

```java
// Blocking IO — thread blocks on read() until data arrives
InputStream in = socket.getInputStream();
byte[] buffer = new byte[1024];
int n = in.read(buffer); // BLOCKS HERE until data arrives or connection closes
// Thread is doing nothing useful while waiting

// Non-blocking NIO — thread can do other things
Selector selector = Selector.open();
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);  // ← key: non-blocking mode
channel.register(selector, SelectionKey.OP_READ);

// One thread can monitor many channels
while (true) {
    selector.select(); // blocks only until ANY channel has data
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) {
            SocketChannel ch = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(1024);
            ch.read(buf); // data is ready — this won't block
        }
    }
}
```

---

## FileChannel and Memory-Mapped Files

Think of reading a massive book. The normal way (stream I/O) is like sitting at a library desk and asking the librarian to bring you one page at a time. The librarian walks to the shelf, brings page 1, you read it, she takes it back, brings page 2...

Memory-mapped files are like the library installing a glass window directly into the bookshelf wall. You can see the entire book through the window. You "read" by just looking at any page through the window — no copying, no librarian trips. The OS handles bringing pages into view as you look at them (demand paging).

```java
// Normal file read — copies bytes from OS kernel buffer to Java heap
byte[] data = Files.readAllBytes(Path.of("bigfile.dat")); // copies everything to heap

// Memory-mapped — the file IS the memory (zero-copy reads)
try (FileChannel channel = FileChannel.open(Path.of("bigfile.dat"),
        StandardOpenOption.READ)) {

    MappedByteBuffer mapped = channel.map(
        FileChannel.MapMode.READ_ONLY,
        0,                     // start position
        channel.size()         // how much to map
    );

    // Access bytes directly — OS pages them in on demand
    byte firstByte = mapped.get(0);       // access byte at position 0
    byte byteAt1M  = mapped.get(1_000_000); // access byte 1 million — no seek needed
}
// Extremely fast for random access, large files, and shared-memory between processes
```

---

## WatchService (File System Events)

Imagine a night security guard patrolling a building. Without `WatchService`, you would have to keep checking every door yourself (polling): "Is this door open? Is that window broken? Is the filing cabinet disturbed?" — checking every few seconds, wasting energy.

`WatchService` is like installing motion sensors on every door and window. Instead of you checking, the sensors alert you when something happens: "Room 3 door opened," "Filing cabinet in Room 7 was accessed." You just sit at the security desk and respond to alerts.

```java
// Without WatchService — polling (wasteful)
while (true) {
    long lastModified = new File("config.properties").lastModified();
    if (lastModified != previousModified) {
        reloadConfig();
        previousModified = lastModified;
    }
    Thread.sleep(5000); // wasteful polling every 5 seconds
}

// With WatchService — event-driven (efficient)
WatchService watcher = FileSystems.getDefault().newWatchService();
Path dir = Path.of("/etc/myapp");

dir.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

// Block waiting for events — no CPU waste
WatchKey key = watcher.take(); // blocks until an event occurs

for (WatchEvent<?> event : key.pollEvents()) {
    WatchEvent.Kind<?> kind = event.kind();
    Path filename = (Path) event.context();
    System.out.println(kind.name() + ": " + filename);
}
key.reset(); // must reset to receive future events
```

---

## Charset/Encoding

Think of a document written in Morse code. The dots and dashes (bytes) are the raw data. But without knowing whether it is Standard Morse, Continental Morse, or some other variant (charset), the same dots and dashes can mean completely different letters.

In computing, the same byte sequence `0xC3 0xA9` is `é` in UTF-8, but gibberish in ISO-8859-1, and something entirely different in Windows-1252. If you do not specify the charset, Java uses the platform default — which varies by OS and locale, causing bugs on different machines.

```java
// Platform default — DANGEROUS, varies by OS
FileReader fr = new FileReader("file.txt"); // uses Charset.defaultCharset()
// Works on your Mac (UTF-8), breaks on Windows (Cp1252)

// Explicit charset — SAFE and portable
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("file.txt"),
            StandardCharsets.UTF_8))) {        // always explicit
    String line = reader.readLine();
}

// Modern API — charset is a first-class citizen
String content = Files.readString(Path.of("file.txt"), StandardCharsets.UTF_8);
Files.writeString(Path.of("output.txt"), content, StandardCharsets.UTF_8);

// Detecting what charset a file uses (heuristic, not perfect)
// Libraries like ICU4J or juniversalchardet can detect encoding from byte patterns
byte[] bytes = Files.readAllBytes(path);
// Check for BOM (Byte Order Mark)
if (bytes.length >= 3 && bytes[0] == (byte)0xEF
        && bytes[1] == (byte)0xBB && bytes[2] == (byte)0xBF) {
    System.out.println("UTF-8 with BOM");
}
```
