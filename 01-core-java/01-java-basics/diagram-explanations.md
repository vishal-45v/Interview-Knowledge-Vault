# Java Basics — Diagram Explanations

---

## JVM Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                      JVM MEMORY                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                       HEAP                              │   │
│  │                                                         │   │
│  │   Young Generation                  Old Generation      │   │
│  │   ┌──────────────────────────┐   ┌───────────────────┐  │   │
│  │   │  Eden    │ S0  │   S1   │   │    Tenured Space  │  │   │
│  │   │  (new    │(sur-│(surv-  │   │  (long-lived obj) │  │   │
│  │   │  objects)│vivor│ ivor)  │   │                   │  │   │
│  │   └──────────────────────────┘   └───────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              METASPACE (non-heap, native memory)        │   │
│  │  Class metadata, method bytecodes, static fields        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           THREAD STACKS (one per thread)                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │  Thread 1   │  │  Thread 2   │  │  Thread N   │    │   │
│  │  │ Stack Frame │  │ Stack Frame │  │ Stack Frame │    │   │
│  │  │ (main())    │  │ (run())     │  │ (worker())  │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## String Pool vs Heap

```
 String s1 = "hello";
 String s2 = "hello";
 String s3 = new String("hello");

┌──────────────────────────────────────────────────────────────┐
│                         HEAP MEMORY                          │
│                                                              │
│   String Pool:                                               │
│   ┌────────────────────────────────────────────────┐        │
│   │  "hello"  ◄──── s1                             │        │
│   │            ◄──── s2   (same object!)            │        │
│   └────────────────────────────────────────────────┘        │
│                                                              │
│   General Heap:                                              │
│   ┌────────────────────────────────────────────────┐        │
│   │  "hello" (copy)  ◄──── s3  (different object)  │        │
│   └────────────────────────────────────────────────┘        │
│                                                              │
│   s1 == s2   → true   (same pool reference)                 │
│   s1 == s3   → false  (pool vs heap)                        │
│   s1.equals(s3) → true  (same content)                      │
└──────────────────────────────────────────────────────────────┘
```

---

## ClassLoader Hierarchy

```
        ┌──────────────────────────┐
        │  Bootstrap ClassLoader   │ ← loads java.lang.*, java.util.*
        │  (native code)           │
        └─────────────┬────────────┘
                      │ parent
                      ▼
        ┌──────────────────────────┐
        │  Platform ClassLoader    │ ← loads JDK extensions (Java 9+)
        │  (Java code)             │
        └─────────────┬────────────┘
                      │ parent
                      ▼
        ┌──────────────────────────┐
        │  Application ClassLoader │ ← loads your app's classes
        │  (classpath)             │
        └─────────────┬────────────┘
                      │ (custom loaders can chain here)
                      ▼
        ┌──────────────────────────┐
        │  Custom ClassLoader      │ ← plugin systems, OSGi, etc.
        │  (URLClassLoader, etc.)  │
        └──────────────────────────┘

  Request flow: child asks parent first → if not found, child loads
```

---

## Object Creation in Memory

```
User user = new User("Alice", 30);

Stack (Thread 1):              Heap:
┌─────────────────┐           ┌──────────────────────────────────────────┐
│ main() frame    │           │                                          │
│ ┌─────────────┐ │           │   User object @ 0x7f3c4b90               │
│ │user  0x7f3c │─┼──────────►│   ┌──────────────────────────────────┐   │
│ └─────────────┘ │           │   │ name ref ──────────►"Alice" in   │   │
│                 │           │   │               pool               │   │
│                 │           │   │ age  = 30                        │   │
│                 │           │   └──────────────────────────────────┘   │
│                 │           │                                          │
└─────────────────┘           └──────────────────────────────────────────┘
```
