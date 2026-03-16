# Spring Core DI — Diagram Explanations

> ASCII diagrams showing Spring DI internals, proxy mechanics, and container flow.

---

## Diagram 1: Spring IoC Container — Bean Creation Flow

```
  @SpringBootApplication
         │
         ▼
  SpringApplication.run()
         │
         ▼
  ┌──────────────────────────────────────────────────────────┐
  │           ApplicationContext Initialization               │
  │                                                          │
  │  1. Create BeanFactory                                   │
  │  2. Load BeanDefinitions                                 │
  │     ├── Scan @Component, @Service, @Repository           │
  │     ├── Process @Configuration classes                   │
  │     └── Register @Bean methods                           │
  │  3. Run BeanFactoryPostProcessors                        │
  │     └── PropertySourcesPlaceholderConfigurer             │
  │         (resolves @Value placeholders)                   │
  │  4. Register BeanPostProcessors                          │
  │     ├── AutowiredAnnotationBeanPostProcessor             │
  │     └── CommonAnnotationBeanPostProcessor                │
  │  5. Instantiate Singleton Beans (eager)                  │
  │     ├── Constructor called                               │
  │     ├── Dependencies injected                            │
  │     ├── @PostConstruct called                            │
  │     └── Bean stored in singletonObjects map              │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  Application Ready ✓
```

---

## Diagram 2: @Autowired Resolution Algorithm

```
  @Autowired
  PaymentGateway gateway;
         │
         ▼
  Find all beans of type PaymentGateway
  ┌─────────────────────────────┐
  │  stripeGateway  (Stripe)    │
  │  paypalGateway  (PayPal)    │
  │  squareGateway  (Square)    │
  └─────────────────────────────┘
         │
         ▼
  Is there a @Qualifier annotation?
  ├── YES → Filter by qualifier name
  │          └── Use matching bean ✓
  └── NO  ─────────────────────────────────────────────────┐
                                                           │
                                                           ▼
                                                   Is there a @Primary bean?
                                                   ├── YES → Use @Primary bean ✓
                                                   └── NO  ──────────────────────┐
                                                                                 │
                                                                                 ▼
                                                                   Does field name match a bean name?
                                                                   ├── YES → Use that bean ✓
                                                                   └── NO  → NoUniqueBeanDefinitionException ✗
```

---

## Diagram 3: Spring Proxy — How @Transactional Works

```
  OrderController
        │
        │ calls orderService.placeOrder()
        ▼
  ┌──────────────────────────────────────────────────────┐
  │          CGLIB Proxy (OrderServiceProxy)              │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  @Transactional interceptor                     │  │
  │  │  1. BEGIN TRANSACTION                           │  │
  │  │  2. ─────────────────────────────────────────┐ │  │
  │  │                                               │ │  │
  │  │          ┌──────────────────────────┐        │ │  │
  │  │          │   Real OrderService      │        │ │  │
  │  │          │   placeOrder() executes  │        │ │  │
  │  │          └──────────────────────────┘        │ │  │
  │  │                                               │ │  │
  │  │  3. ◄────────────────────────────────────────┘ │  │
  │  │  4. COMMIT (or ROLLBACK on exception)           │  │
  │  └────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────┘

  SELF-INVOCATION PROBLEM:
  orderService.placeOrder() calls this.sendConfirmation()
  → Calls directly on Real OrderService
  → BYPASSES the proxy
  → @Transactional on sendConfirmation() has NO effect
```

---

## Diagram 4: Three-Level Cache for Circular Dependency Resolution

```
  Creating Bean A (depends on B)
  Creating Bean B (depends on A)

  Step 1: Start creating A
  ┌──────────────────────────────────────────────────┐
  │  Level 3 (singletonFactories):  {"a": factory_A}  │
  │  Level 2 (earlySingletonObjects): {}               │
  │  Level 1 (singletonObjects):    {}                 │
  └──────────────────────────────────────────────────┘

  Step 2: A needs B → start creating B
  Step 3: B needs A → check caches:
    - Level 1: not found
    - Level 2: not found
    - Level 3: found factory_A → invoke factory → get early A reference

  ┌──────────────────────────────────────────────────┐
  │  Level 3 (singletonFactories):  {}                │
  │  Level 2 (earlySingletonObjects): {"a": earlyA}   │
  │  Level 1 (singletonObjects):    {}                 │
  └──────────────────────────────────────────────────┘

  Step 4: B created successfully (holds reference to earlyA)
  ┌──────────────────────────────────────────────────┐
  │  Level 2 (earlySingletonObjects): {"a": earlyA}   │
  │  Level 1 (singletonObjects):    {"b": fullyB}     │
  └──────────────────────────────────────────────────┘

  Step 5: A finishes initialization (now holds reference to fullyB)
  ┌──────────────────────────────────────────────────┐
  │  Level 2 (earlySingletonObjects): {}              │
  │  Level 1 (singletonObjects):    {"a": fullyA,     │
  │                                  "b": fullyB}     │
  └──────────────────────────────────────────────────┘

  NOTE: This ONLY works for setter/field injection.
  Constructor injection cannot provide an early reference.
```

---

## Diagram 5: Prototype Bean in Singleton — The Problem and Fix

