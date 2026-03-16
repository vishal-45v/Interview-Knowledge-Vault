# Testing — Analogy Explanations

## Unit Test

A unit test is like testing a single light bulb in isolation. You take the bulb out of the lamp, connect it directly to a power source, and check if it glows. You are not testing the lamp, the wall socket, the house wiring, or the power grid — just that one bulb. If it works in isolation, you have confidence in the bulb itself. Unit tests check that a single class or method does exactly what it is supposed to do, isolated from everything else.

---

## Integration Test

An integration test is like testing a lamp with all its parts assembled and plugged into the wall. You want to verify that the bulb, the socket, the cord, the switch, and the wall outlet all work together correctly. Individual parts might test fine in isolation but fail when combined — a bulb rated for 120V connected to a 240V socket works fine alone (with the right power supply) but fails in the actual setup. Integration tests catch problems that only appear when real components interact.

---

## Mock vs Real Object

A mock is like a crash-test dummy. When testing a car's safety systems in a collision, you do not use a real person — that would be dangerous and unrepeatable. A crash-test dummy is a stand-in that behaves enough like a real person for the test to be meaningful. Similarly, in tests you replace a real database or external API with a mock: an object that behaves like the real thing for testing purposes, is controllable, and does not have side effects. A real object is the actual production implementation — used in integration tests when you want to verify real behavior.

---

## Test Double (Mock, Stub, Spy, Fake, Dummy)

Think of making a film that requires a dangerous stunt. You have several stand-in options:

- **Dummy**: A cardboard cutout of a person placed in the background. It is never interacted with — it just fills a required slot. In testing: an object passed as an argument but never used. `null` often works.

- **Stub**: A non-acting stand-in who delivers pre-rehearsed lines when prompted. They always say the same thing regardless of context. In testing: an object that returns hard-coded values. `when(repo.findById(1)).thenReturn(Optional.of(user))`.

- **Mock**: An actor specifically hired to record and verify interactions. At the end of the scene, you can confirm "did the director call 'action' exactly once?" In testing: an object that records method calls so you can verify them with `verify()`.

- **Spy**: The real actor doing the real stunt, but with a camera specifically watching certain moves. Most behavior is real; certain interactions are observed or partially overridden. In testing: a Mockito `@Spy` wraps a real object but lets you stub or verify specific methods.

- **Fake**: A real working implementation that takes shortcuts for testing. Like building a simplified working prop instead of using the real thing — it functions but is not production-grade. An in-memory database (H2) is a fake for PostgreSQL.

---

## TDD (Test-Driven Development)

TDD is like writing a recipe before you cook the dish. Before you make a single cut or add any ingredient, you write down exactly what the finished dish should taste, look, and smell like. Then you cook to meet that specification. If the dish does not match at any point, you adjust. You know you are done when the dish matches the recipe exactly.

In coding terms: write a failing test that specifies the behavior you want (Red), write the minimum code to make it pass (Green), then clean up the code while keeping tests passing (Refactor). The test is your specification — written before the implementation.

---

## Test Pyramid

The test pyramid describes how many tests of each type you should have. Imagine a building:

- **Foundation (Unit tests)**: Thousands of small, fast, cheap tests at the base. They run in milliseconds, require no external systems, and cover individual classes and methods. Like bricks — you need many of them.

- **Middle floor (Integration tests)**: Hundreds of tests verifying that components work together. They are slower (seconds each), require a database or server, and cover the interaction between layers. Like beams — you need some, but not as many as bricks.

- **Top floor (End-to-end / UI tests)**: Tens of tests that exercise the entire system through the real UI or API. They are the slowest (minutes), the most brittle (any change anywhere can break them), and the hardest to debug. Like the roof — you need it, but you want it as light as possible.

Inverting the pyramid (many E2E tests, few unit tests) is an anti-pattern called the "ice cream cone" — it produces slow, unreliable test suites.

---

## @Mock vs @InjectMocks

`@Mock` is like hiring an actor to play a supporting role. You control their script entirely.

`@InjectMocks` is like the main character who does all the real work but gets supporting actors handed to them. Mockito creates the class under test and automatically injects all the `@Mock` objects into it (via constructor, setter, or field injection). The `@InjectMocks` object runs real production code; its collaborators are the mocks.

```
@Mock UserRepository userRepo   ← fake stand-in, you control it
@Mock EmailService emailService ← fake stand-in, you control it
@InjectMocks UserService userService ← real class, gets the mocks injected into it
```

You verify that `userService` (real) calls `userRepo` (mock) and `emailService` (mock) correctly.

---

## ArgumentCaptor

ArgumentCaptor is like a security camera that records exactly what was handed to a teller at a bank counter. You may not care that the teller called the vault — you want to know what specific envelope was handed over. `ArgumentCaptor` captures the actual argument passed to a mocked method so you can assert on it in detail.

Example: your service calls `emailService.send(emailObject)`, but `emailObject` is built internally by the service. You cannot easily construct the same object for comparison. ArgumentCaptor grabs the exact `emailObject` that was passed, and you can then assert on its fields (`email.getSubject()`, `email.getRecipient()`, etc.).

---

## Verify Behavior vs Assert State

These are two complementary testing philosophies:

**Assert state** is like checking a recipe result by tasting the dish. You do not care how it was cooked — only whether the final outcome is correct. `assertEquals(expectedResult, actualResult)`. You check the output or the changed state of objects.

**Verify behavior** is like checking the cooking process by reviewing the CCTV footage. You do not taste the dish — you verify that the chef chopped the onions (method was called), used exactly one cup of flour (called with specific arguments), and stirred three times (called exactly 3 times). `verify(service).sendEmail(expectedEmail)`. You check interactions.

Good tests usually combine both: assert the state returned by the method AND verify critical interactions (e.g., that an audit log was written).

---

## Test Isolation

Test isolation means each test is a completely independent experiment in a clean laboratory. Imagine a chemistry lab where each student gets a fresh, clean workbench and unused reagents. If one student spills something, it does not contaminate the next student's experiment.

Without isolation, tests can interfere: test A saves data to a database, test B queries that same data and gets an unexpected result, test B fails — not because of any bug in the code it is testing, but because of test A's side effects. In JUnit 5, each test method gets a new instance of the test class by default. Mockito resets mocks before each test (with `MockitoExtension`). Spring's `@Transactional` on tests rolls back database changes after each test method.
