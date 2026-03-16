# Testing — Structured Answers

## 1. What Is the Difference Between a Mock, Stub, Spy, and Fake?

These are all types of test doubles — objects that stand in for real dependencies during testing. Gerard Meszaros coined the canonical definitions:

**Dummy**: An object that is passed but never used. Its only purpose is to satisfy a parameter requirement. `null` often serves as a dummy when the parameter is irrelevant to the test.

**Stub**: Returns pre-configured answers to calls during the test. It does not record calls or verify behavior. `when(repo.findById(1L)).thenReturn(Optional.of(user))` is a stub.

**Mock**: Pre-programmed with expectations of calls it should receive. Verifies that the expected interactions occurred. `verify(emailService).send(any())` uses a mock.

**Spy**: A wrapper around a real object. Real methods execute by default; you can selectively stub or verify individual methods. Mockito's `@Spy` wraps an actual instance.

**Fake**: A working implementation that takes shortcuts not suitable for production. An in-memory `HashMap`-based repository, or an H2 database, is a fake for a production PostgreSQL database.

```java
// Stub — returns a canned value, no verification
when(userRepo.findById(1L)).thenReturn(Optional.of(testUser));

// Mock — records calls, assertions about interactions
verify(auditLog, times(1)).record(any(AuditEvent.class));

// Spy — wraps real object, overrides one method
@Spy
List<String> list = new ArrayList<>();
doReturn(99).when(list).size(); // only size() is overridden; add() is real

// Fake — simplified working implementation
public class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    public Optional<User> findById(Long id) { return Optional.ofNullable(store.get(id)); }
    public User save(User u) { store.put(u.getId(), u); return u; }
}
```

In practice, most developers use "mock" to mean any test double. In code and interviews, the distinction matters when choosing the right tool.

---

## 2. How Does Mockito Work Internally?

Mockito uses byte-code generation and a thread-local invocation recorder to intercept calls and match them to stubs or record them for verification.

**Creating a mock**: Mockito uses `ByteBuddy` (formerly CGLIB/javassist) to generate a subclass of the class (or implementation of the interface) at runtime. Every method in this generated subclass calls into Mockito's `InvocationHandler` instead of the real code.

**Stubbing** (`when(...).thenReturn(...)`):
1. `when(mock.findById(1L))` calls `mock.findById(1L)`, which is intercepted by the handler and recorded on a thread-local `OngoingStubbing` stack.
2. `.thenReturn(Optional.of(user))` pops the last recorded invocation and stores it in a `StubbingRegistry` with the return value.

**Execution**: When `findById(1L)` is called in the actual test, the handler checks the `StubbingRegistry` for a matching stub. If found, it returns the configured value. All invocations are recorded in an `InvocationContainer`.

**Verification** (`verify(mock).findById(1L)`):
1. `verify(mock)` puts the mock into verification mode.
2. `findById(1L)` is intercepted; Mockito checks `InvocationContainer` for matching recorded invocations.
3. If the expected count matches, verification passes; otherwise `WantedButNotInvoked` or `TooManyActualInvocations` is thrown.

```java
// Mockito internal flow (simplified)
MockInterceptor:
  received call: findById(1L)
  → check if in "recording" mode (called from when()) → store in OngoingStubbing
  → check if in "verification" mode (called from verify()) → match against invocations
  → otherwise → look up in StubbingRegistry → return configured value
```

**Limitation**: Mockito cannot mock `final` classes or `static` methods without the `mockito-inline` module, because byte-code subclassing requires non-final classes.

---

## 3. When Should You Use @Mock vs @InjectMocks vs @Spy?

**@Mock**: Use for every dependency of the class under test that you want to control and verify. The mock returns null/default values unless you stub specific methods.

**@InjectMocks**: Use on exactly one class per test — the class whose behavior you are testing. Mockito instantiates it and injects all `@Mock` and `@Spy` fields. This class runs real production code.

**@Spy**: Use when you want mostly real behavior from a collaborator but need to override or verify one or two specific methods. Common use cases: partial testing of a legacy class, overriding one method that calls an external service.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    OrderRepository orderRepo;     // fake — all methods return null by default

    @Mock
    PaymentGateway paymentGateway; // fake — controls payment behavior in tests

    @Spy
    PriceCalculator calculator = new PriceCalculator(); // mostly real, but can stub one method

    @InjectMocks
    OrderService orderService;     // REAL class under test — gets mocks injected

    @Test
    void placeOrder_chargesCorrectAmount() {
        Order order = new Order(List.of(new Item("Widget", 9.99)));
        when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.SUCCESS);

        orderService.placeOrder(order, creditCard);

        verify(paymentGateway).charge(eq(creditCard), eq(new BigDecimal("9.99")));
        verify(orderRepo).save(any(Order.class));
    }
}
```

A common mistake is putting `@InjectMocks` on a mock — only put it on the class you are testing. Another mistake is creating `@Spy` on an interface — spies must wrap a concrete instance.

---

## 4. How Do You Test a Method That Calls Another Method in the Same Class?

This is the self-invocation problem. If `methodA()` calls `methodB()` within the same class, and you want to stub `methodB()` while testing `methodA()`, you have a few options:

**Option 1 (preferred): Extract methodB to a collaborator class**

```java
// Before — hard to test in isolation
class ReportService {
    public Report generateReport(long userId) {
        List<Order> orders = fetchOrders(userId);  // calls self
        return buildReport(orders);
    }
    private List<Order> fetchOrders(long userId) { /* DB call */ }
}

