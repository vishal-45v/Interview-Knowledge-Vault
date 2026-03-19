# Chapter 10: Python Internals — Structured Answers

## Q1: Explain CPython's memory management — reference counting + cyclic GC

**Answer:**

CPython uses a two-layer garbage collection system.

**Layer 1: Reference Counting**

Every Python object has an `ob_refcnt` field (a C `Py_ssize_t`). When a name is bound to an object, the count increments. When the name goes out of scope, is rebound, or deleted, the count decrements. When `ob_refcnt` reaches 0, CPython immediately deallocates the object and decrements counts of all objects it referenced.

```python
import sys

x = [1, 2, 3]
print(sys.getrefcount(x))   # 2 (x + getrefcount's argument)

y = x                        # second binding
print(sys.getrefcount(x))   # 3

del y
print(sys.getrefcount(x))   # 2 again

# When x goes out of scope (function returns, or del x):
# ob_refcnt hits 0 → list freed immediately
# → ob_refcnt of integers 1, 2, 3 each decremented
```

**The problem: Reference Cycles**

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

a = Node(1)
b = Node(2)
a.next = b    # a references b
b.next = a    # b references a — cycle!

del a, del b
# a.ob_refcnt: was 2 (name 'a' + b.next), after del: 1 (b.next still points to a)
# b.ob_refcnt: was 2 (name 'b' + a.next), after del: 1 (a.next still points to b)
# Neither reaches 0. Memory leaks without cyclic GC.
```

**Layer 2: Generational Cyclic GC**

The `gc` module implements a mark-and-sweep collector for cycles. It only tracks *container* objects (lists, dicts, sets, instances — not ints, strings).

Three generations with thresholds:
- Generation 0: new objects. Collected when len(gen0) > 700. Collected most frequently.
- Generation 1: survived one gen0 collection. Collected when gen1 promotions > 10.
- Generation 2: survived one gen1 collection. Collected occasionally (longest-lived objects).

```python
import gc

# Tune thresholds for long-running servers (reduce pause frequency)
# Default: (700, 10, 10)
gc.set_threshold(1000, 15, 15)

# Disable cyclic GC for performance-critical sections (if you know no cycles exist)
gc.disable()
# ... performance critical code ...
gc.enable()

# Force a collection (useful in tests or after bulk deletion)
gc.collect()

# Find objects in the GC's tracking list
print(gc.get_count())      # (gen0_count, gen1_count, gen2_count)
print(len(gc.get_objects()))  # total tracked objects
```

---

## Q2: Explain the GIL — what it is, why it exists, and its future

**Answer:**

The GIL (Global Interpreter Lock) is a mutex in CPython that allows only one thread to execute Python bytecode at a time. It protects CPython's non-thread-safe internal state: reference counts, memory allocator, and the interpreter's data structures.

**Why it exists:** CPython's C implementation was designed in an era before multicore CPUs were common. Making the interpreter thread-safe at a fine-grained level would require locking on every reference count operation — an enormous performance overhead. The GIL is a coarse-grained lock that is simpler and faster for single-threaded programs.

**How it switches:**
```python
import sys

# Default: switch interval of 5ms (CPython 3.2+)
print(sys.getswitchinterval())   # 0.005 seconds

# Change switch interval (reduce for I/O bound, increase for CPU bound)
sys.setswitchinterval(0.001)     # switch every 1ms

# The eval breaker: after each "tick", CPython checks:
# 1. Has the switch interval elapsed? → release GIL, let another thread take it
# 2. Are there pending signals? → process them
# 3. Is there a pending garbage collection?
```

**Impact on threads:**
```
CPU-bound: threading does NOT help (only one core used)
I/O-bound: threading DOES help (GIL released during I/O syscalls)

Two CPU-bound threads on 8-core machine:
  Without GIL: 2x speedup
  With GIL:    ~1x (or slightly worse due to GIL contention)

Two I/O-bound threads:
  GIL released during time.sleep(), socket.recv(), file.read()
  Both threads make progress concurrently → speedup proportional to I/O time
