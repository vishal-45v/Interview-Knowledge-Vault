# Chapter 08: Packaging & Tooling — Follow-Up Traps

## Trap 1: "pyproject.toml replaces setup.py entirely"

**What most people say:** "We moved to pyproject.toml so we don't need setup.py anymore."

**Correct answer:** `pyproject.toml` is the configuration file; the *build backend* (setuptools, flit, hatchling, poetry-core) is what actually performs the build. With `setuptools`, you can still have a `setup.py` for complex dynamic configuration that cannot be expressed in static TOML (e.g., Cython extension compilation). The distinction: `pyproject.toml` declares *what* to build and *which backend* to use; the backend's `setup.py` (if any) handles *how*. For most pure-Python packages, `pyproject.toml` + setuptools with no `setup.py` is the modern, complete setup.

```toml
# pyproject.toml — full modern setup, no setup.py needed
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "mypackage"
version = "1.2.3"
requires-python = ">=3.9"
dependencies = ["requests>=2.28", "pydantic>=2.0"]
```

---

## Trap 2: "Locking the version with `==` in setup.py is the responsible thing to do"

**What most people say:** "I pin `requests==2.31.0` in my library's `install_requires` to be safe."

**Correct answer:** Pinning exact versions in a *library's* `install_requires` is a major anti-pattern. If two libraries both pin the same dependency at different exact versions, pip cannot satisfy both simultaneously — every user of your library gets a dependency conflict. Libraries should specify *minimum compatible versions and exclusions* using `>=` and `!=`. Only *applications* (deployed services, scripts) should pin exact versions, typically through a lock file (`poetry.lock`, `pip freeze > requirements.txt`). The library's job is to say "I need at least requests 2.25"; the application deployer decides the exact version.

```toml
# Library: use ranges
[project]
dependencies = [
    "requests>=2.25,<3.0",    # good: compatible range
    "pydantic>=2.0",           # good: minimum with major version constraint
]

# Application (requirements.txt or poetry.lock): pin exactly
# requests==2.31.0
# pydantic==2.3.0
```

---

## Trap 3: "`__all__` prevents importing names not listed in it"

**What most people say:** "I define `__all__` to control what users can import from my module."

**Correct answer:** `__all__` only affects `from module import *`. Direct imports (`from module import _private_thing`) are never blocked by `__all__`. Python has no true access control for imports. `__all__` is documentation and a `import *` filter, not a security mechanism. Prefixing names with `_` is the conventional signal for "private," but it too can be imported directly. Tools like `pylint` and `ruff` will warn about importing `_private` names, but Python itself will not.

```python
# mymodule.py
__all__ = ["public_func"]

def public_func(): return "public"
def _private_func(): return "private"

# test.py
from mymodule import *
public_func()    # OK
_private_func()  # NameError — * import respected __all__

from mymodule import _private_func   # WORKS! __all__ doesn't block this
_private_func()  # "private" — still accessible
```

---

## Trap 4: "Relative imports are always better inside a package"

**What most people say:** "Inside a package, always use `from . import thing` to avoid hardcoding the package name."

**Correct answer:** Relative imports have real drawbacks. They cannot be used in scripts run directly (a module run as `python mypackage/utils.py` has `__name__ == "__main__"` and no `__package__`, so relative imports fail). They also make it harder to move code between packages because every relative import path must be updated. The Python community generally prefers **absolute imports** (PEP 8 recommends them), even inside packages. Use relative imports when you specifically want to express "this module within this package" and the package may be renamed or redistributed, but never as a blanket rule.

```python
# mypackage/service.py

# Absolute import — works everywhere, clear origin
from mypackage.utils import helper     # preferred

# Relative import — only works when mypackage is a proper package
from .utils import helper              # fails if run directly as a script
```

---

## Trap 5: "A virtual environment IS the Python installation"

**What most people say:** "I activated my venv so I have a separate Python."

**Correct answer:** A virtual environment is not a separate Python — it is a lightweight directory that contains symlinks to the real Python binary and a separate `site-packages` directory. When you activate a venv, `PATH` is modified so the venv's `python` symlink is found first. The Python binary itself is the same CPython installation from `/usr/bin/python3` or wherever. The venv only isolates third-party packages (site-packages). This matters because: if you install a Python version update that changes the symlink target, the venv may break; system Python updates can invalidate venvs.

