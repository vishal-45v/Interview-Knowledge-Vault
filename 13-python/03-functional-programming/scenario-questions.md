# Chapter 03 — Functional Programming: Scenario Questions

---

1. You are building a web framework where routes are registered with decorators like
   `@app.route("/users", methods=["GET", "POST"])`. The decorator must:
   (a) register the function in a route table, (b) preserve the function's original
   signature for documentation, (c) wrap it to add request logging. Design the
   decorator, explaining how nested functions enable parameterised decorators.

2. Your data science team processes large CSV files (multi-GB). They load an entire
   file into a list of dicts before processing. Memory is running out. Rewrite the
   pipeline using generators so that at any point only one row is in memory. The
   pipeline has three stages: read rows, filter by a condition, transform each row.

3. You are implementing a retry decorator that wraps any function and retries it up
   to N times with exponential backoff if it raises a specified exception type. The
   decorator should be configurable: `@retry(times=3, exception=IOError, backoff=2)`.
   Show the full implementation using nested functions.

4. Your team has a utility module with 20 functions. A teammate proposes using
   `functools.lru_cache` on all of them to speed things up. You push back on three of
   the functions. Describe what properties make a function unsuitable for `lru_cache`,
   and give one concrete example function from each problematic category.

5. You are writing a lazy evaluation pipeline to process a stream of log events. Each
   event is a dict. The pipeline must: filter events where `level == "ERROR"`, extract
   the `message` field, deduplicate messages, and yield the first 10 unique ones.
   Show how to implement this entirely with generators without loading all events.

6. A coworker writes the following code and cannot understand why it behaves
   unexpectedly:

   ```python
   functions = []
   for i in range(5):
       functions.append(lambda: i)
   print([f() for f in functions])
   ```

   Explain the bug (it involves closures), what the output is, and show two ways
   to fix it — one using a default argument trick and one using `functools.partial`.

7. You are building a functional-style data transformation library. A user wants to
   compose several functions together: `pipeline = compose(parse_date, normalize,
   validate)` where `pipeline(raw)` applies them left-to-right. Implement `compose`
   using `functools.reduce`. Then implement a lazy version that returns a generator.

8. You are profiling a recursive Fibonacci function called millions of times. It is
   extremely slow. Show how to fix it with `functools.lru_cache`, explain the cache
   invalidation model (when does the cache clear?), and explain why `lru_cache` cannot
   be used on instance methods that have a mutable `self` argument.

9. You need to implement a stateful generator that generates batches of items from an
   input stream. `batch(iterable, size=100)` should yield lists of up to `size` items.
   Show an implementation using `itertools.islice` and explain why generators make
   this pattern elegant compared to a class-based iterator.

10. Your team is debating whether to use `map(func, items)` or `[func(x) for x in
    items]`. Write a benchmark comparing both, explain the theoretical differences,
    and provide a style guide recommendation with three concrete rules for when to
    prefer each form.
