# Spring & Hibernate Basics — Diagram Explanations

---

## 1. Spring IoC Container — Bean Lifecycle

```
ApplicationContext.refresh()
          |
          v
+---------------------------+
| 1. INSTANTIATION          |
|    new BeanClass()        |  <- Constructor called
|    (or via factory)       |
+---------------------------+
          |
          v
+---------------------------+
| 2. POPULATE PROPERTIES    |
|    @Autowired fields set  |  <- DI: dependencies injected
|    Setter injection       |
+---------------------------+
          |
          v
+---------------------------+
| 3. AWARE CALLBACKS        |
|    setBeanName()          |  <- BeanNameAware
|    setApplicationContext()|  <- ApplicationContextAware
+---------------------------+
          |
          v
+---------------------------+
| 4. BeanPostProcessor      |
|    postProcessBefore      |  <- Spring AOP proxies created here!
|    Initialization()       |
+---------------------------+
          |
          v
+---------------------------+
| 5. INITIALIZATION         |
|    @PostConstruct         |  <- Your init code
|    afterPropertiesSet()   |  <- InitializingBean
|    init-method            |
+---------------------------+
          |
          v
+---------------------------+
| 6. BeanPostProcessor      |
|    postProcessAfter       |  <- Further proxy wrapping
|    Initialization()       |
+---------------------------+
          |
          v
    [BEAN IS READY]
    [Used by application]
          |
          v
+---------------------------+
| 7. DESTRUCTION            |
|    @PreDestroy            |  <- Your cleanup code
|    destroy()              |  <- DisposableBean
|    destroy-method         |
+---------------------------+
```

---

## 2. Dependency Injection Types

```
CONSTRUCTOR INJECTION (recommended):
+------------------+
| OrderService     |
|------------------|
| - emailSvc       |<----- Spring injects via constructor
| - paymentSvc     |
| + OrderService(  |
|    EmailService, |
|    PaymentService|
|   )              |
+------------------+

Advantages:
  - Immutable fields (final)
  - Dependencies visible in constructor signature
  - Easy to test with manual instantiation
  - No null pointer issues — either all deps provided or constructor fails

SETTER INJECTION:
  - Optional dependencies
  - Can be re-injected (for testing)
  - Fields are nullable — need null checks

FIELD INJECTION (@Autowired on field):
  - Discouraged: can't use final, can't inject without Spring container
  - Can't easily write unit tests without a Spring context
  - Hidden dependencies — not visible in constructor

Example:
  // BAD
  @Autowired
  private OrderRepository repo;  // Spring can't enforce non-null; hidden dependency

  // GOOD
  private final OrderRepository repo;
  public OrderService(OrderRepository repo) { this.repo = repo; }
```

---

## 3. Spring MVC Request Flow

```
HTTP Request: GET /orders/123
        |
        v
+-------------------+
| DispatcherServlet |  <- Front Controller
| (one per app)     |
+-------------------+
        |
        | Which controller handles this?
        v
+-------------------+
| HandlerMapping    |  <- Finds @GetMapping("/orders/{id}")
+-------------------+
        |
        v
+-------------------+
| HandlerAdapter    |  <- Invokes the controller method
+-------------------+
        |
        v
+-------------------+
| OrderController   |  <- @RestController
| getOrder(id)      |    Calls service, returns OrderDTO
+-------------------+
        |
        v
+-------------------+
| OrderService      |  <- @Service, @Transactional
| findById(id)      |    Business logic
+-------------------+
        |
        v
+-------------------+
| OrderRepository   |  <- @Repository (Spring Data JPA)
| findById(id)      |    SQL: SELECT * FROM orders WHERE id=?
+-------------------+
        |  (reverse path)
        v
+-------------------+
| MessageConverter  |  <- Jackson: OrderDTO -> JSON
+-------------------+
        |
        v
HTTP Response: 200 OK + {"id":123,"status":"SHIPPED",...}

Exception path:
  Controller throws ResourceNotFoundException
        |
        v
  @ControllerAdvice / @ExceptionHandler
  Returns: 404 Not Found + {"error":"Order not found"}
```

---

## 4. Hibernate Entity States

