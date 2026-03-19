# Chapter 01 — Python Basics: Scenario Questions

---

1. Your team is building a configuration loader that reads settings from a file and
   stores them in a dictionary. A junior developer writes:

   ```python
   defaults = {"timeout": 30, "retries": 3}
   config = defaults
   config["timeout"] = 60
   ```

   They are surprised that `defaults["timeout"]` is now 60. How do you explain this,
   and how would you fix the code to avoid this mutation? What if the config dict
   contains nested dicts — how does your fix change?

2. You are reviewing a pull request and you see the following code pattern scattered
   across a codebase:

   ```python
   def process(items=[]):
       items.append("done")
       return items
   ```

   The bug only appears in production under specific call patterns. Identify the issue,
   explain exactly why it occurs, and rewrite the function correctly. Why do many
   developers fall into this trap?

3. Your data pipeline uses a set of status strings like `"pending"`, `"active"`,
   `"done"`. A colleague suggests storing them as plain strings throughout the codebase.
   You suggest using something more Pythonic. What would you recommend and why? Show
   how you would define and use it, and explain what protection it provides.

4. You are writing a function that accepts flexible positional and keyword arguments to
   build an HTTP request:

   ```python
   def make_request(method, url, *headers, **params):
       pass
   ```

   A caller does `make_request("GET", "https://api.example.com", timeout=5)`.
   Is `timeout` accessible via `headers` or `params`? What if someone passes
   `make_request("GET", "https://api.example.com", "Authorization: Bearer token")`?
   Trace exactly what lands where.

5. You are optimizing a data-processing script that builds a filtered list of 1 million
   integers. The current code uses a list comprehension but memory usage is very high.
   You only need to iterate over the result once. How do you refactor it, and what is
   the memory trade-off? Show before and after code.

6. You have a REPL session where you are debugging variable scoping:

   ```python
   x = 10
   def outer():
       x = 20
       def inner():
           print(x)
       inner()
   outer()
   print(x)
   ```

   What does each `print` output? Now change `inner` to do `x += 1` before printing.
   What error occurs and why? How do you fix it while keeping `inner` as a nested
   function?

7. You are reading a large text file line by line and need to find lines that match
   a pattern and assign them to a variable in the same expression to avoid calling
   the match function twice. A coworker suggests using the walrus operator. Show how
   you would use `:=` here, and then critique when this approach might reduce
   readability.

8. Your team processes financial data and a senior engineer says "never use `float`
   for money." You are asked to explain this to an intern and demonstrate the problem
   with a code example, then show the correct approach using the standard library.

9. You have a function that returns multiple values:

   ```python
   def get_stats(data):
       return min(data), max(data), sum(data) / len(data)
   ```

   Show three different ways callers can unpack the return value. Then show how
   extended unpacking (`*`) could be used if the function returned 5 values but
   the caller only cares about the first and last.

10. Your team is writing a script that runs in Python 3.10+. A developer uses an
    `if/elif` chain with 12 branches checking string values. You suggest using
    structural pattern matching (`match`/`case`). Show the before and after, and
    explain which Python version introduced this feature and what it can match beyond
    simple string values.
