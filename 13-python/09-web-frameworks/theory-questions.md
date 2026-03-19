# Chapter 09: Web Frameworks — Theory Questions

1. Explain Django's MTV (Model-Template-View) pattern. How does it differ from MVC? What are the responsibilities of each layer?

2. How does Django's ORM generate SQL? Explain QuerySet laziness, `select_related` vs `prefetch_related`, and when each is appropriate for solving the N+1 query problem.

3. What is the Django migration system? What do `makemigrations` and `migrate` do? How does Django detect model changes, and what is a `squashmigrations`?

4. How do Django signals work? What are `post_save` and `pre_delete`? What are the dangers of using signals, and when should you use them vs. overriding `save()`?

5. Explain Django middleware. What methods does a middleware class expose (`__init__`, `__call__`, `process_view`, `process_exception`, `process_template_response`)? How is the middleware stack ordered?

6. What is the difference between Django class-based views (CBV) and function-based views (FBV)? What are the advantages of each? When would you choose a CBV over a FBV?

7. How does Django REST Framework (DRF) serialization work? What is the difference between `Serializer`, `ModelSerializer`, and `HyperlinkedModelSerializer`? How does validation work?

8. Explain Django authentication vs authorization. What is the `User` model, what is `AbstractUser` vs `AbstractBaseUser`, and when would you create a custom user model?

9. What is FastAPI's dependency injection system? How does `Depends()` work? What is the difference between a dependency that `yield`s vs one that returns?

10. How does FastAPI use Pydantic models? What is the difference between a request body model, a response model, and a query parameter model? How does FastAPI generate OpenAPI documentation automatically?

11. What is the difference between WSGI and ASGI? How does each interface between the web server and the Python application? Why does Django need `asgi.py` even for a "sync" application?

12. How does Uvicorn work as an ASGI server? What is the difference between running `uvicorn myapp:app` and using Gunicorn with the `uvicorn.workers.UvicornWorker`?

13. What is the N+1 query problem in an ORM? Give a concrete Django example and show both the broken and fixed versions. How would you detect this problem in production?

14. Explain Alembic migrations in the context of SQLAlchemy. How do `alembic revision --autogenerate` and `alembic upgrade head` work? What does "autogenerate" actually detect?

15. What is JWT (JSON Web Token) authentication? What are the three parts of a JWT, what is stored in the payload, and what is the difference between access tokens and refresh tokens?

16. What is OAuth2? How does the authorization code flow work? How does FastAPI's `OAuth2PasswordBearer` implement a simplified version of this?

17. What is SQLAlchemy connection pooling? What is `pool_size`, `max_overflow`, `pool_timeout`, and `pool_recycle`? What happens when all connections in the pool are checked out?

18. What are Django management commands? How do you create a custom one? What is `BaseCommand` vs `AppCommand`?

19. What is a background task in FastAPI (`BackgroundTasks`)? How does it differ from running a Celery task? What are its limitations?

20. What are Django settings best practices for different environments (dev/staging/prod)? What is `django-environ` or `django-split-settings`? Why should `SECRET_KEY` and database credentials never be in `settings.py`?
