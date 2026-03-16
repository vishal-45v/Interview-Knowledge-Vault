# Java Basics — Structured Answers

> Deep-dive structured answers for the most commonly asked Java basics topics.

---

## Topic 1: JVM Architecture — Complete Breakdown

### Class Loading Subsystem
1. **Loading:** Finds and loads .class bytecode (Bootstrap → Platform → Application classloader chain)
2. **Verification:** Ensures bytecode is valid and doesn't violate JVM security constraints
3. **Preparation:** Allocates memory for static fields and sets default values
4. **Resolution:** Replaces symbolic references with direct references
5. **Initialization:** Executes static initializers and assigns static field values

### Runtime Data Areas
- **Method Area (Metaspace):** Class structure, method data, constant pool
- **Heap:** All object instances and arrays; split into Young (Eden + Survivor) and Old
- **Java Stacks:** Per-thread; each method call creates a stack frame
- **PC Registers:** Per-thread; points to current instruction
- **Native Method Stacks:** For JNI/native code

### Execution Engine
- **Interpreter:** Reads and executes bytecode one instruction at a time (slow but immediate)
- **JIT Compiler:** Compiles hot bytecode to native machine code (C1 + C2 tiered compilation)
- **GC:** Automatic memory reclamation

---

## Topic 2: String Immutability — Complete Explanation

### Why String is Designed to be Immutable

**Technical implementation:**
```java
public final class String implements Comparable<String>, CharSequence {
    private final byte[] value;          // Java 9+: compact strings (byte[] not char[])
    private final byte coder;            // LATIN1 or UTF16
    private int hash;                    // cached hashCode, default 0
    private boolean hashIsZero;          // disambiguate hash=0 from uncomputed

    // No public method modifies value after construction
}
```

**Five reasons for immutability:**
1. String pool safety (shared objects cannot be modified)
2. Security (class names, URLs, passwords)
3. Thread safety (no synchronization needed)
4. HashCode caching (HashMap performance)
5. ClassLoader integrity

### String Operations and Memory
```java
String s = "Hello";          // "Hello" in pool
s = s + " World";            // creates "Hello World" in heap/pool
                             // original "Hello" eligible for GC if no other reference

// StringBuilder is mutable — uses a resizable byte array internally
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");         // modifies same object — no new String created
```

---

## Topic 3: Garbage Collection — How It Works

### Object Lifecycle
```
new Object() → allocated in Eden (Young Gen)
              ↓ (if Eden full, Minor GC runs)
     Survives Minor GC → moved to Survivor space (S0 or S1)
              ↓ (after N GC cycles, default age=15)
     Old enough → promoted to Old Gen (Tenured)
              ↓ (if Old Gen full, Major/Full GC runs)
     No more references → collected
```

### GC Algorithms (G1 — Default since Java 9)
- Heap divided into equal-sized regions (default 1-32 MB)
- Some regions designated Young, some Old, some Humongous (large objects)
- GC focuses on regions with most garbage first (Garbage First = G1)
- Concurrent marking runs alongside application
- Stop-the-world pauses for evacuation (copying live objects)

### Key GC Tuning
```bash
-XX:+UseG1GC                    # use G1 (default Java 9+)
-Xmx4g -Xms4g                   # fix heap size (prevents resize overhead)
-XX:MaxGCPauseMillis=200        # target pause time
-XX:G1HeapRegionSize=16m        # region size
-Xlog:gc*:file=gc.log:time:filecount=5,filesize=20m  # GC logging
```
