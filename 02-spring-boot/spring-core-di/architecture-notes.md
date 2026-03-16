# Spring Core DI — Architecture Notes

> Internal architecture of the Spring IoC container, bean factory internals, and proxy mechanics.

---

## Spring IoC Container Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Spring IoC Container                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              ApplicationContext                           │   │
│  │    (ClassPathXmlApplicationContext /                      │   │
│  │     AnnotationConfigApplicationContext /                  │   │
│  │     SpringApplication in Spring Boot)                     │   │
│  │                                                           │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │              BeanFactory                           │   │   │
│  │  │  (DefaultListableBeanFactory)                      │   │   │
│  │  │                                                    │   │   │
│  │  │  beanDefinitionMap: Map<String, BeanDefinition>    │   │   │
│  │  │  singletonObjects: Map<String, Object>             │   │   │
│  │  │  earlySingletonObjects: Map<String, Object>        │   │   │
│  │  │  singletonFactories: Map<String, ObjectFactory>    │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Bean Definition Registration Flow

```
@ComponentScan / @Configuration
         │
         ▼
  ClassPathBeanDefinitionScanner
  (scans classpath for @Component, @Service, @Repository, @Controller)
         │
         ▼
  BeanDefinitionRegistry.registerBeanDefinition()
  (Stores BeanDefinition: class, scope, init method, depends-on, etc.)
         │
         ▼
  BeanFactoryPostProcessor runs
  (Can modify BeanDefinitions before instantiation)
  e.g., PropertySourcesPlaceholderConfigurer resolves @Value
         │
         ▼
  BeanPostProcessor registered
  (AutowiredAnnotationBeanPostProcessor, CommonAnnotationBeanPostProcessor)
         │
         ▼
  Singleton beans instantiated (eager by default)
```

---

## Three-Level Cache for Circular Dependency Resolution

Spring uses three maps to handle circular dependencies with setter/field injection:

```
Level 1 (singletonObjects):       Fully initialized beans
Level 2 (earlySingletonObjects):  Partially initialized (properties not set)
Level 3 (singletonFactories):     ObjectFactory for creating early references

Creation of Bean A that depends on Bean B which depends on Bean A:

1. Start creating A
2. Place A's ObjectFactory in singletonFactories (Level 3)
3. Inject B into A → start creating B
4. Inject A into B → A not in Level 1 or Level 2
5. Get A from Level 3 (via ObjectFactory, gets early reference/proxy)
6. Move early A reference to Level 2
7. B fully initialized → placed in Level 1
8. A continues initialization with B injected
9. A fully initialized → moved from Level 2 to Level 1
```

```java
// This is why circular deps work with setter/field injection
// but NOT with constructor injection:

// Constructor injection: A needs B to construct, B needs A to construct
// → deadlock, cannot get early reference

// Setter injection: A constructed first (no deps needed for constructor)
// → early reference available for B to use
```

---

## CGLIB vs JDK Dynamic Proxy

Spring creates proxies for AOP, @Transactional, @Async, @Cacheable, etc.

### JDK Dynamic Proxy

```
Requirements:
- Bean must implement an interface
- Proxy implements the same interface

Client ──→ [JDK Proxy (implements Interface)] ──→ [Real Bean]

           java.lang.reflect.Proxy.newProxyInstance()
           InvocationHandler handles method calls
```

### CGLIB Proxy

```
Requirements:
- No interface needed
- Class must not be final
- Methods must not be final

Client ──→ [CGLIB Subclass of Bean] ──→ [Real Bean / super calls]

           ByteBuddy/CGLIB generates a subclass at runtime
           Overrides methods to add AOP behavior
```

### How Spring Chooses

```java
@Configuration
public class AopConfig {
    // proxyTargetClass = false (default) → JDK proxy if interface exists
    // proxyTargetClass = true → always CGLIB
}

// Spring Boot default: proxyTargetClass = true (CGLIB)
// Reason: handles cases where interface is not defined
```

### Implications

```java
// Problem with CGLIB proxy: final methods cannot be intercepted
@Service
public class PaymentService {

    @Transactional  // This method WILL NOT be proxied!
    public final void processPayment(Payment payment) {
        // final method — CGLIB cannot override it
        // @Transactional has NO effect here
    }
}
```

---

## BeanPostProcessor — How Spring Annotations Work

```java
// This is how @Autowired works internally
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Find all @Autowired fields and methods
        // Resolve dependencies from BeanFactory
        // Inject them via reflection
        return bean;
    }
}

// @PostConstruct is handled by:
public class CommonAnnotationBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Find @PostConstruct methods
        // Invoke them via reflection
        return bean;
    }
}
```

