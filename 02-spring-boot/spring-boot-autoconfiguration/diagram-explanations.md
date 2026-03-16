# Spring Boot Auto-Configuration — Diagram Explanations

---

## Diagram 1: Auto-Configuration Decision Flow

```
  Spring Boot Startup
        │
        ▼
  Read AutoConfiguration.imports (150+ entries)
        │
        ▼
  For each auto-configuration class:
  ┌─────────────────────────────────────────────┐
  │  1. @ConditionalOnClass                     │
  │     Is the required class on classpath?     │
  │     NO ─────────────────────── SKIP ✗       │
  │     YES ▼                                   │
  │  2. @ConditionalOnMissingBean               │
  │     Does the required bean NOT exist?       │
  │     NO (bean exists) ────────── SKIP ✗      │
  │     YES (no bean) ▼                         │
  │  3. @ConditionalOnProperty                  │
  │     Is the property set correctly?          │
  │     NO ─────────────────────── SKIP ✗       │
  │     YES ▼                                   │
  │  APPLY auto-configuration ✓                 │
  └─────────────────────────────────────────────┘
```

---

## Diagram 2: User Bean vs Auto-Configured Bean

```
  Auto-configuration (DataSourceAutoConfiguration):
  @Bean
  @ConditionalOnMissingBean(DataSource.class)
  public DataSource dataSource() { ... }

  SCENARIO A: No user DataSource defined
  ┌─────────────────────────────────────────────┐
  │  Context scan: no DataSource bean found      │
  │  @ConditionalOnMissingBean → TRUE            │
  │  Auto-config creates DataSource (HikariCP)  ✓│
  └─────────────────────────────────────────────┘

  SCENARIO B: User defines DataSource
  ┌─────────────────────────────────────────────┐
  │  User @Bean DataSource defined first         │
  │  @ConditionalOnMissingBean → FALSE           │
  │  Auto-config SKIPPED (backs off)           ✓ │
  │  User's DataSource used                      │
  └─────────────────────────────────────────────┘

  PRINCIPLE: User configuration always wins over auto-configuration
```

---

## Diagram 3: spring.factories vs AutoConfiguration.imports

```
  Spring Boot < 2.7:
  META-INF/spring.factories
  ┌───────────────────────────────────────────────┐
  │  org.springframework.boot.autoconfigure       │
  │    .EnableAutoConfiguration=\                  │
  │      com.example.AutoConfig1,\                 │
  │      com.example.AutoConfig2                   │
  │  org.springframework.context                   │
  │    .ApplicationContextInitializer=\            │
  │      com.example.MyInitializer                 │
  │  (mixed multiple types in one file)            │
  └───────────────────────────────────────────────┘

  Spring Boot 2.7+:
  META-INF/spring/
  org.springframework.boot.autoconfigure.AutoConfiguration.imports
  ┌───────────────────────────────────────────────┐
  │  com.example.AutoConfig1                       │
  │  com.example.AutoConfig2                       │
  │  (dedicated file, one class per line)          │
  └───────────────────────────────────────────────┘

  Benefits of new format:
  - Each concern has its own file
  - Easier to read and maintain
  - Better IDE support
  - Works with GraalVM native compilation
```
