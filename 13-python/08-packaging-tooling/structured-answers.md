# Chapter 08: Packaging & Tooling — Structured Answers

## Q1: Walk through a complete modern `pyproject.toml` for a publishable library

**Answer:**

The modern Python packaging standard (PEP 517/518/621) uses `pyproject.toml` as the single source of truth for project metadata and build configuration.

```toml
# pyproject.toml

[build-system]
# Declares which tool builds the package and what it needs installed to do so
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "myawesome-lib"
version = "1.2.0"
description = "A library for awesome things"
readme = "README.md"
license = { text = "MIT" }
authors = [{ name = "Alice Smith", email = "alice@example.com" }]
requires-python = ">=3.9"
keywords = ["awesome", "library"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
]

# Runtime dependencies — use ranges, not exact pins (this is a library)
dependencies = [
    "requests>=2.28,<3.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
# pip install myawesome-lib[postgres]
postgres = ["psycopg2-binary>=2.9"]
# pip install myawesome-lib[dev]
dev = [
    "pytest>=7.0",
    "pytest-cov",
    "mypy>=1.0",
    "ruff>=0.1",
]
# pip install myawesome-lib[all]
all = ["myawesome-lib[postgres]"]

[project.scripts]
# Creates a CLI command: awesome-cli → myawesome_lib.cli:main()
awesome-cli = "myawesome_lib.cli:main"

[project.urls]
Homepage = "https://github.com/alice/myawesome-lib"
Repository = "https://github.com/alice/myawesome-lib"
Documentation = "https://myawesome-lib.readthedocs.io"

# Tool configuration (linters, formatters, type checkers)
[tool.ruff]
line-length = 88
select = ["E", "W", "F", "I", "B", "C4", "UP"]

[tool.mypy]
strict = true
python_version = "3.9"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=myawesome_lib --cov-branch --cov-fail-under=85"
```

**Building and publishing:**
```bash
pip install build twine
python -m build              # creates dist/*.whl and dist/*.tar.gz
twine upload dist/*          # publish to PyPI
```

---

## Q2: Explain the `tox` workflow for multi-version testing

**Answer:**

`tox` orchestrates isolated virtual environments for running tests, linting, and type checking across different Python versions and configurations. It installs your package into each environment via the build backend, so you're testing the installed artifact — not just the source.

```ini
# tox.ini (or [tool.tox] in pyproject.toml)
[tox]
env_list = py{39,310,311,312}-{unit,integration}, lint, type, docs

[testenv]
deps =
    pytest>=7.0
    pytest-cov
    # extras from pyproject.toml
    .[postgres]
commands =
    unit:        pytest tests/unit/ {posargs}
    integration: pytest tests/integration/ {posargs:-m "not slow"}
setenv =
    integration: DATABASE_URL = sqlite:///test.db

[testenv:lint]
skip_install = true
deps = ruff>=0.1
commands = ruff check src/ tests/

[testenv:type]
deps = mypy>=1.0
commands = mypy src/myawesome_lib/

[testenv:docs]
deps = sphinx, sphinx-rtd-theme
commands = sphinx-build -b html docs/ docs/_build/html
```

**Key tox commands:**
```bash
tox                        # run all environments
tox -e py311-unit          # run specific environment
tox -e py311-unit,lint     # run multiple environments
tox -p auto                # parallel execution
tox -r                     # recreate environments (after dep changes)
tox list                   # show all environments
```

**GitHub Actions integration:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install tox tox-gh-actions
      - run: tox
