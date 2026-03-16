# Chapter 08: Advanced Java — Diagram Explanations

ASCII diagrams showing the architecture and flow of advanced Java features.

---

## 1. Generic Type Erasure: Compile Time vs Runtime

At compile time, the compiler sees the full generic type. After compilation, all type parameters are replaced with their bounds (or `Object`). The runtime never sees `<String>` — only `Object` or the bounded type.

```
COMPILE TIME                          RUNTIME (after erasure)
─────────────────────────────────     ─────────────────────────────────

  List<String> list                    List list
       │                                    │
       ├─ add(String s) ✓                   ├─ add(Object o)
       ├─ add(Integer i) ✗ ERROR            ├─ add(Object o)
       └─ String get(0)                     └─ Object get(0)
                                                   │
                                            (checkcast String)
                                            inserted by compiler

  Pair<Integer, String>                Pair
       │                                    │
       ├─ T  → Integer (known)              ├─ first  → Object
       └─ V  → String (known)              └─ second → Object

  Type tokens in .class file:
  ┌─────────────────────────────────────────────────────────────┐
  │  Signature attr: Ljava/util/ArrayList<Ljava/lang/String;>;  │  ← for tools/reflection only
  │  Bytecode:       invokevirtual ArrayList.add(Object)        │  ← what JVM actually runs
  └─────────────────────────────────────────────────────────────┘

  Proof:
    List<String> a = new ArrayList<>();
    List<Integer> b = new ArrayList<>();
    a.getClass() == b.getClass()  →  true   (both are ArrayList.class)
```

---

## 2. Wildcard Variance — PECS Rule (Producer Extends, Consumer Super)

```
  Type Hierarchy:

       Object
          │
        Number
        /    \
   Integer  Double

  ─────────────────────────────────────────────────────────────────────

  PRODUCER  (? extends Number) — you can READ from it

    List<? extends Number>   can hold:  List<Integer>, List<Double>, List<Number>
         │
         ├─ READ:  Number n = list.get(0);  ✓  (safe — always at least a Number)
         └─ WRITE: list.add(1);             ✗  (unsafe — list might be List<Double>)

  ─────────────────────────────────────────────────────────────────────

  CONSUMER  (? super Integer) — you can WRITE to it

    List<? super Integer>    can hold:  List<Integer>, List<Number>, List<Object>
         │
         ├─ WRITE: list.add(42);            ✓  (safe — Integer fits in all above)
         └─ READ:  Integer i = list.get(0); ✗  (unsafe — might be Number or Object)
                   Object   o = list.get(0); ✓  (only safe as Object)

  ─────────────────────────────────────────────────────────────────────

  PECS VISUAL — Collections.copy signature:

    static <T> void copy(List<? super T> dest,  List<? extends T> src)
                              ↑                        ↑
                         CONSUMER                  PRODUCER
                      (writes T into dest)     (reads T from src)

  ─────────────────────────────────────────────────────────────────────

  UNBOUNDED  (?) — neither producer nor consumer

    void printAll(List<?> list) — can only call methods on Object
    Useful for: "I have a list, I don't care what's in it"
```

---

## 3. ClassLoader Hierarchy

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   Bootstrap ClassLoader                      │
  │           (native C code, no Java class for it)             │
  │   Loads: java.lang.*, java.util.*, java.io.*, rt.jar        │
  │   getParent() returns null                                   │
  └──────────────────────┬──────────────────────────────────────┘
                         │  parent
                         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              Platform ClassLoader (Java 9+)                  │
  │              Extension ClassLoader (Java 8)                  │
  │   Loads: java.se, javax.*, named platform modules            │
  └──────────────────────┬──────────────────────────────────────┘
                         │  parent
                         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                 Application ClassLoader                      │
  │   Loads: -classpath / -cp entries, your app's .jar files     │
  └──────────────────────┬──────────────────────────────────────┘
                         │  parent
                         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  Custom ClassLoader(s)                       │
  │   e.g. PluginLoader, URLClassLoader, Tomcat WebappLoader    │
  │   Loads: plugins, hot-reload classes, isolated modules       │
  └─────────────────────────────────────────────────────────────┘

  DELEGATION MODEL — how loadClass("com.example.Foo") flows:

    AppClassLoader.loadClass("com.example.Foo")
         │
         ├─→ delegate to PlatformClassLoader
         │       │
         │       ├─→ delegate to BootstrapClassLoader
         │       │       │
         │       │       └─ "Not found in rt.jar" → returns null
         │       │
         │       └─ "Not found in platform modules" → ClassNotFoundException
         │
         └─ "Found in classpath!" → load & return Class object

  WHY THIS MATTERS:
    • Prevents user code from replacing java.lang.String
    • Each ClassLoader has its own namespace
    • Same .class file loaded by two different ClassLoaders = two distinct Class objects
