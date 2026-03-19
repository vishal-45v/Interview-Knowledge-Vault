# Testing — Theory Questions

> 20 theory questions on JUnit 5, Mockito, Spring Boot Test, TDD, and testing best practices for Senior Java Backend Engineers.

---

**Q1. What is the difference between unit tests, integration tests, and end-to-end tests?**

**Test Pyramid:**
```
         /\
        /E2E\          ← Few, slow, expensive (Selenium, RestAssured)
       /------\
      /  Integ  \      ← Some (Spring Boot tests, @DataJpaTest)
     /------------\
    /  Unit Tests  \   ← Many, fast, isolated (JUnit + Mockito)
   /________________\
```

**Unit tests:** Test a single class/method in isolation. All dependencies mocked. No Spring context. Fast (milliseconds).

**Integration tests:** Test interaction between components (e.g., service + repository + database). Uses real database (H2 or Testcontainers). Slower (seconds).

**End-to-end tests:** Test the full system from HTTP request to database response. Validates user flows. Very slow (minutes). Few in number.

**Rule of thumb:** Many fast unit tests, some integration tests, very few E2E tests.

---

**Q2. What are the key JUnit 5 annotations?**

```java
@Test               // Marks a test method
@BeforeEach         // Runs before each test (setup)
@AfterEach          // Runs after each test (teardown)
@BeforeAll          // Runs once before all tests in class (must be static)
@AfterAll           // Runs once after all tests (must be static)

@DisplayName("Human-readable test name")
@Disabled("reason") // Skip test

@ParameterizedTest  // Run test with multiple inputs
@ValueSource(ints = {1, 2, 3})
@CsvSource({"1,one", "2,two"})
@MethodSource("provideArguments")

@Nested             // Group related tests in inner class
@Tag("unit")        // Tag for selective test execution
@Timeout(5)         // Fail if test takes more than 5 seconds

@ExtendWith(MockitoExtension.class)   // JUnit 5 extension (Mockito)
@ExtendWith(SpringExtension.class)    // Spring test extension
```

---

**Q3. What is Mockito and what are its key features?**

Mockito is a mocking framework that creates fake objects substituting real dependencies:

```java
// Create mock
OrderRepository mockRepo = Mockito.mock(OrderRepository.class);
// Or with annotation:
@Mock OrderRepository mockRepo;  // Requires @ExtendWith(MockitoExtension.class)
@InjectMocks OrderService service;  // Creates service and injects mocks

// Stubbing — define what mock returns
when(mockRepo.findById(1L))
    .thenReturn(Optional.of(testOrder));
when(mockRepo.findById(anyLong()))
    .thenThrow(new DataAccessException("DB down"));

// Verification — check what was called
verify(mockRepo, times(1)).save(any(Order.class));
verify(mockRepo, never()).deleteById(anyLong());
verify(mockRepo).save(argThat(order ->
    order.getStatus().equals("PENDING") && order.getTotal().compareTo(BigDecimal.TEN) > 0));

// Capture arguments
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(mockRepo).save(captor.capture());
Order savedOrder = captor.getValue();
assertThat(savedOrder.getStatus()).isEqualTo("PENDING");
```

---

**Q4. What is the difference between `@Mock`, `@Spy`, and `@MockBean`?**

```java
// @Mock — complete mock, all methods return null/0/false by default
@Mock OrderRepository orderRepository;
// All calls: orderRepository.findById(1L) → Optional.empty() (default)
// Must stub everything you need

// @Spy — wraps real object, calls real methods unless stubbed
@Spy OrderService realService;
// realService.createOrder(request) → calls REAL method
// doReturn(fakeOrder).when(realService).findOrder(anyLong());  // Stub specific method

// @MockBean — Spring-specific, replaces actual bean in Spring context
@MockBean OrderRepository orderRepository;
// Replaces the real OrderRepository in Spring ApplicationContext
// Use in @SpringBootTest or @WebMvcTest
// Resets after each test by default

// @SpyBean — Spring-specific spy (wraps real bean)
@SpyBean EmailService emailService;
```

**When to use each:**
- `@Mock`: unit tests with no Spring context
- `@Spy`: want real behavior but override specific methods
- `@MockBean`: integration tests with Spring context (`@SpringBootTest`, `@WebMvcTest`)

---

**Q5. What are the Spring Boot testing annotations?**

