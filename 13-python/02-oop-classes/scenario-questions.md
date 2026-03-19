# Chapter 02 — OOP & Classes: Scenario Questions

---

1. You are building a payment processing system with multiple payment providers
   (Stripe, PayPal, Square). Each provider has different APIs but must support
   `charge(amount)`, `refund(transaction_id)`, and `get_balance()`. How would you
   design this using Python's OOP features? Show the class hierarchy and explain
   how you would enforce the interface contract.

2. Your team is building a configuration system where settings can come from
   environment variables, a config file, or command-line arguments. When a setting
   is requested, it should be found in whichever source has it, in priority order.
   A coworker suggests using multiple inheritance with one class per source. How would
   you design this and what MRO issues might arise? Show the MRO diagram.

3. You are designing a `Vector2D` class for a physics engine. It needs to support
   `+`, `-`, `*`, `==`, `abs()`, `str()` representation, and be usable as a
   dictionary key. What dunder methods do you implement, and what constraints does
   "usable as a dict key" impose on your design?

4. You have a class `DatabaseConnection` that holds an open socket. A junior developer
   notices that objects are not being cleaned up properly, causing connection leaks.
   How would you implement `__enter__` and `__exit__` to make it a context manager?
   Also show how to implement `__del__` as a fallback and explain its limitations.

5. You are reviewing code and find 15 data-carrying classes that each have `__init__`,
   `__repr__`, `__eq__`, and `__hash__` — all written by hand with identical patterns.
   Suggest a refactor using a Python standard library feature. Show before and after,
   and explain what the feature generates automatically and what its limitations are.

6. You are building a caching decorator that needs to preserve the original function's
   name, docstring, and signature. A junior developer writes a simple wrapper but
   `help()` and introspection tools show the wrapper's name instead of the original.
   How do you fix this, and what does `functools.wraps` do internally?

7. Your team uses a `User` class that is passed around in many places. A new requirement
   says `User` objects must be immutable and hashable so they can be stored in sets and
   used as dict keys. What changes would you make to the existing class? Show at least
   two approaches.

8. You are implementing a plugin system where third-party code can register new
   "handler" classes. You want to ensure every registered class implements specific
   methods (`process`, `validate`) and raises a clear error at class definition time
   (not at call time) if it does not. How would you implement this?

9. You need to create a `Singleton` class in Python — only one instance should ever
   exist. Show at least two implementations: one using `__new__` and one using a
   metaclass. Discuss the thread-safety implications of each approach.

10. A performance-sensitive section of code creates millions of small objects of the
    same class. Memory profiling shows these objects consume far more memory than
    expected. A `__dict__` per instance is the culprit. How do you fix this, and
    what is the trade-off in terms of flexibility?
