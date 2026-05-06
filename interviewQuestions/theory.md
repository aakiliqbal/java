---
layout: default
title: Theory Questions
---

# Java Backend Theory Questions

<div class="page-intro">
  <p>This page merges the common theory questions from both original README files and removes duplicate answers. It keeps the strongest explanations and adds missing fundamentals from the broader notes.</p>
</div>

## Quick Navigation

- [Core Java](#core-java)
- [Java 8 And Java 21](#java-8-and-java-21)
- [Multithreading](#multithreading)
- [Spring Boot](#spring-boot)
- [Spring Security](#spring-security)
- [Hibernate And Database](#hibernate-and-database)
- [Microservices](#microservices)
- [Kafka And Messaging](#kafka-and-messaging)
- [REST API](#rest-api)
- [Design Patterns And SOLID](#design-patterns-and-solid)
- [Testing, Tools, And AI](#testing-tools-and-ai)

## Core Java

### 1. What are OOP principles with real-world examples?

Object-Oriented Programming models software as objects that combine state and behavior. The four core principles are encapsulation, inheritance, polymorphism, and abstraction.

| Principle | Meaning | Real-world example |
|---|---|---|
| Encapsulation | Hide internal state and expose controlled methods | `BankAccount` keeps `balance` private and exposes `deposit()` or `withdraw()` |
| Inheritance | Reuse behavior from a parent type | `Car` extends `Vehicle` |
| Polymorphism | Same operation behaves differently for different types | `Payment.pay()` behaves differently for UPI, card, and net banking |
| Abstraction | Show what an object does, hide how it does it | `NotificationService.send()` hides email/SMS implementation |

```java
interface Payment {
    void pay(BigDecimal amount);
}

class UpiPayment implements Payment {
    public void pay(BigDecimal amount) {
        System.out.println("Paid by UPI: " + amount);
    }
}

class CardPayment implements Payment {
    public void pay(BigDecimal amount) {
        System.out.println("Paid by card: " + amount);
    }
}
```

In interviews, connect OOP to maintainability: encapsulation protects data, abstraction reduces coupling, polymorphism avoids large `if-else` chains, and inheritance should be used carefully because composition is often safer.

### 2. Interface vs abstract class

An interface defines a contract. An abstract class defines a partial implementation for related classes.

| Point | Interface | Abstract class |
|---|---|---|
| Purpose | Contract/capability | Shared base behavior |
| Multiple inheritance | A class can implement many interfaces | A class can extend only one abstract class |
| Fields | Constants by default | Instance fields allowed |
| Constructor | No constructor for object state | Constructor allowed |
| Best use | Service contracts, strategies, adapters | Common code for related classes |

Use an interface for unrelated implementations like `PaymentStrategy`, `NotificationSender`, or `StorageProvider`. Use an abstract class when several related classes share state or reusable template logic.

### 3. How does `HashMap` work internally?

`HashMap` stores key-value pairs in an internal bucket array. When `put(key, value)` is called:

1. Java calculates the key's `hashCode()`.
2. The hash is spread to reduce poor hash distribution.
3. Bucket index is calculated using `(n - 1) & hash`.
4. If the bucket is empty, the node is inserted.
5. If another key maps to the same bucket, Java compares keys using `equals()`.
6. Collisions are stored as a linked list, and since Java 8, a heavily collided bucket can become a red-black tree.

Important values:

| Concept | Default / behavior |
|---|---|
| Initial capacity | `16` |
| Load factor | `0.75` |
| Resize threshold | capacity * load factor |
| Treeification threshold | bucket reaches `8` nodes and table capacity is at least `64` |
| Average complexity | `O(1)` for `put()` and `get()` |
| Worst case after treeification | `O(log n)` |

`hashCode()` decides the bucket. `equals()` decides whether an existing key is the same logical key. If these methods are implemented incorrectly, `HashMap`, `HashSet`, and `ConcurrentHashMap` can behave incorrectly.

### 4. What is the `equals()` and `hashCode()` contract?

If two objects are equal according to `equals()`, they must return the same `hashCode()`. The reverse is not required because different objects can have the same hash code.

```java
class Employee {
    private final Long id;

    Employee(Long id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Employee employee)) return false;
        return Objects.equals(id, employee.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

If only `equals()` is overridden, a logically equal object may go to a different hash bucket and lookups may fail.

### 5. `HashMap` vs `ConcurrentHashMap`

| Feature | `HashMap` | `ConcurrentHashMap` |
|---|---|---|
| Thread safety | Not thread-safe | Thread-safe for concurrent access |
| Null keys/values | Allows one null key and null values | Does not allow null keys or values |
| Locking | No locking | CAS plus fine-grained bucket locking |
| Iteration | Fail-fast | Weakly consistent |
| Use case | Single-threaded or externally synchronized access | Shared map in multi-threaded code |

In Java 8+, `ConcurrentHashMap` avoids locking the whole map. Reads are mostly lock-free, updates use CAS where possible, and synchronization is applied at bucket level when needed. Its iterators are weakly consistent: they do not throw `ConcurrentModificationException`, but they may reflect some concurrent updates and miss others.

### 6. `ArrayList` vs `LinkedList`

| Point | `ArrayList` | `LinkedList` |
|---|---|---|
| Internal structure | Dynamic array | Doubly linked nodes |
| Random access | Fast, `O(1)` | Slow, `O(n)` |
| Insert/delete at middle | Requires shifting elements | Efficient only if node reference is already known |
| Memory | Lower overhead | Higher overhead due to node references |
| Common backend choice | Preferred for most read-heavy lists | Rare, useful for queue-like operations |

In real projects, `ArrayList` is usually better because iteration and indexed reads are common, and CPU cache locality matters.

### 7. `String` vs `StringBuilder` vs `StringBuffer`

| Type | Mutable | Thread-safe | Best use |
|---|---|---|---|
| `String` | No | Safe because immutable | Constants, keys, normal text values |
| `StringBuilder` | Yes | No | Fast string construction in one thread |
| `StringBuffer` | Yes | Yes, synchronized | Legacy synchronized string construction |

Use `StringBuilder` inside loops or formatting logic where many concatenations happen. Use `String` for values that should not change, especially map keys or DTO fields.

### 8. Checked vs unchecked exceptions

Checked exceptions extend `Exception` but not `RuntimeException`; the compiler forces handling or declaration. Unchecked exceptions extend `RuntimeException`; they usually represent programming errors or invalid runtime state.

| Checked | Unchecked |
|---|---|
| `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| Must catch or declare | Handling is optional |
| Used for recoverable external failures | Used for bugs, validation errors, invalid state |

In Spring services, unchecked custom exceptions are common because global exception handlers can convert them into proper HTTP responses.

### 9. `final`, `finally`, and `finalize()`

`final` is a keyword used to prevent reassignment, overriding, or inheritance. `finally` is a block that runs after `try-catch` for cleanup. `finalize()` was a GC callback method, is deprecated, and should not be used.

```java
final class TokenValidator {
    final boolean validate(String token) {
        return token != null && !token.isBlank();
    }
}
```

Use `try-with-resources` instead of relying on `finally` for resources, and never use `finalize()` for cleanup.

### 10. How do you create an immutable class?

An immutable object cannot change after construction.

Rules:

1. Make the class `final`.
2. Make fields `private final`.
3. Do not expose setters.
4. Validate and initialize through constructor.
5. Defensively copy mutable inputs and outputs.

```java
public final class EmployeeProfile {
    private final String name;
    private final List<String> skills;

    public EmployeeProfile(String name, List<String> skills) {
        this.name = name;
        this.skills = List.copyOf(skills);
    }

    public String name() {
        return name;
    }

    public List<String> skills() {
        return skills;
    }
}
```

Immutability is naturally thread-safe and makes objects safer as map keys, cache values, and DTOs.

### 11. `Comparable` vs `Comparator`

`Comparable` defines natural ordering inside the class. `Comparator` defines external custom ordering.

| Point | `Comparable` | `Comparator` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo()` | `compare()` |
| Location | Inside model class | Separate class or lambda |
| Use case | One default order | Multiple ordering choices |

```java
class Employee implements Comparable<Employee> {
    private String name;

    public int compareTo(Employee other) {
        return this.name.compareTo(other.name);
    }
}

Comparator<Employee> byJoiningDate = Comparator.comparing(Employee::joiningDate);
```

### 12. Fail-fast vs fail-safe iterator

Fail-fast iterators detect structural modification and throw `ConcurrentModificationException`. Examples include `ArrayList` and `HashMap` iterators.

Fail-safe is a common interview term, but for `ConcurrentHashMap`, the more accurate term is weakly consistent. These iterators do not throw concurrent modification exceptions, but they are not guaranteed to show a perfect snapshot.

```java
List<String> names = new ArrayList<>(List.of("A", "B"));
for (String name : names) {
    names.remove(name); // ConcurrentModificationException
}
```

Use an explicit iterator's `remove()`, collect items to remove later, or use concurrent collections when concurrent access is required.

### 13. JVM memory areas and garbage collection

JVM memory includes heap, stack, metaspace, PC register, and native method stack.

| Area | Purpose |
|---|---|
| Heap | Objects and arrays |
| Stack | Method frames, local variables, references |
| Metaspace | Class metadata |
| PC register | Current instruction per thread |
| Native method stack | Native call support |

The heap is usually divided into young generation and old generation. New objects are allocated in Eden. Objects that survive several young GCs are promoted to old generation.

Garbage collection removes objects that are no longer reachable from GC roots such as thread stacks, static fields, JNI references, and active class loaders.

Common collectors:

| GC | Use case |
|---|---|
| Serial GC | Small applications |
| Parallel GC | Throughput-focused workloads |
| G1 GC | Balanced latency and throughput; default since Java 9 |
| ZGC | Very low pause time |
| Shenandoah | Low-pause concurrent collection |

## Java 8 And Java 21

### 14. What are important Java 8 features?

Java 8 introduced lambdas, functional interfaces, method references, Stream API, Optional, default/static methods in interfaces, `CompletableFuture` improvements, and the new Date/Time API.

```java
List<String> result = names.stream()
    .filter(name -> name.startsWith("A"))
    .map(String::toUpperCase)
    .toList();
```

These features made Java backend code more expressive, especially for collection processing, asynchronous workflows, and null handling.

### 15. `map()` vs `flatMap()`

`map()` is one-to-one transformation. `flatMap()` is one-to-many transformation followed by flattening.

```java
List<Integer> lengths = List.of("java", "spring").stream()
    .map(String::length)
    .toList();

List<String> items = List.of(List.of("A", "B"), List.of("C")).stream()
    .flatMap(Collection::stream)
    .toList();
```

Use `map()` when every input produces one output. Use `flatMap()` when every input produces a collection, optional, stream, or nested structure.

### 16. What is `Optional`?

`Optional<T>` represents a value that may or may not exist. It makes null handling explicit.

```java
String displayName = Optional.ofNullable(user.getName())
    .filter(name -> !name.isBlank())
    .orElse("Guest");
```

Prefer `map()`, `flatMap()`, `orElseGet()`, and `ifPresent()` over `isPresent()` followed by `get()`. Do not use `Optional` for entity fields or DTO fields unless your project has a strong convention for it.

### 17. `Predicate`, `Function`, `Consumer`, and `Supplier`

| Interface | Input | Output | Common use |
|---|---|---|---|
| `Predicate<T>` | `T` | `boolean` | Validation/filtering |
| `Function<T,R>` | `T` | `R` | Conversion/mapping |
| `Consumer<T>` | `T` | `void` | Logging/saving/printing |
| `Supplier<T>` | none | `T` | Lazy creation |

```java
Predicate<Integer> isEven = number -> number % 2 == 0;
Function<String, Integer> length = String::length;
Consumer<String> logger = System.out::println;
Supplier<UUID> idGenerator = UUID::randomUUID;
```

### 18. What are key Java 21 features?

Important Java 21 features include virtual threads, sequenced collections, record patterns, pattern matching for `switch`, generational ZGC, and several preview features.

Virtual threads are the biggest backend feature because they allow high concurrency for blocking I/O without forcing reactive programming.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> paymentClient.call());
    executor.submit(() -> inventoryClient.call());
}
```

Virtual threads are not a fix for CPU-heavy work. They are most useful for I/O-bound workloads such as database calls, HTTP calls, and file operations.

## Multithreading

### 19. Thread lifecycle

Java thread states are `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`.

| State | Meaning |
|---|---|
| `NEW` | Thread object created, not started |
| `RUNNABLE` | Ready/running on CPU |
| `BLOCKED` | Waiting for monitor lock |
| `WAITING` | Waiting indefinitely |
| `TIMED_WAITING` | Waiting with timeout |
| `TERMINATED` | Finished execution |

In thread dumps, `BLOCKED` usually indicates lock contention, while many `WAITING` threads can be normal for worker pools.

### 20. `Runnable` vs `Callable`

| Point | `Runnable` | `Callable<V>` |
|---|---|---|
| Return value | No | Yes |
| Checked exception | Cannot throw directly | Can throw |
| Execution | `Thread`, `ExecutorService` | `ExecutorService` |
| Result holder | None | `Future<V>` |

```java
Callable<Integer> task = () -> 42;
Future<Integer> result = executorService.submit(task);
Integer value = result.get();
```

### 21. `synchronized` vs `Lock` vs `ReentrantLock`

`synchronized` is simple JVM-level locking. `Lock` is an interface with advanced capabilities. `ReentrantLock` supports timeout, fairness, interruptible locking, and manual unlock.

```java
if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try {
        updateSharedState();
    } finally {
        lock.unlock();
    }
}
```

Use `synchronized` for simple critical sections. Use `ReentrantLock` when you need timeout, fairness, or finer control.

### 22. What is deadlock and how do you prevent it?

Deadlock happens when threads wait forever for each other's locks.

```text
Thread A holds Lock 1 and waits for Lock 2
Thread B holds Lock 2 and waits for Lock 1
```

Prevention strategies:

1. Always acquire locks in the same order.
2. Avoid nested locks.
3. Keep locked sections short.
4. Use `tryLock()` with timeout.
5. Prefer concurrent utilities such as `ConcurrentHashMap`, `BlockingQueue`, and database transactions.

### 23. `AtomicInteger` and CAS

`AtomicInteger` provides lock-free thread-safe numeric operations using Compare-And-Swap.

```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
counter.compareAndSet(10, 11);
```

Use it for simple shared counters. Do not use it when multiple fields must be updated atomically together; use a lock or a single immutable state object.

### 24. What is `volatile`?

`volatile` guarantees visibility of the latest value across threads. It does not make compound operations atomic.

```java
private volatile boolean running = true;

void stop() {
    running = false;
}
```

Use `volatile` for flags and simple state visibility. Do not use it for `count++` because increment is read-modify-write and still needs atomicity.

### 25. What is `ExecutorService`?

`ExecutorService` manages a pool of worker threads and separates task submission from thread creation.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
Future<String> future = executor.submit(() -> externalApi.call());
```

In production, avoid unbounded queues and tune pool size based on workload. CPU-bound tasks should generally use around CPU-core count. I/O-bound tasks can use more threads, but metrics should guide the number.

### 26. What is `CompletableFuture`?

`CompletableFuture` supports asynchronous pipelines and combining independent tasks.

```java
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> userClient.getUser(id));
CompletableFuture<List<Order>> orderFuture = CompletableFuture.supplyAsync(() -> orderClient.getOrders(id));

UserSummary summary = userFuture.thenCombine(orderFuture, UserSummary::new).join();
```

Use it when independent calls can run in parallel. Always handle timeouts and exceptions, otherwise failures can be hidden until `join()` or `get()`.

## Spring Boot

### 27. What is `@SpringBootApplication`?

`@SpringBootApplication` combines three annotations:

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

It marks the main configuration class, enables conditional auto-configuration, and scans components from the package of the main class downward.

### 28. `@Component` vs `@Service` vs `@Repository`

All three register Spring beans during component scanning.

| Annotation | Intent | Extra behavior |
|---|---|---|
| `@Component` | Generic bean | None |
| `@Service` | Business service | Semantic clarity |
| `@Repository` | Data access | Persistence exception translation |

Use the annotation that communicates the layer correctly. `@Repository` is not only cosmetic because Spring can translate persistence exceptions into `DataAccessException`.

### 29. `@Controller` vs `@RestController`

`@Controller` is for MVC views. `@RestController` is for REST APIs and combines `@Controller` with `@ResponseBody`.

| Point | `@Controller` | `@RestController` |
|---|---|---|
| Return value | View name/model | Response body |
| Common output | HTML template | JSON/XML |
| Use case | Thymeleaf/JSP MVC | REST backend |

### 30. What is `DispatcherServlet`?

`DispatcherServlet` is Spring MVC's front controller. Every matching HTTP request enters it first.

```text
Client -> DispatcherServlet -> HandlerMapping -> Controller -> MessageConverter/ViewResolver -> Response
```

In Spring Boot, it is auto-configured and usually mapped to `/`.

### 31. How does auto-configuration work?

`@EnableAutoConfiguration` loads auto-configuration classes from Spring Boot metadata. Modern Spring Boot uses:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Auto-config classes use conditions such as `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `@ConditionalOnProperty`. For example, adding `spring-boot-starter-data-jpa` can auto-configure Hibernate, `EntityManagerFactory`, transaction manager, and HikariCP if required classes and properties exist.

### 32. What is bean lifecycle?

Spring bean lifecycle is:

```text
Instantiate -> Populate dependencies -> Aware callbacks -> BeanPostProcessor before init -> @PostConstruct/init -> BeanPostProcessor after init -> Ready -> @PreDestroy/destroy
```

Use `@PostConstruct` for initialization that requires dependencies. Use `@PreDestroy` for cleanup. Avoid heavy network calls in bean initialization unless startup dependency is intentional.

### 33. Bean scopes

| Scope | Meaning |
|---|---|
| `singleton` | One bean per Spring context; default |
| `prototype` | New instance each request from container |
| `request` | One bean per HTTP request |
| `session` | One bean per HTTP session |
| `application` | One bean per servlet context |

Most services, repositories, and controllers are singleton. Singleton beans must not keep request-specific mutable state in fields.

### 34. Dependency injection types

Constructor injection is recommended because dependencies are mandatory, testable, and can be `final`.

```java
@Service
class OrderService {
    private final PaymentClient paymentClient;

    OrderService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }
}
```

Setter injection is useful for optional dependencies. Field injection is discouraged because it hides dependencies and makes unit testing harder.

### 35. `@Value` vs `@ConfigurationProperties`

| Point | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Best use | Single property | Grouped config |
| Type safety | Limited | Strong |
| Validation | Manual | Supports validation |
| Relaxed binding | Limited | Yes |

Use `@ConfigurationProperties` for application-level configuration like timeouts, URLs, retry settings, and security options.

### 36. How does `@Transactional` work?

Spring uses AOP proxies. When a transactional method is called through the proxy, Spring opens or joins a transaction, executes the method, commits on success, and rolls back on unchecked exceptions by default.

```java
@Transactional(rollbackFor = Exception.class)
public void transfer(Long from, Long to, BigDecimal amount) {
    debit(from, amount);
    credit(to, amount);
}
```

Important caveat: self-invocation does not trigger the proxy. If one method in the same class calls another `@Transactional` method directly, transaction advice may not run.

### 37. What is Actuator?

Spring Boot Actuator exposes operational endpoints such as:

```text
/actuator/health
/actuator/metrics
/actuator/info
/actuator/loggers
/actuator/prometheus
```

Expose only necessary endpoints and secure sensitive ones. In production, Actuator is often connected with Prometheus, Grafana, New Relic, Datadog, or Elastic APM.

## Spring Security

### 38. Authentication vs authorization

Authentication verifies who the user is. Authorization verifies what the authenticated user can access.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated()
);
```

### 39. Spring Security filter chain

Spring Security works through servlet filters before the request reaches the controller.

Common responsibilities:

1. Load or create `SecurityContext`.
2. Extract credentials or token.
3. Authenticate user.
4. Handle authentication/authorization exceptions.
5. Enforce access rules.

For JWT, a custom `OncePerRequestFilter` usually extracts the bearer token, validates it, and sets authentication in `SecurityContextHolder`.

### 40. JWT vs session

| Point | JWT | Session |
|---|---|---|
| Storage | Client holds token | Server holds session |
| Scaling | Stateless | Needs sticky session/shared store |
| Revocation | Harder; needs blacklist/versioning | Easy; delete session |
| Common use | APIs, microservices, mobile clients | Server-rendered web apps |

JWT should be short-lived, signed securely, sent over HTTPS, and paired with refresh-token strategy if long login is needed.

### 41. OAuth2 vs JWT

OAuth2 is an authorization framework. JWT is a token format. OAuth2 flows can use JWT access tokens, but OAuth2 itself is not the same as JWT.

Use OAuth2/OpenID Connect when delegating login to an identity provider such as Keycloak, Auth0, Okta, Google, or Azure AD. Use JWT as the stateless token format when appropriate.

## Hibernate And Database

### 42. JPA vs Hibernate

JPA is a specification. Hibernate is an implementation of that specification.

| Point | JPA | Hibernate |
|---|---|---|
| Type | Standard/specification | Framework/provider |
| Package | `jakarta.persistence` | `org.hibernate` |
| Role | Defines APIs and annotations | Implements ORM behavior |

Using JPA interfaces keeps code portable, while Hibernate-specific features can be used when the project needs them.

### 43. How does Hibernate work internally?

Hibernate wraps JDBC and maps entities to database rows.

Core flow:

1. Entity enters persistence context.
2. First-level cache tracks managed entity.
3. Dirty checking detects changes.
4. Flush generates SQL.
5. JDBC executes SQL using a connection from the pool.
6. Transaction commits or rolls back.

```text
Entity -> Persistence Context -> Dirty Checking -> SQL -> JDBC -> Database
```

### 44. Lazy vs eager loading

| Point | Lazy | Eager |
|---|---|---|
| Load timing | When accessed | Immediately |
| Default for | `@OneToMany`, `@ManyToMany` | `@ManyToOne`, `@OneToOne` |
| Risk | `LazyInitializationException` | Unnecessary joins and data loading |

Prefer lazy loading by default and fetch required associations explicitly using `JOIN FETCH`, projections, or entity graphs.

### 45. What is N+1 problem?

N+1 happens when one query loads parent records and then one additional query runs for each parent's lazy association.

```java
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order order : orders) {
    order.getItems().size(); // N more queries
}
```

Fixes:

1. `JOIN FETCH`.
2. `@EntityGraph`.
3. Batch fetching with `@BatchSize`.
4. DTO projections.

### 46. `save()` vs `saveAndFlush()`

`save()` persists the entity in the persistence context. SQL may be delayed until flush or transaction commit. `saveAndFlush()` immediately flushes pending SQL to the database but does not commit the transaction.

Use `save()` normally. Use `saveAndFlush()` only when the database-generated value or constraint result is needed before transaction end.

### 47. Transaction isolation levels

| Isolation | Prevents |
|---|---|
| `READ_UNCOMMITTED` | Almost nothing; dirty reads possible |
| `READ_COMMITTED` | Dirty reads |
| `REPEATABLE_READ` | Dirty and non-repeatable reads |
| `SERIALIZABLE` | Dirty, non-repeatable, and phantom reads |

Higher isolation improves consistency but can reduce concurrency. Pick based on business correctness, not guesswork.

### 48. Optimistic vs pessimistic locking

Optimistic locking assumes conflicts are rare and uses a version field.

```java
@Version
private Long version;
```

Pessimistic locking locks rows during transaction.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);
```

Use optimistic locking for normal web concurrency. Use pessimistic locking when conflicts are frequent and correctness requires blocking.

### 49. Indexing and pagination

Indexes speed up filtering, joining, and sorting, but slow down writes and consume storage. Add indexes on columns used in `WHERE`, `JOIN`, and `ORDER BY`.

Pagination avoids loading huge data sets into memory.

```java
Page<Order> page = orderRepository.findByStatus(status, PageRequest.of(0, 20));
```

For very large tables, keyset pagination can perform better than high-offset pagination.

### 50. Connection pool

A connection pool reuses database connections because creating a connection for every request is expensive. Spring Boot uses HikariCP by default.

Key metrics:

1. Active connections.
2. Idle connections.
3. Pending threads.
4. Connection timeout.
5. Slow query time.

Pool size should be tuned with database capacity and service concurrency in mind. Bigger pools can overload the database.

## Microservices

### 51. Microservices vs monolith

A monolith is one deployable application. Microservices split business capabilities into independently deployable services.

| Point | Monolith | Microservices |
|---|---|---|
| Deployment | One unit | Independent services |
| Scaling | Whole app | Per service |
| Data | Often shared DB | Usually database per service |
| Complexity | Lower operational complexity | Higher distributed complexity |
| Best for | Small/simple systems | Large teams and independently scalable domains |

Microservices are not automatically better. They introduce network failures, data consistency challenges, observability requirements, and operational overhead.

### 52. How do microservices communicate?

Synchronous communication is request-response: REST, gRPC, Feign, WebClient. It is useful when the caller needs an immediate result.

Asynchronous communication uses events/messages: Kafka, RabbitMQ, SQS. It is useful for decoupling, background work, notifications, audit trails, and eventual consistency.

### 53. API Gateway

An API Gateway is the single entry point for clients.

Responsibilities:

1. Routing.
2. Authentication.
3. Rate limiting.
4. Request/response transformation.
5. TLS termination.
6. Central logging.

Examples include Spring Cloud Gateway, Kong, Nginx, AWS API Gateway, and Apigee.

### 54. Service discovery

Service discovery lets services find dynamic instances without hardcoded IPs.

Options:

1. Client-side discovery using Eureka/Consul.
2. Server-side discovery using Kubernetes Services, AWS ALB/NLB, or service mesh.

In Kubernetes, DNS and Services often replace older Eureka-style discovery.

### 55. Circuit breaker, retry, timeout, and bulkhead

These patterns protect services from cascading failures.

| Pattern | Purpose |
|---|---|
| Timeout | Stop waiting forever |
| Retry | Retry transient failures |
| Circuit breaker | Stop calling unhealthy dependency |
| Bulkhead | Isolate resources so one dependency does not exhaust all threads |

Retries should use backoff and should not retry non-idempotent operations blindly.

### 56. Saga pattern

Saga manages distributed transactions using local transactions and compensating actions.

```text
Create order -> Charge payment -> Reserve inventory -> Confirm order
If inventory fails -> Refund payment -> Cancel order
```

Choreography uses events with no central coordinator. Orchestration uses a central workflow component. Choreography is simpler initially but can become hard to trace. Orchestration is more explicit but adds a coordinator dependency.

### 57. Distributed tracing

Distributed tracing follows a request across services using trace IDs and spans.

```text
API Gateway [traceId=abc] -> Order Service -> Payment Service -> Database
```

Tools include OpenTelemetry, Micrometer Tracing, Jaeger, Zipkin, Grafana Tempo, Datadog, and New Relic. Good tracing shows latency per hop and makes slow dependency calls visible.

## Kafka And Messaging

### 58. Kafka vs RabbitMQ

| Point | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed commit log | Message broker/queue |
| Retention | Messages retained by policy | Usually removed after acknowledgement |
| Throughput | Very high | High |
| Ordering | Per partition | Per queue |
| Best use | Event streaming, replay, audit logs | Task queues, routing workflows |

Use Kafka when consumers may replay events or multiple systems need the same event stream. Use RabbitMQ for routing-heavy command/task messaging.

### 59. Kafka partitions and consumer groups

A topic is split into partitions. Each partition is an ordered log. In a consumer group, each partition is assigned to only one consumer at a time.

Key rules:

1. Ordering is guaranteed only within one partition.
2. Same key should go to the same partition when ordering by key is required.
3. More partitions allow more parallelism, but also more overhead.
4. Offsets track consumer progress.

### 60. What happens when a Kafka consumer crashes?

The group coordinator detects missed heartbeats and triggers a rebalance. Partitions from the failed consumer are assigned to remaining consumers. New consumers resume from the last committed offset.

To reduce duplicate processing, commit offsets only after successful processing. To handle unavoidable duplicate delivery, make consumers idempotent.

### 61. Kafka replication and ordering

Replication copies partitions across brokers for fault tolerance. One broker is leader for a partition, and followers replicate from it.

Ordering is guaranteed within a partition, not across the whole topic. If order matters for an entity such as `orderId`, use that entity ID as the Kafka message key.

## REST API

### 62. What makes an API RESTful?

REST APIs expose resources through URLs and use HTTP methods correctly.

| Method | Meaning |
|---|---|
| `GET` | Read resource |
| `POST` | Create or trigger non-idempotent operation |
| `PUT` | Replace resource; idempotent |
| `PATCH` | Partial update |
| `DELETE` | Delete resource; should be idempotent |

Use correct status codes, validation errors, pagination, sorting, and consistent response formats.

### 63. API versioning

Common approaches:

1. URI versioning: `/api/v1/orders`.
2. Header versioning: `Accept: application/vnd.company.v1+json`.
3. Query versioning: `/orders?version=1`.

URI versioning is simple and common. Header versioning is cleaner but harder for some clients to test manually.

### 64. Request validation

Use Bean Validation for incoming JSON.

```java
record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}

@PostMapping("/users")
ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
    return ResponseEntity.ok(userService.create(request));
}
```

Handle `MethodArgumentNotValidException` globally and return a field-level error response.

## Design Patterns And SOLID

### 65. SOLID principles

| Principle | Meaning |
|---|---|
| Single Responsibility | A class should have one reason to change |
| Open/Closed | Open for extension, closed for modification |
| Liskov Substitution | Subtypes must honor parent contracts |
| Interface Segregation | Prefer small focused interfaces |
| Dependency Inversion | Depend on abstractions, not concrete classes |

SOLID is useful when it reduces change risk. Do not over-engineer small features with unnecessary abstractions.

### 66. Singleton pattern

Singleton ensures one instance exists.

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

In Spring, singleton is the default bean scope. That does not mean the class is thread-safe; singleton beans must avoid unsafe mutable shared fields.

### 67. Strategy, Factory, and Repository patterns

Strategy selects behavior at runtime.

```java
interface PaymentStrategy {
    void pay(BigDecimal amount);
}
```

Factory centralizes object creation when creation depends on type or input.

Repository separates data access from business logic.

```java
@Repository
interface OrderRepository extends JpaRepository<Order, Long> {
}
```

In Spring Boot, Strategy is commonly implemented by injecting `Map<String, InterfaceType>` where each implementation is a bean.

## Testing, Tools, And AI

### 68. `@Mock` vs `@MockBean`

`@Mock` is Mockito-only and used for unit tests without Spring context. `@MockBean` registers a mock inside the Spring application context and is useful for Spring Boot integration/slice tests.

Use `@Mock` for fast unit tests. Use `@MockBean` when a Spring bean must be replaced in the context.

### 69. Git merge conflicts

A merge conflict occurs when Git cannot automatically combine changes.

Typical process:

```bash
git status
# edit conflicted files
git add <resolved-files>
git commit
```

Always run tests after resolving conflicts because a syntactic merge can still be behaviorally wrong.

### 70. How should developers use AI tools?

AI tools can help explain code, generate boilerplate, draft tests, and suggest refactors. They must not be treated as final authority.

Validate AI-generated code by reviewing logic, running tests, checking edge cases, checking security risks, and aligning with project standards.

