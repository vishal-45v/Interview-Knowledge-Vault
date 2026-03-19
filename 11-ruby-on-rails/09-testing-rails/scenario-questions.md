# Chapter 09 — Testing Rails: Scenario Questions

---

**Scenario 1:** Your test suite takes 18 minutes to run. The team is avoiding running tests because they're too slow. Walk through how you would profile and optimize the test suite.

---

**Scenario 2:** You need to test a controller action that sends an email and enqueues a background job. Write a complete request spec that verifies: correct HTTP status, correct response body, email enqueued (not sent), and job enqueued.

---

**Scenario 3:** A developer added a test that uses `let(:user) { create(:user) }` but the test doesn't use `user`. The test still passes but is slower than it needs to be. Explain why and fix it. Now imagine another test in the same group does use `user` — does `let` help or hurt?

---

**Scenario 4:** You're testing a service object that makes an HTTP call to a third-party API. The API requires authentication and sometimes returns rate-limit errors. How do you test all the scenarios without making real HTTP calls?

---

**Scenario 5:** Your system tests intermittently fail with "element not found" or "element not visible" errors, but the feature works fine manually. These are called "flaky tests." What are the common causes and how do you fix them?

---

**Scenario 6:** You need to test a complex financial calculation that depends on: current time, exchange rates from an external API, and user account balance. Write a unit test that tests the calculation in isolation from all external dependencies.

---

**Scenario 7:** Your factory has a deeply nested association: `User → Organization → Subscription → Plan`. Every test that creates a user with `:create` hits the database 4 times. How do you optimize factory performance?

---

**Scenario 8:** You're building a test for a multi-step form wizard (3 steps). Write a system test that fills out all steps, submits, and verifies the final state in the database and the confirmation page.

---

**Scenario 9:** Your test creates data in a `before(:all)` block (shared across examples) but individual tests modify the data. Some tests pass or fail depending on the order they run. What's the problem and how do you fix it?

---

**Scenario 10:** You need to add test coverage for a legacy controller that has no tests. The controller is 300 lines with complex conditional logic. What is your strategy for adding tests incrementally without refactoring the controller first?

---

**Scenario 11:** Write a complete spec for a Pundit policy that checks: owner can update, admin can update, other users cannot, unauthenticated users are rejected.

---

**Scenario 12:** Your team wants to add mutation testing to the project. What is mutation testing? How does it differ from code coverage? What tool would you use and what would it reveal?

---
