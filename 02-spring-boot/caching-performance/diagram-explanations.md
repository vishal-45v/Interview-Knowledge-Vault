# Caching & Performance — Diagram Explanations

---

## Diagram 1: Cache-Aside Pattern

```
  First access (cold):
  App ──► check cache ──► MISS
    └──► load from DB
    └──► store in cache
    └──► return data

  Subsequent access (warm):
  App ──► check cache ──► HIT ──► return data
         (no DB call)

  After update:
  App ──► update DB
    └──► evict from cache  ← important!
         (next access re-loads from DB)
```

---

## Diagram 2: Cache TTL and Expiry

```
  Time →  0    5min   10min  15min  20min  25min  30min
  
  Entry added at t=0:
  ┌──────────────────────────────────────────┐
  │     Cache entry (TTL = 30 min)           │ → EXPIRED
  └──────────────────────────────────────────┘

  Access at t=5:  HIT ✓ (15 min remaining)
  Access at t=25: HIT ✓ (5 min remaining)
  Access at t=31: MISS ✗ → load from DB → new entry added

  Stampede risk at t=30:
  Many users hit at t=30 → all MISS → all hit DB simultaneously
```

---

## Diagram 3: Local vs Distributed Cache

```
  Single instance (local cache OK):
  [App Instance] ──► [Caffeine Cache] ──► [DB]
  All requests hit same cache ✓

  Multiple instances (need distributed cache):
  [Instance 1] ──► [Caffeine L1] ──► \
  [Instance 2] ──► [Caffeine L1] ──► ─► [DB]  ← Inconsistency!
  [Instance 3] ──► [Caffeine L1] ──► /
  Each instance has different cache state ✗

  Correct with Redis:
  [Instance 1] ──► \
  [Instance 2] ──► ─► [Redis] ──► [DB]
  [Instance 3] ──► /
  All instances share same cache ✓
```
