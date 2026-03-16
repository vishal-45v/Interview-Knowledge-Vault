# Lazy vs Eager Loading — Architecture Notes

---

## How Hibernate Implements Lazy Loading

Hibernate implements lazy loading using bytecode proxies:

```
@OneToMany(fetch = LAZY)
List<OrderItem> items;

When Order is loaded from DB:
  items = [Hibernate Proxy / PersistentBag]
  → Proxy knows: "load from order_items WHERE order_id = ?"
  → SQL NOT executed yet

When items.size() or items.get(0) called:
  → Proxy checks: is Session open?
    YES → execute SQL, populate list, return result
    NO  → LazyInitializationException
```

---

## OSIV Request Lifecycle

```
Without OSIV (spring.jpa.open-in-view=false):
  Request arrives
  [Filter/Interceptor]
  Controller.method()
    └── service.method()  ← @Transactional — TX begins
          └── DB queries
          └── return entity  ← TX ends here
    └── entity.getLazyCollection()  ← LazyInitializationException!

With OSIV (spring.jpa.open-in-view=true):
  Request arrives
  [OpenSessionInViewInterceptor opens Session + binds to thread]
  Controller.method()
    └── service.method()  ← @Transactional — TX begins
          └── DB queries
          └── return entity  ← TX ends (but Session stays open!)
    └── entity.getLazyCollection()  ← Works! Session is open
        └── SQL executed (outside TX, auto-commit)
  [OpenSessionInViewInterceptor closes Session]

OSIV Cost: DB connection held for entire request duration
```

---

## Fetch Strategy Decision Matrix

```
Scenario                           | Strategy
────────────────────────────────────────────────────────────
Always need association            | EAGER (rare — mainly for @ManyToOne in perf-critical)
Sometimes need it                  | LAZY + load explicitly when needed
Paginated list of entities         | LAZY + @BatchSize on collections
Detail page (always full data)     | LAZY + JOIN FETCH or @EntityGraph
Report query (DTO projection)      | Neither — use JPQL DTO constructor
Count queries                      | LAZY — avoid loading unnecessary associations
```
