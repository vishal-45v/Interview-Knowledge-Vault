# Chapter 09: File I/O — Diagram Explanations

ASCII diagrams for Java I/O and NIO architecture.

---

## 1. Java I/O Stream Hierarchy

```
  BYTE STREAMS (raw bytes)
  ════════════════════════════════════════════════════════════════

  InputStream (abstract)
  ├── FileInputStream            — reads bytes from a file
  ├── ByteArrayInputStream       — reads bytes from a byte[]
  ├── PipedInputStream           — reads from a PipedOutputStream
  ├── FilterInputStream (abstract)
  │   ├── BufferedInputStream    — adds in-memory buffer
  │   ├── DataInputStream        — reads primitives (readInt, readDouble...)
  │   └── CheckedInputStream     — tracks CRC/checksum
  └── ObjectInputStream          — deserializes Java objects

  OutputStream (abstract)
  ├── FileOutputStream           — writes bytes to a file
  ├── ByteArrayOutputStream      — writes bytes to an in-memory byte[]
  ├── PipedOutputStream          — writes to a PipedInputStream
  ├── FilterOutputStream (abstract)
  │   ├── BufferedOutputStream   — adds write buffer, reduces system calls
  │   ├── DataOutputStream       — writes primitives (writeInt, writeLong...)
  │   └── PrintStream            — System.out is a PrintStream
  └── ObjectOutputStream         — serializes Java objects

  ────────────────────────────────────────────────────────────────

  CHARACTER STREAMS (text, charset-aware)
  ════════════════════════════════════════════════════════════════

  Reader (abstract)
  ├── InputStreamReader          — bridge: InputStream → Reader (with charset)
  │   └── FileReader             — reads chars from file (platform default charset!)
  ├── StringReader               — reads chars from a String
  ├── BufferedReader             — adds buffer + readLine()
  └── CharArrayReader            — reads from char[]

  Writer (abstract)
  ├── OutputStreamWriter         — bridge: OutputStream → Writer (with charset)
  │   └── FileWriter             — writes chars to file (platform default charset!)
  ├── StringWriter               — writes to an in-memory StringBuffer
  ├── BufferedWriter             — adds write buffer + newLine()
  └── PrintWriter                — formatted text output (printf, println)

  ────────────────────────────────────────────────────────────────

  WRAPPING PATTERN (Decorator):

  new BufferedReader(
      new InputStreamReader(
          new FileInputStream("file.txt"),  ← raw bytes from disk
          StandardCharsets.UTF_8            ← decode bytes to chars
      )                                     ← bridge InputStream → Reader
  )                                         ← add buffering + readLine()
```

---

## 2. Buffered I/O — How Buffering Reduces System Calls

```
  WITHOUT BUFFERING (FileInputStream directly):

  Your Code          Java VM            OS Kernel          Disk
  ─────────          ────────           ─────────          ────
  read() ─────────→  read(1 byte) ───→  syscall ─────────→ spin
         ←─────────  return byte  ←───  return 1 byte ←─── disk read
  read() ─────────→  read(1 byte) ───→  syscall ─────────→ spin
         ←─────────  return byte  ←───  return 1 byte ←─── disk read
  ... (1000 syscalls for 1000 bytes) ...

  ──────────────────────────────────────────────────────────────────

  WITH BUFFERING (BufferedInputStream, default buffer = 8192 bytes):

  Your Code          BufferedInputStream    OS Kernel          Disk
  ─────────          ─────────────────      ─────────          ────
  read() ─────────→  buffer empty!          syscall ─────────→ spin
                     read(8192 bytes) ─────────────────────→ disk read
                     ←─── 8192 bytes loaded into buffer ←────────
         ←─────────  return buffer[0]
  read() ─────────→  return buffer[1]   (no syscall!)
  read() ─────────→  return buffer[2]   (no syscall!)
  ...               (8191 more reads from buffer — no syscalls)
  read() ─────────→  buffer empty again!
                     read(8192 bytes) ─────→ next disk block
  ...

  RESULT: 1 syscall per 8192 bytes instead of 1 syscall per byte
          ~8000x fewer context switches to kernel mode

  ──────────────────────────────────────────────────────────────────

  BUFFER SIZE TUNING:

  Default (8192 bytes) — good for most cases
  Large files / network: 64KB-256KB buffer may improve throughput
  SSD random access: smaller buffers fine
  HDD sequential read: larger buffers amortize seek time

  new BufferedInputStream(in, 64 * 1024); // 64KB buffer
```

