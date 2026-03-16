# Spring Boot Auto-Configuration — Structured Answers

> Deep-dive answers for auto-configuration, conditional beans, and spring.factories.

---

## Q1: How Does Spring Boot Auto-Configuration Work End-to-End?

```
1. @SpringBootApplication includes @EnableAutoConfiguration
2. @EnableAutoConfiguration imports AutoConfigurationImportSelector
3. AutoConfigurationImportSelector reads:
   META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   (lists hundreds of auto-configuration classes)
4. Each auto-configuration class has @Conditional annotations
5. Spring evaluates each condition against the current environment
6. Matching configurations are applied — beans are registered
7. User-defined beans take precedence via @ConditionalOnMissingBean
```

---

## Q2: @ConditionalOnClass vs @ConditionalOnBean

```java
@ConditionalOnClass(RedisConnectionFactory.class)
// Condition: Is the CLASS on the classpath?
// Use: "Apply this config only if the Redis library is available"
// Evaluated at: BeanDefinition registration time
// Does NOT load the class (safe if not present)

@ConditionalOnBean(RedisConnectionFactory.class)
// Condition: Is there a BEAN of this type in the context?
// Use: "Apply this config only if Redis is already configured"
// Evaluated at: Bean instantiation time
// Requires the bean to already exist

// Common pattern — combine both:
@ConditionalOnClass(RedisConnectionFactory.class)  // Library available
@ConditionalOnBean(RedisConnectionFactory.class)    // AND configured by user
public class RedisCacheAutoConfiguration { }
```

---

## Q3: @ConditionalOnMissingBean — The Override Pattern

`@ConditionalOnMissingBean` enables the "user-wins" principle:

```java
// Auto-configuration provides a sensible default:
@Bean
@ConditionalOnMissingBean(ObjectMapper.class)
public ObjectMapper objectMapper() {
    return new ObjectMapper();  // Default configuration
}

// User can override by defining their own:
@Bean  // User's bean — no @ConditionalOnMissingBean needed
public ObjectMapper objectMapper() {
    return new ObjectMapper()
        .configure(FAIL_ON_UNKNOWN_PROPERTIES, false);
}
// Spring sees user's ObjectMapper → condition is false → auto-config backs off
```

---

## Q4: Order of Auto-Configuration Processing

1. User-defined `@Configuration` classes are processed first
2. Auto-configuration classes are processed after (they're "auto")
3. Within auto-configurations, ordering can be specified with `@AutoConfigureBefore`/`@AutoConfigureAfter`
4. `@Order` on auto-configurations controls within-phase ordering

This ensures user configurations always take precedence over auto-configurations.

---

## Q5: How to Debug Why Auto-Configuration Is/Isn't Applied

```bash
# Method 1: --debug flag
java -jar app.jar --debug
# Prints full conditions evaluation report

# Method 2: logging level
logging.level.org.springframework.boot.autoconfigure=DEBUG

# Method 3: Actuator endpoint
GET /actuator/conditions
# Returns JSON with all positive/negative matches and their conditions

# Method 4: Auto-configure report annotation processor
# Add spring-boot-autoconfigure-processor to see reports in IDE
```
