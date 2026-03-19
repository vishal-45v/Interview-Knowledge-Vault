# Chapter 04 — Concurrency & Async: Scenario Questions

---

1. Your service needs to call 50 external REST APIs to aggregate a dashboard. Currently
   it calls them sequentially and takes 25 seconds. Your SLA requires a response in
   under 2 seconds. The calls are independent. Describe three approaches to speed this
   up (threading, multiprocessing, asyncio) and explain which is most appropriate and
   why. Show a working implementation using your chosen approach.

2. You are debugging a production issue where your Flask application occasionally
   serves incorrect data — specifically, a counter that should increment per request
   is sometimes skipped or incremented twice. The counter is a module-level integer.
   Identify the root cause, explain why it is a race condition, and show how to fix
   it using the appropriate threading primitive.

3. Your data science team runs compute-intensive image preprocessing on a batch of
   10,000 images. Currently it runs sequentially on one core. All 10,000 images are
   independent. Write a `multiprocessing.Pool`-based solution, handle errors gracefully,
   and explain why you chose `Pool.map()` vs `Pool.imap()`. What is the pickle
   constraint in multiprocessing?

4. You are building a real-time chat server using asyncio. The server must handle
   1,000 simultaneous WebSocket connections with a single process. A blocking database
   call is needed on each message. Show how you would structure the async event loop,
   handle the blocking DB call without freezing other connections, and limit concurrent
   DB connections to 10.

5. A coworker writes:

   ```python
   async def fetch_all(urls):
       results = []
       for url in urls:
           result = await fetch(url)   # fetch is an async function
           results.append(result)
       return results
   ```

   Explain why this is sequential (not concurrent) despite using `async/await`, then
   rewrite it to be truly concurrent using `asyncio.gather()` or `asyncio.create_task()`.
   What is the practical speedup for 10 URLs each taking 1 second?

6. You are building a worker queue system. Multiple producers generate tasks and
   multiple consumers process them. You need: (a) tasks processed in FIFO order,
   (b) a maximum of 5 tasks in-flight at any time, (c) graceful shutdown when all
   tasks are done. Show a complete implementation using `asyncio.Queue`.

7. Your application uses `concurrent.futures.ProcessPoolExecutor` to parallelise PDF
   rendering. You receive a `PicklingError` on some tasks. Explain what causes this,
   why multiprocessing requires pickling, and how you would refactor the code to
   avoid the issue.

8. You need to implement a rate-limited API client that makes at most 10 requests
   per second, even when called from multiple concurrent threads. Show an implementation
   using `threading.Semaphore` and explain why a plain `time.sleep()` would be
   insufficient.

9. You are migrating a codebase from synchronous requests to async aiohttp. A
   coworker argues that since all network calls are already wrapped in helper
   functions, they can just change the function signatures to `async def` and
   add `await` in front of the call. Explain what other changes are required and
   what will break if only the signatures are changed.

10. Your long-running asyncio application sometimes hangs indefinitely. Suspicious
    that a particular `await` is the culprit. Show how to add timeout handling to any
    awaitable using `asyncio.wait_for()`, how to distinguish a timeout from a
    `CancelledError`, and how to implement a watchdog that cancels stale tasks.
