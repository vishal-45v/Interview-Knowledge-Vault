# High CPU Debugging

---

## Common Causes of High CPU in Java

1. Infinite loops or tight loops without blocking
2. GC thrashing (too frequent GC due to heap pressure)
3. CPU-intensive computation (image processing, crypto, serialization)
4. Thread contention (many threads competing for locks)
5. Regex with catastrophic backtracking
6. Reflection/deserialization in tight loops

---

## Identifying the Problem

```bash
# Step 1: Which process is consuming CPU?
top -H -p <java_pid>  # Shows per-thread CPU on Linux

# Step 2: Note high-CPU thread IDs (in decimal)
# Convert decimal to hex (for jstack matching):
printf "%x\n" <thread_id>

# Step 3: Generate thread dump
jstack <pid> > /tmp/threads.txt

# Step 4: Find thread in dump by hex ID
grep "nid=0x<hex_id>" /tmp/threads.txt -A 20

# Step 5: The stack trace shows what the thread is doing!
```

---

## GC Thrashing

```
Symptom: 
  - High CPU% in threads named "GC task thread"
  - Short GC cycles happening very frequently
  - Application response time degraded

Diagnosis:
  jstat -gcutil <pid> 5000  # GC stats every 5 seconds
  # If GC% is > 5-10%, you have GC pressure

Causes:
  - Heap too small → GC constantly trying to make room
  - Too many short-lived objects → Young gen fills fast
  - Memory leak → Old gen grows until full GC

Fix:
  - Increase heap: -Xmx2g
  - Tune GC: -XX:+UseG1GC -XX:MaxGCPauseMillis=200
  - Fix the memory leak (see memory-leak-debugging.md)
  - Reduce object allocation rate
```

---

## CPU Profiling with Async-Profiler

```bash
# Install async-profiler
curl -L https://github.com/async-profiler/async-profiler/releases/latest/download/async-profiler.tar.gz | tar xz

# Profile CPU for 30 seconds, generate flame graph
./profiler.sh -d 30 -f /tmp/flamegraph.html <pid>
# Opens flame graph in browser

# The flame graph shows:
# - Width = % of CPU time
# - Stacks = call chains
# - Hottest method = widest block at top

# Look for unexpected wide blocks:
# - Jackson serialization in tight loop → cache ObjectMapper
# - Pattern.compile() every request → compile once at startup
# - String.format() in hot path → use StringBuilder
```

---

## Catastrophic Regex Backtracking

```java
// DANGEROUS: Exponential backtracking
Pattern dangerous = Pattern.compile("(a+)+$");
String input = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaab";
// This can take MINUTES to fail! CPU-intensive.

// Safe alternatives:
Pattern.compile("[a-z]+$");  // No nested quantifiers

// Prevention:
// 1. Timeout on regex match:
new Thread(() -> {
    if (!pattern.matcher(input).matches()) throw new InvalidInputException();
}).start();
// Better: use libraries that detect catastrophic backtracking

// 2. Validate input length before regex
if (input.length() > 1000) throw new InputTooLongException();
```

---

## Production High CPU Response

```
Step 1: Is it expected load or bug?
  → Check traffic volume vs historical baselines
  → If traffic is normal but CPU is high → likely a bug

Step 2: Identify the thread
  → top -H -p <pid> → get thread ID
  → jstack → find thread stack trace

Step 3: Common diagnoses:
  → Thread stuck in loop: fix the loop condition
  → GC thrashing: increase heap, fix leak
  → Hot method: profile with async-profiler, optimize or cache

Step 4: Short-term: Scale out more instances
Step 5: Long-term: Fix root cause
```
