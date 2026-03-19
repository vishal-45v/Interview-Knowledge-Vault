# Chapter 05 — Standard Library: Scenario Questions

---

1. You are building a word frequency analyser for a large corpus of documents.
   You need to: (a) count word occurrences across all documents, (b) find the 20
   most frequent words, (c) merge frequency counts from different processes.
   Show how `collections.Counter` handles each requirement, and explain what the
   arithmetic operators on Counter objects do.

2. Your service processes log lines from multiple sources and needs to group them
   by severity level, then by service name, and count occurrences of each error
   message. You want this in a nested structure with zero-initialised counts.
   Show how `collections.defaultdict` enables this cleanly, including a
   `defaultdict(lambda: defaultdict(Counter))` pattern.

3. You are building a context manager that wraps database transactions. The context
   manager must: (a) begin a transaction on entry, (b) commit on clean exit,
   (c) rollback on exception. Show two implementations — one using a class with
   `__enter__`/`__exit__` and one using `@contextlib.contextmanager`. When would
   you choose each?

4. You have a script that reads files, processes them, and writes output. Currently
   all file paths are strings manipulated with `os.path.join`, `os.path.dirname`,
   and string slicing. A new developer suggests migrating to `pathlib`. Show the
   before and after for: (a) building a path, (b) checking if a file exists,
   (c) reading text, (d) listing all `.py` files recursively.

5. Your application receives timestamps as ISO strings from an API and must convert
   them to the user's local timezone for display, and to UTC for storage. Show the
   full workflow using `datetime`, `timezone`, and `zoneinfo` (Python 3.9+), and
   explain the difference between naive and aware datetimes and why mixing them
   causes bugs.

6. You are writing a JSON serialiser for API responses that include custom types:
   `datetime` objects, `Decimal` amounts, `UUID` identifiers, and `Enum` values.
   Show how to write a custom `JSONEncoder` subclass, and alternatively how to use
   the `default` parameter of `json.dumps()`. What error do you get without custom handling?

7. Your application has extensive `print()` debugging that a senior engineer says
   must be replaced with proper logging before the code goes to production. Show
   the migration: configure a logger with a file handler, a console handler with
   different levels, a custom formatter, and explain how to avoid duplicate log
   records from the root logger.

8. You are writing a CLI tool that: accepts a required `input_file`, an optional
   `--output` flag defaulting to stdout, a `--verbose` flag, and a `--format`
   option accepting `json` or `csv`. Show the complete `argparse` setup, including
   type validation, help text, and how to test it without running the script.

9. You are building a type-safe configuration system. A config object has known
   fields with specific types, optional fields, and a field that can be either
   a string or a list of strings. Show how to model this using `TypedDict`,
   `Optional`, `Union`, and `Literal` from the `typing` module.

10. Your codebase has a function that should accept any object that has a `read()`
    method returning `str`. It should work with `io.StringIO`, open file objects,
    and custom reader classes without requiring inheritance. Show how to define and
    use a `Protocol` for this, and explain how it differs from using an ABC.
