# Testing — Follow-Up Traps

## Trap 1: @Mock vs @MockBean — When to Use Which?

**Trap**: Using `@MockBean` everywhere, even in pure unit tests.

**Reality**: `@Mock` is a Mockito annotation. It creates a mock object and injects it via `@InjectMocks` — no Spring context is involved. Tests run in milliseconds.

`@MockBean` is a Spring Boot Test annotation. It creates a Mockito mock AND registers it in the Spring ApplicationContext, replacing any existing bean of the same type. It requires loading a Spring context, making the test 10–100× slower.

- Use `@Mock` + `@InjectMocks` + `@ExtendWith(MockitoExtension.class)` for unit tests of a single class.
- Use `@MockBean` only when you need a Spring context (`@WebMvcTest`, `@SpringBootTest`) and need to replace a real bean with a mock.

```java
// Pure unit test — no Spring, fast
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;          // Mockito mock, no Spring
    @InjectMocks UserService service;   // Mockito creates and injects
}

// Spring web layer test — needs Spring context
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService service;      // Spring context mock replacement
}
```

---

## Trap 2: @Spy vs @Mock — What's the Difference?

**Trap**: "They do the same thing — both allow stubbing."

**Reality**: A `@Mock` creates a completely fake object. Every method returns null/0/false by default unless stubbed. No real code ever runs.

A `@Spy` wraps a real instance. Real code runs by default unless you explicitly stub specific methods. If you forget to stub a method on a spy, the real implementation executes — including any side effects.

```java
// @Mock — all methods return null by default
@Mock List<String> mockList;
mockList.add("hello");   // does nothing (fake)
System.out.println(mockList.size()); // prints 0 (not 1 — no real state)

// @Spy — wraps a real ArrayList, real methods execute
@Spy List<String> spyList = new ArrayList<>();
spyList.add("hello");   // REAL add() — modifies the real list
System.out.println(spyList.size()); // prints 1

// Stubbing a spy (override specific method, keep real behavior elsewhere)
doReturn(10).when(spyList).size(); // override just size()
spyList.add("world");  // still real
System.out.println(spyList.size()); // 10 (stubbed)
```

Use `@Spy` when you want to test a real object but stub one or two expensive/external calls within it.

---

## Trap 3: verify() Checking Interaction Count

**Trap**: Forgetting that `verify(mock).method()` defaults to `times(1)` and failing to check important interaction counts.

**Reality**: Mockito's `verify(mock).method()` is shorthand for `verify(mock, times(1)).method()`. It asserts the method was called exactly once. If it was called zero times or two times, the verification fails.

```java
// These are equivalent:
verify(emailService).sendWelcome(user);
verify(emailService, times(1)).sendWelcome(user);

// Verify never called:
verify(emailService, never()).sendWelcome(any());

// Verify at least once:
verify(emailService, atLeastOnce()).sendWelcome(any());

// Verify at most twice:
verify(emailService, atMost(2)).sendWelcome(any());

// Verify called exactly 3 times:
verify(emailService, times(3)).sendWelcome(any());

// Verify no more interactions after specific calls:
verifyNoMoreInteractions(emailService);
// Fails if ANY method on emailService was called beyond what was verified
```

A common mistake is calling `verify()` and trusting the test passes, not realizing the method was called twice when it should only be called once (e.g., sending two emails when one is expected).

---

## Trap 4: Testing Void Methods with Mockito

**Trap**: "I can't test a void method because there's nothing to return."

**Reality**: Void methods are tested by verifying their side effects — either state changes or interactions with collaborators.

```java
// Approach 1: verify the interaction
service.deactivateUser(userId);
verify(userRepository).save(argThat(u -> !u.isActive()));
verify(emailService).sendDeactivationEmail(userId);

// Approach 2: assert state change
user.setActive(true);
service.deactivate(user);
assertFalse(user.isActive());

// Stubbing void methods to throw exceptions (for testing error handling):
doThrow(new RuntimeException("DB down"))
    .when(userRepository).save(any());

// Stubbing void methods with a callback (doAnswer):
doAnswer(invocation -> {
    User u = invocation.getArgument(0);
    u.setId(42L); // simulate DB-assigned ID
    return null;
}).when(userRepository).save(any(User.class));
```

---

## Trap 5: doReturn vs thenReturn — What's the Difference?

