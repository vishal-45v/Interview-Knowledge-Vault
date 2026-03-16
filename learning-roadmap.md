# Learning Roadmap — Java Backend Engineering Interview Preparation

> A structured 12-week program to take you from Java fundamentals to senior-level interview readiness.

---

## Overview

This roadmap is divided into four phases:

| Phase | Duration | Focus |
|-------|----------|-------|
| Foundation | Weeks 1–3 | Core Java fundamentals |
| Intermediate | Weeks 4–6 | Spring Boot + Databases |
| Advanced | Weeks 7–9 | Architecture, Cloud, DevOps |
| Expert | Weeks 10–12 | Production systems, debugging, mock interviews |

---

## Phase 1: Foundation (Weeks 1–3)

### Goals
- Master Java fundamentals deeply enough to discuss internals
- Understand JVM behavior, memory model, and class loading
- Be confident about OOP design, exceptions, and collections

### Week 1: JVM Internals & Java Basics

**Topics:**
- JVM vs JDK vs JRE — understand the difference and their roles
- Bytecode compilation and class loading
- String pool and String immutability
- Primitive types vs wrapper types, autoboxing/unboxing pitfalls
- Pass-by-value vs pass-by-reference (Java is always pass-by-value)
- `==` vs `.equals()` and `hashCode()` contract
- `static` keyword: static fields, methods, blocks, inner classes
- `final` keyword: final variables, methods, classes
- Access modifiers: public, protected, package-private, private

**Files to Study:**
- [01-java-basics/theory-questions.md](./01-core-java/01-java-basics/theory-questions.md)
- [01-java-basics/scenario-questions.md](./01-core-java/01-java-basics/scenario-questions.md)
- [01-java-basics/structured-answers.md](./01-core-java/01-java-basics/structured-answers.md)

**Daily Plan:**
- Day 1–2: JVM internals, bytecode, classloading
- Day 3–4: Strings, primitives, autoboxing
- Day 5–7: static/final, access modifiers, type casting

**Practice Questions:** Aim to answer 10 theory + 5 scenario questions per day

---

### Week 2: OOP Design & Exception Handling

**Topics:**
- Four pillars: Abstraction, Encapsulation, Inheritance, Polymorphism
- Interfaces vs abstract classes — when to use each
- Composition over inheritance principle
- SOLID principles with real-world examples
- Method overloading vs overriding
- Checked vs unchecked exceptions
- try-with-resources and AutoCloseable
- Exception chaining and suppressed exceptions
- Custom exception hierarchy design

**Files to Study:**
- [02-oop-design/theory-questions.md](./01-core-java/02-oop-design/theory-questions.md)
- [02-oop-design/scenario-questions.md](./01-core-java/02-oop-design/scenario-questions.md)
- [05-exception-handling/theory-questions.md](./01-core-java/05-exception-handling/theory-questions.md)

**Daily Plan:**
- Day 1–3: OOP pillars, interfaces vs abstract classes, SOLID
- Day 4–5: Overloading, overriding, polymorphism traps
- Day 6–7: Exception handling best practices

---

### Week 3: Collections & Multithreading Basics

**Topics:**
- Collection hierarchy: List, Set, Map, Queue
- ArrayList vs LinkedList internals
- HashMap: hashCode, equals, bucket structure, load factor, rehashing
- ConcurrentHashMap vs HashMap vs Hashtable
- TreeMap, LinkedHashMap, PriorityQueue
- Fail-fast vs fail-safe iterators
- Thread lifecycle states
- Creating threads: Runnable vs Thread
- synchronized keyword: method vs block
- volatile keyword and visibility guarantee

**Files to Study:**
- [06-collections/theory-questions.md](./01-core-java/06-collections/theory-questions.md)
- [06-collections/scenario-questions.md](./01-core-java/06-collections/scenario-questions.md)
- [03-multithreading/theory-questions.md](./01-core-java/03-multithreading/theory-questions.md)

---

## Phase 2: Intermediate (Weeks 4–6)

### Goals
- Master concurrency patterns and Java 8 features
- Understand Spring Boot's core concepts deeply
- Be able to reason about JPA/Hibernate behavior

### Week 4: Advanced Concurrency & Java 8

**Topics:**
- ExecutorService and thread pools
- CompletableFuture and async programming
- CountDownLatch, CyclicBarrier, Semaphore use cases
- ReentrantLock, ReadWriteLock
- Deadlock detection and prevention
- Java Memory Model (happens-before relationship)
- Lambda expressions and functional interfaces
- Stream API: map, filter, reduce, collect, flatMap
- Optional: correct usage patterns
- Method references (4 types)

