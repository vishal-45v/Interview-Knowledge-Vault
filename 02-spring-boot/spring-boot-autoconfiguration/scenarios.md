# Spring Boot Auto-Configuration — Scenarios

> Real-world scenarios covering auto-configuration, @Conditional, and spring.factories.

---

## Scenario 1: Understanding How Auto-Configuration Works

**Question:** How does Spring Boot know to auto-configure a `DataSource` when you add `spring-boot-starter-data-jpa` to your classpath?

**Answer:**

1. Spring Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+) or `META-INF/spring.factories` (older versions).
2. This file lists all auto-configuration classes, including `DataSourceAutoConfiguration`.
3. `DataSourceAutoConfiguration` is annotated with `@ConditionalOnClass(DataSource.class)` — it only activates if `javax.sql.DataSource` is on the classpath (which it is when you add the JPA starter).
4. Spring applies the configuration, creating a `DataSource` bean from your `application.yml` properties.

```java
// What DataSourceAutoConfiguration looks like internally (simplified):
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class,
              DataSourceConfiguration.Tomcat.class,
              DataSourceConfiguration.Dbcp2.class })
    static class PooledDataSourceConfiguration { }
}
```

---

## Scenario 2: Disabling Specific Auto-Configuration

**Problem:** You want to manage the `DataSource` yourself and don't want Spring Boot's auto-configuration to create one:

```java
// Option 1: @SpringBootApplication exclude
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class MyApplication { }

// Option 2: Property-based exclusion
// application.yml:
// spring:
//   autoconfigure:
//     exclude:
//       - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

// Option 3: @EnableAutoConfiguration on a @Configuration class
@Configuration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
public class NoAutoDataSource { }
```

**When to use:** When the auto-configured bean conflicts with your custom configuration, or when you need complete control over a component (e.g., custom connection pool settings).

---

## Scenario 3: Writing Your Own Auto-Configuration

**Problem:** You're building a shared library for your company that should auto-configure a `MetricsService` bean when included in any Spring Boot project:

```java
// 1. Create the auto-configuration class
@AutoConfiguration
@ConditionalOnClass(MetricsService.class)  // Only if our library is on classpath
@ConditionalOnMissingBean(MetricsService.class)  // Don't override user's bean
@EnableConfigurationProperties(MetricsProperties.class)
public class MetricsAutoConfiguration {

    @Bean
    public MetricsService metricsService(MetricsProperties props) {
        return new MetricsService(props.getEndpoint(), props.getApiKey());
    }
}

// 2. Create configuration properties
@ConfigurationProperties(prefix = "company.metrics")
public class MetricsProperties {
    private String endpoint = "https://metrics.company.com";
    private String apiKey;
    // getters/setters
}

// 3. Register the auto-configuration
// src/main/resources/META-INF/spring/
// org.springframework.boot.autoconfigure.AutoConfiguration.imports
// Add one line: com.company.library.MetricsAutoConfiguration
```

Any project that includes your library JAR will automatically get `MetricsService` configured.

---

## Scenario 4: @ConditionalOnProperty for Feature Flags

**Problem:** You have a notification feature that should be enabled/disabled via configuration:

```java
@Configuration
@ConditionalOnProperty(
    name = "app.notifications.enabled",
    havingValue = "true",
    matchIfMissing = false  // Don't create if property not set
)
public class NotificationConfiguration {

    @Bean
    public EmailNotificationService emailNotificationService() {
        return new EmailNotificationService();
    }

    @Bean
    public SmsNotificationService smsNotificationService() {
        return new SmsNotificationService();
    }
}

// application-prod.yml
// app:
//   notifications:
//     enabled: true

// application-test.yml
// app:
//   notifications:
//     enabled: false  (or omit entirely)
```

---

## Scenario 5: @ConditionalOnBean and @ConditionalOnMissingBean

**Problem:** A caching layer should be configured with Redis if a `RedisConnectionFactory` is available, otherwise fall back to a simple in-memory cache:

```java
@Configuration
public class CacheConfiguration {

    // Use Redis if RedisConnectionFactory bean exists
    @Bean
    @ConditionalOnBean(RedisConnectionFactory.class)
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
    }

    // Fall back to in-memory if NO Redis
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager simpleCacheManager() {
        return new ConcurrentMapCacheManager("products", "users");
    }
}
```

---

## Scenario 6: @Profile for Environment-Specific Beans

**Problem:** You need different implementations for development and production environments:

```java
public interface FileStorage {
    void store(String filename, byte[] data);
    byte[] retrieve(String filename);
}

@Service
@Profile("dev")  // Only active in 'dev' profile
public class LocalFileStorage implements FileStorage {

    private final Path storageDir = Paths.get("/tmp/uploads");

    @Override
    public void store(String filename, byte[] data) {
        Files.write(storageDir.resolve(filename), data);
    }

    @Override
    public byte[] retrieve(String filename) {
        return Files.readAllBytes(storageDir.resolve(filename));
    }
}

@Service
@Profile("prod")  // Only active in 'prod' profile
public class S3FileStorage implements FileStorage {

    private final AmazonS3 s3Client;
    private final String bucketName;

    @Override
    public void store(String filename, byte[] data) {
        s3Client.putObject(bucketName, filename, new ByteArrayInputStream(data), null);
    }

    @Override
    public byte[] retrieve(String filename) {
        S3Object obj = s3Client.getObject(bucketName, filename);
        return obj.getObjectContent().readAllBytes();
    }
}
```

```bash
# Activate profile:
java -jar app.jar --spring.profiles.active=prod
# or in application.yml:
# spring.profiles.active: prod
```

