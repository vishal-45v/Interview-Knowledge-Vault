# REST API Design — Diagram Explanations

---

## Diagram 1: Richardson Maturity Model

```
  Level 0: The Swamp of POX (Plain Old XML/JSON)
    All requests to same URL, same method
    POST /api  {"action": "getProduct", "id": 123}

  Level 1: Resources
    Separate URLs per resource
    GET /products/123
    POST /products

  Level 2: HTTP Verbs
    Proper HTTP methods: GET, POST, PUT, DELETE
    Proper status codes: 200, 201, 404, etc.
    ← Most REST APIs are at this level

  Level 3: Hypermedia (HATEOAS)
    Responses include links to next actions
    {
      "id": 123,
      "_links": {
        "self": "/products/123",
        "reviews": "/products/123/reviews",
        "purchase": "/cart/add/123"
      }
    }
    ← True REST by Fielding's definition
```

---

## Diagram 2: API Versioning Strategies Comparison

```
  Strategy          | URL                           | Header
  ──────────────────┼───────────────────────────────┼─────────────────────
  URL Path          | GET /api/v2/products/1        | (none)
  ──────────────────┼───────────────────────────────┼─────────────────────
  Query Parameter   | GET /api/products/1?version=2 | (none)
  ──────────────────┼───────────────────────────────┼─────────────────────
  Request Header    | GET /api/products/1           | API-Version: 2
  ──────────────────┼───────────────────────────────┼─────────────────────
  Accept Header     | GET /api/products/1           | Accept: application/
  (Content Neg.)    |                               | vnd.co.v2+json

  Recommendation: URL path versioning for most cases
  - Most visible and understandable
  - Easily cacheable by CDN/proxies
  - Works with all clients (browsers, curl, etc.)
  - Easy to test in browser
```

---

## Diagram 3: Pagination Response Structure

```
  GET /products?page=2&size=10&sort=price,asc

  Response:
  {
    "content": [
      {"id": 21, "name": "Product 21", "price": 45.00},
      ...
      {"id": 30, "name": "Product 30", "price": 67.00}
    ],
    "pageable": {
      "pageNumber": 2,       ← current page (0-indexed)
      "pageSize": 10,
      "sort": {"sorted": true, "orders": [{"property": "price", "direction": "ASC"}]}
    },
    "totalElements": 523,    ← total matching records
    "totalPages": 53,
    "first": false,
    "last": false,
    "numberOfElements": 10,  ← elements in THIS page
    "empty": false
  }
```

---

## Diagram 4: Idempotency Pattern for Safe Retries

```
  Client generates: idempotency-key = UUID.randomUUID()

  Attempt 1:
  Client ──POST /payments, Idempotency-Key: abc123──► Server
                                                         │
                                                         ├─ Process payment
                                                         ├─ Charge: $100
                                                         ├─ Store result for "abc123"
                                                         └─► 201 Created {paymentId: 456}

  [Network failure — client didn't get response]

  Attempt 2 (retry with SAME key):
  Client ──POST /payments, Idempotency-Key: abc123──► Server
                                                         │
                                                         ├─ Lookup "abc123"
                                                         ├─ Found! Return cached result
                                                         └─► 201 Created {paymentId: 456}
                                                             (NO duplicate charge!)

  Attempt 3 (new operation with NEW key):
  Client ──POST /payments, Idempotency-Key: xyz999──► Server
                                                         │
                                                         ├─ "xyz999" not found
                                                         ├─ Process payment
                                                         └─► 201 Created {paymentId: 789}
                                                             (NEW payment, intentional)
```
