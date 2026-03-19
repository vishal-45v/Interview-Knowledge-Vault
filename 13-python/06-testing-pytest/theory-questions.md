# Chapter 06: Testing & pytest — Theory Questions

1. What is pytest's test discovery mechanism? What naming conventions must files, classes, and functions follow for pytest to automatically collect them?

2. Explain assert rewriting in pytest. How does pytest produce rich, introspective failure messages from a plain `assert` statement without any extra messaging from the developer?

3. What is a pytest fixture? How does it differ from a `setUp`/`tearDown` method in `unittest`? What are the four fixture scopes (`function`, `class`, `module`, `session`) and when is each appropriate?

4. What is `conftest.py`? Where can it be placed in a project, and how does pytest resolve fixture lookup when there are multiple `conftest.py` files in nested directories?

5. Explain the difference between `@pytest.mark.parametrize` and putting a loop inside a test function. Why does parametrize give better output, better isolation, and integrate better with coverage tools?

6. What is a factory fixture? Give an example scenario where returning a callable from a fixture is superior to returning an object directly.

7. Describe the `tmp_path` and `tmp_path_factory` built-in fixtures. What do they provide, how do their scopes differ, and how does pytest clean them up?

8. What does `monkeypatch` do in pytest? List five distinct methods it provides (`setattr`, `delattr`, `setenv`, `delenv`, `syspath_prepend`) and a concrete use case for each.

9. What is `capsys`? How do you use it to assert that a function printed specific text to stdout? What is `capfd` and how is it different?

10. Explain the difference between `unittest.mock.Mock` and `unittest.mock.MagicMock`. When would you prefer a plain `Mock`, and when does `MagicMock` save you configuration work?

11. What is `spec` (and `spec_set`) in `unittest.mock`? Why is using spec considered a best practice, and what problem does it prevent that you would not catch without it?

12. Explain `side_effect` on a Mock object. What are the three types of values you can assign to `side_effect` and what does each one do?

13. How does `patch` differ from `patch.object`? Explain the "patch where it is used, not where it is defined" rule with an example. What happens if you patch the wrong location?

14. What is `pytest-mock`'s `mocker` fixture? How does it improve on using `unittest.mock.patch` as a context manager or decorator in pytest tests? What does `mocker.spy` do?

15. What is branch coverage and how does it differ from line coverage? Give an example where line coverage reports 100% but branch coverage exposes a gap.

16. What is property-based testing? How does the `hypothesis` library generate and shrink test cases? What advantage does it provide over hand-written example-based tests?

17. Distinguish between the four classic test doubles: **stub**, **fake**, **spy**, and **mock**. Give a concrete Python scenario for each. Which of these does `unittest.mock.Mock` most closely represent?

18. What is the difference between integration tests and unit tests from a practical standpoint? Is test size, execution speed, or isolation the defining factor for a "unit" test?

19. How do `@pytest.mark.skip`, `@pytest.mark.skipif`, and `@pytest.mark.xfail` behave? What is the difference between a test that is `xfail` and one that passes unexpectedly (`xpass`)? How do you make `xpass` cause the suite to fail?

20. Explain how `pytest-cov` generates coverage reports. What is `--cov-fail-under`, when would you use it in CI, and what is the danger of chasing 100% coverage without also measuring branch coverage?
