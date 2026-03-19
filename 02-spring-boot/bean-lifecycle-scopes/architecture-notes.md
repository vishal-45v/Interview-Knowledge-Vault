# Bean Lifecycle & Scopes — Architecture Notes

> Spring bean lifecycle internals, BeanDefinition structure, and scope proxy mechanics.

---

## BeanDefinition — What Spring Stores Per Bean

Before any bean is instantiated, Spring creates a `BeanDefinition` for each discovered bean:

```
BeanDefinition {
    beanClassName:     "com.example.service.UserService"
    scope:             "singleton"
    lazyInit:          false
    dependsOn:         []
    constructorArgs:   []
    propertyValues:    []
    initMethodName:    "init"      // from @Bean(initMethod)
    destroyMethodName: "cleanup"   // from @Bean(destroyMethod)
    factoryBeanName:   null        // or @Configuration class name
    factoryMethodName: null        // or @Bean method name
    primary:           false
    abstract:          false
    autowireMode:      AUTOWIRE_CONSTRUCTOR
}
```

`BeanFactoryPostProcessor` can modify these definitions before instantiation.

---

## Startup Sequence — Detailed Flow

```
SpringApplication.run()
        │
        ▼
1. Create ApplicationContext
        │
        ▼
2. Load BeanDefinitions
   ├── ClassPathBeanDefinitionScanner → @Component classes
   ├── ConfigurationClassParser     → @Configuration/@Bean
   └── XML/annotation based config
        │
        ▼
3. BeanFactoryPostProcessors run (in order)
   ├── ConfigurationClassPostProcessor (processes @Configuration)
   └── PropertySourcesPlaceholderConfigurer (resolves ${...})
        │
        ▼
4. BeanPostProcessors registered (not yet invoked)
        │
        ▼
5. Pre-instantiate all non-lazy singletons
   For each bean (in dependency order):
   ├── Create instance (constructor)
   ├── Inject dependencies
   ├── BeanPostProcessor.postProcessBeforeInitialization()
   ├── @PostConstruct / afterPropertiesSet() / initMethod
   └── BeanPostProcessor.postProcessAfterInitialization()
        │
        ▼
6. ContextRefreshedEvent published
        │
        ▼
7. CommandLineRunner / ApplicationRunner execute
        │
        ▼
8. ApplicationReadyEvent published
        │
        ▼
Application running...
        │
        ▼
Shutdown signal received
        │
        ▼
9. ContextClosedEvent published
        │
        ▼
10. Singleton beans destroyed (reverse order)
    ├── @PreDestroy / destroy() / destroyMethod
    └── Resources released
```

---

## Scope Proxy Internal Mechanics

```
ScopedProxyMode.TARGET_CLASS creates a CGLIB proxy:

Singleton bean (AuditService)
    │
    │ @Autowired UserContext userContext
    │
    └── UserContext proxy (CGLIB subclass)
        ┌────────────────────────────────────────────────┐
        │  UserContextProxy extends UserContext          │
        │                                                │
        │  String getUserId() {                          │
        │      // Delegate to real bean for this scope   │
        │      UserContext real =                        │
        │          scopeRegistry.get("userContext",      │
        │                            "request");         │
        │      return real.getUserId();                  │
        │  }                                             │
        └────────────────────────────────────────────────┘

Request 1 (Thread A):
    AuditService.userContext.getUserId()
    → Proxy looks up Thread A's request scope
    → Returns UserContext{userId="alice"}

Request 2 (Thread B):
    AuditService.userContext.getUserId()
    → Proxy looks up Thread B's request scope
    → Returns UserContext{userId="bob"}
```

---

## SmartLifecycle — Phase-Based Startup/Shutdown

```
Phase ordering for startup:
    Phase INT_MIN_VALUE starts first
    Phase 0 (default Lifecycle) starts next
    Phase INT_MAX_VALUE starts last

Shutdown is the reverse:
    Phase INT_MAX_VALUE stops first
    Phase 0 stops next
    Phase INT_MIN_VALUE stops last

Example phases:
    DataSource:          Phase -1000 (start early)
    TransactionManager:  Phase -500
    Application beans:   Phase 0 (default)
    MessageConsumers:    Phase 1000 (start last)
    └── Start consuming only after all processing beans are ready
```

---

## Three-Phase Object Instantiation

```
Phase 1: Instantiation
  ├── BeanDefinition says: use constructor
  ├── Spring resolves constructor parameters
  └── Calls constructor → object created

Phase 2: Population (Dependency Injection)
  ├── AutowiredAnnotationBeanPostProcessor.postProcessBeforeInstantiation()
  │   → Finds @Autowired fields and methods
  │   → Resolves beans for each dependency
  │   → Injects via reflection (Field.set() or Method.invoke())
  └── @Value placeholders resolved via PropertySourcesPlaceholderConfigurer

Phase 3: Initialization
  ├── Aware interfaces called (BeanNameAware, ApplicationContextAware, etc.)
  ├── BeanPostProcessor.postProcessBeforeInitialization()
  ├── @PostConstruct method
  ├── InitializingBean.afterPropertiesSet()
  ├── initMethod from @Bean
  └── BeanPostProcessor.postProcessAfterInitialization()
      └── AOP proxy creation happens here
```
