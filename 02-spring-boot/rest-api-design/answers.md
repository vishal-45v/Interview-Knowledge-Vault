# REST API Design — Structured Answers

---

## Q1: REST Constraints

REST has 6 architectural constraints: Client-Server, Stateless, Cacheable, Uniform Interface, Layered System, and Code on Demand (optional).

Key constraint for interviews: **Stateless** — each request must contain all information needed. Server holds no client session state. This enables horizontal scaling.

---

## Q2: PUT vs PATCH

PUT performs a full replacement of the resource. All fields must be provided; missing fields are set to null. PUT is idempotent.

PATCH performs a partial update with only the changed fields. Other fields remain unchanged. PATCH idempotency depends on the operation semantics.

---

## Q3: HTTP Status Codes Reference

| Code | Name | When to Use |
|------|------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST that creates resource |
| 202 | Accepted | Async operation started |
| 204 | No Content | Successful DELETE, no body |
| 400 | Bad Request | Validation errors, malformed request |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but no permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate creation, state conflict |
| 422 | Unprocessable Entity | Semantically invalid request |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Maintenance, overload |

---

## Q4: Idempotency

An operation is idempotent if calling it multiple times produces the same result as calling it once.

- GET, PUT, DELETE: Idempotent
- POST: NOT idempotent (creates new resource each time)
- PATCH: Depends on operation

Idempotency is critical for retry logic. Clients can safely retry idempotent operations after network failures.

---

## Q5: Difference Between 401 and 403

- 401 Unauthorized: No or invalid authentication credentials. Client should authenticate.
- 403 Forbidden: Valid credentials, but insufficient permissions. Client needs elevated access.

---

## Q6: Best Practices for REST URL Design

Use plural nouns: /orders, /products, /users
Use sub-resources for relationships: /orders/123/items
Use POST for non-CRUD actions: POST /orders/123/cancel
Keep URLs lowercase with hyphens: /product-categories
Version in the URL path: /api/v1/products
