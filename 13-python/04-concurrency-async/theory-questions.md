# Chapter 04 — Concurrency & Async: Theory Questions

---

1. What is the GIL (Global Interpreter Lock)? Why does CPython have it? What problems
   does it solve and what problems does it create? Is the GIL being removed from
   Python, and what is the current status?

2. Explain the difference between concurrency and parallelism. Which Python mechanism
   provides true parallelism for CPU-bound tasks? Which provides concurrency for
   I/O-bound tasks?

3. What is a thread in Python? What does the `threading` module provide? What is a
   daemon thread and how does it differ from a non-daemon thread?

4. What is a race condition? Give a concrete Python code example where a race condition
   occurs with threads. How do you fix it using a `threading.Lock`?

5. What is a deadlock? Give the four necessary conditions for a deadlock to occur.
   How does Python's `threading.RLock` differ from `threading.Lock`?

6. What is the multiprocessing module? How does it differ from threading? What is the
   cost of creating a process vs a thread? How does inter-process communication work?

7. What is an event loop in asyncio? What does it mean for code to be "asynchronous"?
   How does a coroutine differ from a regular function and from a thread?

8. What is `async def`, `await`, and `asyncio.run()`? What happens if you call an
   `async def` function without `await`? What does `await` actually do?

9. What is `asyncio.Task` vs a coroutine? What is `asyncio.gather()`? When should
   you use `asyncio.create_task()` vs `await coroutine()` directly?

10. What is `asyncio.Queue`? How does it help coordinate producers and consumers
    in an async program? How does it differ from `queue.Queue` for threads?

11. What are `concurrent.futures.ThreadPoolExecutor` and `ProcessPoolExecutor`? How
    do they differ from directly using `threading.Thread` and `multiprocessing.Process`?

12. What is `asyncio.sleep(0)` and why is it sometimes used? What is the difference
    between `asyncio.sleep(0)` and `asyncio.sleep(1)`?

13. What is the difference between `threading.Semaphore` and `threading.BoundedSemaphore`?
    What is a semaphore used for? Give a concurrency pattern where a semaphore is
    appropriate.

14. What is `subprocess`? How does `subprocess.run()` differ from `subprocess.Popen()`?
    When would you use `subprocess` vs `os.system()`?

15. How does `asyncio.shield()` work? What is `asyncio.timeout()` (or `asyncio.wait_for()`)
    and why is timeout handling important in async code?

16. What is the `queue.Queue` class? How is it thread-safe? What is the difference
    between `put()`, `put_nowait()`, `get()`, and `get_nowait()`? What is `task_done()`?

17. How do you run blocking code inside an async function without blocking the event
    loop? What is `loop.run_in_executor()`? When must you use it?

18. What are coroutines, tasks, and futures in asyncio? What is the relationship
    between them? How does `asyncio.ensure_future()` relate to `asyncio.create_task()`?

19. What is `aiohttp` and why is it needed instead of using `requests` in async code?
    What happens if you `await` a blocking `requests.get()` call inside an async function?

20. Explain Python's GIL-release mechanism. What kinds of operations release the GIL
    during execution, and what is the significance for I/O-bound threading?
