# Chapter 09: Web Frameworks — Scenario Questions

1. Your Django application's `/api/orders/` endpoint takes 3 seconds to respond. Django Debug Toolbar shows 847 SQL queries for a single request that returns 50 orders. Each order has a customer and 5 line items. Walk through diagnosing and fixing the N+1 problem. Show the QuerySet before and after your fix, and explain what SQL change your fix causes.

2. You are building a FastAPI microservice for user authentication. It needs to: validate a JWT token on every request, look up the user from the database, and inject the user object into every route handler that needs it. Show the complete FastAPI dependency injection setup for this, including the `get_current_user` dependency, how it uses `Depends()`, and how you would override it in tests.

3. A Django REST Framework API returns sensitive user data. Some fields (like `social_security_number`) should be writable on create but never returned in any response. Other fields are read-only. The API must also validate that a user's email is unique. Show the complete DRF serializer for this scenario.

4. Your team is deploying a FastAPI application and debates between: (A) Uvicorn directly with multiple workers, (B) Gunicorn with UvicornWorker, (C) nginx + Gunicorn + UvicornWorker. Walk through the tradeoffs of each deployment option for a service that handles 500 req/sec with some endpoints doing async database calls.

5. You are adding a feature to a Django application where, whenever an order is marked as "shipped," an email must be sent and a webhook must be fired to a third-party service. A junior developer suggests using `post_save` signals. A senior developer says "use a service layer instead." What is the case for each approach, and what would you recommend?

6. A FastAPI endpoint does three independent database queries before returning a response. Currently they are sequential. Each query takes 80ms. Total latency is ~250ms. How do you parallelize them using `asyncio.gather`? Show the before and after code. What are the gotchas with async SQLAlchemy?

7. Your Django application has a `User` model that was created with the default `auth.User`. Now, 2 years into production, the CTO wants to add a `phone_number` field to User and use email as the login field instead of username. What is the problem, what is the correct way to handle custom user models, and how do you migrate existing data?

8. You are building a public API with FastAPI that serves both a web frontend and third-party developers. The frontend uses session cookies, third-party developers use API keys, and mobile apps use JWT. How do you design a flexible authentication system that supports all three, using FastAPI's dependency injection and `Security()` declarations?

9. An Alembic autogenerate migration is producing a migration that drops and recreates a table even though you only added a new nullable column. What could cause this, and how do you debug and fix it? What parts of a schema does Alembic NOT detect automatically?

10. Your company runs a Django application. A new data privacy regulation requires that every API response be logged with the user ID, endpoint, response status code, and timestamp for audit purposes. You must not touch every individual view. How would you implement this using Django middleware, and what are the edge cases you must handle (streaming responses, exception cases, admin URLs)?
