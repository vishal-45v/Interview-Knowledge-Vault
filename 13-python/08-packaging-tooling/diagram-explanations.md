# Chapter 08: Packaging & Tooling — Diagram Explanations

## Diagram 1: Python Package Build Flow (PEP 517/518)

```
SOURCE TREE                    BUILD PROCESS               DISTRIBUTION
─────────────                  ─────────────               ────────────

mylib/                         pip install build
├── pyproject.toml    ────────►  python -m build
├── src/                           │
│   └── mylib/                     │ 1. Read [build-system]
│       ├── __init__.py             │    requires = ["setuptools"]
│       ├── models.py               │    backend = "setuptools..."
│       └── utils.py                │
├── tests/                          │ 2. Install build dependencies
│   └── test_models.py              │    in isolated venv
└── README.md                       │
                                    │ 3. Call backend's
                                    │    build_wheel() and
                                    │    build_sdist() APIs
                                    │
                                    ▼
                              dist/
                              ├── mylib-1.0.0-py3-none-any.whl
                              └── mylib-1.0.0.tar.gz
                                    │
                                    │ twine upload dist/*
                                    ▼
                              PyPI (or private index)
                                    │
                                    │ pip install mylib
                                    ▼
                              User's site-packages/
                              mylib/  (extracted from .whl)
```

---

## Diagram 2: `pyproject.toml` Structure Map

```
pyproject.toml
│
├── [build-system]
│   ├── requires = [...]          ← tools needed to build (e.g., setuptools)
│   └── build-backend = "..."    ← which backend does the building
│
├── [project]                    ← PEP 621 project metadata
│   ├── name
│   ├── version
│   ├── description
│   ├── requires-python
│   ├── dependencies = [...]     ← runtime deps (ranges OK for libraries)
│   ├── optional-dependencies    ← pip install pkg[extra]
│   │   ├── postgres = [...]
│   │   └── dev = [...]
│   ├── scripts                  ← CLI entry points
│   └── urls
│
├── [tool.ruff]                  ← ruff linter config
├── [tool.mypy]                  ← mypy type checker config
├── [tool.pytest.ini_options]    ← pytest config
├── [tool.coverage.run]          ← coverage config
└── [tool.poetry]                ← poetry-specific metadata (if using poetry)
    ├── [tool.poetry.dependencies]
    └── [tool.poetry.group.dev.dependencies]
```

---

## Diagram 3: Virtual Environment Anatomy

```
/usr/bin/python3.11               ← system Python (shared)
         │
         │ symlinked
         ▼
myproject-venv/
├── bin/
│   ├── python  ──────────────── symlink → /usr/bin/python3.11
│   ├── python3 ──────────────── symlink → /usr/bin/python3.11
│   ├── pip     ──────────────── wrapper script
│   ├── pytest  ──────────────── generated entry point wrapper
│   └── mycli   ──────────────── generated entry point wrapper
├── lib/
│   └── python3.11/
│       └── site-packages/       ← ISOLATED packages (not shared)
│           ├── requests/
│           ├── pytest/
│           └── mypackage.egg-link  ← editable install pointer
├── include/
└── pyvenv.cfg                   ← records base Python path
    home = /usr/bin
    version = 3.11.5
    include-system-site-packages = false

KEY ISOLATION POINT:
  ┌─────────────────────────────────────────┐
  │  PATH = myproject-venv/bin:$PATH        │
  │  When you type "python", shell finds    │
  │  myproject-venv/bin/python first        │
  │  which symlinks to system python3.11    │
  │  but loads myproject-venv/site-packages │
  └─────────────────────────────────────────┘
```

---

## Diagram 4: Entry Points — From `pyproject.toml` to CLI Command

