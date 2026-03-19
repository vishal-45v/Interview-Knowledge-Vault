# Chapter 05 — Standard Library: Theory Questions

---

1. What is `collections.defaultdict`? How does it differ from a regular `dict`? What
   is the `default_factory` and what happens if you access a missing key? When should
   you use `setdefault()` instead of `defaultdict`?

2. What is `collections.Counter`? What does it do with elements not in the count?
   How do you find the N most common elements? What arithmetic operations does
   `Counter` support?

3. What is `collections.OrderedDict`? Since Python 3.7+ regular dicts maintain
   insertion order, is `OrderedDict` still useful? What does it provide that a regular
   dict does not?

4. What is `collections.deque`? How does it differ from a list for append/pop
   operations at both ends? What is the `maxlen` parameter and how does it behave
   when the deque is full?

5. What is `collections.namedtuple`? How does it differ from a regular tuple? How
   does it differ from a `@dataclass`? When would you prefer a namedtuple over a
   dataclass?

6. What is `collections.ChainMap`? How does attribute lookup work across the maps?
   How does writing to a `ChainMap` work — which underlying map receives the write?

7. What is `contextlib.contextmanager`? How does a generator-based context manager
   work? What must be true about the `yield` in the generator?

8. What is `contextlib.suppress`? How does it differ from a `try/except/pass` block?
   What is `contextlib.ExitStack` and when is it useful?

9. What is `pathlib.Path`? How does it differ from `os.path` string operations? What
   are the advantages of using `Path` objects for filesystem operations?

10. What is the difference between `datetime.datetime`, `datetime.date`, and
    `datetime.time`? What is the difference between a naive and an aware datetime?
    How do you make a timezone-aware datetime?

11. What is the `json` module? What Python types can be serialised to JSON by default?
    What happens when you try to serialise a `datetime`, a `set`, or a custom object?
    How do you handle custom serialisation?

12. What is the `re` module? What is the difference between `re.match()`, `re.search()`,
    `re.findall()`, and `re.finditer()`? When should you compile a regex with
    `re.compile()`?

13. What is Python's `logging` module? How does it differ from `print()` statements?
    Explain the hierarchy: loggers, handlers, formatters, and levels. What is
    `propagate` and when would you set it to `False`?

14. What is `argparse`? What are the differences between positional arguments and
    optional arguments? What is `nargs`? How do you create subcommands with `argparse`?

15. What is the `typing` module? Explain `Optional[T]`, `Union[T, U]`, `List[T]`,
    `Dict[K, V]`, `Tuple[T, ...]`. What is the difference between `Any` and `object`?

16. What is `TypeVar` and when do you use it? Show an example of a generic function
    that preserves the type of its argument.

17. What is `Protocol` in Python's typing system? How does it enable structural
    subtyping (duck typing with type checking)? How does it differ from ABC?

18. What is `TypedDict`? What does it provide at type-checking time vs runtime?
    How does it differ from a regular `dict` and from a `@dataclass`?

19. What is `enum.Enum`? How does it differ from using constants (module-level
    variables)? What is `enum.IntEnum` and `enum.Flag`? What are the auto() helper?

20. What is `itertools.groupby` and what sorting requirement does it have? What is
    `itertools.chain.from_iterable` and when does it save memory over `chain`?
