# N+1 Problem — Diagram Explanations

---

## Diagram 1: N+1 vs JOIN FETCH Query Count

```
  N+1 Problem (100 orders with lazy items):

  1. SELECT * FROM orders                          ← 1 query
  2. SELECT * FROM order_items WHERE order_id=1    ← query for order 1
  3. SELECT * FROM order_items WHERE order_id=2    ← query for order 2
  4. SELECT * FROM order_items WHERE order_id=3    ← query for order 3
  ...
  101. SELECT * FROM order_items WHERE order_id=100 ← query for order 100
  
  TOTAL: 101 queries

  WITH JOIN FETCH:

  SELECT DISTINCT o.*, i.*
  FROM orders o
  JOIN order_items i ON i.order_id = o.id
  
  TOTAL: 1 query ✓
```

---

## Diagram 2: @BatchSize Loading

```
  100 Orders, @BatchSize(25) on items:

  SELECT * FROM orders        ← 1 query, returns [O1..O100]

  When O1.getItems() accessed:
  SELECT * FROM order_items WHERE order_id IN (1,2,...,25)   ← batch 1
  SELECT * FROM order_items WHERE order_id IN (26,27,...,50)  ← batch 2
  SELECT * FROM order_items WHERE order_id IN (51,52,...,75)  ← batch 3
  SELECT * FROM order_items WHERE order_id IN (76,77,...,100) ← batch 4

  TOTAL: 1 + 4 = 5 queries (vs 101 without @BatchSize)
```
