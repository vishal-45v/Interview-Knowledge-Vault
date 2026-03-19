# REST API Design — Trap Questions

---

## Trap 1: DELETE Should Be Idempotent — But What Status Code for Missing Resource?

**Question:** A client sends DELETE /orders/123 twice. The first call succeeds (order deleted). What should the second call return — 404 or 204?

**Answer:** This is debated. REST spec says DELETE is idempotent, meaning the server state is the same after multiple calls (order doesn't exist). Some argue:
- 204 No Content: The desired end state (order doesn't exist) is achieved
- 404 Not Found: The specific resource targeted doesn't exist

The most pragmatic approach: return 404 for the second call. The idempotency principle concerns state (the order is gone), not necessarily the response code. Most real APIs return 404 when trying to delete a non-existent resource.

---

## Trap 2: POST Is Not Idempotent, But Can Be Made Idempotent

**Question:** How do you make a POST request idempotent?

**Answer:** Use an Idempotency-Key header. The client generates a unique key (UUID) per logical operation and includes it in the header. The server stores the result keyed by this value. On retries with the same key, the server returns the cached result instead of processing again.

This is used by payment processors (Stripe, PayPal) to prevent duplicate charges on network retries.

---

## Trap 3: 200 OK vs 204 No Content for Updates

**Question:** Your PUT /orders/123 endpoint updates an order. Should it return 200 OK with the updated order, or 204 No Content?

**Answer:** Both are valid. The choice depends on your API contract:
- 200 with body: Client gets confirmation of the actual saved state (recommended — what was actually saved may differ from what was sent after server-side transformations)
- 204 No Content: Bandwidth efficient when client doesn't need the response body

Best practice: Return 200 with the updated resource, so clients always know the actual server state.

---

## Trap 4: HATEOAS — Is It Required for REST?

**Question:** Is HATEOAS required for a "true" REST API?

**Answer:** By Roy Fielding's definition, yes — HATEOAS (Hypermedia as the Engine of Application State) is one of the constraints of REST. Without it, technically it's a "REST-like" or "REST-ish" API, not truly RESTful.

In practice, very few production APIs implement HATEOAS. Most API contracts are communicated via documentation rather than hypermedia links. This is widely accepted as a pragmatic tradeoff.

---

## Trap 5: Partial Response with Field Selection

**Question:** A mobile client only needs `id` and `name` from a large product object. How do you implement this efficiently?

**Answer:** Implement partial response with a `fields` query parameter:

```
GET /products/123?fields=id,name
→ Returns: {"id": 123, "name": "Widget"}
```

This reduces bandwidth and client parsing time. Implementation requires filtering the response object before serialization (can use Jackson's `@JsonView` or manual filtering with ObjectMapper).

This pattern is used by Google APIs and is called "partial response" or "field masks."