```java
// Full application context
@SpringBootTest  // Loads complete context
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)  // With embedded server

// Slice tests — only relevant beans loaded (faster)
@WebMvcTest(OrderController.class)     // Controller layer only + security
@DataJpaTest                           // JPA + H2 in-memory (no services, no controllers)
@DataMongoTest                         // MongoDB slice
@RestClientTest(OrderService.class)    // RestTemplate/WebClient tests
@JsonTest                              // JSON serialization/deserialization

// With test containers
@Testcontainers
class IntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

**Q6. What is `@WebMvcTest` and how do you use it?**

`@WebMvcTest` loads only the MVC layer (controllers, filters, exception handlers) without starting a full application context:

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;          // Pre-configured MockMvc
    @MockBean OrderService orderService; // Mock the service layer
    @MockBean JwtTokenProvider jwt;      // Mock security components

    @Test
    void createOrder_returns201WithLocation() throws Exception {
        OrderResponse mockResponse = new OrderResponse("ORD-123", "PENDING", BigDecimal.TEN);
        when(orderService.createOrder(any())).thenReturn(mockResponse);

        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""
                        {"customerId": 1, "items": [{"productId": "P1", "quantity": 2}]}
                        """))
               .andExpect(status().isCreated())
               .andExpect(header().string("Location", containsString("/orders/ORD-123")))
               .andExpect(jsonPath("$.id").value("ORD-123"))
               .andExpect(jsonPath("$.status").value("PENDING"));

        verify(orderService).createOrder(any(CreateOrderRequest.class));
    }

    @Test
    void createOrder_withInvalidBody_returns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{}"))  // Missing required fields
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.errors").isArray());
    }
}
```

---

**Q7. What is `@DataJpaTest` and how do you test repositories?**

```java
@DataJpaTest    // In-memory H2, auto-rollback after each test
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;
    @Autowired TestEntityManager em;  // Direct EntityManager access for test setup

    @Test
    void findByCustomerAndStatus_returnsMatchingOrders() {
        // Setup via EntityManager (bypasses service layer)
        Customer customer = em.persistAndFlush(new Customer("alice@example.com"));
        Order pending = em.persistAndFlush(new Order(customer, "PENDING", BigDecimal.TEN));
        Order shipped = em.persistAndFlush(new Order(customer, "SHIPPED", BigDecimal.ONE));
        em.clear();  // Clear cache — force fresh read

        // Execute
        List<Order> found = orderRepository.findByCustomerAndStatus(customer, "PENDING");

        // Verify
        assertThat(found).hasSize(1)
            .first()
            .extracting(Order::getId)
            .isEqualTo(pending.getId());
    }
}

// For tests against real PostgreSQL (Testcontainers):
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)  // Don't replace with H2
@Testcontainers
class OrderRepositoryPostgresTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
}
```

---

**Q8. What is TDD (Test-Driven Development)?**

TDD cycle: **Red → Green → Refactor**

1. **Red**: Write a failing test for functionality that doesn't exist yet
2. **Green**: Write minimal code to make the test pass
3. **Refactor**: Improve the code without breaking tests

```java
// Red: failing test
@Test
void calculateDiscount_forPremiumCustomer_applies20PercentDiscount() {
    Customer premiumCustomer = new Customer(CustomerType.PREMIUM);
    BigDecimal orderTotal = new BigDecimal("100.00");

    BigDecimal discount = discountService.calculateDiscount(premiumCustomer, orderTotal);

    assertThat(discount).isEqualByComparingTo("20.00");
}
// DiscountService doesn't exist yet → RED

// Green: make it pass
public class DiscountService {
    public BigDecimal calculateDiscount(Customer customer, BigDecimal total) {
        if (customer.getType() == CustomerType.PREMIUM) {
            return total.multiply(new BigDecimal("0.20"));
        }
        return BigDecimal.ZERO;
    }
}
// Test passes → GREEN

// Refactor: improve code quality
// (extract constants, improve naming, add edge case handling)
```

**Benefits:** Forces small, focused design; built-in regression suite; confidence to refactor.

---

**Q9. What is the AAA pattern for test structure?**

**Arrange, Act, Assert** — structure every test in three sections:

