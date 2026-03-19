# Chapter 08: Packaging & Tooling — Scenario Questions

1. You are starting a new Python library that other teams will pip-install. You need to choose between `setuptools + setup.cfg`, `poetry`, and `hatch`. The library will be published to PyPI, needs a CLI tool, and your team uses mypy strictly. Which tool do you choose, how do you configure the `pyproject.toml`, and what does your first `pyproject.toml` look like?

2. A junior developer on your team runs `pip install requests` in the project directory and commits changes to `requirements.txt` with `requests` at the latest version (no pin). Three months later, CI breaks because a new version of `requests` changed an API. How do you restructure the project's dependency management to prevent this class of problem in the future?

3. Your company has internal Python packages that must not be published to public PyPI. You need developers to be able to `pip install company-auth-sdk` seamlessly during development and in CI. Walk through setting up a private PyPI server strategy (devpi or Artifactory), including how you configure `pip` and `poetry` to look at the private index.

4. A library you maintain has grown to have optional dependencies: `psycopg2` for PostgreSQL support, `motor` for MongoDB, and `aioredis` for Redis. Some users only need one backend. How do you structure `pyproject.toml` to support `pip install mylib[postgres]`, `pip install mylib[mongo]`, and `pip install mylib[all]`?

5. Your team's pre-commit hooks run `mypy --strict` on every commit and it takes 45 seconds, causing developers to skip commits or use `git commit --no-verify`. How do you fix the pre-commit configuration to make the type checking faster without removing mypy entirely?

6. You are maintaining a Python package that runs on Python 3.8, 3.9, 3.10, 3.11, and 3.12. You need to ensure all tests pass on all versions before merging a PR. Describe the `tox` setup and the corresponding GitHub Actions CI configuration.

7. A colleague imports your internal utility module with `import utils` at the top of their file, which works fine on their machine because they ran the code from the project root. The code fails in production with `ModuleNotFoundError`. Explain what is happening, why it "works" locally, and how to fix it properly using packaging rather than `sys.path` manipulation.

8. You have a monorepo with three services: `api/`, `worker/`, and `shared/`. Each service is a separate Python application. The `shared/` package is used by both `api/` and `worker/`. You want `shared` to be installable as a package in each service's virtual environment. How do you structure this, and what is the difference between using namespace packages vs regular packages here?

9. After running `pip install -r requirements.txt` a deployment fails with a conflict: `packageA 2.0` requires `requests>=2.28`, but `packageB 1.5` requires `requests<2.28`. You cannot upgrade `packageB` (vendor lock). What are your options for resolving this, and how does `pip-tools` (`pip-compile`) help prevent this situation?

10. Your team uses `ruff` for linting but a senior engineer insists on also running `pylint` because "ruff misses important checks." You have one minute in a code review to make the case for or against this dual-linting approach. What do you say, and what specific `ruff` rule sets would you enable to cover the pylint checks the engineer is worried about?