---

## Scenario 7: @ConditionalOnClass — Safe Classpath Detection

**Problem:** Your auto-configuration should only apply when Jackson is on the classpath, but you don't want a `ClassNotFoundException` if it's not there:

```java
@AutoConfiguration
@ConditionalOnClass(ObjectMapper.class)  // Safe — if class not present, condition is false
public class JacksonAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(FAIL_ON_UNKNOWN_PROPERTIES, false)
            .registerModule(new JavaTimeModule());
    }
}
```

**Note:** `@ConditionalOnClass` does NOT load the class — it just checks if it's available without triggering `ClassNotFoundException`. This is why it's safe to use even when the class might not be on the classpath.

---

## Scenario 8: Overriding Auto-Configured Beans

**Problem:** Spring Boot auto-configures a `Jackson ObjectMapper`, but you need to customize it:

```java
// If you define your own ObjectMapper bean, auto-configuration backs off
// because of @ConditionalOnMissingBean(ObjectMapper.class) in JacksonAutoConfiguration

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(FAIL_ON_UNKNOWN_PROPERTIES, false)
            .configure(WRITE_DATES_AS_TIMESTAMPS, false)
            .registerModule(new JavaTimeModule())
            .setSerializationInclusion(JsonInclude.Include.NON_NULL);
    }
}
```

Spring sees your `ObjectMapper` bean and the `@ConditionalOnMissingBean` in `JacksonAutoConfiguration` evaluates to false — auto-configuration backs off entirely.

---

## Scenario 9: Auto-Configuration Debug and Reporting

**Problem:** You need to understand why a particular auto-configuration class is or isn't being applied:

```bash
# Enable auto-configuration report
java -jar app.jar --debug

# Or in application.yml:
# logging:
#   level:
#     org.springframework.boot.autoconfigure: DEBUG

# Output shows:
# Positive matches (auto-configs that DID apply):
#    DataSourceAutoConfiguration matched:
#      - @ConditionalOnClass found required class 'javax.sql.DataSource'
#
# Negative matches (auto-configs that did NOT apply):
#    MongoAutoConfiguration:
#      Did not match:
#        - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

You can also use the `/actuator/conditions` endpoint (with Spring Boot Actuator):

```bash
curl http://localhost:8080/actuator/conditions
```

---

## Scenario 10: Custom @Conditional

**Problem:** You need to conditionally register a bean based on whether a specific external file exists:

```java
public class FileExistsCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // Get the file path from the annotation attribute
        Map<String, Object> attributes =
            metadata.getAnnotationAttributes(ConditionalOnFileExists.class.getName());
        String filePath = (String) attributes.get("value");

        File file = new File(filePath);
        return file.exists() && file.isFile();
    }
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(FileExistsCondition.class)
public @interface ConditionalOnFileExists {
    String value();
}

// Usage:
@Bean
@ConditionalOnFileExists("/opt/app/license.key")
public LicensedFeatureService licensedFeatureService() {
    return new LicensedFeatureService();
}
```

---

## Scenario 11: Auto-Configuration Order with @AutoConfigureBefore/@AutoConfigureAfter

**Problem:** Your auto-configuration depends on another auto-configuration being applied first:

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
// or equivalently:
@AutoConfiguration
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class LiquibaseMigrationAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(SpringLiquibase.class)
    public SpringLiquibase liquibase(DataSource dataSource) {
        // DataSource is guaranteed to exist because DataSourceAutoConfiguration ran first
        SpringLiquibase liquibase = new SpringLiquibase();
        liquibase.setDataSource(dataSource);
        liquibase.setChangeLog("classpath:db/changelog/db.changelog-master.yml");
        return liquibase;
    }
}
```

---

## Scenario 12: spring.factories vs AutoConfiguration.imports

**Question:** What changed between Spring Boot 2.6 and 2.7 regarding auto-configuration registration?

**Old approach (still works in Spring Boot 3 for compatibility):**

```properties
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyFirstAutoConfiguration,\
  com.example.MySecondAutoConfiguration
```

**New approach (Spring Boot 2.7+ / Spring Boot 3):**

```text
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyFirstAutoConfiguration
com.example.MySecondAutoConfiguration
```

**Reason for change:** The new format is more explicit, doesn't overload `spring.factories` with multiple types of entries, and is easier to process. The old `spring.factories` approach still works but is discouraged for new libraries.

---

## Scenario 13: Testing Auto-Configuration

**Problem:** You've written a custom auto-configuration class and want to test it in isolation:

```java
@Test
void shouldAutoConfigureMetricsService() {
    ApplicationContextRunner contextRunner = new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(MetricsAutoConfiguration.class))
        .withPropertyValues("company.metrics.endpoint=https://test.example.com");

    contextRunner.run(context -> {
        assertThat(context).hasSingleBean(MetricsService.class);
        MetricsService service = context.getBean(MetricsService.class);
        assertThat(service.getEndpoint()).isEqualTo("https://test.example.com");
    });
}

@Test
void shouldNotAutoConfigureWhenBeanAlreadyPresent() {
    ApplicationContextRunner contextRunner = new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(MetricsAutoConfiguration.class))
        .withBean(MetricsService.class, () -> new MockMetricsService());

    contextRunner.run(context -> {
        // Our auto-configuration should back off
        assertThat(context).hasSingleBean(MetricsService.class);
        assertThat(context.getBean(MetricsService.class))
            .isInstanceOf(MockMetricsService.class);
    });
}
```

`ApplicationContextRunner` is the recommended way to test auto-configuration classes in isolation.