```java
@Test
void cancelOrder_whenStatusIsPending_setsStatusToCancelled() {
    // ARRANGE — set up test data and mocks
    Order order = new Order();
    order.setId(1L);
    order.setStatus("PENDING");
    when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
    when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

    // ACT — execute the method under test (single action)
    Order cancelled = orderService.cancelOrder(1L, "Customer request");

    // ASSERT — verify outcomes
    assertThat(cancelled.getStatus()).isEqualTo("CANCELLED");
    assertThat(cancelled.getCancellationReason()).isEqualTo("Customer request");
    verify(orderRepository).save(argThat(o -> "CANCELLED".equals(o.getStatus())));
}
```

---

**Q10. What are AssertJ assertions and why use them over JUnit assertions?**

AssertJ provides fluent, readable assertions with better failure messages:

```java
// JUnit assertions — less readable
assertEquals("PENDING", order.getStatus());
assertTrue(order.getItems().size() > 0);
assertNull(order.getCancelledAt());

// AssertJ — fluent, much better error messages
assertThat(order.getStatus()).isEqualTo("PENDING");
assertThat(order.getItems()).isNotEmpty().hasSize(2);
assertThat(order.getCancelledAt()).isNull();

// Collection assertions
assertThat(orders)
    .hasSize(3)
    .extracting(Order::getStatus)
    .containsExactlyInAnyOrder("PENDING", "SHIPPED", "DELIVERED");

assertThat(orders)
    .filteredOn(o -> o.getTotal().compareTo(BigDecimal.TEN) > 0)
    .hasSize(2);

// Exception assertions
assertThatThrownBy(() -> orderService.cancelOrder(999L, "test"))
    .isInstanceOf(OrderNotFoundException.class)
    .hasMessageContaining("999");

// Soft assertions — collect all failures before reporting
SoftAssertions softly = new SoftAssertions();
softly.assertThat(order.getId()).isNotNull();
softly.assertThat(order.getStatus()).isEqualTo("PENDING");
softly.assertThat(order.getTotal()).isPositive();
softly.assertAll();  // Reports ALL failures at once
```

---

**Q11. What is parameterized testing and when do you use it?**

```java
@ParameterizedTest
@MethodSource("provideOrderScenarios")
void calculateTotal_withDifferentItems_returnsCorrectTotal(
        List<OrderItem> items, BigDecimal expectedTotal) {
    Order order = new Order(items);
    assertThat(order.calculateTotal()).isEqualByComparingTo(expectedTotal);
}

static Stream<Arguments> provideOrderScenarios() {
    return Stream.of(
        Arguments.of(List.of(item("P1", 2, "10.00")), new BigDecimal("20.00")),
        Arguments.of(List.of(item("P1", 1, "10.00"), item("P2", 3, "5.00")),
                     new BigDecimal("25.00")),
        Arguments.of(List.of(), BigDecimal.ZERO)
    );
}

// Other sources
@ParameterizedTest
@ValueSource(strings = {"PENDING", "CONFIRMED", "PROCESSING"})
void canCancel_whenStatusIsActive_returnsTrue(String status) { ... }

@ParameterizedTest
@CsvSource({
    "alice@example.com, true",
    "invalid-email, false",
    "'', false"
})
void isValidEmail(String email, boolean expected) { ... }

@ParameterizedTest
@EnumSource(value = OrderStatus.class, names = {"SHIPPED", "DELIVERED"})
void cannotCancel_whenStatusIsTerminal(OrderStatus status) { ... }
```

---

**Q12. What are test doubles? Describe mock, stub, spy, fake, dummy.**

| Type | Description | Use |
|------|-------------|-----|
| **Dummy** | Placeholder, never actually used | Fill parameter lists |
| **Stub** | Returns pre-programmed responses | Control indirect inputs |
| **Fake** | Working implementation, simplified | In-memory database, fake service |
| **Mock** | Verifies expected interactions | Verify behavior was triggered |
| **Spy** | Real object with selective stubbing | Partial mocking |

```java
// Dummy — just filling a parameter
void test() {
    Logger dummyLogger = mock(Logger.class);  // Never called, just satisfies constructor
    new Service(dummyLogger).method();
}

// Stub — returns canned data
when(repo.findById(1L)).thenReturn(Optional.of(testOrder));

// Fake — simplified real implementation
class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    @Override public Optional<Order> findById(Long id) { return Optional.ofNullable(store.get(id)); }
    @Override public Order save(Order order) { store.put(order.getId(), order); return order; }
}

// Mock — verify interactions
verify(emailService).sendOrderConfirmation(any(Order.class));

// Spy
@Spy
EmailService emailService = new RealEmailService(mockSmtp);
// Calls real sendOrderConfirmation unless you stub it
```

