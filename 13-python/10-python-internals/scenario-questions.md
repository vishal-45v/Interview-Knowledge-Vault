# Chapter 10: Python Internals — Scenario Questions

1. A colleague reports that their Python program has a memory leak. They have no cyclic references visible in their code. The `gc` module reports it found and collected 0 objects. Yet memory grows over time. What are the possible causes, and how would you investigate? Consider CPython internals, C extension allocations, and `__del__` methods.

2. You are debugging a production crash. The traceback shows a `RecursionError: maximum recursion depth exceeded`. You investigate and find the function is not infinitely recursive — it processes deeply nested JSON (200 levels deep). What is Python's default recursion limit, how do you change it, and what is the real fix for processing arbitrarily deep nested structures?

3. A developer on your team writes this code and is confused why it "doesn't work":
   ```python
   import threading
   counter = 0
   def increment():
       global counter
       for _ in range(100000):
           counter += 1
   threads = [threading.Thread(target=increment) for _ in range(10)]
   for t in threads: t.start()
   for t in threads: t.join()
   print(counter)  # Expected: 1,000,000. Actual: ~700,000
   ```
   Explain exactly why this is a race condition despite the GIL, including the bytecode-level reason. How do you fix it?

4. You are writing a plugin system where users can install plugins as Python packages. You want to load a plugin from a file path at runtime, without it being on `sys.path`. Walk through using `importlib` to load the module dynamically. What are the security implications?

5. A Python web service has been running for 3 days and its response latency has occasional spikes of 200-500ms. Everything else looks normal. A colleague suggests it might be GC pauses from the cyclic garbage collector. How do you verify this hypothesis? What can you tune in the `gc` module to reduce pauses, and what is the tradeoff?

6. You need to implement a caching proxy class that intercepts all attribute access on a wrapped object, logs which attributes are accessed, and returns the real attribute value. Show how you would use the descriptor protocol and/or `__getattr__`/`__getattribute__` to implement this. What is the difference between `__getattr__` and `__getattribute__`?

7. A team member is trying to use Python 3.10's structural pattern matching to parse API response dicts. They write a `match` statement that always hits the `case _` default even when the response matches an earlier case. What are the common mistakes with `match`/`case` for dict patterns, and how do you fix them?

8. You are implementing a custom import hook to load `.yaml` files as Python modules (so `import config` loads `config.yaml`). Walk through implementing a `MetaPathFinder` and `Loader` using `importlib`. What is the risk of this approach?

9. Two modules `A` and `B` have a circular import: `A` imports `B` at the top level, and `B` imports `A` at the top level. The code works sometimes and fails with `ImportError: cannot import name 'X' from partially initialized module 'A'` other times. Explain CPython's module initialization order and why the error is intermittent. How do you fix it without restructuring all code?

10. You are analyzing a CPython performance regression between Python 3.11 and 3.12. Using `dis`, you notice that a hot function has different bytecode between versions. Python 3.11 uses `CALL_FUNCTION` and Python 3.12 uses `CALL`. What changed in CPython 3.11/3.12's bytecode? What is the "specializing adaptive interpreter" and how does it optimize hot paths?