**Files to Study:**
- [03-multithreading/theory-questions.md](./01-core-java/03-multithreading/theory-questions.md)
- [03-multithreading/scenario-questions.md](./01-core-java/03-multithreading/scenario-questions.md)
- [14-java8-functional/theory-questions.md](./01-core-java/14-java8-functional/theory-questions.md)
- [14-java8-functional/scenario-questions.md](./01-core-java/14-java8-functional/scenario-questions.md)

---

### Week 5: Spring Boot Core

**Topics:**
- IoC container and dependency injection
- Constructor vs setter vs field injection
- Bean scopes: singleton, prototype, request, session
- Bean lifecycle: creation, initialization, destruction
- @PostConstruct, @PreDestroy
- BeanFactoryPostProcessor vs BeanPostProcessor
- @SpringBootApplication and auto-configuration
- @Conditional annotations
- Spring profiles

**Files to Study:**
- [spring-core-di/scenarios.md](./02-spring-boot/spring-core-di/scenarios.md)
- [spring-core-di/answers.md](./02-spring-boot/spring-core-di/answers.md)
- [bean-lifecycle-scopes/scenarios.md](./02-spring-boot/bean-lifecycle-scopes/scenarios.md)
- [spring-boot-autoconfiguration/scenarios.md](./02-spring-boot/spring-boot-autoconfiguration/scenarios.md)

---

### Week 6: Spring Boot REST, Transactions, JPA

**Topics:**
- REST API design principles
- @RestController, @RequestMapping family
- Content negotiation and versioning
- Exception handling with @ControllerAdvice
- @Transactional propagation levels
- Isolation levels in Spring transactions
- Self-invocation problem with @Transactional
- JPA entity states: transient, persistent, detached, removed
- Dirty checking mechanism
- First-level vs second-level cache
- N+1 problem detection and solutions

**Files to Study:**
- [rest-api-design/scenarios.md](./02-spring-boot/rest-api-design/scenarios.md)
- [transactions-propagation/scenarios.md](./02-spring-boot/transactions-propagation/scenarios.md)
- [hibernate-jpa-internals/scenarios.md](./02-spring-boot/hibernate-jpa-internals/scenarios.md)
- [n-plus-one-problem/scenarios.md](./02-spring-boot/n-plus-one-problem/scenarios.md)

---

## Phase 3: Advanced (Weeks 7–9)

### Goals
- Understand containerization and cloud deployment patterns
- Master database and caching strategies
- Be able to design systems at scale

### Week 7: Docker & Cloud Fundamentals

**Topics:**
- Docker architecture: daemon, client, registry
- Dockerfile best practices for Java apps
- Multi-stage builds for Spring Boot
- JVM memory flags in containers
- Docker Compose for local development
- AWS core services for Java backends
- EC2 vs ECS vs EKS decision matrix
- Kubernetes: Pods, Deployments, Services, Ingress
- HPA and resource limits

**Files to Study:**
- [03-docker-java-backend/dockerizing-springboot.md](./03-docker-java-backend/dockerizing-springboot.md)
- [03-docker-java-backend/jvm-memory-in-containers.md](./03-docker-java-backend/jvm-memory-in-containers.md)
- [04-cloud-devops/aws-springboot-deployment.md](./04-cloud-devops/aws-springboot-deployment.md)
- [04-cloud-devops/kubernetes-basics.md](./04-cloud-devops/kubernetes-basics.md)

---

### Week 8: Database & Caching Mastery

**Topics:**
- B-tree vs hash indexes
- Composite indexes and covering indexes
- EXPLAIN plan analysis
- Transaction isolation levels: dirty reads, phantom reads, lost updates
- Hibernate performance tuning
- Redis caching strategies: cache-aside, write-through, write-behind
- Cache invalidation and thundering herd problem
- Database migrations with Flyway/Liquibase
- Elasticsearch basics

**Files to Study:**
- [05-database-caching/sql-indexing-optimization.md](./05-database-caching/sql-indexing-optimization.md)
- [05-database-caching/transactions-isolation-levels.md](./05-database-caching/transactions-isolation-levels.md)
- [05-database-caching/redis-caching-strategies.md](./05-database-caching/redis-caching-strategies.md)
- [05-database-caching/hibernate-performance-tuning.md](./05-database-caching/hibernate-performance-tuning.md)

---

### Week 9: System Design

**Topics:**
- System design interview framework (STAR approach)
- CAP theorem and implications
- Horizontal vs vertical scaling decisions
- CDN, load balancing algorithms
- Message queues and event-driven architecture
- Designing for high availability
- Trade-offs: consistency vs availability
- Real designs: URL shortener, rate limiter, notification service
- High-level designs: Twitter, Netflix, Uber

