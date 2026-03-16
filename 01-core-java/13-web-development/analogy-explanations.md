# Web Development — Analogy Explanations

## Servlet Lifecycle

A Servlet is like a restaurant employee hired for the duration of the restaurant's operation. The restaurant manager (Servlet container, e.g., Tomcat) goes through three stages with this employee:

1. **init()** — Orientation day. The employee gets their uniform, learns the menu, and is briefed on procedures. This happens once when the restaurant opens (or when the first customer arrives, depending on `load-on-startup`). Expensive setup like loading configuration happens here.

2. **service()** — Actual work. Every time a customer (HTTP request) comes in, the employee handles their order. Many customers can arrive simultaneously, so the employee must handle concurrent requests correctly. This is called repeatedly, once per request.

3. **destroy()** — End of employment. When the restaurant closes (application shuts down), the employee is given time to wrap up, return their keys, and release any resources they were holding. Called once during shutdown.

---

## HTTP Request/Response

An HTTP request/response is like sending a letter and receiving a reply. Your letter (request) has:
- An address (URL) — where to send it.
- A subject line (HTTP method: GET, POST, PUT, DELETE) — what you want done.
- Headers (envelope annotations) — metadata like language preference, content type.
- A body (letter contents) — the actual data, for POST/PUT requests.

The reply (response) has:
- A status code (outcome notification) — "200 OK" is like writing "Done, here is what you asked for." "404 Not Found" is "I searched everywhere but couldn't find it." "500 Internal Server Error" is "I had a problem processing your request."
- Headers — metadata about the response.
- A body — the actual data sent back (JSON, HTML, etc.).

---

## Filter vs Interceptor

A **Filter** is like airport security before the terminal. Every single person (HTTP request) going anywhere in the airport must pass through security, regardless of their destination. Security can reject someone entirely (return 403), inspect their luggage (examine request body), or let them through. When they leave, security checks departures too. Filters are part of the Servlet container — they run before any Spring code.

An **Interceptor** is like the airline's gate agent inside the terminal. They only handle passengers going on that specific airline's flights (requests mapped to Spring controllers). They can see the ticket (controller method that will handle the request) and can check the passenger's boarding pass (authentication tokens, permissions). Interceptors are a Spring MVC concept — they have access to Spring context, handler information, and model attributes.

---

## Session Management

A **session** is like a coat check at a theater. When you arrive, the attendant gives you a ticket with a number (session ID, stored in a cookie). Your coat (your data: name, preferences, shopping cart) is stored in the back room under that number. Every time you interact with the theater staff (make an HTTP request), you show your ticket, and they retrieve your coat to verify who you are and what you were doing. When you leave (session expires or logout), the attendant discards your coat from the back room and invalidates your ticket.

The problem in distributed systems: if the theater has multiple coat check rooms in different buildings, the ticket from building A is meaningless in building B. This is the distributed session problem — solved either by always routing you to the same building (sticky sessions) or by sharing all coat check rooms over a network (session replication/Redis).

---

## Cookie

A cookie is like a loyalty card stamped by a coffee shop. On your first visit, the barista creates a card with your ID written on it and hands it to you. On every subsequent visit, you present the card, and the barista looks up your preferences and purchase history. You (the browser) hold the card; the shop (server) holds the data. The cookie is just the identifier — a small piece of text the browser sends automatically on every request to the same domain. The server uses that identifier to look up the session data it stores on the backend.

---

## CORS (Cross-Origin Resource Sharing)

CORS is like the rule that says an employee at Company A cannot walk into Company B's office and take documents without Company B's permission, even if both offices are in the same building.

On the web, a web page loaded from `https://myapp.com` is "inside" that origin. When JavaScript on that page tries to call `https://api.otherdomain.com`, the browser (like a security guard) asks: "Has `api.otherdomain.com` given permission for code from `myapp.com` to access it?" The browser sends a preflight `OPTIONS` request to check. If `api.otherdomain.com` responds with the right `Access-Control-Allow-Origin` header, the browser allows the request. If not, the browser blocks it — not the server, but the browser. CORS is a browser security mechanism, not a server firewall.

---

## REST Principles (Stateless, Uniform Interface)

REST is like a vending machine. Each interaction is complete and self-contained:

- **Stateless**: Every press of a button includes full context. The machine does not remember your previous selection. You press "A3", insert money, and get your item — all in one complete interaction. The machine has no memory of you between interactions. Every HTTP request must contain all the information the server needs to process it.

- **Uniform interface**: Every vending machine in the world works the same way — you press a code, insert money, collect item. There is no "VIP mode" or special protocol for returning customers. REST's uniform interface means every resource is accessed the same way (standard HTTP methods on URLs) regardless of what the resource is.

- **Resource-based**: The machine stocks items at fixed slots (A1, B3). Each slot (URL) represents a specific resource. You interact with the slot, not with the internal machinery.

---

## Content Negotiation

Content negotiation is like ordering at a restaurant that serves the same dish in multiple forms. You tell the waiter "I'd like the pasta, and I prefer gluten-free if you have it, but regular is fine too." The kitchen checks: do we have gluten-free pasta? Yes — serve that. If not — serve regular.

In HTTP, the client sends `Accept: application/json, application/xml;q=0.9` in the request header. The server checks if it can produce JSON (preferred) or XML (acceptable). If it can produce JSON, it responds with `Content-Type: application/json`. If it can only produce XML, it uses that. If it cannot produce either, it responds with `406 Not Acceptable`. This negotiation happens on every request transparently.

---

## HTTP Status Codes (2xx, 4xx, 5xx)

Status codes are like the outcome labels on a package delivery:

- **2xx (Success)**: "Delivered successfully." The request was received, understood, and processed.
  - 200: Normal delivery — here is what you asked for.
  - 201: New item delivered and placed in your mailbox — here is where it is located (Location header).
  - 204: Delivered, but nothing to show you (e.g., a DELETE that worked).

- **4xx (Client Error)**: "Return to sender — the problem is with how you addressed the package."
  - 400: Bad address format — your request is malformed.
  - 401: No ID shown — you must authenticate first.
  - 403: ID shown but access denied — authenticated but not authorized.
  - 404: Address does not exist — resource not found.

- **5xx (Server Error)**: "The delivery truck broke down — the problem is on our end."
  - 500: Truck crashed — unexpected server error.
  - 502: Wrong truck arrived at the warehouse — bad gateway (proxy got an invalid response).
  - 503: Truck is not running today — service temporarily unavailable.

---

## Idempotency

Idempotency is like a light switch. Pressing the "off" switch once or ten times produces the same outcome — the light is off. The number of times you perform the operation does not matter; the result is the same.

In HTTP terms: an operation is idempotent if performing it N times has the same effect as performing it once.
- `GET /users/1` — idempotent. Reading never changes data.
- `DELETE /users/1` — idempotent. First call deletes the user; subsequent calls get 404, but the state of the system (no user with ID 1) is the same.
- `PUT /users/1` with a full representation — idempotent. Replacing the same resource with the same data always results in the same state.
- `POST /users` — NOT idempotent. Each call creates a new user. Pressing the button 5 times creates 5 users.

Idempotency is critical for API reliability: clients can safely retry idempotent requests after a network timeout without causing duplicate processing.