---

## Trap 6: "poetry.lock should not be committed for libraries"

**What most people say:** "Lock files are for applications, not libraries, so I don't commit poetry.lock."

**Correct answer:** This is partially correct but needs nuance. The *distributed* package (what users pip-install) should NOT include a lock file — users' installers resolve their own environment. However, the *repository* for the library should commit `poetry.lock` for **reproducible development and CI**. When contributors clone the repo and run `poetry install`, they all get identical environments, preventing "works on my machine" issues. The lock file is for the development experience, not the published artifact. The published package's pinned deps come from `pyproject.toml`'s `dependencies`, not from the lock file.

---

## Trap 7: "entry_points console_scripts just copies the Python file to /usr/local/bin"

**What most people say:** "When I install a CLI tool, it puts the Python script in /usr/local/bin."

**Correct answer:** pip generates a small wrapper script (on Unix, a shell script or stub; on Windows, a `.exe` launcher) that calls the specified Python entry point function using the virtual environment's Python. The wrapper sets up the Python path and calls your function. This is why `mypackage` installed in a venv gives you a `mypackage` binary that uses that venv's Python — the generated wrapper has the absolute path to the venv's Python baked in.

```toml
# pyproject.toml
[project.scripts]
mycli = "mypackage.cli:main"

# pip generates approximately this wrapper at venv/bin/mycli:
# #!/path/to/venv/bin/python
# from mypackage.cli import main
# import sys
# sys.exit(main())
```

---

## Trap 8: "tox just runs pytest in a loop"

**What most people say:** "tox is pytest for multiple Python versions, nothing more."

**Correct answer:** `tox` does much more: (1) Creates isolated virtual environments for each test environment defined in `tox.ini`. (2) Builds and installs your package into each environment using the configured build backend — this means you're testing the *installed* package, not just the source tree (catching packaging errors like missing `package_data`). (3) Supports non-test environments for docs, linting, type checking, etc. (4) Provides environment reuse (`tox -r` to recreate), parallel execution (`tox -p auto`), and factor-based environment selection. Testing via `pytest` alone tests your source tree; `tox` tests your *installable package*.

```ini
# tox.ini
[tox]
env_list = py39, py310, py311, py312, lint, type

[testenv]
deps = pytest, pytest-cov
commands = pytest {posargs}

[testenv:lint]
deps = ruff
commands = ruff check src/ tests/

[testenv:type]
deps = mypy
commands = mypy src/
```

---

## Trap 9: "`pip freeze` gives me the requirements.txt I need"

**What most people say:** "I run `pip freeze > requirements.txt` to capture my dependencies."

**Correct answer:** `pip freeze` captures every package installed in the environment — including transitive dependencies (dependencies of your dependencies) and development-only packages. If a colleague installs from this file on a different OS or Python version, some compiled packages may not be available at those exact versions. The better approach: use `pip-tools` (`pip-compile`) which takes a high-level `requirements.in` (just your direct deps) and generates a full `requirements.txt` with all hashes. Or use poetry/hatch which separate direct from transitive deps in the lock file. Also, `pip freeze` on a fresh venv with only your dependencies still captures pip and setuptools themselves.

---

## Trap 10: "namespace packages need an empty `__init__.py`"

**What most people say:** "Every Python package directory needs `__init__.py`."

**Correct answer:** Namespace packages (PEP 420) explicitly work *without* `__init__.py`. Python will find and merge directories without `__init__.py` across different locations on `sys.path` into a single namespace package. This is used for large organizations where different teams own parts of a namespace: `company.auth`, `company.billing`, `company.reporting` — each in a separate repo and pip-installed separately, but all importable as `from company.auth import ...`. The catch: if you have `__init__.py` in one location, Python treats that directory as a regular package and will NOT merge it with other directories on the path under the same name.

```
# Namespace package — no __init__.py
company-auth-sdk/
    company/          ← no __init__.py here!
        auth/
            __init__.py
            login.py

company-billing-sdk/
    company/          ← no __init__.py here!
        billing/
            __init__.py
            invoice.py

# After pip-installing both:
from company.auth import login      # works!
from company.billing import invoice # works!
# Python merged both "company/" dirs into one namespace package
```
