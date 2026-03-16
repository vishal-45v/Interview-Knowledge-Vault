# Testing — Scenario Questions

> 15 real-world scenarios on test design, Spring Boot testing, and production-quality test suites.

---

## Scenario 1: Testing a REST Controller Layer

**Situation:** Write comprehensive tests for `OrderController.createOrder()`.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean OrderService orderService;

    private static final CreateOrderRequest VALID_REQUEST = new CreateOrderRequest(
        1L,
        List.of(new OrderItemRequest("PROD-1", 2, null)),
        "123 Main St",
        null
    );

    @Test
    @DisplayName("POST /api/orders - success returns 201 with order details")
    void createOrder_withValidRequest_returns201() throws Exception {
        OrderResponse expectedResponse = new OrderResponse(
            "ORD-123", "PENDING", new BigDecimal("100.00"), Instant.now());

        when(orderService.createOrder(any(CreateOrderRequest.class)))
            .thenReturn(expectedResponse);

        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(VALID_REQUEST))
                    .header("Authorization", "Bearer " + validToken()))
               .andExpect(status().isCreated())
               .andExpect(header().exists("Location"))
               .andExpect(jsonPath("$.id").value("ORD-123"))
               .andExpect(jsonPath("$.status").value("PENDING"))
               .andExpect(jsonPath("$.total").value(100.00));

        ArgumentCaptor<CreateOrderRequest> captor =
            ArgumentCaptor.forClass(CreateOrderRequest.class);
        verify(orderService).createOrder(captor.capture());
        assertThat(captor.getValue().customerId()).isEqualTo(1L);
    }

    @Test
    @DisplayName("POST /api/orders - missing customerId returns 400 with field errors")
    void createOrder_withMissingCustomerId_returns400() throws Exception {
        String invalidJson = """
            {"items": [{"productId": "P1", "quantity": 1}]}
            """;

        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(invalidJson))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.errors.customerId").exists());

        verifyNoInteractions(orderService);  // Service never called
    }

    @Test
    @DisplayName("POST /api/orders - service throws OrderNotFoundException returns 404")
    void createOrder_whenProductNotFound_returns404() throws Exception {
        when(orderService.createOrder(any()))
            .thenThrow(new ProductNotFoundException("PROD-999"));

        mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(VALID_REQUEST)))
               .andExpect(status().isNotFound())
               .andExpect(jsonPath("$.title").value("Product Not Found"));
    }
}
```

---

## Scenario 2: Integration Testing with Testcontainers

**Situation:** Write integration tests for OrderRepository against real PostgreSQL.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)  // Use real DB, not H2
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withReuse(true);  // Reuse container across test classes for speed

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository orderRepository;
    @Autowired TestEntityManager em;

    @Test
    void findByCustomerIdAndStatusIn_returnsCorrectOrders() {
        // Setup
        Customer customer = em.persistAndFlush(
            new Customer("alice@example.com", "Alice"));
        Order pending = em.persistAndFlush(
            new Order(customer.getId(), "PENDING", new BigDecimal("100.00")));
        Order shipped = em.persistAndFlush(
            new Order(customer.getId(), "SHIPPED", new BigDecimal("200.00")));
        Order other = em.persistAndFlush(
            new Order(customer.getId(), "CANCELLED", new BigDecimal("50.00")));
        em.clear();

        // Execute
        List<Order> active = orderRepository.findByCustomerIdAndStatusIn(
            customer.getId(), List.of("PENDING", "SHIPPED"));

        // Assert
        assertThat(active)
            .hasSize(2)
            .extracting(Order::getId)
            .containsExactlyInAnyOrder(pending.getId(), shipped.getId());
    }

    @Test
    void findTopOrdersByRevenue_returnsOrderedByTotal() {
        // Setup test data
        IntStream.range(1, 11)
            .mapToObj(i -> new Order(1L, "SHIPPED", new BigDecimal(i * 100)))
            .forEach(em::persistAndFlush);
        em.clear();

        // Execute
        List<Order> top3 = orderRepository.findTopOrdersByRevenue(PageRequest.of(0, 3));

        // Assert — should be in descending order
        assertThat(top3).hasSize(3)
            .extracting(o -> o.getTotal().intValue())
            .containsExactly(1000, 900, 800);
    }
}
```

