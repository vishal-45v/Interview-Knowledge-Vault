# Lazy vs Eager Loading — Diagram Explanations

---

## Diagram 1: Lazy vs Eager Loading Comparison

```
  LAZY Loading (@OneToMany default):

  Session.find(Order.class, 1L):
  SQL: SELECT * FROM orders WHERE id=1  ← Only order row
  
  order.getItems():
  SQL: SELECT * FROM order_items WHERE order_id=1  ← Triggered on access
  
  ┌─────────────────────────────────────────────────────────┐
  │  +: No unnecessary data loaded                           │
  │  +: Faster initial load                                  │
  │  -: LazyInitializationException if accessed outside TX   │
  │  -: N+1 problem in loops                                 │
  └─────────────────────────────────────────────────────────┘

  EAGER Loading (@ManyToOne default):

  Session.find(Order.class, 1L):
  SQL: SELECT o.*, c.* FROM orders o JOIN customers c ON c.id = o.customer_id WHERE o.id=1
       ← Customer always loaded with order
  
  ┌─────────────────────────────────────────────────────────┐
  │  +: No lazy loading issues                               │
  │  +: Predictable data loading                             │
  │  -: Always loads, even when not needed                   │
  │  -: Performance cost for bulk queries                    │
  └─────────────────────────────────────────────────────────┘
```

---

## Diagram 2: OSIV — Open Session in View

```
  With OSIV (spring.jpa.open-in-view=true):
  ┌──────────────────────────────────────────────────────────────┐
  │  HTTP Request lifecycle                                      │
  │                                                              │
  │  ← Session opened, Connection from pool ─────────────────► │
  │  [Interceptor]  [Controller]  [@Transactional Service]       │
  │                     │              │                         │
  │                     │ TX begins ───┘                         │
  │                     │ DB queries                             │
  │                     │ TX ends ─────                          │
  │                     │              │                         │
  │  Lazy loading still │ works here! ◄┘ (Session still open)   │
  │  JSON serialization │ can trigger lazy loads                 │
  │  ← Connection held for entire request duration ───────────► │
  └──────────────────────────────────────────────────────────────┘
  COST: Connection held during JSON serialization, view rendering
```
