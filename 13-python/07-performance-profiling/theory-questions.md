# Chapter 07: Performance & Profiling — Theory Questions

1. What is `cProfile` and how does it differ from `profile`? What does each column in `cProfile` output represent: `ncalls`, `tottime`, `cumtime`, `percall`?

2. What is `pstats` and how do you use it to sort, filter, and print profiling data programmatically? What does sorting by `cumulative` vs `tottime` tell you about different types of bottlenecks?

3. What is `line_profiler` and how does it differ from `cProfile`? When would you reach for `line_profiler` after already using `cProfile`?

4. What is `memory_profiler` and what decorator does it use? How do you interpret its output? What are the limitations of its measurement approach?

5. What is `tracemalloc` (built into Python 3.4+)? How do you use it to take snapshots and find the top memory-consuming lines? How does it differ from `memory_profiler`?

6. How does `timeit` work? What is the difference between `timeit.timeit()` and `timeit.repeat()`? Why does `timeit` suppress garbage collection by default and what does that mean for your benchmarks?

7. Explain `functools.lru_cache`. What is the difference between `lru_cache(maxsize=None)` and `functools.cache` (Python 3.9+)? What are the two conditions a function must meet to be safely cacheable?

8. What are `__slots__`? How do they reduce per-instance memory? What functionality do you lose when you define `__slots__` on a class?

9. Explain why looking up a global variable in Python is slower than a local variable. What is the bytecode difference between `LOAD_GLOBAL` and `LOAD_FAST`? How do you exploit this in hot loops?

10. What is string concatenation with `+` inefficient in Python? What is the correct pattern for building strings in a loop, and why is it faster?

11. Explain the performance difference between a generator expression and a list comprehension. When is the list comprehension faster despite using more memory?

12. What does `sys.getsizeof` measure? What does it *not* measure (shallow vs deep size)? How would you measure the true deep size of a nested structure?

13. What is a `weakref`? In what scenario would you use `weakref.ref` or `weakref.WeakValueDictionary` for performance? What happens when the referent is garbage collected?

14. What are the dangers of implementing `__del__` in Python? How can it cause memory leaks by resurrecting objects or preventing garbage collection of reference cycles?

15. What is object pooling? Give a practical example of when you would implement a pool rather than creating and destroying objects repeatedly.

16. What is NumPy vectorization and why is it faster than a Python loop? What happens "under the hood" that makes element-wise operations on `ndarray` so much faster?

17. What is lazy evaluation and how do generators implement it? Compare the memory profile of `list(range(10_000_000))` vs `range(10_000_000)` and explain why.

18. What does PyPy offer over CPython for performance? What limitations does PyPy have that prevent it from being a drop-in replacement for all workloads?

19. Explain the concept of a benchmark microbenchmark trap — what are the three main ways that microbenchmarks lie about real-world performance?

20. What is `functools.cache` vs `functools.lru_cache(maxsize=None)` vs a plain dict cache? When would you use each?
