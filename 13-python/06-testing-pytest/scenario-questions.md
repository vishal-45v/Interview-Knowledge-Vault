# Chapter 06: Testing & pytest — Scenario Questions

1. Your team has a `PaymentService` class that calls an external Stripe API to charge customers. Unit tests are hitting the real API during the CI run, sometimes failing due to network issues and always taking 8+ seconds. How would you restructure the tests so they are fast, deterministic, and do not touch the network? Walk through the patching strategy you would use.

2. You have a pytest suite of 400 tests. A new developer joins and notices that several tests share a complex database setup that takes 2 seconds each time it runs, inflating total suite time to 14 minutes. The setup creates the same schema every time. How would you use fixtures and their scopes to fix this, and what tradeoffs do you accept?

3. A colleague writes this test and it passes, but the code has a bug that causes the wrong email to be sent under certain edge cases. The test covers the happy path but misses the edge case. How would you use `@pytest.mark.parametrize` and/or `hypothesis` to harden this test?

4. You are reviewing a PR and see that `mock.patch('myapp.services.email.smtplib.SMTP')` is used. The tests pass locally but fail in CI with `AttributeError: <module 'myapp.services.email'> does not have attribute 'smtplib'`. What is the likely cause and how would you fix the patch target?

5. Your Django application has a `send_weekly_digest` management command that reads users from the database, renders an email template, and sends via SendGrid. You want 100% test coverage on this command without touching the DB or making HTTP calls. Describe the test setup: what you mock, what fixtures you use, and how you assert the right output was sent.

6. A data pipeline function reads a CSV file, transforms each row, and writes to a Parquet file. You need to test it without creating real files on disk. How would you use `tmp_path`, `monkeypatch`, and/or `StringIO` to make this test hermetic?

7. Your team's CI pipeline takes 25 minutes. The CTO asks you to get it under 8 minutes without deleting tests. Describe a strategy using pytest markers, pytest-xdist parallel execution, test selection, and coverage scoping to achieve this.

8. You need to test a class that uses `datetime.datetime.now()` internally to stamp records. The result is non-deterministic. How do you freeze time in the test? Compare using `monkeypatch`, `unittest.mock.patch`, and the `freezegun` library approaches.

9. A legacy codebase has zero tests. You are asked to add tests before refactoring a 300-line `process_order` function. The function reads from a database, calls two external APIs, sends an email, and updates Redis. Describe your strategy for adding the first tests, including which test doubles you use and how you structure the test file.

10. Your team uses `hypothesis` for property-based testing of a custom JSON serializer/deserializer. A new engineer accidentally breaks the round-trip invariant but the CI passes because the generated examples did not include the edge case. After the bug is found manually, how do you ensure `hypothesis` will catch this class of bug in the future? What is the `@example` decorator for?