```
1. DEVELOPER DEFINES in pyproject.toml:
   ─────────────────────────────────────
   [project.scripts]
   awesome-cli = "mylib.cli:main"
            │              │
            │              └── function "main" in module "mylib.cli"
            └── name of the command

2. PIP INSTALLS and GENERATES wrapper:
   ────────────────────────────────────
   venv/bin/awesome-cli:
   ┌─────────────────────────────────────────┐
   │ #!/path/to/venv/bin/python              │
   │ import re, sys                          │
   │ from mylib.cli import main              │
   │ if __name__ == '__main__':              │
   │     sys.exit(main())                    │
   └─────────────────────────────────────────┘

3. USER RUNS:
   ──────────
   $ awesome-cli --verbose report.csv

   Shell finds: venv/bin/awesome-cli  (because venv/bin is in PATH)
              ↓
   Executes Python wrapper
              ↓
   Imports mylib.cli
              ↓
   Calls main(["--verbose", "report.csv"])
              ↓
   sys.exit(return_code)

4. PLUGIN SYSTEMS use group entry points:
   ──────────────────────────────────────
   [project.entry-points."myapp.plugins"]
   my-plugin = "mypluginpkg.plugin:MyPlugin"

   # Host app discovers plugins at runtime:
   import importlib.metadata
   for ep in importlib.metadata.entry_points(group="myapp.plugins"):
       plugin_class = ep.load()   # dynamically loads MyPlugin
```

---

## Diagram 5: Dependency Version Specifiers (PEP 440)

```
VERSION SPECIFIER CHEAT SHEET
─────────────────────────────────────────────────────────────────────

Specifier     Meaning                         Example Match Range
─────────────────────────────────────────────────────────────────────
==2.3.1       Exact version                   Only 2.3.1
==2.3.*       Prefix match                    2.3.0, 2.3.1, 2.3.99
!=2.3.1       Exclusion                       Anything except 2.3.1
>=2.3         At least                        2.3, 2.4, 3.0, ...
<=2.3         At most                         2.3, 2.2, 1.0, ...
>2.3          Strictly greater                2.4, 3.0, ...
<2.3          Strictly less                   2.2, 1.0, ...
~=2.3         Compatible release              >=2.3, <3.0  (MINOR compat)
~=2.3.1       Compatible release (patch)      >=2.3.1, <2.4 (PATCH compat)
─────────────────────────────────────────────────────────────────────

USE CASES:
  Library install_requires:
    "requests>=2.28,<3.0"    ← safe range
    "pydantic~=2.0"          ← compatible with 2.x, not 3.x

  Application requirements.txt (pinned):
    requests==2.31.0         ← exact for reproducibility
    pydantic==2.3.0

  Tooling (pyproject.toml dev deps):
    pytest>=7.0              ← minimum version, latest patch is fine


PEP 440 PRE-RELEASE VERSIONS:
  1.0a1   → alpha 1
  1.0b2   → beta 2
  1.0rc1  → release candidate 1
  1.0     → final release

  By default, pip skips pre-releases unless you do:
  pip install mypackage --pre
  # or: mypackage>=1.0a1  (explicit pre-release specifier)
```

---

## Diagram 6: Namespace Package vs Regular Package

```
REGULAR PACKAGE (with __init__.py)
───────────────────────────────────────────────────────────────

company_auth_sdk/
    company/
        __init__.py        ← EXISTS — this IS the "company" package
        auth/
            __init__.py
            login.py

company_billing_sdk/       ← separate repo / pip package
    company/
        __init__.py        ← ALSO EXISTS — CONFLICT!
        billing/
            __init__.py
            invoice.py

Result:
  pip install company-auth-sdk company-billing-sdk
  import company  ← Python finds FIRST "company" on sys.path
                    and IGNORES the second one
  from company.billing import invoice  ← ImportError! (auth was found first)


NAMESPACE PACKAGE (without __init__.py in the namespace root)
──────────────────────────────────────────────────────────────

company_auth_sdk/
    company/               ← NO __init__.py here
        auth/
            __init__.py    ← __init__.py exists only in sub-packages
            login.py

company_billing_sdk/
    company/               ← NO __init__.py here
        billing/
            __init__.py
            invoice.py

sys.path = [..., "company_auth_sdk/", "company_billing_sdk/"]

Result:
  Python merges BOTH "company/" directories into one namespace package
  from company.auth import login       ← works!
  from company.billing import invoice  ← works!

DETECTION:
  import company
  print(company.__path__)
  # _NamespacePath(['company_auth_sdk/company', 'company_billing_sdk/company'])
```