```

---

## 4. Dynamic Proxy Architecture

```
  CLIENT CODE                    PROXY LAYER                    REAL OBJECT
  ──────────                     ───────────                    ───────────

  MyService proxy = (MyService)
    Proxy.newProxyInstance(...)

  proxy.doWork(args)
       │
       ▼
  ┌────────────────────┐
  │   Proxy Object     │    ← generated at runtime, implements MyService
  │  (auto-generated)  │
  │                    │
  │  doWork(args) {    │
  │    handler.invoke( │
  │      this,         │
  │      doWorkMethod, │
  │      args          │
  │    );              │
  │  }                 │
  └──────────┬─────────┘
             │ calls
             ▼
  ┌────────────────────────────────┐
  │      InvocationHandler         │
  │  (your code — e.g. logging,   │
  │   transactions, caching)       │
  │                                │
  │  invoke(proxy, method, args) { │
  │    // BEFORE logic             │
  │    method.invoke(target, args) │──────→  ┌─────────────────┐
  │    // AFTER logic              │         │  RealService     │
  │  }                             │ ←──────  │  doWork(args)   │
  └────────────────────────────────┘         └─────────────────┘

  KEY CONSTRAINT:
  ┌────────────────────────────────────────────────────────────┐
  │  java.lang.reflect.Proxy can ONLY proxy INTERFACES.        │
  │  To proxy a concrete class → use CGLIB or ByteBuddy.       │
  │  Spring uses JDK Proxy when interface exists,              │
  │  CGLIB when only a concrete class is available.            │
  └────────────────────────────────────────────────────────────┘
```

---

## 5. Annotation Processing Pipeline

```
  SOURCE CODE (.java)
  ───────────────────
  @Entity
  @Table(name = "users")
  public class User { ... }

       │
       ▼  javac invokes annotation processors (APT)
  ┌─────────────────────────────────────────────────────────────┐
  │            Annotation Processor (compile time)              │
  │                                                             │
  │  @SupportedAnnotationTypes("javax.persistence.Entity")      │
  │  public class EntityProcessor extends AbstractProcessor {   │
  │      process(annotations, roundEnv) {                       │
  │          // Read @Entity classes → generate boilerplate     │
  │          // e.g. Lombok, MapStruct, Dagger, JPA Metamodel   │
  │      }                                                       │
  │  }                                                          │
  └──────────────┬──────────────────────────────────────────────┘
                 │ may generate new .java files
                 ▼
  COMPILED BYTECODE (.class)
  ──────────────────────────
  Annotation with @Retention(CLASS) → stored in .class, invisible at runtime
  Annotation with @Retention(RUNTIME) → stored in .class, readable via reflection
  Annotation with @Retention(SOURCE) → discarded after compilation (@Override, @SuppressWarnings)

       │
       ▼  JVM loads class
  ┌─────────────────────────────────────────────────────────────┐
  │            Runtime Annotation Inspection                    │
  │                                                             │
  │  Class<?> clazz = User.class;                               │
  │  Table table = clazz.getAnnotation(Table.class);           │
  │  System.out.println(table.name()); // "users"               │
  └─────────────────────────────────────────────────────────────┘

  Retention Policy Summary:

    SOURCE  ──→  Compiler reads → discards → NOT in .class
    CLASS   ──→  Compiler reads → keeps in .class → NOT at runtime (default)
    RUNTIME ──→  Compiler reads → keeps in .class → AVAILABLE at runtime
```

---

## 6. Java Module System (JPMS)

```
  BEFORE MODULES (Classpath — the "Big Ball of Mud")
  ────────────────────────────────────────────────────

    [app.jar] ─── can access ──→ [everything on classpath]
    [lib-a.jar]                   including internal APIs
    [lib-b.jar]                   including sun.misc.Unsafe!
    [lib-c.jar]

    Problem: split packages, version conflicts, no encapsulation

  ─────────────────────────────────────────────────────────────

  WITH MODULES (module-info.java defines the contract)

    ┌──────────────────────────────────────────────────────┐
    │  module com.myapp.web                                │
    │  ┌────────────────────────────────────────────────┐ │
    │  │  requires com.myapp.service;  // hard dep       │ │
    │  │  requires java.net.http;      // JDK module     │ │
    │  │                                                 │ │
    │  │  exports com.myapp.web.api;   // public surface │ │
    │  │  // com.myapp.web.internal  ← NOT exported      │ │
    │  └────────────────────────────────────────────────┘ │
    └──────────────────────────────────────────────────────┘
             │ requires                    │ requires
             ▼                             ▼
    ┌────────────────────┐      ┌────────────────────────┐
    │ com.myapp.service  │      │     java.net.http       │
    │  exports:          │      │   (JDK platform module) │
    │  - service.api     │      └────────────────────────┘
    │  - service.dto     │
    │  internal: LOCKED  │
    └────────────────────┘

  Key Directives:
  ┌──────────────────────────────────────────────────────────────┐
  │  requires           → compile + runtime dependency           │
  │  requires transitive → dependency exposed to your consumers  │
  │  exports            → makes package accessible to all        │
  │  exports X to Y     → makes package accessible only to Y     │
  │  opens X            → allows deep reflection into package X  │
  │  opens X to Y       → allows reflection only from module Y   │
  │  uses               → declares a ServiceLoader dependency    │
  │  provides X with Y  → registers a ServiceLoader provider     │
  └──────────────────────────────────────────────────────────────┘
```