// After — collaborator is injectable and mockable
class ReportService {
    private final OrderFetcher orderFetcher;

    public Report generateReport(long userId) {
        List<Order> orders = orderFetcher.fetch(userId);  // mockable!
        return buildReport(orders);
    }
}
```

**Option 2: Use @Spy to partially stub the class under test**

```java
@ExtendWith(MockitoExtension.class)
class ReportServiceTest {
    @Spy
    @InjectMocks
    ReportService reportService;  // spy wraps real instance

    @Test
    void generateReport_buildsReportFromOrders() {
        List<Order> fakeOrders = List.of(new Order(...));
        doReturn(fakeOrders).when(reportService).fetchOrders(anyLong());  // stub self-call

        Report report = reportService.generateReport(1L);
        assertNotNull(report);
    }
}
```

Option 2 works but is a design smell — if you need it, Option 1 is the better long-term solution.

---

## 5. What Is ArgumentCaptor and When Is It Useful?

`ArgumentCaptor` captures the actual argument passed to a mocked method so you can make assertions on it. It is useful when the object is created internally by the class under test and you cannot easily construct the same object for `eq()` matching.

```java
@Test
void createUser_sendsWelcomeEmailWithCorrectDetails() {
    ArgumentCaptor<EmailMessage> emailCaptor =
        ArgumentCaptor.forClass(EmailMessage.class);

    userService.registerUser("Alice", "alice@example.com");

    verify(emailSender).send(emailCaptor.capture());

    EmailMessage sentEmail = emailCaptor.getValue();
    assertThat(sentEmail.getTo()).isEqualTo("alice@example.com");
    assertThat(sentEmail.getSubject()).isEqualTo("Welcome to our platform, Alice!");
    assertThat(sentEmail.getBody()).contains("confirm your email address");
}
```

`ArgumentCaptor` is also useful for capturing multiple invocations:

```java
// Service sends an email for each user in a batch
userService.sendMonthlyNewsletter(List.of(user1, user2, user3));

ArgumentCaptor<EmailMessage> captor = ArgumentCaptor.forClass(EmailMessage.class);
verify(emailSender, times(3)).send(captor.capture());

List<EmailMessage> allEmails = captor.getAllValues();
assertThat(allEmails).hasSize(3);
assertThat(allEmails).extracting(EmailMessage::getTo)
    .containsExactlyInAnyOrder("u1@x.com", "u2@x.com", "u3@x.com");
```

**When to prefer `argThat` over ArgumentCaptor**: if your assertion is simple (e.g., just checking one field), `argThat` with a lambda is more concise. Use ArgumentCaptor when you need multiple assertions on the captured object.

---

## 6. How Do You Test Spring Boot REST Controllers with MockMvc?

`MockMvc` performs a simulated HTTP request through the full Spring MVC stack (filters, interceptors, argument resolvers, response converters) without starting a real HTTP server.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    UserService userService;  // replace real service with mock in Spring context

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void getUser_returnsUserJson_whenFound() throws Exception {
        User user = new User(1L, "Alice", "alice@example.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));

        mockMvc.perform(get("/api/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    void getUser_returns404_whenNotFound() throws Exception {
        when(userService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    void createUser_returnsCreated_withLocation() throws Exception {
        CreateUserRequest req = new CreateUserRequest("Bob", "bob@example.com");
        User created = new User(2L, "Bob", "bob@example.com");
        when(userService.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/api/users/2")));
    }

    @Test
    void createUser_returns400_whenEmailBlank() throws Exception {
        CreateUserRequest req = new CreateUserRequest("Bob", "");

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isBadRequest());
    }
}
```

For testing with security:

```java
@Test
@WithMockUser(roles = "ADMIN")
void deleteUser_succeeds_withAdminRole() throws Exception {
    mockMvc.perform(delete("/api/users/1"))
        .andExpect(status().isNoContent());
}
```

---

## 7. How Do You Test Database Repositories with @DataJpaTest?

