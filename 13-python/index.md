# Python Interview Knowledge Vault

A comprehensive interview preparation guide covering core Python concepts, internals,
patterns, and best practices. Each chapter contains theory questions, scenario questions,
follow-up traps, structured answers, analogies, and diagrams.

---

## Chapters

### Chapter 01 — Python Basics
`01-python-basics/`

Covers the foundational building blocks of Python: data types and their internals,
mutability vs immutability, type coercion rules, variable scoping (LEGB rule),
operators, control flow, comprehensions, walrus operator, f-strings, truthiness,
unpacking, and *args/**kwargs. Essential for any Python interview.

---

### Chapter 02 — Object-Oriented Programming & Classes
`02-oop-classes/`

Deep dive into Python's object model: class definition and instantiation, instance vs
class vs static methods, inheritance chains, super() mechanics, multiple inheritance and
MRO (C3 linearization), dunder/magic methods, properties, descriptors, __slots__,
dataclasses, abstract base classes, metaclasses, and class decorators.

---

### Chapter 03 — Functional Programming
`03-functional-programming/`

Explores Python's functional side: first-class functions, closures and their cell
objects, function and class-based decorators, functools utilities, lambdas, map/filter/zip,
generators and the yield machinery, iterators protocol, itertools, all comprehension
forms, higher-order functions, and immutability patterns.

---

### Chapter 04 — Concurrency & Async
`04-concurrency-async/`

Tackles Python's concurrency model from the ground up: the GIL and why it exists,
threading with its limitations and use cases, multiprocessing for CPU-bound work,
asyncio's event loop and coroutine model, async/await syntax, tasks and gather,
concurrent.futures, aiohttp, subprocess, and synchronization primitives (locks,
semaphores, queues). Includes when-to-use guidance for each approach.

---

### Chapter 05 — Standard Library
`05-standard-library/`

Practical mastery of Python's standard library: collections module (defaultdict, Counter,
OrderedDict, deque, namedtuple, ChainMap), itertools recipes, contextlib patterns,
pathlib for filesystem work, datetime and timezone handling, json/csv I/O, regex with re,
logging best practices, argparse, the typing module and modern type hints, and enum.

---

### Chapter 06 — Testing & Pytest
`06-testing-pytest/`

Writing and organizing tests with pytest and unittest, mock and patch patterns, fixtures,
parametrize, coverage, debugging with pdb/breakpoint(), and common testing strategies
for real-world Python projects.

---

### Chapter 07 — Performance & Profiling
`07-performance-profiling/`

CPython performance patterns: profiling with cProfile and line_profiler, memory
profiling, numpy vectorization, slots optimization, and profiling-driven optimization
workflows for production Python code.

---

### Chapter 08 — Packaging & Tooling
`08-packaging-tooling/`

Modern Python packaging: pyproject.toml, setup.cfg, virtual environments, pip and
pip-tools, poetry, module vs package, __init__.py conventions, relative imports,
namespace packages, entry points, and distributing packages on PyPI.

---

### Chapter 09 — Web Frameworks
`09-web-frameworks/`

Python web development: Flask and Django fundamentals, FastAPI and async frameworks,
WSGI vs ASGI, middleware patterns, ORM usage, REST API design, and common web
interview questions.

---

### Chapter 10 — Python Internals
`10-python-internals/`

CPython internals: bytecode and the dis module, reference counting and garbage
collection, the memory model, small integer caching, string interning, the import
system, and metaclass machinery under the hood.

---

## How to Use This Vault

1. Start with `theory-questions.md` to self-assess your knowledge gaps.
2. Try to answer each question before reading `structured-answers.md`.
3. Use `analogy-explanations.md` when a concept does not click immediately.
4. Study `diagram-explanations.md` to build strong mental models.
5. Practice `scenario-questions.md` out loud — simulate a real interview setting.
6. Drill `follow-up-traps.md` last — these separate good candidates from great ones.
