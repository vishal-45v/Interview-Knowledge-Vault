# Testing — Diagram Explanations

## 1. Test Pyramid

The pyramid shape communicates quantity, speed, and cost ratios across test types.

```
                        /\
                       /  \
                      / E2E\        ~10 tests
                     / Tests\       Slow (minutes), brittle
                    /        \      Real browser/API, full stack
                   /──────────\
                  / Integration\    ~100 tests
                 /    Tests     \   Medium speed (seconds)
                /                \  Real DB, real Spring context
               /──────────────────\
              /    Unit Tests       \  ~1000+ tests
             /                      \ Fast (milliseconds)
            /   (fast, isolated,      \ No external deps
           /     cheap to write)       \
          /________________________________\

  ┌──────────────────┬──────────┬───────────┬──────────────┐
  │ Type             │ Count    │ Speed     │ Isolation    │
  ├──────────────────┼──────────┼───────────┼──────────────┤
  │ Unit             │ Many     │ ~1ms      │ Full (mocks) │
  │ Integration      │ Some     │ ~500ms    │ Partial      │
  │ End-to-End       │ Few      │ ~30s      │ None         │
  └──────────────────┴──────────┴───────────┴──────────────┘

  Anti-pattern — "Ice Cream Cone" (inverted):
           /────────────────────────────\
          /      Many E2E Tests          \   ← slow, brittle
         /────────────────────────────────\
        /    Some Integration Tests        \
       /──────────────────────────────────\ \
      /    Very Few Unit Tests              \ \  ← fragile foundation
     /________________________________________\ \
```

---

## 2. Mockito Mock Lifecycle

Mockito mocks go through four phases: creation, stubbing, acting, and verification.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  Phase 1: CREATE                                                │
  │  @Mock UserRepository repo  ──►  Mockito generates a dynamic   │
  │                                  proxy implementing the         │
  │                                  UserRepository interface       │
  │                                  All methods return null/0/false│
  └────────────────────────────────────────────────┬────────────────┘
                                                   │
  ┌────────────────────────────────────────────────▼────────────────┐
  │  Phase 2: STUB (optional but usually needed)                    │
  │  when(repo.findById(1L))                                        │
  │      .thenReturn(Optional.of(user));                            │
  │                                                                 │
  │  Mockito stores: "when findById is called with arg=1L,          │
  │                   return Optional.of(user)"                     │
  └────────────────────────────────────────────────┬────────────────┘
                                                   │
  ┌────────────────────────────────────────────────▼────────────────┐
  │  Phase 3: ACT                                                   │
  │  userService.getUser(1L);  ──►  internally calls repo.findById  │
  │                                 Mockito intercepts via proxy    │
  │                                 returns stubbed value           │
  └────────────────────────────────────────────────┬────────────────┘
                                                   │
  ┌────────────────────────────────────────────────▼────────────────┐
  │  Phase 4: VERIFY                                                │
  │  verify(repo).findById(1L);                                     │
  │  verify(repo, times(1)).findById(anyLong());                    │
  │  verify(repo, never()).deleteById(any());                       │
  │                                                                 │
  │  Mockito checks its internal invocation log                     │
  │  Throws AssertionError if expectations not met                  │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 3. Spring Test Slices

Spring Boot test slices load only the relevant portion of the application context, dramatically reducing startup time compared to `@SpringBootTest`.

```
  @SpringBootTest
  ┌─────────────────────────────────────────────────────────────────┐
  │  Full Application Context                                       │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
  │  │Controllers│ │ Services │ │   DAOs   │ │ Security Config  │  │
  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
  │  │DataSource │ │JPA Setup │ │ Caching  │  ← everything loaded  │
  │  └──────────┘ └──────────┘ └──────────┘                        │
  └─────────────────────────────────────────────────────────────────┘

  @WebMvcTest(UserController.class)
  ┌─────────────────────────────────────────────────────────────────┐
  │  Web Layer Only                                                 │
  │  ┌──────────────┐ ┌────────────┐ ┌──────────────────────────┐  │
  │  │UserController│ │ MockMvc    │ │ Spring Security (partial) │  │
  │  └──────────────┘ └────────────┘ └──────────────────────────┘  │
  │  Services, DAOs, DataSource ── NOT loaded (use @MockBean)       │
  └─────────────────────────────────────────────────────────────────┘

  @DataJpaTest
  ┌─────────────────────────────────────────────────────────────────┐
  │  JPA/Database Layer Only                                        │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐   │
  │  │JPA Entities  │ │  Repositories│ │ Embedded DB (H2)      │   │
  │  └──────────────┘ └──────────────┘ └──────────────────────┘   │
  │  Controllers, Services ── NOT loaded                            │
  │  @Transactional by default ── each test rolls back             │
  └─────────────────────────────────────────────────────────────────┘

  Startup time comparison:
  @SpringBootTest  ──► 8–15 seconds (loads everything)
  @WebMvcTest      ──► 1–3 seconds  (web layer only)
  @DataJpaTest     ──► 2–5 seconds  (JPA layer only)
  Pure @ExtendWith  ──► <100ms      (no Spring at all)
```

---

## 4. ArgumentCaptor Flow

