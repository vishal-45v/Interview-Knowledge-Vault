# Bean Lifecycle & Scopes — Diagram Explanations

---

## Diagram 1: Complete Bean Lifecycle

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Spring Bean Lifecycle                         │
  │                                                                 │
  │  ① Instantiation                                               │
  │     Constructor called (dependencies injected via constructor)  │
  │                         │                                       │
  │  ② Property Population                                         │
  │     @Autowired fields, @Value injected via reflection           │
  │                         │                                       │
  │  ③ Aware Interfaces                                            │
  │     setBeanName(), setBeanFactory(), setApplicationContext()    │
  │                         │                                       │
  │  ④ BeanPostProcessor.postProcessBeforeInitialization()         │
  │     Custom pre-init logic, bean transformation                  │
  │                         │                                       │
  │  ⑤ Initialization                                              │
  │     @PostConstruct → afterPropertiesSet() → initMethod()        │
  │                         │                                       │
  │  ⑥ BeanPostProcessor.postProcessAfterInitialization()          │
  │     ← AOP PROXY CREATION HAPPENS HERE (@Transactional, @Async) │
  │                         │                                       │
  │           ┌─────────────▼──────────────────┐                   │
  │           │      BEAN READY FOR USE          │                  │
  │           └─────────────┬──────────────────┘                   │
  │                         │                                       │
  │         ┌───────────────▼────────────────────┐                 │
  │         │      APPLICATION RUNNING             │                │
  │         └───────────────┬────────────────────┘                 │
  │                         │                                       │
  │  ⑦ Destruction                                                 │
  │     @PreDestroy → destroy() → destroyMethod()                   │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Bean Scope Comparison

```
  SINGLETON (default):
  ┌──────────────────────────────────────────────────────────────┐
  │  ApplicationContext                                          │
  │                                                              │
  │  Request 1 → getBean("service") → ─────────────────────┐    │
  │  Request 2 → getBean("service") → ─────────────────┐   │    │
  │  Request 3 → getBean("service") → ─────────────┐   │   │    │
  │                                                 ▼   ▼   ▼   │
  │                                         ┌──────────────────┐ │
  │                                         │  Service Bean #1  │ │
  │                                         │  (shared)         │ │
  │                                         └──────────────────┘ │
  └──────────────────────────────────────────────────────────────┘

  PROTOTYPE:
  ┌──────────────────────────────────────────────────────────────┐
  │  ApplicationContext                                          │
  │                                                              │
  │  Request 1 → getBean("processor") → [Processor #1] (new!)   │
  │  Request 2 → getBean("processor") → [Processor #2] (new!)   │
  │  Request 3 → getBean("processor") → [Processor #3] (new!)   │
  │                                                              │
  │  Each caller gets a UNIQUE instance                         │
  │  Spring does NOT call @PreDestroy on these ✗                 │
  └──────────────────────────────────────────────────────────────┘

  REQUEST SCOPE (web apps):
  ┌──────────────────────────────────────────────────────────────┐
  │  HTTP Request 1 → [ShoppingCart #1] → destroyed after resp  │
  │  HTTP Request 2 → [ShoppingCart #2] → destroyed after resp  │
  │  HTTP Request 3 → [ShoppingCart #3] → destroyed after resp  │
  │                                                              │
  │  Each HTTP request gets its own bean instance               │
  └──────────────────────────────────────────────────────────────┘
```

---

## Diagram 3: Scoped Proxy — How Singleton Uses Request Bean

```
  Without Scoped Proxy (BROKEN):
  ┌────────────────────────────────────────────────────────────┐
  │  Singleton: CheckoutService                                │
  │                                                            │
  │  userCart = [ShoppingCart #0]  ← Set at startup           │
  │                                                            │
  │  Request 1: this.userCart → ShoppingCart #0  ← WRONG!     │
  │  Request 2: this.userCart → ShoppingCart #0  ← SAME!      │
  │  All requests share same cart ✗                           │
  └────────────────────────────────────────────────────────────┘

  With Scoped Proxy (CORRECT):
  ┌────────────────────────────────────────────────────────────┐
  │  Singleton: CheckoutService                                │
  │                                                            │
  │  userCart = [ShoppingCartProxy] ← CGLIB proxy             │
  │                                                            │
  │  Request 1: this.userCart.addItem()                        │
  │    → Proxy.addItem()                                       │
  │    → scope.get("shoppingCart", "request-1")                │
  │    → ShoppingCart #1.addItem() ✓                           │
  │                                                            │
  │  Request 2: this.userCart.addItem()                        │
  │    → Proxy.addItem()                                       │
  │    → scope.get("shoppingCart", "request-2")                │
  │    → ShoppingCart #2.addItem() ✓                           │
  └────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: BeanPostProcessor — Bean Transformation

```
  Each bean passes through ALL BeanPostProcessors:

  ┌────────────────────────────────────────────────────────────────┐
  │  Bean A (raw object, just constructed)                         │
  │         │                                                       │
  │         ▼                                                       │
  │  BPP1.postProcessBeforeInitialization(beanA, "beanA")          │
  │         │  [Returns: beanA or a wrapper]                        │
  │         ▼                                                       │
  │  @PostConstruct / afterPropertiesSet() / initMethod            │
  │         │                                                       │
  │         ▼                                                       │
  │  BPP1.postProcessAfterInitialization(beanA, "beanA")           │
  │         │  [Returns: beanA or a PROXY]                          │
  │         ▼                                                       │
  │  BPP2.postProcessAfterInitialization(beanA, "beanA")           │
  │         │  [Can wrap beanA further]                             │
  │         ▼                                                       │
  │  Final bean stored in ApplicationContext                        │
  │  (may be the original bean or a proxy/wrapper)                 │
  └────────────────────────────────────────────────────────────────┘

  Example transformation:
  UserService → [after postProcessAfterInitialization]
             → UserService$$EnhancerBySpringCGLIB (AOP proxy)

  Callers inject and use the PROXY, not the original bean!
```

---

## Diagram 5: SmartLifecycle Phase Ordering

```
  Application Context Startup:
  ─────────────────────────────────────────────────────────────
  Phase -2147483648 (Integer.MIN_VALUE)
    └── DataSource, SecurityFilter (start very early)

  Phase -1000
    └── Cache infrastructure

  Phase 0 (default)
    └── Most application beans (services, controllers)

  Phase 1000
    └── Message consumers, background workers

  Phase 2147483647 (Integer.MAX_VALUE)
    └── Health check endpoint (start last — app fully ready)

  Application Context Shutdown (REVERSE ORDER):
  ─────────────────────────────────────────────────────────────
  Phase 2147483647 (stops first)
    └── Health check → reports UNHEALTHY (stop accepting traffic)

  Phase 1000
    └── Message consumers stop (no new work accepted)

  Phase 0
    └── In-flight requests finish

  Phase -1000
    └── Cache flush

  Phase -2147483648 (stops last)
    └── DataSource closes connections
```