```
                  [TRANSIENT]
                  new Order()
                  Not tracked by Hibernate
                  Not in DB
                        |
                  entityManager.persist(order)
                  OR session.save(order)
                        |
                        v
                  [PERSISTENT]
                  Within active Session
                  Tracked by Hibernate (1st level cache)
                  Any field changes auto-detected
                        |
              ---------+---------
              |                  |
   session.close()         entityManager.remove()
   or detach()             or session.delete()
              |                  |
              v                  v
         [DETACHED]          [REMOVED]
         Outside Session     Still in Session
         Changes NOT         Scheduled for DELETE
         tracked             on flush/commit
              |
    entityManager.merge()
              |
              v
         [PERSISTENT again]
         (merged copy tracked)

Practical:
  @Controller calls @Service
    Service method is @Transactional -> Session opens
    repo.findById(id) -> entity is PERSISTENT (tracked)
    entity.setStatus(SHIPPED) -> dirty (change detected)
    method returns -> Session flushes -> UPDATE SQL -> Session closes
    entity is now DETACHED (returned to controller)

    controller calls entity.getItems() -> LazyInitializationException!
    (Session is closed — lazy collection can't be loaded)
```

---

## 5. First-Level Cache (Session Cache) — Hit vs Miss Flow

```
Session scope (= one transaction):

First findById(1):
  Code: User user1 = em.find(User.class, 1L);
          |
          | Check 1st level cache for (User, 1)
          v
     CACHE MISS
          |
          v
     SQL: SELECT * FROM users WHERE id = 1
          |
          v
     Store in cache: (User, 1) -> user1 object
          |
          v
     Return user1

Second findById(1) in same Session:
  Code: User user2 = em.find(User.class, 1L);
          |
          | Check 1st level cache for (User, 1)
          v
     CACHE HIT
          |
          v
     Return same object reference (user1 == user2 -> true!)
     NO SQL issued

Cross-session (new Session = new cache):
  User userA = sessionA.find(User.class, 1L);  // SQL issued
  sessionA.close();  // cache cleared
  User userB = sessionB.find(User.class, 1L);  // SQL issued again
```

---

## 6. Spring AOP — Proxy Wrapping a Bean

```
Without AOP (direct call):
  Client --> OrderService.placeOrder()

With AOP (@Transactional):
  Spring creates a proxy at startup:

  Client --> [OrderServiceProxy]
                  |
                  | @Before: begin transaction
                  |
                  v
             [Real OrderService.placeOrder()]
                  |
                  | @After: commit transaction
                  |         (or rollback on exception)
                  v
  Client <-- response

  The proxy implements the same interface (or subclasses OrderService)
  The client never knows it's talking to a proxy

JDK Dynamic Proxy (interface-based):
  Client -> Proxy(implements IOrderService) -> Real OrderService

CGLIB Proxy (class-based, no interface needed):
  Client -> ProxySubclass(extends OrderService) -> Real OrderService

Self-invocation trap:
  class OrderService {
    @Transactional
    void placeOrder() {
      this.validateOrder();  // calls 'this' — bypasses proxy!
    }
    @Transactional(propagation=REQUIRES_NEW)
    void validateOrder() { ... }  // transaction NOT applied
  }
  Fix: inject self, or extract to separate bean
```

---

## 7. @Transactional Propagation — REQUIRED vs REQUIRES_NEW

```
PROPAGATION.REQUIRED (default):
  Join existing transaction if one exists; create new if not.

  ServiceA.methodA() @Transactional(REQUIRED) {
    serviceB.methodB();  // @Transactional(REQUIRED)
  }

  Timeline:
  [methodA starts TX1]
       |
       +-- [methodB joins TX1, no new TX]
       |         |
       |         +-- methodB finishes (no commit yet)
       |
  [methodA commits TX1]
  -- One commit for everything
  -- If methodB throws, methodA's TX is also rolled back

PROPAGATION.REQUIRES_NEW:
  Always create a new, independent transaction.
  Suspend the outer transaction while inner runs.

  ServiceA.methodA() @Transactional(REQUIRED) {
    serviceB.methodB();  // @Transactional(REQUIRES_NEW)
  }

  Timeline:
  [methodA starts TX1]
       |
       +-- [TX1 SUSPENDED]
           [methodB starts TX2]
                |
           [methodB commits TX2] <- committed independently!
           [TX1 RESUMED]
       |
  [methodA commits TX1 or rollbacks TX1]
  -- TX2 is already committed regardless of TX1's fate

  Use case: audit logging that must be saved even if the main transaction fails
  ServiceA saves order (TX1) -> fails -> TX1 rolled back
  But AuditService.log() (REQUIRES_NEW TX2) already committed -> audit preserved
```