ArgumentCaptor intercepts arguments passed to a mock so you can make detailed assertions on objects that were created internally by the class under test.

```
  Class under test: NotificationService
  ──────────────────────────────────────────────────────────────────
  void notifyUser(User user) {
      Email email = Email.builder()
          .to(user.getEmail())
          .subject("Welcome, " + user.getName())
          .body(templateEngine.render("welcome", user))
          .build();
      emailSender.send(email);   ◄── this call is what we want to capture
  }

  Test:
  ──────────────────────────────────────────────────────────────────
  ArgumentCaptor<Email> captor = ArgumentCaptor.forClass(Email.class);
         │
         │  1. Act — call the real method
         ▼
  notificationService.notifyUser(testUser);
         │
         │  2. Verify and capture
         ▼
  verify(emailSender).send(captor.capture());
         │
         │  3. Extract and assert
         ▼
  Email captured = captor.getValue();
  assertThat(captured.getTo()).isEqualTo("alice@example.com");
  assertThat(captured.getSubject()).contains("Welcome, Alice");
  assertThat(captured.getBody()).contains("confirm your email");

  ┌────────────────────────────────────────────────────────────────┐
  │  Without captor: you'd need to mock Email.builder() or          │
  │  expose the Email object — both are worse design choices.       │
  │  Captor lets you assert on the ACTUAL object passed to the mock │
  └────────────────────────────────────────────────────────────────┘
```

---

## 5. TDD Cycle (Red → Green → Refactor)

TDD enforces a tight feedback loop that drives design from the outside in.

```
                    ┌─────────────────────────────────┐
                    │                                 │
          ┌─────────▼──────────┐                      │
          │        RED         │                      │
          │   Write a failing  │                      │
          │   test that        │                      │
          │   specifies the    │                      │
          │   desired behavior │                      │
          │   Test MUST fail   │                      │
          │   (proves test     │                      │
          │   actually tests   │                      │
          │   something)       │                      │
          └─────────┬──────────┘                      │
                    │                                 │
                    ▼                                 │
          ┌─────────────────────┐                     │
          │        GREEN        │                     │
          │  Write the MINIMUM  │                     │
          │  code to make the   │                     │
          │  test pass.         │                     │
          │  No gold-plating.   │                     │
          │  Hardcoding is OK   │                     │
          │  at this stage.     │                     │
          └─────────┬───────────┘                     │
                    │                                 │
                    ▼                                 │
          ┌─────────────────────┐                     │
          │      REFACTOR       │                     │
          │  Clean up the code. │                     │
          │  Remove duplication.│                     │
          │  Improve names.     │─────────────────────┘
          │  Tests still pass.  │  ← next cycle begins
          └─────────────────────┘

  Example cycle:
  ─────────────────────────────────────────────────────────────────
  RED:     @Test void addTwoNumbers() { assertEquals(5, calc.add(2,3)); }
           // fails — Calculator class doesn't exist yet

  GREEN:   class Calculator { int add(int a, int b) { return 5; } }
           // passes — hardcoded but test passes

  RED:     @Test void addDifferentNumbers() { assertEquals(7, calc.add(3,4)); }
           // fails — hardcoded 5 is wrong

  GREEN:   int add(int a, int b) { return a + b; }
           // both tests pass

  REFACTOR: rename, extract, clean — tests still green
```

---

## 6. Test Isolation — Each Test Has Its Own State

Test isolation ensures that tests do not share mutable state. Failures stay local and results are reproducible regardless of execution order.

```
  Without isolation (shared state — DANGEROUS):
  ─────────────────────────────────────────────────────────────────
  static List<Order> orders = new ArrayList<>();  // shared!

  test1: orders.add(new Order("A"));   → orders = [A]
  test2: assertEquals(0, orders.size()) → FAILS because test1 left state

  With isolation (Mockito + JUnit 5 default lifecycle):
  ─────────────────────────────────────────────────────────────────
  @ExtendWith(MockitoExtension.class)
  class OrderServiceTest {
      @Mock OrderRepository repo;   ← fresh mock per test
      @InjectMocks OrderService svc; ← new instance per test
  }

  Test 1 lifecycle:                 Test 2 lifecycle:
  ┌────────────────────────┐        ┌────────────────────────┐
  │ New OrderServiceTest   │        │ New OrderServiceTest   │
  │ New @Mock repo         │        │ New @Mock repo         │
  │ New @InjectMocks svc   │        │ New @InjectMocks svc   │
  │ setUp() / @BeforeEach  │        │ setUp() / @BeforeEach  │
  │ test1() runs           │        │ test2() runs           │
  │ tearDown() / @AfterEach│        │ tearDown() / @AfterEach│
  └────────────────────────┘        └────────────────────────┘
         ↑ completely independent ↑

  Database isolation with @Transactional:
  ─────────────────────────────────────────────────────────────────
  @DataJpaTest
  class UserRepositoryTest {
      // Each @Test method runs in a transaction that ROLLS BACK after the test
      // Database returns to clean state for each test
  }

  Real isolation: @DirtiesContext — recreates Spring context entirely
  (expensive — use only when context state is irreparably polluted)
```
