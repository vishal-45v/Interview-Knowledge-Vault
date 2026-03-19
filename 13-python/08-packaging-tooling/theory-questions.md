# Chapter 08: Packaging & Tooling — Theory Questions

1. What is the difference between `setup.py`, `setup.cfg`, and `pyproject.toml`? Why did the community move from `setup.py` to `pyproject.toml`, and what PEPs drove this change?

2. What is a build backend in the context of Python packaging? How does the `[build-system]` table in `pyproject.toml` specify which backend to use? Name three common backends and their associated tools.

3. What is `pip install -e .` (editable install)? How does it work mechanically? What is the difference between the legacy `setup.py develop` mechanism and the PEP 660 mechanism used by modern backends?

4. What is the difference between `install_requires` and `extras_require` (or `[project.optional-dependencies]` in `pyproject.toml`)? Give a concrete example of when you would use extras.

5. What is semantic versioning (SemVer) and how does PEP 440 relate to it? What are PEP 440 version specifiers (`~=`, `>=`, `!=`, `==`) and when is each appropriate in dependency specifications?

6. What is the purpose of `__init__.py`? How do namespace packages (PEP 420) differ from regular packages, and when would you use a namespace package?

7. What is `__all__` in a Python module? What does it affect? What does it not affect (hint: it does not prevent `import *` from accessing private names via direct import)?

8. Explain the difference between absolute and relative imports. What are the rules for when each can be used? What does `from . import utils` mean in a package?

9. How do entry points and `console_scripts` work? What happens when you run a CLI tool installed by pip? Walk through the mechanism from `pyproject.toml` to the command running in your terminal.

10. What is `tox` and what problem does it solve? How does it differ from running `pytest` directly? What is the `tox.ini` / `pyproject.toml` configuration for a basic multi-version test matrix?

11. What is `pre-commit`? How does it integrate with git hooks? What is the difference between a pre-commit hook that runs `ruff` and one that runs `mypy`, and why might you put one in pre-commit but not the other?

12. What is `ruff` and how does it differ from `flake8`? What is the performance advantage and what linting categories does it support?

13. What is `mypy` and what is the difference between `--strict` mode and default mode? What does `py.typed` marker file indicate?

14. What is poetry? What is `poetry.lock` and why should it be committed to version control for applications but not necessarily for libraries? What is the difference between `poetry add` and `poetry add --group dev`?

15. What is a private PyPI server? What is `devpi` and what would you configure in `pip.conf` or `pyproject.toml` to install packages from a private index alongside PyPI?

16. What is `sys.path` and how does Python use it to find modules? What are the dangers of manipulating `sys.path` directly, and what is a better alternative for most use cases?

17. What is the difference between `pip install package` and `pip install package==1.2.3`? In a production deployment, which do you use and why? What is `pip freeze` for?

18. What is `hatch` and what features does it offer beyond what a simple `pyproject.toml` + `setuptools` provides? What problem does it solve compared to poetry?

19. What is the purpose of a `py.typed` marker file? How do type checkers like mypy use it? What is the difference between inline types and stub files (`.pyi`)?

20. Explain dependency resolution in pip. What algorithm does pip use (as of pip 20.3+)? What is a dependency conflict and how does `pip install --dry-run` help diagnose it?