---

## 3. NIO Channel + Buffer Model vs Classic IO

```
  CLASSIC IO (Blocking, Stream-based):

  Thread ──→ InputStream.read() ──→ [BLOCKED] ──→ OS copies data to heap
                ↑ Thread is stuck here until data arrives

  One thread per connection model:
  Thread-1 ── handles connection A (may block for minutes)
  Thread-2 ── handles connection B
  Thread-3 ── handles connection C
  ...
  Thread-N ── handles connection N   (memory: N × ~1MB stack each)

  ──────────────────────────────────────────────────────────────────

  NIO (Non-Blocking, Channel + Buffer + Selector):

                        ┌────────────────────────────┐
                        │         Selector           │
                        │  (monitors N channels)     │
                        │                            │
                        │  select() blocks until     │
                        │  ANY channel is ready      │
                        └────────────┬───────────────┘
                                     │ ready events
            ┌────────────────────────┼─────────────────────────┐
            ▼                        ▼                          ▼
    SocketChannel A          SocketChannel B           SocketChannel C
    (READ ready)             (not ready)               (WRITE ready)

  Single thread handles all ready channels — O(1) processing per event

  ──────────────────────────────────────────────────────────────────

  BUFFER LIFECYCLE (NIO always uses Buffers, never raw bytes):

  ByteBuffer buf = ByteBuffer.allocate(1024);

  [          capacity=1024           ]
  0                                 1024
  position=0, limit=1024

  After channel.read(buf):   // fills buffer
  [XXXXXXXXXXXXXXXX          ]
  0               256       1024
  position=256, limit=1024

  buf.flip():                // prepare to read what was written
  [XXXXXXXXXXXXXXXX          ]
  0               256       1024
  position=0, limit=256    ← limit moves to old position

  After buf.get() reads:
  [XXXXXXXXXXXXXXXX          ]
       position=X, limit=256 ← position advances as you read

  buf.compact() or buf.clear() to reuse for next write
```

---

## 4. Serialization Object Graph

```
  OBJECT GRAPH IN MEMORY:

  Order ──→ Customer ──→ Address
    │            └──→ List<Order>  (back-reference to Order!)
    └──→ List<OrderItem>
               ├──→ Product
               └──→ Product (same Product referenced twice)

  ──────────────────────────────────────────────────────────────────

  HOW ObjectOutputStream HANDLES IT:

  1. Assign handle #1 to Order
  2. Write Order class descriptor (class name, serialVersionUID, fields)
  3. Encounter Customer field → assign handle #2 → write Customer
  4. Encounter Address field → assign handle #3 → write Address
  5. Encounter List<Order> in Customer → contains Order...
       → Order already written as handle #1 → write "reference to #1" (no duplicate)
  6. Encounter List<OrderItem> → write each item
  7. Encounter Product in item[0] → assign handle #4 → write Product
  8. Encounter Product in item[1] → same Product? handle #4 already exists
       → write "reference to #4" (shared object preserved!)

  SERIALIZED BYTE STREAM:

  [STREAM_MAGIC][STREAM_VERSION]
  [TC_OBJECT][ClassDesc:Order][serialVersionUID]
    [field: customer → TC_OBJECT][ClassDesc:Customer]
      [field: address → TC_OBJECT][ClassDesc:Address]...
      [field: orders → TC_REFERENCE #1]   ← circular ref handled!
    [field: items → TC_OBJECT][ClassDesc:List]
      [TC_OBJECT][ClassDesc:OrderItem]
        [field: product → TC_OBJECT][ClassDesc:Product][handle #4]
      [TC_OBJECT][ClassDesc:OrderItem]
        [field: product → TC_REFERENCE #4]  ← shared object, not duplicated!

  ──────────────────────────────────────────────────────────────────

  serialVersionUID IMPORTANCE:

  Version 1 of class (serialVersionUID = 1L):          Serialize →  bytes
  Version 2 adds a field (still serialVersionUID = 1L): Deserialize ✓  OK
  Version 2 changes serialVersionUID = 2L:              Deserialize ✗  InvalidClassException
```