```
  WITHOUT FIX (Bug):
  ┌──────────────────────────────────────────────────┐
  │            SingletonService (Scope: singleton)    │
  │                                                   │
  │  prototypeBean ──────────────► [PrototypeBean #1] │
  │                 (injected once at startup)         │
  │                                                   │
  │  processA() → uses PrototypeBean #1               │
  │  processB() → uses PrototypeBean #1  ← Same!      │
  │  processC() → uses PrototypeBean #1  ← Same!      │
  └──────────────────────────────────────────────────┘
  Expected: New PrototypeBean each call
  Actual: Same PrototypeBean every call

  WITH FIX (ObjectProvider):
  ┌──────────────────────────────────────────────────┐
  │            SingletonService (Scope: singleton)    │
  │                                                   │
  │  objectProvider ─────────────► ObjectProvider     │
  │                 (injected once at startup)         │
  │                                                   │
  │  processA() → provider.getObject() → [PB #1] NEW  │
  │  processB() → provider.getObject() → [PB #2] NEW  │
  │  processC() → provider.getObject() → [PB #3] NEW  │
  └──────────────────────────────────────────────────┘
  Each call creates a new PrototypeBean ✓
```

---

## Diagram 6: Bean Scopes Comparison

```
  ┌──────────────────┬─────────────────────────────────────────────┐
  │  Scope           │  Description                                 │
  ├──────────────────┼─────────────────────────────────────────────┤
  │  singleton       │  One instance per ApplicationContext (default)│
  │  (default)       │  ┌────┐                                     │
  │                  │  │ B1 │ ◄── All requests get same bean      │
  │                  │  └────┘                                     │
  ├──────────────────┼─────────────────────────────────────────────┤
  │  prototype       │  New instance every time it's requested     │
  │                  │  ┌────┐                                     │
  │                  │  │ B1 │ ◄── Request 1                       │
  │                  │  ├────┤                                     │
  │                  │  │ B2 │ ◄── Request 2                       │
  │                  │  ├────┤                                     │
  │                  │  │ B3 │ ◄── Request 3                       │
  │                  │  └────┘                                     │
  ├──────────────────┼─────────────────────────────────────────────┤
  │  request         │  New instance per HTTP request (web only)   │
  ├──────────────────┼─────────────────────────────────────────────┤
  │  session         │  New instance per HTTP session (web only)   │
  ├──────────────────┼─────────────────────────────────────────────┤
  │  application     │  One instance per ServletContext (web only) │
  └──────────────────┴─────────────────────────────────────────────┘
```

---

## Diagram 7: @Configuration Full Mode vs Lite Mode

```
  FULL MODE (@Configuration):
  ┌────────────────────────────────────────────────────────┐
  │  @Configuration  ← CGLIB proxy wraps the class         │
  │  class AppConfig {                                      │
  │                                                         │
  │    @Bean ServiceA serviceA() {                          │
  │      return new ServiceA(serviceB()); ─────────────┐   │
  │    }                            intercepts call     │   │
  │                                                     ▼   │
  │    @Bean ServiceB serviceB() {    Spring returns existing bean  │
  │      return new ServiceB();  ←── instead of creating new       │
  │    }                                                    │
  │  }                                                      │
  │                                                         │
  │  Result: serviceA and serviceB share SAME ServiceB      │
  └────────────────────────────────────────────────────────┘

  LITE MODE (@Component with @Bean):
  ┌────────────────────────────────────────────────────────┐
  │  @Component  ← NOT CGLIB proxied                        │
  │  class AppConfig {                                      │
  │                                                         │
  │    @Bean ServiceA serviceA() {                          │
  │      return new ServiceA(serviceB()); ─────────────┐   │
  │    }                         plain Java method call │   │
  │                                                     ▼   │
  │    @Bean ServiceB serviceB() {    Creates NEW ServiceB  │
  │      return new ServiceB();  ← each time called        │
  │    }                                                    │
  │  }                                                      │
  │                                                         │
  │  Result: serviceA gets DIFFERENT ServiceB than context  │
  └────────────────────────────────────────────────────────┘
```

---

## Diagram 8: Spring ApplicationContext Hierarchy (Web App)

```
  Traditional Spring MVC (pre-Boot):

  ┌────────────────────────────────────────────────────┐
  │         Root ApplicationContext                     │
  │  (loaded by ContextLoaderListener)                  │
  │  - @Service beans                                   │
  │  - @Repository beans                                │
  │  - DataSource, TransactionManager                   │
  │  - Security config                                  │
  └──────────────────────┬─────────────────────────────┘
                         │  parent reference
  ┌──────────────────────▼─────────────────────────────┐
  │         Web ApplicationContext                       │
  │  (loaded by DispatcherServlet)                       │
  │  - @Controller beans                                 │
  │  - ViewResolver                                      │
  │  - HandlerMapping                                    │
  │  - @ControllerAdvice                                 │
  │                                                      │
  │  Can access parent beans ✓                           │
  │  Parent cannot access these beans ✗                  │
  └──────────────────────────────────────────────────────┘

  Spring Boot:
  ┌────────────────────────────────────────────────────┐
  │         Single ApplicationContext                   │
  │  - All beans in one context                         │
  │  - No parent/child split by default                 │
  └────────────────────────────────────────────────────┘
```