```

**CPython 3.13 Free-Threaded Build (PEP 703):**
```bash
# Install the free-threaded build
python3.13t --version
# CPython 3.13.0 (free-threaded build)

# Check if GIL is active
import sys
print(sys._is_gil_enabled())  # False in free-threaded build

# Enable/disable the GIL at runtime (experimental)
sys._set_gil_enabled(False)  # risky with non-thread-safe extensions
```

**For CPU-bound work without the GIL:** use `multiprocessing` (separate processes, each with their own GIL and memory space) or C extensions that release the GIL (`with nogil:` in Cython, NumPy does this automatically).

---

## Q3: Explain Python bytecode and the `dis` module

**Answer:**

CPython compiles Python source to bytecode (`.pyc` files) — a sequence of fixed-size instructions for the CPython virtual machine. Each instruction has an opcode (1 byte) and an argument (1 byte, or more with `EXTENDED_ARG`).

```python
import dis

def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

dis.dis(fibonacci)
```

**Sample output (Python 3.12):**
```
  2           0 RESUME                   0

  3           2 LOAD_FAST                0 (n)
              4 LOAD_CONST               1 (2)
              6 COMPARE_OP               2 (<)
             10 POP_JUMP_IF_FALSE        4 (to 20)

  4          12 LOAD_FAST                0 (n)
             14 RETURN_VALUE

  5          16 LOAD_GLOBAL              1 (NULL + fibonacci)
             26 LOAD_FAST                0 (n)
             28 LOAD_CONST               1 (1)
             30 BINARY_OP               10 (-)
             34 CALL                     1
             44 LOAD_GLOBAL              1 (NULL + fibonacci)
             54 LOAD_FAST                0 (n)
             56 LOAD_CONST               3 (2)
             58 BINARY_OP               10 (-)
             62 CALL                     1
             72 BINARY_OP                0 (+)
             76 RETURN_VALUE
```

**Key instructions:**
| Instruction | Meaning |
|---|---|
| `LOAD_FAST` | Push local variable (direct array index) |
| `LOAD_GLOBAL` | Push global variable (dict lookup) |
| `LOAD_CONST` | Push constant (from co_consts) |
| `STORE_FAST` | Pop and store to local variable |
| `CALL` | Call callable with N args (Python 3.11+) |
| `BINARY_OP` | Perform binary operation (+, -, etc.) |
| `POP_JUMP_IF_FALSE` | Conditional jump |
| `RETURN_VALUE` | Return top of stack |
| `MAKE_FUNCTION` | Create function object from code object |
| `RESUME` | Entry point for tracing/profiling hooks |

**Inspecting code objects:**
```python
code = fibonacci.__code__
print(code.co_varnames)     # ('n',) — local variable names
print(code.co_consts)       # (None, 2, 1) — constants
print(code.co_names)        # ('fibonacci',) — global names referenced
print(code.co_argcount)     # 1 — number of arguments
print(code.co_filename)     # source file path
print(code.co_firstlineno)  # 1 — first line number
```

**Python 3.11+ Specializing Adaptive Interpreter:**
CPython 3.11 introduced "quickening" — frequently-executed bytecode is replaced in-place with specialized versions. `LOAD_GLOBAL` on a module-level function might become `LOAD_GLOBAL_MODULE` (skips the generic dict lookup path). This is transparent to users but provides 10-60% speedup on typical code.

---

## Q4: Explain the descriptor protocol with a complete example

**Answer:**

The descriptor protocol is how Python implements attribute access. An object is a *descriptor* if it defines any of `__get__`, `__set__`, or `__delete__`. When you access `obj.attr`, Python's attribute lookup follows this priority:

1. Data descriptors from `type(obj).__mro__` (define both `__get__` and `__set__`)
2. Instance `__dict__`
3. Non-data descriptors from `type(obj).__mro__` (define only `__get__`)

```python
class TypedField:
    """A data descriptor that enforces type on assignment."""

    def __set_name__(self, owner, name):
        # Called when the class is created (Python 3.6+)
        self.public_name = name
        self.private_name = f"_{name}"

    def __init__(self, expected_type):
        self.expected_type = expected_type

    def __get__(self, obj, objtype=None):
        if obj is None:
            # Accessed on the class itself, not an instance
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.public_name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        setattr(obj, self.private_name, value)

    def __delete__(self, obj):
        delattr(obj, self.private_name)