---

## Scenario 3: Testing Service Layer with Mocks

**Situation:** Test complex business logic in OrderService with multiple dependencies.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepository;
    @Mock InventoryService inventoryService;
    @Mock PaymentService paymentService;
    @Mock EmailService emailService;
    @InjectMocks OrderService orderService;

    @Test
    void createOrder_withSufficientInventory_savesAndSendsEmail() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest(
            1L, List.of(new OrderItemRequest("P1", 2, null)), "123 Main St", null);

        when(inventoryService.isAvailable("P1", 2)).thenReturn(true);
        when(inventoryService.reserve("P1", 2)).thenReturn(new ReservationId("RES-123"));
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order o = inv.getArgument(0);
            o.setId(1L);
            return o;
        });

        // Act
        Order result = orderService.createOrder(request);

        // Assert
        assertThat(result.getStatus()).isEqualTo("CONFIRMED");
        assertThat(result.getId()).isNotNull();

        verify(inventoryService).reserve("P1", 2);
        verify(orderRepository).save(any());
        verify(emailService).sendOrderConfirmation(eq(1L));
    }

    @Test
    void createOrder_withInsufficientInventory_throwsExceptionAndNoPayment() {
        // Arrange
        when(inventoryService.isAvailable("P1", 5)).thenReturn(false);
        when(inventoryService.getAvailableQuantity("P1")).thenReturn(2);

        // Act & Assert
        assertThatThrownBy(() ->
            orderService.createOrder(new CreateOrderRequest(1L,
                List.of(new OrderItemRequest("P1", 5, null)), "Addr", null)))
            .isInstanceOf(InsufficientInventoryException.class)
            .hasMessageContaining("P1")
            .hasMessageContaining("requested=5")
            .hasMessageContaining("available=2");

        // Verify payment was never attempted
        verifyNoInteractions(paymentService);
        verify(orderRepository, never()).save(any());
    }

    @Test
    void cancelOrder_whenAlreadyShipped_throwsException() {
        // Arrange
        Order shippedOrder = anOrder().withStatus("SHIPPED").build();
        when(orderRepository.findById(1L)).thenReturn(Optional.of(shippedOrder));

        // Act & Assert
        assertThatThrownBy(() -> orderService.cancelOrder(1L, "Changed mind"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Cannot cancel a shipped order");
    }
}
```

---

## Scenario 4: Parameterized Test for Validation Logic

**Situation:** Test all edge cases of email validation with a single parameterized test.

```java
@ExtendWith(MockitoExtension.class)
class EmailValidatorTest {

    private final EmailValidator validator = new EmailValidator();

    @ParameterizedTest(name = "{0} → {1}")
    @MethodSource("emailValidationCases")
    void validateEmail(String email, boolean expectedValid, String reason) {
        boolean result = validator.isValid(email);
        assertThat(result)
            .as("Email '%s' should be %s because: %s",
                email, expectedValid ? "valid" : "invalid", reason)
            .isEqualTo(expectedValid);
    }

    static Stream<Arguments> emailValidationCases() {
        return Stream.of(
            // Valid cases
            Arguments.of("alice@example.com", true, "standard email"),
            Arguments.of("alice+tag@example.co.uk", true, "plus addressing and country TLD"),
            Arguments.of("a@b.io", true, "short but valid"),
            Arguments.of("first.last@example.com", true, "dots in local part"),

            // Invalid cases
            Arguments.of("", false, "empty string"),
            Arguments.of(null, false, "null"),
            Arguments.of("notanemail", false, "no @ symbol"),
            Arguments.of("@example.com", false, "missing local part"),
            Arguments.of("alice@", false, "missing domain"),
            Arguments.of("alice@.com", false, "domain starts with dot"),
            Arguments.of("alice@example", false, "no TLD"),
            Arguments.of("alice @example.com", false, "space in local part"),
            Arguments.of("alice@example..com", false, "consecutive dots in domain")
        );
    }
}
```

---

## Scenario 5: Testing Caching Behavior

**Situation:** Verify that `@Cacheable` works correctly — first call hits DB, second returns cached result.

```java
@SpringBootTest
@Transactional
class ProductServiceCacheTest {

    @Autowired ProductService productService;
    @MockBean ProductRepository productRepository;

    @Autowired CacheManager cacheManager;

    @BeforeEach
    void clearCache() {
        cacheManager.getCache("products").clear();
    }

    @Test
    void getProduct_calledTwice_onlyHitsRepositoryOnce() {
        // Arrange
        Product product = new Product("P1", "Widget", new BigDecimal("9.99"));
        when(productRepository.findById("P1")).thenReturn(Optional.of(product));

        // Act
        Product first = productService.getProduct("P1");
        Product second = productService.getProduct("P1");  // Should return cached

        // Assert
        assertThat(first).isEqualTo(second);
        verify(productRepository, times(1)).findById("P1");  // Called only once!
    }

    @Test
    void updateProduct_invalidatesCache() {
        // Arrange
        Product product = new Product("P1", "Widget", new BigDecimal("9.99"));
        when(productRepository.findById("P1")).thenReturn(Optional.of(product));

        productService.getProduct("P1");  // Populate cache
        verify(productRepository, times(1)).findById("P1");

        // Update invalidates cache
        productService.updatePrice("P1", new BigDecimal("12.99"));

        // Next read hits repository again
        productService.getProduct("P1");
        verify(productRepository, times(2)).findById("P1");  // Called again after cache eviction
    }
}
```

---

## Scenario 6: Testing Async/Reactive Code

**Situation:** Test a reactive WebFlux endpoint that processes orders asynchronously.

```java
@WebFluxTest(ReactiveOrderController.class)
class ReactiveOrderControllerTest {

    @Autowired WebTestClient webTestClient;
    @MockBean ReactiveOrderService orderService;

    @Test
    void getOrder_returnsMonoWithOrder() {
        when(orderService.getOrder(1L)).thenReturn(Mono.just(testOrder));

        webTestClient.get()
            .uri("/api/v2/orders/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(OrderResponse.class)
            .value(response -> {
                assertThat(response.id()).isEqualTo("ORD-1");
                assertThat(response.status()).isEqualTo("PENDING");
            });
    }

    @Test
    void streamOrders_returnsFluxOfOrders() {
        Flux<OrderEvent> events = Flux.just(
            new OrderEvent("ORD-1", "CREATED"),
            new OrderEvent("ORD-1", "CONFIRMED"),
            new OrderEvent("ORD-1", "SHIPPED")
        ).delayElements(Duration.ofMillis(100));

        when(orderService.streamOrderEvents(1L)).thenReturn(events);

        webTestClient.get()
            .uri("/api/v2/orders/1/events")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(OrderEvent.class)
            .hasSize(3);
    }
}

// Testing service-level reactive
@ExtendWith(MockitoExtension.class)
class ReactiveOrderServiceTest {

    @Mock ReactiveOrderRepository orderRepository;
    @InjectMocks ReactiveOrderService orderService;

    @Test
    void createOrder_returnsCreatedOrder() {
        when(orderRepository.save(any()))
            .thenReturn(Mono.just(testOrder));

        StepVerifier.create(orderService.createOrder(createRequest))
            .expectNextMatches(order -> "PENDING".equals(order.getStatus()))
            .verifyComplete();
    }

    @Test
    void createOrder_onRepositoryError_returnsMono_error() {
        when(orderRepository.save(any()))
            .thenReturn(Mono.error(new DataAccessException("DB down")));

        StepVerifier.create(orderService.createOrder(createRequest))
            .expectError(ServiceUnavailableException.class)
            .verify();
    }
}
```

---

## Scenario 7: Testing with Custom Test Fixtures

**Situation:** Create reusable test fixtures and builders to reduce duplication across tests.

```java
// Test fixture class — shared setup
public class OrderTestFixture {

    public static Customer testCustomer() {
        return Customer.builder()
            .id(1L)
            .email("alice@example.com")
            .name("Alice Smith")
            .type(CustomerType.PREMIUM)
            .build();
    }

    public static Product testProduct() {
        return Product.builder()
            .id("PROD-1")
            .name("Test Widget")
            .price(new BigDecimal("49.99"))
            .category("Electronics")
            .stockQuantity(100)
            .build();
    }

    public static Order pendingOrder() {
        return anOrder()
            .withStatus("PENDING")
            .forCustomer(1L)
            .withTotal("49.99")
            .build();
    }

    public static Order shippedOrder() {
        return anOrder()
            .withStatus("SHIPPED")
            .withTrackingNumber("TRK-12345")
            .build();
    }

    // Create test data in DB using Spring's test support
    @Component
    public static class TestDataSetup {

        @Autowired CustomerRepository customerRepository;
        @Autowired ProductRepository productRepository;

        @Transactional
        public TestContext setupBasicTestData() {
            Customer customer = customerRepository.save(testCustomer());
            Product product = productRepository.save(testProduct());
            return new TestContext(customer, product);
        }
    }
}

// Usage across tests
class OrderServiceTest {
    private final Customer testCustomer = OrderTestFixture.testCustomer();
    private final Product testProduct = OrderTestFixture.testProduct();

    @BeforeEach
    void setUp() {
        when(customerRepository.findById(1L))
            .thenReturn(Optional.of(testCustomer));
    }
}
```

---

## Scenario 8: Testing Exception Handling

**Situation:** Verify that the global exception handler returns correct HTTP status and error body for different exceptions.

```java
@WebMvcTest
@ContextConfiguration(classes = {GlobalExceptionHandler.class, TestController.class})
class GlobalExceptionHandlerTest {

    @Autowired MockMvc mockMvc;

    @RestController
    static class TestController {
        @GetMapping("/test/not-found")
        public void notFound() { throw new OrderNotFoundException("ORD-999"); }

        @GetMapping("/test/validation")
        public void validation() { throw new ConstraintViolationException(Set.of()); }

        @GetMapping("/test/server-error")
        public void serverError() { throw new RuntimeException("Unexpected"); }
    }

    @Test
    void handleNotFound_returns404WithProblemDetails() throws Exception {
        mockMvc.perform(get("/test/not-found"))
               .andExpect(status().isNotFound())
               .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
               .andExpect(jsonPath("$.type").value("/errors/order-not-found"))
               .andExpect(jsonPath("$.title").value("Order Not Found"))
               .andExpect(jsonPath("$.status").value(404))
               .andExpect(jsonPath("$.correlationId").exists());
    }

    @Test
    void handleServerError_returns500WithGenericMessage() throws Exception {
        mockMvc.perform(get("/test/server-error"))
               .andExpect(status().isInternalServerError())
               .andExpect(jsonPath("$.detail").value("An unexpected error occurred"))
               // Should NOT expose internal error message
               .andExpect(jsonPath("$.detail").doesNotContain("Unexpected"));
    }
}
```

---

## Scenario 9: Testing Kafka Consumer

**Situation:** Test a Kafka consumer without a real Kafka broker.

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = "order-events")
class OrderEventConsumerTest {

    @Autowired KafkaTemplate<String, String> kafkaTemplate;
    @Autowired OrderEventConsumer consumer;
    @MockBean OrderService orderService;
    @Autowired CountDownLatchBean latch;

    @Test
    void consume_orderCreatedEvent_callsOrderService() throws Exception {
        // Send message to embedded Kafka
        String payload = """
            {"eventType": "ORDER_CREATED", "orderId": "ORD-123", "customerId": 1}
            """;
        kafkaTemplate.send("order-events", "ORD-123", payload);

        // Wait for consumer to process (with timeout)
        boolean processed = latch.getCountDownLatch().await(10, TimeUnit.SECONDS);
        assertThat(processed).isTrue();

        // Verify service was called
        verify(orderService).processOrderCreated(
            argThat(event -> "ORD-123".equals(event.getOrderId())));
    }

    @Bean
    public CountDownLatchBean countDownLatchBean() {
        return new CountDownLatchBean(new CountDownLatch(1));
    }
}

// Alternative: test consumer logic directly without Kafka
@ExtendWith(MockitoExtension.class)
class OrderEventConsumerUnitTest {

    @Mock OrderService orderService;
    @InjectMocks OrderEventConsumer consumer;

    @Test
    void handleOrderCreated_parsesAndDelegates() throws JsonProcessingException {
        String payload = "{\"eventType\":\"ORDER_CREATED\",\"orderId\":\"ORD-123\"}";
        ConsumerRecord<String, String> record =
            new ConsumerRecord<>("order-events", 0, 100L, "ORD-123", payload);

        consumer.handleOrderEvent(record);

        verify(orderService).processOrderCreated(
            argThat(event -> "ORD-123".equals(event.getOrderId())));
    }
}
```

---

## Scenario 10: Testing Database Migration

**Situation:** Verify that Flyway migrations run cleanly and schema matches expectations.

```java
@SpringBootTest
@Testcontainers
class FlywayMigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");
    }

    @Autowired JdbcTemplate jdbcTemplate;

    @Test
    void allMigrationsApply_withoutError() {
        // If we reach this point, Flyway ran all migrations successfully
        // (Spring Boot fails to start if migrations fail)
    }

    @Test
    void ordersTable_hasExpectedColumns() {
        Map<String, String> columns = jdbcTemplate
            .queryForList(
                "SELECT column_name, data_type FROM information_schema.columns " +
                "WHERE table_name = 'orders' ORDER BY ordinal_position")
            .stream()
            .collect(Collectors.toMap(
                row -> (String) row.get("column_name"),
                row -> (String) row.get("data_type")
            ));

        assertThat(columns).containsKeys("id", "status", "total", "customer_id", "created_at");
        assertThat(columns.get("total")).isEqualTo("numeric");
        assertThat(columns.get("created_at")).isEqualTo("timestamp with time zone");
    }

    @Test
    void requiredIndexes_exist() {
        List<String> indexes = jdbcTemplate.queryForList(
            "SELECT indexname FROM pg_indexes WHERE tablename = 'orders'",
            String.class);

        assertThat(indexes)
            .contains("idx_orders_customer_id", "idx_orders_status_created_at");
    }
}
```

---

## Scenario 11: Testing Circuit Breaker

**Situation:** Test that circuit breaker opens after N failures and returns fallback.

```java
@SpringBootTest
class CircuitBreakerTest {

    @Autowired InventoryService inventoryService;
    @MockBean InventoryApiClient inventoryApiClient;

    @Autowired CircuitBreakerRegistry circuitBreakerRegistry;

    @BeforeEach
    void resetCircuitBreaker() {
        circuitBreakerRegistry.circuitBreaker("inventory").reset();
    }

    @Test
    void circuitBreaker_opensAfterFailures_returnsFallback() {
        // Configure mock to always fail
        when(inventoryApiClient.getStock(any()))
            .thenThrow(new RuntimeException("Service down"));

        // Trigger failures to open circuit
        IntStream.range(0, 10).forEach(i -> {
            try { inventoryService.getStock("P1"); }
            catch (Exception e) { /* expected */ }
        });

        // Verify circuit is open
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("inventory");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // When circuit is open, fallback should be returned immediately
        InventoryStatus fallback = inventoryService.getStock("P1");
        assertThat(fallback.isAvailable()).isFalse();
        assertThat(fallback.isFromFallback()).isTrue();

        // Verify client was NOT called when circuit is open (fast fail)
        // 10 calls during closing phase
        verify(inventoryApiClient, times(10)).getStock(any());
    }
}
```

---

## Scenario 12: Testing with @TestConfiguration

**Situation:** Override specific beans in integration tests without changing production config.

```java
@SpringBootTest
class OrderServiceIntegrationTest {

    // Override only the email service — use fake instead of real SMTP
    @TestConfiguration
    static class TestConfig {

        @Bean
        @Primary  // Overrides production EmailService
        public EmailService fakeEmailService() {
            return new FakeEmailService();  // Records sent emails without actual SMTP
        }

        @Bean
        @Primary
        public Clock testClock() {
            return Clock.fixed(Instant.parse("2024-01-15T10:00:00Z"), ZoneOffset.UTC);
        }
    }

    @Autowired OrderService orderService;
    @Autowired FakeEmailService emailService;

    @Test
    void createOrder_sendConfirmationEmail() {
        Order order = orderService.createOrder(validRequest);

        assertThat(emailService.getSentEmails())
            .hasSize(1)
            .first()
            .satisfies(email -> {
                assertThat(email.getTo()).isEqualTo("customer@example.com");
                assertThat(email.getSubject()).contains("ORD-");
            });
    }
}

// Fake email service
class FakeEmailService implements EmailService {
    private final List<EmailMessage> sentEmails = new ArrayList<>();

    @Override
    public void sendOrderConfirmation(Order order) {
        sentEmails.add(new EmailMessage(
            order.getCustomerEmail(),
            "Order Confirmation: " + order.getId(),
            "Your order has been confirmed."
        ));
    }

    public List<EmailMessage> getSentEmails() { return List.copyOf(sentEmails); }
    public void clearEmails() { sentEmails.clear(); }
}
```

---

## Scenario 13: Load Testing with Gatling

**Situation:** Define a Gatling load test for the order creation endpoint.

```scala
// OrderSimulation.scala
class OrderSimulation extends Simulation {

    val httpConfig = http
        .baseUrl("https://api.example.com")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer ${token}")

    val createOrder = scenario("Create Order")
        .exec(http("POST /api/orders")
            .post("/api/orders")
            .body(StringBody("""
                {"customerId": 1, "items": [{"productId": "P1", "quantity": 1}],
                 "deliveryAddress": "123 Main St"}
            """))
            .check(status.is(201))
            .check(jsonPath("$.id").saveAs("orderId")))
        .pause(1)
        .exec(http("GET /api/orders/${orderId}")
            .get("/api/orders/${orderId}")
            .check(status.is(200)))

    setUp(
        createOrder.inject(
            rampUsersPerSec(1) to 100 during (60 seconds),  // Ramp up
            constantUsersPerSec(100) during (120 seconds)    // Sustained load
        ).protocols(httpConfig)
    ).assertions(
        global.responseTime.p95.lt(500),   // P95 < 500ms
        global.responseTime.p99.lt(1000),  // P99 < 1s
        global.failedRequests.percent.lt(1) // < 1% failures
    )
}
```

---

## Scenario 14: Testing Spring Security Rules

**Situation:** Verify that authorization rules work correctly — only order owners and admins can view orders.

```java
@WebMvcTest(OrderController.class)
class OrderSecurityTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;

    @Test
    @WithMockUser(username = "alice", authorities = {"SCOPE_orders:read"})
    void getOrder_withCorrectOwner_returns200() throws Exception {
        Order aliceOrder = anOrder().forCustomer(1L).build();
        when(orderService.getOrder(1L)).thenReturn(aliceOrder);

        // Alice (user ID 1) accessing her own order
        mockMvc.perform(get("/api/orders/1")
                    .with(user("alice").roles("USER")))
               .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "bob", authorities = {"SCOPE_orders:read"})
    void getOrder_forDifferentOwner_returns403() throws Exception {
        Order aliceOrder = anOrder().forCustomer(1L).build();
        when(orderService.getOrder(1L)).thenReturn(aliceOrder);

        // Bob (user ID 2) accessing Alice's order
        mockMvc.perform(get("/api/orders/1")
                    .with(user("bob").roles("USER")))
               .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void getOrder_withAdminRole_canAccessAnyOrder() throws Exception {
        when(orderService.getOrder(anyLong())).thenReturn(testOrder);

        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk());
    }

    @Test
    void getOrder_withoutAuthentication_returns401() throws Exception {
        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void deleteOrder_withUserRole_returns403() throws Exception {
        mockMvc.perform(delete("/api/orders/1"))
               .andExpect(status().isForbidden());
    }
}
```

---

## Scenario 15: Test Coverage and Quality Metrics

**Situation:** Establish a test quality baseline with JaCoCo and mutation testing.

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <configuration>
        <excludes>
            <exclude>**/config/**</exclude>
            <exclude>**/*Config.class</exclude>
            <exclude>**/dto/**</exclude>    <!-- DTOs are data carriers, not logic -->
            <exclude>**/Application.class</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>  <!-- 80% branch coverage -->
                            </limit>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.85</minimum>  <!-- 85% line coverage -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>

<!-- PIT mutation testing -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.service.*</param>  <!-- Only test services -->
        </targetClasses>
        <mutationThreshold>70</mutationThreshold>  <!-- 70% mutations must be killed -->
    </configuration>
</plugin>
```