**Trap**: "They both stub a return value — same thing."

**Reality**: The difference matters for `@Spy` and protected methods.

`when(mock.method()).thenReturn(value)` actually calls `mock.method()` during the stubbing setup. For a `@Mock`, the call is intercepted. For a `@Spy`, the real method executes during setup — this can cause `NullPointerException` or unwanted side effects.

`doReturn(value).when(mock).method()` sets up the stub without calling the method. The method is only intercepted on the actual test call, not during setup.

```java
@Spy
UserService realService = new UserService(repo);

// DANGEROUS on spy — real getUserCount() called during stubbing
when(realService.getUserCount()).thenReturn(10);  // may throw if DB not ready

// SAFE on spy — no actual call during stubbing setup
doReturn(10).when(realService).getUserCount();    // always safe

// For mocks, both are equivalent in practice:
when(mockRepo.findAll()).thenReturn(List.of(user1));
doReturn(List.of(user1)).when(mockRepo).findAll();
```

---

## Trap 6: @ExtendWith(MockitoExtension.class) vs MockitoAnnotations.openMocks()

**Trap**: "I can use either — they do the same thing."

**Reality**: Both initialize `@Mock`, `@Spy`, `@Captor`, and `@InjectMocks` annotations. The difference is lifecycle and cleanup.

`@ExtendWith(MockitoExtension.class)` is a JUnit 5 extension. It automatically initializes mocks before each test and validates mock usage after each test (catches unnecessary stubbing with `UnnecessaryStubbingException`). It is the modern, preferred approach for JUnit 5.

`MockitoAnnotations.openMocks(this)` is a programmatic approach, typically called in `@BeforeEach`. It initializes mocks but does not automatically call `closeable.close()` after each test — you must do it manually in `@AfterEach`, or risk mock contamination across tests.

```java
// Modern JUnit 5 approach — extension handles everything
@ExtendWith(MockitoExtension.class)
class ServiceTest {
    @Mock UserRepository repo;
}

// Legacy approach — you manage lifecycle manually
class ServiceTest {
    @Mock UserRepository repo;
    private AutoCloseable closeable;

    @BeforeEach
    void setUp() { closeable = MockitoAnnotations.openMocks(this); }

    @AfterEach
    void tearDown() throws Exception { closeable.close(); }
}
```

---

## Trap 7: Testing Private Methods — Should You?

**Trap**: "I need to test private methods — let me use reflection or PowerMock."

**Reality**: The need to directly test a private method is almost always a design smell. Private methods are implementation details. If the private method does something important, it should be testable through the public interface that calls it.

If a private method is so complex it seems to need its own test, it likely deserves to be extracted into a separate class with a public method — at which point it is naturally testable.

```java
// Smell: testing private logic directly
class CryptoService {
    private String hashPassword(String password) { ... }  // complex logic
}
// Don't do: use reflection to call hashPassword() directly

// Fix: extract to a separate class
class PasswordHasher {
    public String hash(String password) { ... }  // now testable normally
}
class CryptoService {
    private final PasswordHasher hasher;
    // ...
}
```

If you truly must test private code (legacy systems), PowerMock or Mockito's inline mock maker can do it — but treat it as a temporary measure, not a pattern.

---

## Trap 8: Testcontainers vs H2 for Integration Tests

**Trap**: "H2 in-memory database is good enough for testing database code."

**Reality**: H2 is a different database with different SQL dialect, different behavior, and different constraints. Tests pass on H2 but fail in production PostgreSQL for reasons like:

- H2 does not support PostgreSQL-specific functions (`array_agg`, `jsonb_set`, `ON CONFLICT DO UPDATE`)
- Case sensitivity rules differ
- Sequence/auto-increment behavior differs
- Stored procedures syntax differs
- Transaction isolation behavior is not identical

Testcontainers starts a real PostgreSQL Docker container for your tests:

```java
@Testcontainers
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired UserRepository repo;

    @Test
    void testComplexQuery() {
        // runs against real PostgreSQL — results you can trust
    }
}
```

Testcontainers is slower than H2 but catches real database compatibility issues. Use H2 for basic CRUD tests; use Testcontainers for anything involving SQL dialect features, stored procedures, or database-specific behavior.

---

## Trap 9: @Transactional in Test Classes — Rollback Behavior