---

## ApplicationContext Hierarchy

```
                    ┌──────────────────────────┐
                    │   Root ApplicationContext  │
                    │   (Spring Security, JPA,   │
                    │    common services)        │
                    └──────────────┬───────────┘
                                   │  parent
                    ┌──────────────▼───────────┐
                    │  Child ApplicationContext  │
                    │  (Spring MVC web layer)    │
                    │  @Controller, ViewResolver │
                    └──────────────────────────┘

Child can access parent beans, but NOT vice versa.
Beans in child override beans in parent with same name.
```

In Spring Boot, there is typically one `ApplicationContext` (no parent/child split) unless using `SpringApplication.setParent()`.

---

## @Configuration Class — Full vs Lite Mode

### Full @Configuration (CGLIB-enhanced)

```java
@Configuration  // Full mode — class is CGLIB-proxied
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // Calls serviceB() method
        // Spring intercepts this call — returns same ServiceB bean
        // NOT a new ServiceB() instance
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

### Lite Mode (@Component with @Bean)

```java
@Component  // Lite mode — NOT CGLIB-proxied
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // Calls serviceB() DIRECTLY
        // Creates a NEW ServiceB() — NOT the Spring-managed bean!
        // Risk: two different ServiceB instances exist
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

**Use `@Configuration` (not `@Component`) for config classes that have inter-bean dependencies.**

---

## Dependency Injection in Tests

```
Test DI Strategy Decision Tree:

Is it a unit test?
├── Yes → Don't use Spring at all
│         new MyService(mockRepo, mockEmail)
│         Use Mockito for mocks
│
└── No (integration test) → Use Spring context
    │
    ├── Full context? → @SpringBootTest
    │   (slow, full app context)
    │
    ├── Web layer only? → @WebMvcTest
    │   (MockMvc, no service/repo beans)
    │
    ├── Data layer only? → @DataJpaTest
    │   (in-memory H2, JPA beans only)
    │
    └── Specific slice? → @TestConfiguration
        (add only needed beans)
```

```java
// Pure unit test — no Spring
class UserServiceTest {
    private UserRepository mockRepo = mock(UserRepository.class);
    private EmailService mockEmail = mock(EmailService.class);
    private UserService userService = new UserService(mockRepo, mockEmail);

    @Test
    void shouldRegisterUser() {
        when(mockRepo.save(any())).thenReturn(testUser);
        userService.register(testUser);
        verify(mockEmail).sendWelcome(testUser);
    }
}

// Integration test with slice
@DataJpaTest
class UserRepositoryIntegrationTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        userRepository.save(new User("test@example.com"));
        Optional<User> found = userRepository.findByEmail("test@example.com");
        assertThat(found).isPresent();
    }
}
```

---

## Conditional Beans — @ConditionalOnProperty / @Profile

```
Bean Registration Decision Flow:

@ConditionalOnProperty(name="feature.payments.stripe", havingValue="true")
         │
         ▼
  PropertySourcesPlaceholderConfigurer reads application.yml/properties
         │
         ▼
  Condition evaluated at context startup
  (ConditionContext provides access to properties, environment, BeanFactory)
         │
  ┌──────┴──────┐
  │ true        │ false
  ▼             ▼
Register     Skip bean
bean         registration
```

```java
// Multiple conditions combined with AND logic
@Bean
@ConditionalOnProperty(name = "payment.stripe.enabled", havingValue = "true")
@ConditionalOnClass(name = "com.stripe.Stripe")  // Stripe library on classpath
@ConditionalOnMissingBean(PaymentGateway.class)  // No other PaymentGateway bean
public StripeGateway stripeGateway() {
    return new StripeGateway(stripeApiKey);
}
```

---

## Startup Failure Analysis

```
Common Startup Failures:

1. NoSuchBeanDefinitionException
   Cause: Bean not found by type/name
   Fix:   Check @Component scan, @Bean method, correct class/interface

2. NoUniqueBeanDefinitionException
   Cause: Multiple beans match required type
   Fix:   Add @Primary or @Qualifier

3. BeanCurrentlyInCreationException
   Cause: Circular dependency (constructor injection)
   Fix:   Redesign, @Lazy, setter injection

4. UnsatisfiedDependencyException
   Cause: Dependency cannot be resolved
   Fix:   Check bean exists, correct type, no missing config

5. BeanCreationException (caused by something in init)
   Cause: Exception in @PostConstruct or afterPropertiesSet()
   Fix:   Fix the initialization logic, check DB/external service availability
```
