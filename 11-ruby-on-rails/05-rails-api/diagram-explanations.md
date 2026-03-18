# Chapter 05 — Rails API: Diagram Explanations

---

## Diagram 1: JWT Authentication Flow

```
INITIAL LOGIN:

  Mobile App                 Rails API              Database
      │                          │                      │
      │  POST /api/v1/auth/sessions                      │
      │  { email, password }      │                      │
      │──────────────────────────▶│                      │
      │                          │  User.find_by(email)  │
      │                          │─────────────────────▶│
      │                          │◀─────────────────────│
      │                          │  authenticate(pw)     │
      │                          │                       │
      │                          │  JwtService.generate  │
      │                          │  ┌─────────────────────────┐
      │                          │  │ access_token (15 min)   │
      │                          │  │ refresh_token (30 days) │
      │                          │  └─────────────────────────┘
      │                          │  RefreshToken.create!│
      │                          │─────────────────────▶│
      │◀──────────────────────── │                      │
      │  { access_token,          │                      │
      │    refresh_token }        │                      │

─────────────────────────────────────────────────────────────────────

AUTHENTICATED REQUEST:

  Mobile App                 Rails API
      │                          │
      │  GET /api/v1/users/me     │
      │  Authorization: Bearer eyJ...   │
      │──────────────────────────▶│
      │                          │  JwtService.decode_access(token)
      │                          │  → verify signature
      │                          │  → check expiry
      │                          │  → extract user_id
      │                          │  → User.find(user_id)
      │                          │
      │◀──────────────────────── │
      │  200 { data: { user } }   │

─────────────────────────────────────────────────────────────────────

TOKEN REFRESH:

  Mobile App                 Rails API              Database
      │                          │                      │
      │  [access token expires]   │                      │
      │  POST /api/v1/auth/token_refreshes               │
      │  { refresh_token: "eyJ..." }                     │
      │──────────────────────────▶│                      │
      │                          │  decode_refresh(token)│
      │                          │  RefreshToken.find!   │
      │                          │─────────────────────▶│
      │                          │  token_record.revoke! │ ← rotation
      │                          │─────────────────────▶│
      │                          │  generate_tokens(user)│
      │                          │  RefreshToken.create! │
      │                          │─────────────────────▶│
      │◀──────────────────────── │                      │
      │  { new access_token,      │                      │
      │    new refresh_token }    │                      │
```

---

## Diagram 2: API Serialization Pipeline

```
Controller Action
       │
       ▼
  @users = User.includes(:posts, :profile).all
       │
       ▼
  ┌────────────────────────────────────────┐
  │          SERIALIZER                    │
  │  UserBlueprint.render(@users,          │
  │    view: :detailed)                    │
  │                                        │
  │  For each user:                        │
  │    Extract allowed fields              │
  │    Compute derived fields              │
  │    Include associations                │
  │    Format dates/values                 │
  │    Apply view-specific logic           │
  └────────────────────────────────────────┘
       │
       ▼
  Ruby Hash / Array
       │
       ▼
  JSON encoder (Oj or Ruby's JSON)
       │
       ▼
  JSON String (response body)


WHAT SERIALIZER PROTECTS AGAINST:
  User model attributes:
    id ✓ (public)
    name ✓ (public)
    email ✓ (public)
    password_digest ✗ (never exposed)
    reset_token ✗ (never exposed)
    stripe_id ✗ (never in mobile/public view)
    admin_notes ✗ (only admin view)
    internal_score ✗ (business logic, not public)
```

---

## Diagram 3: CORS Preflight Flow

```
Browser (https://myapp.com) makes POST request to https://api.myapp.com/api/v1/users

STEP 1: Browser sends OPTIONS preflight (automatically):
  OPTIONS /api/v1/users HTTP/1.1
  Host: api.myapp.com
  Origin: https://myapp.com
  Access-Control-Request-Method: POST
  Access-Control-Request-Headers: Authorization, Content-Type

STEP 2: API responds (rack-cors handles this):
  HTTP/1.1 200 OK
  Access-Control-Allow-Origin: https://myapp.com     ← Origin allowed
  Access-Control-Allow-Methods: GET, POST, DELETE     ← Methods allowed
  Access-Control-Allow-Headers: Authorization, Content-Type  ← Headers allowed
  Access-Control-Allow-Credentials: true              ← Cookies allowed
  Access-Control-Max-Age: 7200                        ← Cache preflight 2 hours

STEP 3: Browser sends actual POST request:
  POST /api/v1/users HTTP/1.1
  Host: api.myapp.com
  Origin: https://myapp.com
  Authorization: Bearer eyJ...
  Content-Type: application/json
  { "user": { "name": "Alice" } }

STEP 4: API responds with CORS headers on the actual response too:
  HTTP/1.1 201 Created
  Access-Control-Allow-Origin: https://myapp.com
  Access-Control-Allow-Credentials: true
  { "data": { "id": "1", "type": "user" } }

STEP 5: Browser allows JavaScript to read the response ✓

─────────────────────────────────────────────────────────────────────

CORS FAILURE SCENARIOS:

  ✗ Wrong origin:
    API: Access-Control-Allow-Origin: https://myapp.com
    Request: Origin: https://malicious.com
    → Browser blocks the response (CORS error)

  ✗ Wildcard with credentials:
    API: Access-Control-Allow-Origin: *
    Request: credentials: 'include'
    → Browser rejects: can't use wildcard with credentials

  ✗ Missing OPTIONS route:
    Rack::Cors not before router → OPTIONS hits 404 → preflight fails
```

