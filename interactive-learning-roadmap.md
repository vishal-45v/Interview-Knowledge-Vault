# Interactive Learning Roadmap

> Click any topic to navigate directly to that section. Track your progress by checking off completed topics.

---

## Quick-Start Guide

**Are you a beginner?** → Start at [Phase 1: Foundation](#phase-1-foundation)
**Intermediate Java developer?** → Jump to [Phase 2: Spring Boot](#phase-2-spring-boot--databases)
**Experienced engineer refreshing?** → Go to [Phase 3: Advanced Systems](#phase-3-advanced-systems)
**Interview in < 2 weeks?** → See [Rapid Review Track](#rapid-review-track-2-weeks)

---

## Rapid Review Track (2 Weeks)

If your interview is soon, focus on these high-impact topics:

### Week 1 — Core Concepts
| Day | Topic | Link | Time |
|-----|-------|------|------|
| 1 | HashMap internals + ConcurrentHashMap | [Collections](./01-core-java/06-collections/theory-questions.md) | 2h |
| 1 | String pool + immutability | [Java Basics](./01-core-java/01-java-basics/theory-questions.md) | 1h |
| 2 | Thread lifecycle + deadlock prevention | [Multithreading](./01-core-java/03-multithreading/theory-questions.md) | 3h |
| 3 | CompletableFuture + ExecutorService | [Multithreading Scenarios](./01-core-java/03-multithreading/scenario-questions.md) | 2h |
| 3 | Java 8 Streams + Optional | [Java 8](./01-core-java/14-java8-functional/theory-questions.md) | 2h |
| 4 | @Transactional propagation + isolation | [Transactions](./02-spring-boot/transactions-propagation/scenarios.md) | 2h |
| 4 | N+1 problem + solutions | [N+1](./02-spring-boot/n-plus-one-problem/scenarios.md) | 1h |
| 5 | Spring DI + bean lifecycle | [Spring DI](./02-spring-boot/spring-core-di/scenarios.md) | 2h |
| 5 | Spring Security + JWT | [Security](./02-spring-boot/spring-security-jwt/scenarios.md) | 2h |
| 6-7 | System Design practice | [System Design](./06-system-design/) | 4h |

### Week 2 — Scenarios & Practice
| Day | Topic | Link | Time |
|-----|-------|------|------|
| 8 | Debugging scenarios | [Debugging](./07-backend-debugging-scenarios/) | 3h |
| 9 | Production failure cases | [Failures](./08-production-failure-scenarios/) | 2h |
| 10 | Docker + AWS deployment | [Docker](./03-docker-java-backend/) | 2h |
| 11 | Database optimization | [DB/Cache](./05-database-caching/) | 2h |
| 12-14 | Mock interviews + follow-up trap questions | All trap files | 6h |

---

## Phase 1: Foundation

### 1.1 Java Language Fundamentals
> Build a rock-solid understanding of how Java works under the hood

- [ ] **JVM Architecture** → [theory-questions.md](./01-core-java/01-java-basics/theory-questions.md)
  - JVM vs JDK vs JRE
  - Bytecode and class loading
  - Heap vs Stack memory
  - Garbage collection basics

- [ ] **Data Types & Variables** → [theory-questions.md](./01-core-java/01-java-basics/theory-questions.md)
  - Primitive types and their sizes
  - Autoboxing and unboxing
  - Integer cache (-128 to 127)
  - Pass-by-value semantics

- [ ] **String Handling** → [theory-questions.md](./01-core-java/01-java-basics/theory-questions.md)
  - String pool and interning
  - String immutability reasons
  - `==` vs `.equals()` trap
  - StringBuilder vs StringBuffer

- [ ] **Keywords Deep Dive** → [scenario-questions.md](./01-core-java/01-java-basics/scenario-questions.md)
  - `static`: fields, methods, blocks
  - `final`: variables, methods, classes
  - `transient`, `volatile`, `synchronized`
  - Access modifiers matrix

**Check your understanding:** [Follow-up traps](./01-core-java/01-java-basics/follow-up-traps.md)

---

### 1.2 Object-Oriented Design
> Master OOP principles that every senior interview expects

- [ ] **Four Pillars** → [theory-questions.md](./01-core-java/02-oop-design/theory-questions.md)
  - Abstraction: hiding complexity
  - Encapsulation: data + behavior bundling
  - Inheritance: is-a relationships
  - Polymorphism: compile-time vs runtime

- [ ] **Interfaces vs Abstract Classes** → [scenario-questions.md](./01-core-java/02-oop-design/scenario-questions.md)
  - When to choose each
  - Default methods in interfaces (Java 8+)
  - Diamond problem and resolution
  - Marker interfaces

- [ ] **SOLID Principles** → [structured-answers.md](./01-core-java/02-oop-design/structured-answers.md)
  - Single Responsibility Principle
  - Open/Closed Principle
  - Liskov Substitution Principle
  - Interface Segregation Principle
  - Dependency Inversion Principle

- [ ] **Composition vs Inheritance** → [analogy-explanations.md](./01-core-java/02-oop-design/analogy-explanations.md)

---

### 1.3 Exception Handling
> Handle errors like a production engineer

- [ ] **Exception Hierarchy** → [theory-questions.md](./01-core-java/05-exception-handling/theory-questions.md)
- [ ] **try-with-resources** → [scenario-questions.md](./01-core-java/05-exception-handling/scenario-questions.md)
- [ ] **Custom Exceptions Design** → [structured-answers.md](./01-core-java/05-exception-handling/structured-answers.md)
- [ ] **Exception Chaining** → [follow-up-traps.md](./01-core-java/05-exception-handling/follow-up-traps.md)

---

### 1.4 Collections Framework
> One of the most heavily tested topics in Java interviews

- [ ] **List Implementations** → [theory-questions.md](./01-core-java/06-collections/theory-questions.md)
  - ArrayList vs LinkedList
  - Vector (and why it's deprecated)
  - CopyOnWriteArrayList

- [ ] **Map Implementations** → [theory-questions.md](./01-core-java/06-collections/theory-questions.md)
  - HashMap internals (buckets, load factor, rehashing)
  - LinkedHashMap vs TreeMap
  - ConcurrentHashMap segmentation

- [ ] **Set Implementations** → [scenario-questions.md](./01-core-java/06-collections/scenario-questions.md)
  - HashSet, LinkedHashSet, TreeSet
  - How HashSet uses HashMap internally

- [ ] **Queue & Deque** → [scenario-questions.md](./01-core-java/06-collections/scenario-questions.md)
  - PriorityQueue ordering
  - ArrayDeque vs LinkedList

**Visual guide:** [diagram-explanations.md](./01-core-java/06-collections/diagram-explanations.md)

---

## Phase 2: Spring Boot & Databases

### 2.1 Spring Core & Dependency Injection
> Understand Spring's magic, not just its annotations

- [ ] **IoC Container** → [scenarios.md](./02-spring-boot/spring-core-di/scenarios.md)
- [ ] **Injection Types** → [answers.md](./02-spring-boot/spring-core-di/answers.md)
  - Constructor injection (recommended)
  - Setter injection
  - Field injection (anti-pattern why?)
- [ ] **@Qualifier and @Primary** → [trap-questions.md](./02-spring-boot/spring-core-di/trap-questions.md)
- [ ] **Circular Dependencies** → [scenarios.md](./02-spring-boot/spring-core-di/scenarios.md)

**Architecture overview:** [architecture-notes.md](./02-spring-boot/spring-core-di/architecture-notes.md)

---

### 2.2 Bean Lifecycle & Scopes
> Know what happens before your @Bean method returns

- [ ] **Bean Scopes** → [scenarios.md](./02-spring-boot/bean-lifecycle-scopes/scenarios.md)
- [ ] **Lifecycle Hooks** → [answers.md](./02-spring-boot/bean-lifecycle-scopes/answers.md)
- [ ] **BeanPostProcessor** → [trap-questions.md](./02-spring-boot/bean-lifecycle-scopes/trap-questions.md)

---

### 2.3 Auto-Configuration
> How Spring Boot's magic actually works

- [ ] **@SpringBootApplication** → [scenarios.md](./02-spring-boot/spring-boot-autoconfiguration/scenarios.md)
- [ ] **spring.factories** → [answers.md](./02-spring-boot/spring-boot-autoconfiguration/answers.md)
- [ ] **Custom Starters** → [trap-questions.md](./02-spring-boot/spring-boot-autoconfiguration/trap-questions.md)

---

### 2.4 REST API Design
> Design APIs that stand up to real-world scrutiny

- [ ] **RESTful Principles** → [scenarios.md](./02-spring-boot/rest-api-design/scenarios.md)
- [ ] **Versioning Strategies** → [answers.md](./02-spring-boot/rest-api-design/answers.md)
- [ ] **HATEOAS & Pagination** → [trap-questions.md](./02-spring-boot/rest-api-design/trap-questions.md)

---

### 2.5 Transactions & JPA
> The most failure-prone area in Spring Boot applications

- [ ] **@Transactional Propagation** → [scenarios.md](./02-spring-boot/transactions-propagation/scenarios.md)
  - REQUIRED (default)
  - REQUIRES_NEW
  - NESTED
  - SUPPORTS, NOT_SUPPORTED, NEVER, MANDATORY

- [ ] **Self-Invocation Problem** → [trap-questions.md](./02-spring-boot/transactions-propagation/trap-questions.md)

- [ ] **JPA Entity Lifecycle** → [scenarios.md](./02-spring-boot/hibernate-jpa-internals/scenarios.md)
  - Transient → Persistent → Detached → Removed
  - Dirty checking
  - EntityManager operations

- [ ] **N+1 Problem** → [scenarios.md](./02-spring-boot/n-plus-one-problem/scenarios.md)
  - Detection with Hibernate statistics
  - JOIN FETCH solution
  - @EntityGraph solution
  - @BatchSize solution

---

### 2.6 Spring Security & JWT
> Authentication and authorization patterns

- [ ] **Security Filter Chain** → [scenarios.md](./02-spring-boot/spring-security-jwt/scenarios.md)
- [ ] **JWT Flow** → [answers.md](./02-spring-boot/spring-security-jwt/answers.md)
- [ ] **Method Security** → [trap-questions.md](./02-spring-boot/spring-security-jwt/trap-questions.md)

---

## Phase 3: Advanced Systems

### 3.1 Multithreading Deep Dive
> Concurrency is hard — master it and stand out

- [ ] **Thread Pools** → [theory-questions.md](./01-core-java/03-multithreading/theory-questions.md)
  - Core pool size vs max pool size
  - Work queue types
  - Rejection policies

- [ ] **CompletableFuture** → [scenario-questions.md](./01-core-java/03-multithreading/scenario-questions.md)
  - thenApply vs thenCompose
  - exceptionally and handle
  - allOf vs anyOf

- [ ] **Locks & Synchronization** → [follow-up-traps.md](./01-core-java/03-multithreading/follow-up-traps.md)
  - ReentrantLock advantages over synchronized
  - ReadWriteLock use case
  - StampedLock

- [ ] **Deadlock, Livelock, Starvation** → [structured-answers.md](./01-core-java/03-multithreading/structured-answers.md)

---

### 3.2 Docker & Containerization
> Run Java apps reliably in containers

- [ ] **Docker Basics** → [docker-fundamentals.md](./03-docker-java-backend/docker-fundamentals.md)
- [ ] **Spring Boot Containerization** → [dockerizing-springboot.md](./03-docker-java-backend/dockerizing-springboot.md)
- [ ] **JVM in Containers** → [jvm-memory-in-containers.md](./03-docker-java-backend/jvm-memory-in-containers.md)
- [ ] **Production Mistakes** → [docker-production-mistakes.md](./03-docker-java-backend/docker-production-mistakes.md)

---

### 3.3 Cloud & AWS
> Deploy and operate Java applications in the cloud

- [ ] **AWS Core Services** → [aws-springboot-deployment.md](./04-cloud-devops/aws-springboot-deployment.md)
- [ ] **Kubernetes** → [kubernetes-basics.md](./04-cloud-devops/kubernetes-basics.md)
- [ ] **CI/CD Pipelines** → [ci-cd-pipelines.md](./04-cloud-devops/ci-cd-pipelines.md)
- [ ] **Monitoring & Tracing** → [logging-monitoring-tracing.md](./04-cloud-devops/logging-monitoring-tracing.md)

---

### 3.4 Database & Caching
> Every backend system lives or dies by its data layer

- [ ] **SQL Optimization** → [sql-indexing-optimization.md](./05-database-caching/sql-indexing-optimization.md)
- [ ] **Isolation Levels** → [transactions-isolation-levels.md](./05-database-caching/transactions-isolation-levels.md)
- [ ] **Redis Patterns** → [redis-caching-strategies.md](./05-database-caching/redis-caching-strategies.md)
- [ ] **Cache Invalidation** → [cache-invalidation-scenarios.md](./05-database-caching/cache-invalidation-scenarios.md)

---

## Phase 4: Expert Level

### 4.1 System Design
> Design systems that handle millions of users

- [ ] **Interview Framework** → [system-design-interview-strategy.md](./06-system-design/system-design-interview-strategy.md)
- [ ] **Core Concepts** → [core-system-design-concepts.md](./06-system-design/core-system-design-concepts.md)
- [ ] **Low-Level Design** → [low-level-design-questions.md](./06-system-design/low-level-design-questions.md)
  - URL shortener
  - Rate limiter
  - Parking lot
  - Vending machine

- [ ] **High-Level Design** → [high-level-architecture-questions.md](./06-system-design/high-level-architecture-questions.md)
  - Twitter feed
  - Netflix streaming
  - Uber dispatch
  - WhatsApp messaging
  - Payment system

---

### 4.2 Production Debugging
> Show you can actually debug real problems

- [ ] **Thread Deadlocks** → [thread-deadlock-debugging.md](./07-backend-debugging-scenarios/thread-deadlock-debugging.md)
- [ ] **Memory Leaks** → [memory-leak-debugging.md](./07-backend-debugging-scenarios/memory-leak-debugging.md)
- [ ] **High CPU** → [high-cpu-debugging.md](./07-backend-debugging-scenarios/high-cpu-debugging.md)
- [ ] **API Latency** → [api-latency-debugging.md](./07-backend-debugging-scenarios/api-latency-debugging.md)
- [ ] **DB Performance** → [database-performance-debugging.md](./07-backend-debugging-scenarios/database-performance-debugging.md)
- [ ] **Microservice Failures** → [microservice-failure-debugging.md](./07-backend-debugging-scenarios/microservice-failure-debugging.md)

---

### 4.3 Production Failure Post-Mortems
> Learn from real outages before your interview

- [ ] **Database Outage** → [database-outage-scenario.md](./08-production-failure-scenarios/database-outage-scenario.md)
- [ ] **Cache Meltdown** → [cache-meltdown.md](./08-production-failure-scenarios/cache-meltdown.md)
- [ ] **Thread Pool Exhaustion** → [thread-pool-exhaustion.md](./08-production-failure-scenarios/thread-pool-exhaustion.md)
- [ ] **Distributed Partition** → [distributed-system-partition.md](./08-production-failure-scenarios/distributed-system-partition.md)
- [ ] **Message Queue Backlog** → [message-queue-backlog.md](./08-production-failure-scenarios/message-queue-backlog.md)
- [ ] **Deployment Failure** → [deployment-failure.md](./08-production-failure-scenarios/deployment-failure.md)

---

## Interview Readiness Checklist

### Core Java
- [ ] Can explain HashMap put() operation step by step including hash collision handling
- [ ] Can describe all thread states and transitions
- [ ] Can spot a deadlock in code and explain prevention
- [ ] Can explain Java Memory Model and happens-before
- [ ] Can write a correct singleton with double-checked locking
- [ ] Can explain why String is immutable in Java
- [ ] Can describe what happens during autoboxing with Integer cache
- [ ] Can write a generic class with bounded wildcards

### Spring Boot
- [ ] Can explain the difference between REQUIRED and REQUIRES_NEW propagation
- [ ] Can describe what happens when @Transactional annotates a private method
- [ ] Can explain the N+1 problem and demonstrate 3 solutions
- [ ] Can describe Spring Security filter chain order
- [ ] Can explain how auto-configuration works with @Conditional
- [ ] Can describe bean initialization order with multiple beans

### System Design
- [ ] Can design a URL shortener with 1 billion URLs
- [ ] Can explain CAP theorem with concrete examples
- [ ] Can design a rate limiter (token bucket or sliding window)
- [ ] Can describe trade-offs of microservices vs monolith
- [ ] Can design a notification system for 10M users

### Production Readiness
- [ ] Can analyze a thread dump to find deadlocks
- [ ] Can identify a memory leak from a heap dump description
- [ ] Can describe what to check when API latency spikes
- [ ] Can explain how to handle a Redis cluster failure gracefully
- [ ] Can describe a deployment rollback procedure

---

## Diagram Reference

All architectural diagrams:
- [System Design Patterns](./diagrams/system-design-patterns.md)
- [Distributed Systems Patterns](./diagrams/distributed-systems-patterns.md)
- [Backend Architecture Patterns](./diagrams/backend-architecture-patterns.md)
- [Database Scaling Patterns](./diagrams/database-scaling-patterns.md)
