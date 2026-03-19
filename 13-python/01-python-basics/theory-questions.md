# Chapter 01 — Python Basics: Theory Questions

---

1. What are Python's built-in data types? Categorize them as mutable or immutable and
   explain why the distinction matters in practice.

2. What is the difference between `is` and `==`? When can two objects be `==` but not
   `is`? Give a concrete example where `is` returns `True` unexpectedly due to
   interpreter optimization.

3. Explain the LEGB rule. In what order does Python resolve variable names, and what
   happens when the same name exists at multiple scopes simultaneously?

4. What is the `global` keyword? What is the `nonlocal` keyword? When do you need each,
   and what are the risks of using them excessively?

5. What is type coercion in Python? Is Python strongly typed or weakly typed? Is it
   statically typed or dynamically typed? Justify your answer with examples.

6. What is the difference between `int`, `float`, and `complex`? What does `0.1 + 0.2`
   produce and why? How do you handle floating-point precision issues in production code?

7. Explain Python's truthiness and falsiness rules. Which values evaluate to `False` in
   a boolean context? Give at least 8 examples covering different types.

8. What is the walrus operator (`:=`)? In which contexts is it useful and in which
   contexts is its use considered bad style or harmful to readability?

9. Explain string interning in Python. Why does `'hello' is 'hello'` return `True` for
   short strings but `a = 'hello world'; b = 'hello world'; a is b` may return `False`?
   What determines whether a string is interned?

10. What is the difference between `*args` and `**kwargs`? How are they collected and
    unpacked? Can you have both in the same function signature — what is the required
    ordering of parameters?

11. Explain list, dict, set, and generator comprehensions. What are the performance
    differences between a list comprehension and an equivalent `for` loop? When does a
    list comprehension become harder to maintain than a loop?

12. What is the difference between `break`, `continue`, and `pass`? What does an
    `else` clause on a `for` or `while` loop do, and when does it execute?

13. How does Python's `range` object work internally? Why is `range(10**9)` memory-safe
    but `list(range(10**9))` is not?

14. Explain sequence unpacking and extended unpacking with `*`. What does
    `a, *b, c = [1, 2, 3, 4, 5]` produce? Can `*` appear more than once in an
    unpacking expression?

15. What are f-strings and how do they differ from `%`-formatting and `str.format()`?
    What advanced features do f-strings support (e.g., format specs, expressions,
    the `=` debugging specifier introduced in Python 3.8)?

16. What is the difference between `bytes`, `bytearray`, and `str` in Python 3? How do
    you encode and decode between them, and what encodings should you be aware of?

17. How does Python handle integer overflow? What is the maximum size of a Python `int`?
    What are the performance implications of very large integer arithmetic?

18. What does `id()` return? Is the `id` of an object guaranteed to be unique for the
    entire duration of the program? Can two different objects ever share the same `id`?

19. Explain the `None` singleton. Why is `if x is None` preferred over `if x == None`?
    Can you create another instance of `NoneType`? What does `NoneType` inherit from?

20. What string methods would you use to: (a) check if a string starts with a prefix,
    (b) split on any whitespace, (c) strip only leading whitespace, (d) replace the
    first occurrence of a substring, (e) check if every character is a digit?