---

## 5. File System Path Resolution

```
  ABSOLUTE PATH:    /home/user/projects/app/config.yml
  RELATIVE PATH:    ../config/settings.properties   (relative to CWD)

  ──────────────────────────────────────────────────────────────────

  Path resolution operations:

  Path base   = Path.of("/home/user/projects/app");
  Path config = Path.of("../config/settings.properties");

  base.resolve(config):
  → /home/user/projects/app/../config/settings.properties
    (just concatenates, does NOT normalize)

  base.resolve(config).normalize():
  → /home/user/projects/config/settings.properties
    (resolves ".." components)

  base.resolve(config).toRealPath():
  → follows symlinks, verifies file exists, returns canonical path

  ──────────────────────────────────────────────────────────────────

  Path components:

  Path p = Path.of("/home/user/projects/app/Main.java");

  p.getRoot()         → /
  p.getParent()       → /home/user/projects/app
  p.getFileName()     → Main.java
  p.getNameCount()    → 5   (home, user, projects, app, Main.java)
  p.getName(0)        → home
  p.getName(4)        → Main.java
  p.subpath(2, 4)     → projects/app

  ──────────────────────────────────────────────────────────────────

  Relativization (inverse of resolve):

  Path a = Path.of("/home/user/src");
  Path b = Path.of("/home/user/docs/report.pdf");

  a.relativize(b)  →  ../docs/report.pdf
  (how to get from a to b using relative path)
```

---

## 6. Memory-Mapped File (FileChannel + MappedByteBuffer)

```
  NORMAL FILE READ (InputStream):

  Disk ──→ [OS Page Cache] ──copy──→ [Java Heap Buffer] ──→ Your Code
               kernel space               user space

  Two copies: disk→page cache (OS), page cache→heap (JNI/JVM)
  Large file: heap pressure, GC overhead

  ──────────────────────────────────────────────────────────────────

  MEMORY-MAPPED FILE (FileChannel.map()):

  Disk ──→ [OS Page Cache] ══════════════════→ Your Code
               kernel space     virtual memory mapping
                                (no copy to heap!)

  MappedByteBuffer POINTS DIRECTLY into the OS page cache.
  Reading bytes = reading from virtual memory page.
  OS loads pages on demand (page faults) when you access them.

  ──────────────────────────────────────────────────────────────────

  MEMORY MAP LAYOUT:

  File on disk (1 GB):
  ┌──────────────────────────────────────────────────────────────┐
  │ [Page 0][Page 1][Page 2]...[Page N]                          │
  └──────────────────────────────────────────────────────────────┘

  Virtual Address Space (your process):
  ┌──────────────────────────────────────────────────────────────┐
  │ ...code... │ ...heap... │ [MAPPED REGION 0x7f00 - 0xBf00]... │
  └──────────────────────────────────────────────────────────────┘
                             └── points to OS page cache pages ──┘

  mapped.get(1_000_000):
    → access virtual address 0x7f00 + 1,000,000
    → if page not in RAM: OS page fault → loads from disk
    → if page in RAM: instant access (no syscall)

  ──────────────────────────────────────────────────────────────────

  WHEN TO USE:

  ✓ Large file random access (log search, database files)
  ✓ Multiple processes sharing the same file data
  ✓ High-throughput sequential reads of large files
  ✓ Avoiding GC pressure (data is off-heap)

  ✗ Small files (overhead not worth it)
  ✗ Files smaller than OS page size (4KB)
  ✗ Write-heavy workloads on Windows (file locking issues)
```