**Files to Study:**
- [06-system-design/system-design-interview-strategy.md](./06-system-design/system-design-interview-strategy.md)
- [06-system-design/core-system-design-concepts.md](./06-system-design/core-system-design-concepts.md)
- [06-system-design/low-level-design-questions.md](./06-system-design/low-level-design-questions.md)
- [06-system-design/high-level-architecture-questions.md](./06-system-design/high-level-architecture-questions.md)

---

## Phase 4: Expert (Weeks 10–12)

### Goals
- Simulate production debugging scenarios
- Review production failure case studies
- Conduct self-mock interviews
- Fill gaps from previous phases

### Week 10: Debugging & Failure Scenarios

**Topics:**
- Thread dump analysis for deadlocks
- Heap dump analysis for memory leaks
- GC overhead limit exceeded causes
- API latency root cause analysis
- Database lock contention investigation
- Microservice cascading failures

**Files to Study:**
- [07-backend-debugging-scenarios/thread-deadlock-debugging.md](./07-backend-debugging-scenarios/thread-deadlock-debugging.md)
- [07-backend-debugging-scenarios/memory-leak-debugging.md](./07-backend-debugging-scenarios/memory-leak-debugging.md)
- [07-backend-debugging-scenarios/api-latency-debugging.md](./07-backend-debugging-scenarios/api-latency-debugging.md)
- [08-production-failure-scenarios/database-outage-scenario.md](./08-production-failure-scenarios/database-outage-scenario.md)
- [08-production-failure-scenarios/cache-meltdown.md](./08-production-failure-scenarios/cache-meltdown.md)

---

### Week 11: Advanced Topics Review

**Topics:**
- Design patterns: which pattern solves which problem
- Advanced generics and type erasure
- Dynamic proxies and AOP internals
- Spring Security filter chain
- JWT authentication flow
- OAuth2 authorization code flow
- CI/CD pipeline design
- Prometheus metrics and alerting

**Files to Study:**
- [15-design-patterns/theory-questions.md](./01-core-java/15-design-patterns/theory-questions.md)
- [08-advanced-java/theory-questions.md](./01-core-java/08-advanced-java/theory-questions.md)
- [spring-security-jwt/scenarios.md](./02-spring-boot/spring-security-jwt/scenarios.md)
- [04-cloud-devops/ci-cd-pipelines.md](./04-cloud-devops/ci-cd-pipelines.md)

---

### Week 12: Mock Interviews & Consolidation

**Activities:**
1. Pick 5 random questions from each section and answer them out loud
2. Review all follow-up-traps.md files for gotcha questions
3. Practice whiteboarding system design questions
4. Review all analogy-explanations.md for conceptual clarity
5. Do 2 full mock interview sessions (1.5 hours each)

**Mock Interview Format:**
- 15 min: Core Java deep dive (5 questions)
- 20 min: Spring Boot scenario (2–3 scenarios)
- 20 min: System design (1 design question)
- 10 min: Debugging scenario (1 production issue)
- 10 min: Behavioral (use STAR method)
- 5 min: Questions for interviewer

---

## Key Resources by Topic

### Core Java
- [Java Basics](./01-core-java/01-java-basics/)
- [Collections Internals](./01-core-java/06-collections/)
- [Multithreading](./01-core-java/03-multithreading/)
- [Java 8 Features](./01-core-java/14-java8-functional/)
- [Design Patterns](./01-core-java/15-design-patterns/)

### Spring Ecosystem
- [Spring DI](./02-spring-boot/spring-core-di/)
- [Transactions](./02-spring-boot/transactions-propagation/)
- [JPA/Hibernate](./02-spring-boot/hibernate-jpa-internals/)
- [Spring Security](./02-spring-boot/spring-security-jwt/)

### Infrastructure
- [Docker](./03-docker-java-backend/)
- [Cloud/DevOps](./04-cloud-devops/)
- [Database/Caching](./05-database-caching/)

### Design & Debugging
- [System Design](./06-system-design/)
- [Debugging](./07-backend-debugging-scenarios/)
- [Production Failures](./08-production-failure-scenarios/)

---

## Success Criteria

Before each interview, you should be able to:

- [ ] Explain HashMap's internal structure without notes
- [ ] Describe 3 ways a deadlock can occur and how to prevent each
- [ ] Design a REST API with proper exception handling from scratch
- [ ] Explain @Transactional propagation levels with examples
- [ ] Describe the N+1 problem and 3 solutions
- [ ] Design a URL shortener at system scale
- [ ] Walk through a Spring Security JWT authentication flow
- [ ] Analyze a GC overhead scenario and propose fixes
- [ ] Explain CAP theorem with a real-world example
- [ ] Describe a production incident and your role in resolving it
