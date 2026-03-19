# Chapter 03 — Functional Programming: Theory Questions

---

1. What does it mean for functions to be "first-class citizens" in Python? List all
   the things you can do with a function that demonstrate this property.

2. What is a closure? What is a "free variable" and a "cell object"? How do you
   inspect the closed-over variables of a closure using the function's attributes?

3. What is a decorator in Python? Explain the mechanics of how `@decorator` syntax
   works. What does it mean that decorators are "just functions that take functions"?

4. What is `functools.wraps` and why is it necessary when writing decorators? What
   attributes does it copy, and what happens to `help()`, `inspect.signature()`,
   and `__wrapped__` if you omit it?

5. What is `functools.lru_cache`? What algorithm does it use? What are `maxsize` and
   `typed` parameters? When does the cache NOT help (i.e., when is a function
   unsuitable for caching)?

6. What is the difference between `functools.partial` and a lambda? When would you
   use each? What does `partial` return — a new function or a new object?

7. What is a generator function? What does `yield` do to the execution of a function?
   What is the difference between a generator function and a generator iterator?

8. Explain `yield from`. How does it differ from a `for` loop with `yield`? What does
   it do with `send()` values and thrown exceptions?

9. What is the iterator protocol? What two methods does an iterator must define?
   How does a `for` loop translate to iterator protocol calls? What happens when the
   iterator is exhausted?

10. What is the difference between an iterable and an iterator? Can all iterables be
    turned into iterators? Can an iterator also be iterable? Give an example of each.

11. What does `map()` return in Python 3? What is the difference between `map()` and a
    list comprehension in terms of evaluation? When is `map()` preferable?

12. What does `filter()` return? How does it relate to a list comprehension with a
    condition? What is `itertools.filterfalse`?

13. What is `functools.reduce`? How does it differ from `sum()`? Give a use case where
    `reduce` is the clearest solution.

14. What are generator expressions and how do they differ from list comprehensions
    syntactically and semantically? When is a generator expression impossible to use
    (i.e., when must you use a list)?

15. What is `itertools.chain`? What is `itertools.islice`? What is `itertools.groupby`?
    What is the gotcha with `groupby` regarding pre-sorting?

16. What is a higher-order function? Is Python's `sorted()` a higher-order function?
    What about `functools.reduce`? Give three examples from the standard library.

17. What is the difference between a function decorator and a class-based decorator?
    When would you prefer a class-based decorator over a function-based one?

18. What does it mean for data to be "immutable" in a functional programming sense?
    Python does not enforce immutability — what patterns can you use to signal and
    encourage immutability in Python code?

19. What is `itertools.product`? How does it relate to nested `for` loops? What is
    `itertools.combinations` vs `itertools.permutations`? How do the counts differ?

20. What happens if you call `send()` on a freshly created generator that has not
    been advanced yet? What is the "prime" pattern and why is it sometimes necessary?
