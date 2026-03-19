# Chapter 07: Performance & Profiling — Scenario Questions

1. A FastAPI endpoint that processes financial reports is taking 4–6 seconds to respond. Your manager says it needs to be under 500ms. You have never profiled Python code before in a production context. Describe your end-to-end methodology: what tools you use first, what to look for, and how you distinguish a CPU-bound bottleneck from an I/O-bound one.

2. A data science team has a script that processes 10 million rows of sensor data using a for-loop and takes 45 minutes. They ask you to help speed it up without rewriting the algorithm in C or Cython. What Python-native and NumPy-based strategies would you apply, and in what order?

3. A Django web application is leaking memory. The process starts at 200MB and grows to 2GB over 12 hours before being restarted. You suspect a Python-level leak (not a C extension). Walk through how you would use `tracemalloc` and `objgraph` to identify the source, including the specific API calls you would make.

4. A colleague implements a recursive Fibonacci function and calls it in a tight loop. The application is too slow. They ask you to add caching. Walk through using `functools.lru_cache`, explain the cache size tradeoff, and explain why a recursive cached function can hit Python's recursion limit for large inputs and how to work around it.

5. A Python service that processes images is running on an 8-core machine but only uses one core (you can see this in `htop`). The CTO says "Python can't use multiple cores, just rewrite it in Go." How do you respond? What concrete changes could you make to utilize the other cores, and what are the limits of each approach?

6. You are asked to optimize a hot function that does a lot of string building inside a loop. The current code concatenates strings with `+=`. It runs in 2.3 seconds for 100,000 iterations. How do you fix it, and can you quantify the expected improvement using `timeit`?

7. A microservice builds large Python dictionaries from database rows and serializes them to JSON. Memory usage is high. An engineer suggests replacing the dicts with classes that use `__slots__`. Walk through whether that is a good idea, how you would measure the memory savings, and what the tradeoffs are.

8. A batch job reads configuration from a module-level dictionary on every iteration (the loop runs 5 million times). A profiler shows that dict lookup is the top time consumer. How would you restructure the code to make this faster using local variable binding?

9. A web scraping script makes 500 sequential HTTP requests, each taking ~0.2 seconds. Total runtime is ~100 seconds. The developer asks you to use `threading` to parallelize it. What would you recommend and why? What is the performance limit for `threading` on I/O-bound tasks vs CPU-bound tasks, and what does the GIL have to do with it?

10. You are building a high-frequency trading system in Python that needs to process 50,000 price updates per second. The current implementation handles 8,000/second. Describe a strategy that uses object pooling, `__slots__`, local variable caching, and avoiding unnecessary attribute lookups to push performance toward the target.
