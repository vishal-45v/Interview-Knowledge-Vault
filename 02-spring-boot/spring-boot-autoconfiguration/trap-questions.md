# Spring Boot Auto-Configuration — Trap Questions

---

## Trap 1: @ConditionalOnMissingBean with Type vs Name

**Question:** Will the auto-configured bean be created?

```java
// Auto-configuration:
@Bean
@ConditionalOnMissingBean(PaymentService.class)
public PaymentService defaultPaymentService() {
    return new StripePaymentService();
}

// User config:
@Bean("myPaymentService")
public StripePaymentService stripePaymentService() {
    return new StripePaymentService();
}
```

**Answer:** No — auto-config will NOT create a bean. `@ConditionalOnMissingBean` matches by TYPE, not name. Since `StripePaymentService` IS-A `PaymentService`, the condition is false and auto-config backs off.

---

## Trap 2: @Profile vs @ConditionalOnProperty

**Question:** What's the difference between `@Profile("prod")` and `@ConditionalOnProperty(name="app.env", havingValue="prod")`?

**Answer:**
- `@Profile("prod")` checks Spring's active profiles (set via `spring.profiles.active`)
- `@ConditionalOnProperty` checks a specific property value

They're different mechanisms. Profiles are typically set per environment. `@ConditionalOnProperty` gives more granular feature-flag control within a single profile.

You can have `spring.profiles.active=prod` and `app.feature.x.enabled=false` — profile active but feature disabled.

---

## Trap 3: @EnableAutoConfiguration on Non-Boot App

**Question:** Can you use Spring Boot auto-configuration in a plain Spring MVC app?

**Answer:** Yes, but you'd need `@EnableAutoConfiguration` directly. `@SpringBootApplication` is a convenience annotation that combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

In a plain Spring app, you can add `@EnableAutoConfiguration` to any `@Configuration` class to get auto-configuration benefits.

---

## Trap 4: Auto-Configuration for Test Classes

**Question:** If your test uses `@SpringBootTest`, are all auto-configurations applied?

**Answer:** Yes — `@SpringBootTest` loads the full application context including all auto-configurations. This is what makes it "integration test" level.

For faster tests, use test slices:
- `@WebMvcTest` — only web layer auto-config (no JPA, no Redis)
- `@DataJpaTest` — only JPA auto-config (in-memory H2)
- `@DataRedisTest` — only Redis auto-config

Test slices apply only the relevant auto-configurations for faster test startup.

---

## Trap 5: @ConditionalOnBean Ordering Problem

**Question:** Why does this sometimes fail with `NoSuchBeanDefinitionException`?

```java
@AutoConfiguration
public class Config1 {
    @Bean
    @ConditionalOnBean(SomeService.class)
    public AnotherService anotherService() { ... }
}
```

**Answer:** `@ConditionalOnBean` depends on the order beans are registered. If `SomeService` hasn't been registered yet when `Config1` is evaluated, the condition is false even if `SomeService` will be registered later.

Fix: Use `@AutoConfigureAfter(SomeServiceAutoConfiguration.class)` to ensure correct ordering, or use `@ConditionalOnBean` only for beans defined in user configuration (which are processed before auto-configurations).