```

---

## Q3: Explain how pre-commit hooks work and how to configure them effectively

**Answer:**

`pre-commit` manages git hooks as a framework. It installs a hook script at `.git/hooks/pre-commit` that runs your configured checks before each commit. Each check is a "hook" defined in `.pre-commit-config.yaml`, pointing to a repository that provides the hook.

```yaml
# .pre-commit-config.yaml
repos:
  # Fast linter/formatter — runs in milliseconds
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.9
    hooks:
      - id: ruff                  # lint
        args: [--fix]             # auto-fix where possible
      - id: ruff-format           # format (replaces black)

  # Type checking — optionally slower
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.7.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic>=2.0]
        args: [--strict]
        # Only run mypy on changed files — much faster
        pass_filenames: true

  # Prevent secrets from being committed
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets

  # Basic file hygiene
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-merge-conflict
      - id: debug-statements       # catches leftover breakpoint() calls
```

**Installation and usage:**
```bash
pip install pre-commit
pre-commit install           # installs the git hook
pre-commit run --all-files   # run all hooks on all files (CI)
pre-commit autoupdate        # update all hook revisions
```

**Performance strategy for large teams:**
- Keep pre-commit hooks to sub-10-second total for common changes
- Defer slow checks (full mypy type check) to CI, not pre-commit
- Use `--files` mode (default) so mypy only checks changed files
- Run `pre-commit run --all-files` in CI to ensure full coverage

---

## Q4: How does `poetry` manage dependencies differently from pip + requirements.txt?

**Answer:**

`poetry` separates *direct dependencies* from *resolved transitive dependencies*:

- `pyproject.toml`: your direct dependencies with version ranges (what you specify)
- `poetry.lock`: the fully resolved, pinned, hashed snapshot of every package in the graph (what gets installed)

```toml
# pyproject.toml — developer specifies ranges
[tool.poetry.dependencies]
python = "^3.9"
fastapi = "^0.104"
sqlalchemy = "^2.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
mypy = "^1.0"
ruff = "^0.1"
```

```bash
# poetry.lock is auto-generated — contains exact versions + hashes
# fastapi 0.104.1 sha256:abc123...
# sqlalchemy 2.0.23 sha256:def456...
```

**Key commands:**
```bash
poetry add requests              # add runtime dep
poetry add --group dev pytest    # add dev dep
poetry install                   # install from lock file (reproducible)
poetry install --without dev     # production install (no dev tools)
poetry update requests           # update one package, regenerate lock
poetry show --tree               # visualize dependency tree
poetry publish --build           # build and upload to PyPI
poetry run pytest                # run in poetry's venv
poetry env use python3.11        # switch Python version
```

**Lock file for apps vs libraries:**
- Applications (deployed services): commit `poetry.lock` — everyone gets identical environments
- Libraries (published to PyPI): commit `poetry.lock` for dev reproducibility, but it is NOT included in the wheel; users resolve their own environments from `pyproject.toml`

---

## Q5: Explain how `ruff` compares to the traditional linting stack

**Answer:**

Traditional Python linting used multiple separate tools:
- `flake8` (PEP 8 style errors)
- `isort` (import sorting)
- `pyupgrade` (modernize syntax)
- `bandit` (security)
- `pydocstyle` (docstring conventions)
- `flake8-bugbear` (bug patterns)

Each runs as a separate process with separate parsing passes. Collectively, they might take 30-60 seconds on a medium codebase.

`ruff` is a single tool written in Rust that replaces all of the above. It implements 700+ rules from these tools and runs in milliseconds due to parallel processing and no Python interpreter overhead.

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py39"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes (undefined names, unused imports)
    "I",    # isort (import sorting)
    "B",    # flake8-bugbear (bug patterns)
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade (modernize syntax)
    "N",    # pep8-naming
    "SIM",  # flake8-simplify
    "S",    # flake8-bandit (security)
]
ignore = ["E501"]  # ignore line length (handled by ruff format)

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]  # allow assert in tests
```

**ruff vs black:**
`ruff format` is a black-compatible formatter also written in Rust, ~100x faster than black. It produces identical output to black in most cases and can replace black entirely. Use `ruff check` for linting, `ruff format` for formatting.

```bash
ruff check src/ --fix          # lint and auto-fix
ruff format src/               # format (black replacement)
ruff check src/ --select I --fix  # just fix import order
```