---

**Q13. How do you test asynchronous code?**

```java
// Testing @Async methods
@Test
void asyncOrderProcessing_completesSuccessfully() throws Exception {
    CompletableFuture<Order> future = orderService.processOrderAsync(request);

    // Wait for completion with timeout
    Order result = future.get(5, TimeUnit.SECONDS);
    assertThat(result.getStatus()).isEqualTo("CONFIRMED");
}

// Testing reactive (WebFlux)
@Test
void getOrderReactive_returnsOrder() {
    when(orderRepository.findById(1L)).thenReturn(Mono.just(testOrder));

    StepVerifier.create(orderService.getOrderMono(1L))
        .expectNext(testOrder)
        .verifyComplete();
}

// Testing reactive errors
StepVerifier.create(orderService.getOrderMono(999L))
    .expectError(OrderNotFoundException.class)
    .verify();

// Testing time-based reactive
StepVerifier.withVirtualTime(() -> orderService.scheduleReminder(orderId))
    .thenAwait(Duration.ofHours(24))  // Fast-forward time
    .expectNextCount(1)
    .verifyComplete();

// Awaitility for thread-based async
Awaitility.await()
    .atMost(10, TimeUnit.SECONDS)
    .pollInterval(Duration.ofMillis(500))
    .until(() -> orderRepository.findById(orderId)
        .map(o -> "PROCESSED".equals(o.getStatus()))
        .orElse(false));
```

---

**Q14. What is test isolation and why does it matter?**

Test isolation ensures each test is independent — a test should not affect or depend on another test's state.

**Problems with non-isolated tests:**
- Tests pass when run alone but fail in a suite
- Test order matters (brittle)
- Hard to diagnose failures

**Spring test isolation:**
```java
// @DataJpaTest auto-configures @Transactional — rolls back after each test!
// @SpringBootTest does NOT rollback — must annotate explicitly
@SpringBootTest
@Transactional  // Add to rollback after each test
class OrderServiceIntegrationTest { ... }

// Or use @DirtiesContext to reload context after test
@Test
@DirtiesContext  // Reload Spring context after this test (expensive — avoid)
void test_thatChangesStaticState() { ... }

// Testcontainers: start fresh containers per test class (not per test)
@Container  // static = shared across all tests in class
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
// Reset state between tests via @Sql("cleanup.sql") or @Transactional
```

---

**Q15. What are contract tests and consumer-driven contract testing?**

Contract tests verify that a service honors the contract expected by its consumers (API shape, field names, HTTP status codes).

```java
// Spring Cloud Contract — consumer defines the contract
// contracts/order-service/create-order.groovy:
// Contract.make {
//   request { method POST; url "/api/orders" }
//   response { status 201; body([id: anyNonEmptyString(), status: "PENDING"]) }
// }

// On provider side — generated test verifies contract
@SpringBootTest(webEnvironment = WebEnvironment.MOCK)
@AutoConfigureMessageVerifier
class OrderServiceContractTest extends ContractBase {

    @Test
    void createOrder_shouldMatchContract() {
        // Test generated from contract — validates response matches consumer expectation
    }
}

// On consumer side — Pact or Spring Cloud Contract stub
@RunWith(SpringRunner.class)
@PactTestFor(providerName = "order-service")
class OrderClientPactTest {

    @Pact(consumer = "frontend")
    RequestResponsePact createOrderPact(PactDslWithProvider builder) {
        return builder.given("order can be created")
            .uponReceiving("POST to create order")
            .path("/api/orders")
            .method("POST")
            .willRespondWith()
            .status(201)
            .body(new PactDslJsonBody().stringType("id").stringValue("status", "PENDING"))
            .toPact();
    }
}
```

---

**Q16. How do you test Spring Security?**

```java
@WebMvcTest(OrderController.class)
class OrderControllerSecurityTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;

    // Test with authenticated user
    @Test
    @WithMockUser(username = "alice", roles = {"USER"})
    void getOrder_withAuthenticatedUser_returns200() throws Exception {
        when(orderService.getOrder(1L)).thenReturn(testOrder);

        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk());
    }

    // Test unauthenticated
    @Test
    void getOrder_withoutAuth_returns401() throws Exception {
        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isUnauthorized());
    }

    // Test forbidden (wrong role)
    @Test
    @WithMockUser(roles = {"USER"})
    void deleteOrder_withUserRole_returns403() throws Exception {
        mockMvc.perform(delete("/api/orders/1"))
               .andExpect(status().isForbidden());
    }

    // Test with JWT
    @Test
    void getOrder_withValidJwt_returns200() throws Exception {
        String token = jwtTokenProvider.createToken("alice", List.of("USER"));

        mockMvc.perform(get("/api/orders/1")
                    .header("Authorization", "Bearer " + token))
               .andExpect(status().isOk());
    }
}
```