---

## Diagram 4: API Error Response Format

```
CONSISTENT ERROR ENVELOPE (all errors look the same):

  HTTP Status Code (machine-readable)
       │
       ▼
  Response Body:
  {
    "error": {
      "code": "validation_failed",      ← machine-readable code
      "message": "Validation failed",   ← human-readable summary
      "details": {                      ← field-level details (optional)
        "email": ["is invalid", "has already been taken"],
        "name": ["can't be blank"]
      },
      "request_id": "req_abc123",       ← for debugging/support
      "timestamp": "2024-01-15T12:00:00Z"
    }
  }

─────────────────────────────────────────────────────────────────────

HTTP STATUS → ERROR CODE MAPPING:

  400 Bad Request
    code: "bad_request"
    Use: malformed JSON, missing required params, invalid param format

  401 Unauthorized
    code: "unauthorized" or "token_expired"
    Use: no auth, invalid token, expired token

  403 Forbidden
    code: "forbidden"
    Use: authenticated but not authorized for this action/resource

  404 Not Found
    code: "not_found"
    Use: record doesn't exist, or exists but not accessible to user

  409 Conflict
    code: "conflict" or "duplicate_resource"
    Use: unique constraint violation, concurrent modification

  422 Unprocessable Entity
    code: "validation_failed"
    Use: record fails validations, business logic violations

  429 Too Many Requests
    code: "rate_limit_exceeded"
    Use: rate limit hit
    Headers: Retry-After, X-RateLimit-Remaining, X-RateLimit-Reset

  500 Internal Server Error
    code: "server_error"
    Use: unexpected exceptions (never expose stack traces in production)
```

---

## Diagram 5: Pagination Strategies

```
OFFSET PAGINATION:
  GET /api/v1/posts?page=3&per_page=10

  Database: SELECT * FROM posts ORDER BY created_at DESC LIMIT 10 OFFSET 20

  Response:
  {
    "data": [...10 posts...],
    "meta": {
      "current_page": 3,
      "total_pages": 15,
      "total_count": 147,
      "per_page": 10
    },
    "links": {
      "prev": "/api/v1/posts?page=2&per_page=10",
      "next": "/api/v1/posts?page=4&per_page=10"
    }
  }

  Problem: New post added → page shifts
  Page 1 at time T:  [post_10, post_9, post_8, ...]
  Page 2 at time T:  [post_5, post_4, post_3, ...]
  New post added
  Page 2 at time T+1: [post_6, post_5, post_4, ...]  ← post_6 duplicated!

─────────────────────────────────────────────────────────────────────

CURSOR PAGINATION:
  GET /api/v1/posts?cursor=eyJsYXN0X2lkIjo1fQ&per_page=10

  Decoded cursor: { "last_id": 5 }
  Database: SELECT * FROM posts WHERE id < 5 ORDER BY id DESC LIMIT 11

  Response:
  {
    "data": [...10 posts (IDs 4, 3, 2, ...)...],
    "meta": { "has_more": true, "per_page": 10 },
    "cursors": {
      "next": "eyJsYXN0X2lkIjoxfQ"  ← encoded { "last_id": 1 }
    }
  }

  New post added: doesn't affect cursor — cursor tracks by ID, not position
  No duplicate items, no skipped items

─────────────────────────────────────────────────────────────────────

COMPARISON:
  ┌─────────────────┬──────────────────┬──────────────────┐
  │ Feature         │ Offset           │ Cursor           │
  ├─────────────────┼──────────────────┼──────────────────┤
  │ Jump to page N  │ ✓ Yes            │ ✗ No             │
  │ Stable results  │ ✗ Can duplicate  │ ✓ Stable         │
  │ DB performance  │ ✗ OFFSET slow    │ ✓ WHERE id < X   │
  │ Total count     │ ✓ Easy           │ ✗ Harder         │
  │ Real-time feeds │ ✗ Problems       │ ✓ Works well     │
  └─────────────────┴──────────────────┴──────────────────┘
```

