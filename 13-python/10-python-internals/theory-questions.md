# Chapter 10: Python Internals — Theory Questions

1. What is a PyObject? What fields does every Python object have at its base level in CPython? Why does every Python value have a type pointer and a reference count?

2. How does CPython's reference counting garbage collection work? What happens when `ob_refcnt` reaches zero? What is a reference cycle and why can't pure reference counting handle it?

3. Explain CPython's cyclic garbage collector (the `gc` module). What is tri-color marking (white/grey/black)? What are the three generational heaps (generation 0, 1, 2) and what triggers a collection of each?

4. What is the GIL (Global Interpreter Lock)? What does it protect? Why does it exist, and why does it prevent true multi-core parallelism for CPU-bound Python threads?

5. Explain the GIL's safepoints and eval breaker mechanism. How often does the GIL switch between threads in CPython 3.2+ (hint: GIL interval)? What is `sys.getswitchinterval()`?

6. What is the PEP 703 "free-threaded Python" (also called GIL-free Python)? In Python 3.13, how do you enable it? What are the current limitations of the free-threaded build?

7. What is Python bytecode? How do you use the `dis` module to inspect it? Name six important bytecode instructions and explain what each does.

8. What are code objects (`types.CodeType`)? What does `co_code`, `co_varnames`, `co_consts`, `co_names`, and `co_freevars` contain?

9. Explain Python's stack frame model. What is a `PyFrameObject`? What is `f_locals`, `f_globals`, `f_back`? How does the call stack work at the bytecode level?

10. What is the small integer cache in CPython? What is the range (-5 to 256)? Why does `a is b` sometimes return `True` for two separately created integers? What about string interning?

11. Explain Python's import system in terms of `importlib`. What are finders and loaders? What is `sys.meta_path` and what is the difference between a `MetaPathFinder` and a `PathEntryFinder`?

12. What is `sys.modules`? What happens when you `import` the same module twice? How does circular import deadlock occur, and what is the standard way to resolve it?

13. What is structural pattern matching (introduced in Python 3.10)? How does `match`/`case` differ from `if`/`elif` chains? What are capture patterns, guard clauses, and `_` as a wildcard?

14. Explain the descriptor protocol. What are `__get__`, `__set__`, and `__delete__`? What is the difference between a data descriptor and a non-data descriptor, and how does Python resolve attribute lookup order?

15. What is the MRO (Method Resolution Order)? What is the C3 linearization algorithm? Give an example of a diamond inheritance scenario and trace the MRO manually.

16. How does `__slots__` interact with the descriptor protocol? What are slot descriptors and how do they differ from property descriptors?

17. What is `weakref` at the CPython level? What is a `tp_weaklistoffset` and how does the garbage collector track weakly-referenced objects?

18. What are PEP 695 type aliases (`type X = int | str`) introduced in Python 3.12? How do they differ from `TypeAlias` from `typing`?

19. What does `tomllib` (added in Python 3.11 as a built-in) do? Why was it added to the standard library, and what are its limitations (hint: read-only)?

20. Explain Python's memory model for names and objects. When you do `x = y = []`, what is created on the heap and what is on the stack? What happens when you do `x = x + [1]` vs `x += [1]`?