**Trap**: "My test inserts data and the next test sees it — but both are @Transactional."

**Reality**: When `@Transactional` is placed on a test class or test method, Spring wraps each test method in a transaction that is automatically rolled back after the method completes. This keeps tests isolated.

But there is a subtle trap: if your service method is also `@Transactional`, and you call it from a test that is also `@Transactional`, Spring reuses the same transaction (because the default propagation is `REQUIRED`). The rollback at the end of the test undoes everything, including the service's work.

If you need to verify database state after a `@Transactional` service method and you are using `@SpringBootTest`, you might need to either:
1. Use `@Commit` on the test to force commit (and clean up manually in `@AfterEach`)
2. Use `TestTransaction` utilities to manually control transaction boundaries

```java
@Test
@Transactional  // rolls back after test — safe for isolation
void createUser_persistsCorrectly() {
    User saved = userService.createUser("Alice", "alice@example.com");
    assertThat(saved.getId()).isNotNull();
    // No need to clean up — transaction rolls back automatically
}

@Test
@Commit  // use when you need to test behavior that spans transaction boundaries
void createUser_triggersFired() {
    userService.createUser("Bob", "bob@example.com");
    // verify DB triggers or async side effects
}
```

---

## Trap 10: Test Ordering with @TestMethodOrder

**Trap**: "Tests should be independent of order — I never need to order them."

**Reality**: Tests should be independent, but there are legitimate reasons to control order:

1. **Readability**: Ordering tests by logical flow (happy path first, then edge cases, then error cases) makes the test class easier to read.
2. **Performance**: Long-running setup tests can be ordered to run first so developers see results sooner.
3. **End-to-end scenarios**: Some integration/E2E test suites intentionally test a sequence of operations (create → update → delete).

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderWorkflowTest {
    @Test @Order(1)
    void createOrder() { ... }

    @Test @Order(2)
    void addItems() { ... }

    @Test @Order(3)
    void submitOrder() { ... }
}

// Other orderers:
@TestMethodOrder(MethodOrderer.DisplayName.class)      // alphabetical by @DisplayName
@TestMethodOrder(MethodOrderer.MethodName.class)       // alphabetical by method name
@TestMethodOrder(MethodOrderer.Random.class)           // random (for finding order dependencies)
```

Using `MethodOrderer.Random.class` is actually a great diagnostic tool — if your tests fail in random order, they have hidden ordering dependencies that need to be fixed.

---

## Trap 11: assertThrows vs try/catch in Tests

**Trap**: Using try/catch with `fail()` to test exceptions.

**Reality**: JUnit 5's `assertThrows` is cleaner and more precise:

```java
// Old style — verbose and fragile
@Test
void shouldThrowWhenUserNotFound() {
    try {
        userService.getUser(999L);
        fail("Expected UserNotFoundException");
    } catch (UserNotFoundException e) {
        assertEquals("User 999 not found", e.getMessage());
    }
}

// Modern JUnit 5 — cleaner and type-safe
@Test
void shouldThrowWhenUserNotFound() {
    UserNotFoundException ex = assertThrows(
        UserNotFoundException.class,
        () -> userService.getUser(999L)
    );
    assertThat(ex.getMessage()).contains("999");
}

// assertDoesNotThrow — verify no exception
@Test
void validUserIdShouldNotThrow() {
    assertDoesNotThrow(() -> userService.getUser(1L));
}
```

---

## Trap 12: Strict vs Lenient Stubbing in Mockito

**Trap**: Stubbing methods that are never called in a test and not noticing.

**Reality**: By default, Mockito with `MockitoExtension` enforces strict stubbing. If you stub a method (`when(repo.findById(1L)).thenReturn(...)`) but the method is never actually called during the test, Mockito throws `UnnecessaryStubbingException`. This catches dead test code and over-specification.

```java
// This will FAIL with UnnecessaryStubbingException if findById is never called
when(userRepo.findById(1L)).thenReturn(Optional.of(user));

// If you need lenient stubbing for a legitimate reason:
lenient().when(userRepo.findById(1L)).thenReturn(Optional.of(user));

// Or at class level:
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
class MyTest { ... }
```

Unnecessary stubbing often indicates that a test is testing the wrong thing, or that a refactoring has made some setup code stale.