class Product:
    name = TypedField(str)          # descriptor placed on the class
    price = TypedField(float)
    quantity = TypedField(int)

    def __init__(self, name, price, quantity):
        self.name = name            # calls TypedField.__set__
        self.price = price
        self.quantity = quantity

p = Product("Widget", 9.99, 100)
print(p.name)           # calls TypedField.__get__ → "Widget"
p.price = "free"        # calls TypedField.__set__ → TypeError!

# How Python resolves p.name:
# 1. type(p).__mro__ = [Product, object]
# 2. Product.__dict__["name"] = TypedField instance (has __get__ and __set__)
# 3. TypedField is a DATA descriptor (has both __get__ and __set__)
# 4. Data descriptors take priority over instance __dict__
# 5. TypedField.__get__(TypedField_instance, p, Product) is called
```

**Why data vs non-data matters:**
```python
class NonDataDescriptor:
    def __get__(self, obj, objtype=None):
        return "from descriptor"

class MyClass:
    x = NonDataDescriptor()  # non-data: only __get__, no __set__

obj = MyClass()
obj.__dict__["x"] = "from instance dict"  # manually set instance dict

print(obj.x)  # "from instance dict" — instance __dict__ wins over non-data descriptor

# Functions are non-data descriptors:
# method = MyClass.__dict__["my_method"]  (a function object)
# obj.my_method → function.__get__(obj, MyClass) → bound method
```

---

## Q5: Explain Python's import system and how to create a custom import hook

**Answer:**

When you write `import mymodule`, Python follows this sequence:

1. Check `sys.modules` — if found, return cached module
2. Walk `sys.meta_path` — call each finder's `find_spec()` method
3. The finder that matches returns a `ModuleSpec`
4. The spec's loader calls `create_module()` then `exec_module()`
5. The populated module is stored in `sys.modules` and returned

**Default `sys.meta_path` finders:**
- `BuiltinImporter` — for built-in C modules (`sys`, `builtins`)
- `FrozenImporter` — for frozen modules (embedded in the interpreter)
- `PathFinder` — searches `sys.path` for source and package files

**Custom import hook — load YAML as a module:**
```python
import importlib.abc
import importlib.machinery
import importlib.util
import sys
import yaml

class YamlFinder(importlib.abc.MetaPathFinder):
    def find_spec(self, fullname, path, target=None):
        # fullname = "config" (what's being imported)
        # path = None for top-level, or parent package path
        parts = fullname.split(".")
        yaml_file = "/".join(parts) + ".yaml"

        import os
        if os.path.exists(yaml_file):
            return importlib.machinery.ModuleSpec(
                name=fullname,
                loader=YamlLoader(yaml_file),
                origin=yaml_file,
            )
        return None  # let other finders try

class YamlLoader(importlib.abc.Loader):
    def __init__(self, filepath):
        self.filepath = filepath

    def create_module(self, spec):
        return None  # use default module creation

    def exec_module(self, module):
        with open(self.filepath) as f:
            data = yaml.safe_load(f)
        # Add all YAML keys as module attributes
        for key, value in data.items():
            setattr(module, key, value)

# Register the hook
sys.meta_path.insert(0, YamlFinder())

# Now: import config  →  loads config.yaml and exposes its keys as attributes
# import config
# print(config.database_url)
```

**Security warning:** Custom import hooks can execute arbitrary code. Never use them to load untrusted files. Be careful about injection via filename manipulation.

**`importlib.import_module` for dynamic imports:**
```python
import importlib

# Dynamic import by string name (safer than __import__)
module_name = "json"
mod = importlib.import_module(module_name)
result = mod.dumps({"key": "value"})

# Import a submodule
submod = importlib.import_module(".utils", package="mypackage")
```
