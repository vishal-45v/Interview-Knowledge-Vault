# Hibernate & JPA Internals — Diagram Explanations

---

## Diagram 1: JPA Entity States

```
                    new Object()
                         │
                         ▼
              ┌─────────────────────┐
              │      TRANSIENT       │
              │  (not in DB,        │
              │   not tracked)      │
              └──────────┬──────────┘
                         │ persist()
                         ▼
              ┌─────────────────────┐
              │     PERSISTENT      │◄───── merge()
              │  (tracked by EM,   │
              │   dirty checking)  │
              └─────┬──────────────┘
              detach│       │remove()
              close │       ▼
                    │ ┌─────────────────────┐
                    │ │      REMOVED         │
                    │ │  (marked for delete) │
                    │ └─────────────────────┘
                    ▼
              ┌─────────────────────┐
              │     DETACHED        │
              │  (was managed,      │
              │   not tracked now)  │
              └─────────────────────┘
```

---

## Diagram 2: Dirty Checking Mechanism

```
  Transaction begins:
  entityManager.find(Product.class, 1L)
         │
         ▼
  SQL: SELECT * FROM products WHERE id=1
  Result: {id=1, name="Widget", price=9.99, stock=100}
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │  Persistence Context                          │
  │                                               │
  │  entity:   Product{id=1, name="Widget",       │
  │                    price=10.99, stock=95}     │ ← modified!
  │                                               │
  │  snapshot: {id=1, name="Widget",              │
  │              price=9.99, stock=100}           │ ← original state
  └──────────────────────────────────────────────┘
         │
         ▼ flush() (before commit)
  Compare entity to snapshot:
    price changed: 9.99 → 10.99 ✓ DIRTY
    stock changed: 100 → 95     ✓ DIRTY
    name unchanged              ✗ clean
         │
         ▼
  UPDATE products SET price=10.99, stock=95 WHERE id=1
```

---

## Diagram 3: L1 vs L2 Cache Hierarchy

```
  Request 1 (TX 1):
  ┌──────────────────────────────────────────────────────┐
  │  EntityManager (L1 Cache)                            │
  │  findById(1) → SQL → stores Product#1 in L1          │
  │  findById(1) → NO SQL → from L1 ✓                    │
  │  findById(2) → SQL → stores Product#2 in L1          │
  │  TX ends → L1 cleared                                │
  └──────────────────────────────────────────────────────┘
         │ stores to
         ▼
  ┌──────────────────────────────────────────────────────┐
  │  SessionFactory (L2 Cache) — shared across TXs       │
  │  Product#1 cached (if @Cache annotation present)     │
  │  Product#2 cached                                    │
  └──────────────────────────────────────────────────────┘

  Request 2 (TX 2):
  ┌──────────────────────────────────────────────────────┐
  │  EntityManager (L1 Cache) — fresh, empty             │
  │  findById(1) → checks L2 → found! → into L1 → return │
  │  NO SQL ✓                                            │
  │  findById(3) → checks L2 → miss → SQL → store L2    │
  └──────────────────────────────────────────────────────┘
```

---

## Diagram 4: CascadeType Effects

```
  Parent: Order          @OneToMany(cascade = ???)
  Children: OrderItem

  CASCADE_PERSIST:
    order = new Order();
    order.getItems().add(new OrderItem(...));
    entityManager.persist(order);
    → OrderItem persisted automatically ✓

  CASCADE_REMOVE:
    entityManager.remove(order);
    → All OrderItems removed automatically ✓ (or dangerous!)

  CASCADE_ALL + orphanRemoval=true:
    order.getItems().remove(item);
    → item deleted from DB (orphan) ✓

  NO CASCADE:
    entityManager.persist(order);
    → OrderItems NOT persisted — must persist each manually
    → Usually results in TransientPropertyValueException
```