`@DataJpaTest` loads only the JPA layer: entities, repositories, `EntityManager`, and an embedded database (H2 by default). Services and controllers are not loaded.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    UserRepository userRepository;

    @Autowired
    TestEntityManager entityManager;  // JPA test helper for setup

    @Test
    void findByEmail_returnsUser_whenEmailExists() {
        // Arrange — persist via TestEntityManager
        User user = new User();
        user.setName("Alice");
        user.setEmail("alice@example.com");
        entityManager.persistAndFlush(user);  // flush to make visible to repo query
        entityManager.clear();                 // detach to ensure repo loads fresh from DB

        // Act
        Optional<User> found = userRepository.findByEmail("alice@example.com");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }

    @Test
    void findActiveUsersOlderThan_returnsOnlyMatching() {
        entityManager.persist(activeUser("Bob", 30));
        entityManager.persist(activeUser("Carol", 25));
        entityManager.persist(inactiveUser("Dave", 35));
        entityManager.flush();

        List<User> result = userRepository.findActiveUsersOlderThan(28);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("Bob");
    }
}
```

For real database dialect (PostgreSQL), replace the embedded DB with Testcontainers:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryPostgresTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

## 8. What Is the Test Pyramid and How Does It Guide Test Strategy?

The test pyramid is a model (coined by Mike Cohn) describing the ideal distribution of tests across three layers: many fast unit tests at the base, fewer integration tests in the middle, and a small number of end-to-end tests at the top.

**Unit tests (base)**:
- Scope: one class, all dependencies mocked.
- Speed: milliseconds per test.
- Count: hundreds to thousands.
- Value: rapid feedback, precise failure localization.

**Integration tests (middle)**:
- Scope: multiple real components (e.g., service + real database).
- Speed: seconds per test.
- Count: dozens to hundreds.
- Value: verifies that components integrate correctly (real SQL queries, real JPA mappings).

**End-to-end tests (top)**:
- Scope: full system — real HTTP, real database, real queue.
- Speed: tens of seconds to minutes.
- Count: tens.
- Value: verifies the entire user journey; catches integration gaps missed by lower layers.

**Practical guidance**:
- Write a unit test for every business logic path (happy path, edge cases, error handling).
- Write an integration test for every database query that uses custom JPQL/SQL — verify the query actually returns the right data.
- Write an E2E test for the most critical user journeys (checkout flow, authentication flow) but not for every feature.

**Anti-patterns to avoid**:
- **Ice-cream cone**: many E2E tests, few unit tests — slow and brittle.
- **Testing implementation details in unit tests**: tests that break on every refactoring.
- **No integration tests**: unit tests all pass, but the system fails at the database layer.

---

## 9. How Do You Handle Flaky Tests?

A flaky test is one that sometimes passes and sometimes fails for reasons unrelated to code correctness. Flaky tests destroy team confidence in the test suite and should be treated as bugs.

**Common causes and fixes**:

**1. Timing/async issues**:
```java
// Flaky — arbitrary sleep
Thread.sleep(500); // might not be enough
assertThat(asyncResult).isNotNull();

// Robust — await with timeout
await().atMost(5, SECONDS)
       .until(() -> asyncProcessor.isComplete());
// Uses Awaitility library
```

**2. Test ordering dependency**:
```java
// Flaky — depends on another test leaving data in DB
// Fix: use @BeforeEach to set up own data, @AfterEach to clean up
// Or use @Transactional to roll back after each test
```

**3. Non-deterministic data (random IDs, current time)**:
```java
// Flaky — uses real clock
assertEquals(LocalDate.now().toString(), user.getCreatedDate()); // fails tomorrow!

// Robust — inject a fixed clock
Clock fixedClock = Clock.fixed(Instant.parse("2024-01-15T10:00:00Z"), ZoneOffset.UTC);
UserService service = new UserService(repo, fixedClock);
```

**4. Port conflicts in integration tests**:
```java
// Use random ports
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```

**5. Shared mutable static state**: Avoid static collections or fields in test classes; prefer instance fields reset in `@BeforeEach`.

**Process**: When a flaky test is identified, mark it with `@Disabled("flaky — tracked in TICKET-123")` to prevent false CI failures, immediately file a ticket to fix it, and do not re-enable until the root cause is fixed.

---

## 10. What Is Testcontainers and Why Is It Better Than H2 for Integration Tests?

Testcontainers is a Java library that programmatically manages Docker containers for tests. You declare a container in your test class, and Testcontainers pulls the Docker image, starts the container before your tests, provides connection details, and stops the container after tests complete.

**Why H2 falls short**:
- Different SQL dialect: H2 does not support `ON CONFLICT DO UPDATE`, `JSONB`, `ARRAY` types, or PostgreSQL window functions.
- Different behavioral quirks: sequences, auto-increment, case sensitivity, and constraint checking differ.
- A test suite that passes on H2 can fail on production PostgreSQL.

**Testcontainers setup**:

```java
@Testcontainers
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class IntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("testuser")
            .withPassword("testpass");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    void upsertProduct_usesOnConflict() {
        // This SQL fails on H2 but works on real PostgreSQL
        productRepo.upsert(new Product("SKU-001", 29.99));
        productRepo.upsert(new Product("SKU-001", 34.99)); // update price
        assertThat(productRepo.findBySku("SKU-001").getPrice())
            .isEqualByComparingTo("34.99");
    }
}
```

**Performance optimization**: Declare the container as `static` so it is shared across all tests in the class (started once). Use `@Container` on a static field for class-level reuse. Testcontainers also supports reusing containers across test runs with `withReuse(true)` — the container stays running between test executions during development, reducing startup time to near zero.

Testcontainers supports not just PostgreSQL but also Redis, Kafka, Elasticsearch, MongoDB, RabbitMQ, and any Docker image — making it the standard for realistic integration testing across the entire dependency stack.