---

**Q17. What is mutation testing?**

Mutation testing evaluates the quality of your tests by introducing small bugs (mutations) into the code and checking if tests catch them:

```java
// Original code
public boolean isEligibleForDiscount(Order order) {
    return order.getTotal().compareTo(new BigDecimal("100.00")) >= 0;
}

// Mutations introduced:
// 1. Change >= to >          (boundary mutation)
// 2. Change 100.00 to 99.00  (constant mutation)
// 3. Negate the condition    (negation mutation)
// 4. Return true always      (return value mutation)

// If your tests don't catch these mutations, your test coverage is insufficient
// even if line coverage shows 100%

// Tools: PIT (PITest) for Java
// mvn test-compile org.pitest:pitest-maven:mutationCoverage
```

**Why line coverage isn't enough:**
```java
@Test
void test() {
    boolean result = service.isEligibleForDiscount(orderWithTotal100);
    // Line is "covered" but we never actually asserted the return value!
    // Mutation: method returns false always → test still passes!
}
```

---

**Q18. What is Testcontainers and when should you use it?**

Testcontainers spins up real Docker containers for integration tests:

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired OrderRepository orderRepository;

    @Test
    void save_andFindById_returnsPersistedOrder() {
        Order saved = orderRepository.save(new Order("PENDING", BigDecimal.TEN));
        Optional<Order> found = orderRepository.findById(saved.getId());
        assertThat(found).isPresent()
            .get().extracting(Order::getStatus).isEqualTo("PENDING");
    }
}
```

**Use when:** Tests require real database behavior (specific SQL features, constraints, stored procedures), Kafka, Redis pub/sub — anything where H2 in-memory isn't accurate enough.

---

**Q19. What are test data builders and why use them?**

Test data builders create test objects with sensible defaults, allowing tests to specify only what matters:

```java
// Test builder
public class OrderTestBuilder {
    private Long id = 1L;
    private String status = "PENDING";
    private BigDecimal total = new BigDecimal("100.00");
    private Long customerId = 1L;
    private Instant createdAt = Instant.now();
    private List<OrderItem> items = List.of(OrderItemTestBuilder.aOrderItem().build());

    public static OrderTestBuilder anOrder() { return new OrderTestBuilder(); }

    public OrderTestBuilder withStatus(String status) {
        this.status = status; return this;
    }

    public OrderTestBuilder withTotal(String total) {
        this.total = new BigDecimal(total); return this;
    }

    public OrderTestBuilder forCustomer(Long customerId) {
        this.customerId = customerId; return this;
    }

    public Order build() {
        Order order = new Order();
        order.setId(id);
        order.setStatus(status);
        order.setTotal(total);
        order.setCustomerId(customerId);
        order.setCreatedAt(createdAt);
        order.setItems(items);
        return order;
    }
}

// Tests are focused and readable
Order pendingOrder = anOrder().withStatus("PENDING").withTotal("50.00").build();
Order shippedOrder = anOrder().withStatus("SHIPPED").forCustomer(42L).build();
```

---

**Q20. What is the difference between `@SpringBootTest` slice tests and full context?**

```
Full context (@SpringBootTest):
  - All beans loaded (services, repos, controllers, security)
  - Tests run against real application behavior
  - Slow startup (seconds)
  - Use for: end-to-end integration tests, complex interaction testing

Slice tests:
  @WebMvcTest:      Controllers + MVC config. No services (mock them). Fast.
  @DataJpaTest:     JPA + H2 only. No controllers, no services. Fast.
  @JsonTest:        ObjectMapper config only. Tests JSON serialization.
  @RestClientTest:  RestTemplate/WebClient tests with mock server.
  @DataRedisTest:   Spring Data Redis slice with embedded Redis.

Trade-offs:
  Slice test: Fast, focused, but requires mocking boundaries
  Full context: Realistic, catches integration bugs, but slow
```

**Best practice:** Use slice tests for most tests. Use `@SpringBootTest` for key integration scenarios and smoke tests.
