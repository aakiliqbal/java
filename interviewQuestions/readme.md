# Java + Spring Boot + Microservices — Interview Q&A

---

## 📦 1. Core Java

---

### Q1. How does HashMap work internally?

HashMap uses an array of **buckets** (Node[]) with a default initial capacity of 16. When you call `put(key, value)`:

1. `hashCode()` is called on the key, then internally processed via a **hash spread** to reduce collisions.
2. The final index is computed as `(n - 1) & hash`, where `n` is the capacity.
3. If the bucket is empty, the entry is placed directly.
4. If there's a **collision**, entries are stored in a linked list at that bucket.
5. From **Java 8**, when a bucket's linked list exceeds **8 nodes** (and total capacity ≥ 64), it converts to a **Red-Black Tree** — making worst-case lookup O(log n) instead of O(n).

**Key points:**
- Default load factor is `0.75` — at 75% capacity, HashMap **rehashes** (doubles capacity).
- `equals()` is used to check key equality within the same bucket.
- `null` key is always placed at index 0.

---

### Q2. What is the difference between HashMap and ConcurrentHashMap?

| Feature | HashMap | ConcurrentHashMap |
|---|---|---|
| Thread Safety | Not thread-safe | Thread-safe |
| Null keys/values | Allows null key/value | Does NOT allow null |
| Locking | No locking | Segment-level / bucket-level locking (Java 8+) |
| Performance | Faster in single-thread | Optimized for concurrent reads |
| Iteration | Fail-fast iterator | Fail-safe (weakly consistent) |

In Java 8+, `ConcurrentHashMap` uses **CAS (Compare-And-Swap)** operations and synchronized blocks only at the bucket level, not the entire map — making concurrent reads lock-free.

---

### Q3. What is the difference between fail-fast and fail-safe iterators?

**Fail-fast** iterators (e.g., `ArrayList`, `HashMap`) throw `ConcurrentModificationException` if the collection is modified during iteration. They work on the original collection and use a **modCount** counter to detect changes.

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    list.remove(s); // throws ConcurrentModificationException
}
```

**Fail-safe** iterators (e.g., `CopyOnWriteArrayList`, `ConcurrentHashMap`) work on a **snapshot/copy** of the collection — they don't throw exceptions but may not reflect the latest state.

---

### Q4. What is an Immutable class in Java?

An immutable class cannot be changed after creation. To create one:

1. Declare the class as `final`
2. Make all fields `private final`
3. No setters
4. Deep-copy mutable objects in constructor and getters

```java
public final class Employee {
    private final String name;
    private final int age;

    public Employee(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
}
```

`String`, `Integer`, `LocalDate` are classic examples. Immutability is safe for multithreading — no synchronization needed.

---

### Q5. What is the difference between Comparable and Comparator?

| | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Location | Inside the class | External / Lambda |
| Use case | Natural ordering | Custom / multiple orderings |

```java
// Comparable - natural order
class Student implements Comparable<Student> {
    public int compareTo(Student s) { return this.name.compareTo(s.name); }
}

// Comparator - custom order
Comparator<Student> byAge = (s1, s2) -> s1.age - s2.age;
students.sort(byAge);
```

---

### Q6. What is the difference between String, StringBuilder, and StringBuffer?

| Feature | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (by design) | No | Yes (synchronized) |
| Performance | Slow (creates new objects) | Fastest | Slower than StringBuilder |

Use `StringBuilder` in single-threaded scenarios where you need concatenation in loops. Use `StringBuffer` only when thread safety is needed (rare in modern code — prefer other approaches).

---

### Q7. What is the difference between `map()` and `flatMap()` in Java Streams?

- `map()` — transforms each element to exactly **one output element**
- `flatMap()` — transforms each element to a **stream**, then flattens all streams into one

```java
// map: List<String> → List<Integer>
List<Integer> lengths = List.of("hello", "world")
    .stream()
    .map(String::length)
    .toList(); // [5, 5]

// flatMap: List<List<String>> → List<String>
List<List<String>> nested = List.of(List.of("a","b"), List.of("c","d"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .toList(); // [a, b, c, d]
```

---

### Q8. What is the Optional class and why was it introduced?

`Optional<T>` is a container that may or may not hold a non-null value. It was introduced in Java 8 to **avoid NullPointerExceptions** and make null-handling explicit.

```java
Optional<String> name = Optional.ofNullable(user.getName());

// isPresent() - checks if value exists
if (name.isPresent()) { System.out.println(name.get()); }

// ifPresent() - executes action only if value exists (preferred)
name.ifPresent(n -> System.out.println(n));

// orElse / orElseGet
String result = name.orElse("Default");
String result2 = name.orElseGet(() -> computeDefault());
```

`isPresent()` is an older check-then-get pattern. `ifPresent()` is cleaner and more functional.

---

### Q9. What is AtomicInteger and when do you use it?

`AtomicInteger` is a thread-safe integer wrapper in `java.util.concurrent.atomic`. It uses **CAS (Compare-And-Swap)** CPU instructions — no locks needed.

```java
AtomicInteger counter = new AtomicInteger(0);

// Thread-safe increment
counter.incrementAndGet(); // returns new value
counter.getAndIncrement(); // returns old value

// CAS operation
counter.compareAndSet(5, 10); // sets to 10 only if current value is 5
```

Use it when multiple threads update a shared counter — faster than `synchronized` for simple numeric operations.

---

### Q10. What is a Daemon Thread?

A daemon thread is a **background/support thread** that JVM kills automatically when all non-daemon (user) threads finish.

```java
Thread t = new Thread(() -> {
    while (true) System.out.println("Running...");
});
t.setDaemon(true); // must be set BEFORE start()
t.start();
```

Examples: Garbage Collector, JVM monitoring threads.  
Regular threads (user threads) keep the JVM alive even after `main()` exits. Daemon threads do not.

---

### Q11. What is the Thread Lifecycle in Java?

```
NEW → RUNNABLE → RUNNING → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```

| State | Description |
|---|---|
| NEW | Thread created, `start()` not yet called |
| RUNNABLE | Ready to run, waiting for CPU |
| RUNNING | Executing |
| BLOCKED | Waiting to acquire a lock |
| WAITING | Waiting indefinitely (`wait()`, `join()`) |
| TIMED_WAITING | Waiting for a timeout (`sleep()`, `wait(ms)`) |
| TERMINATED | Finished execution |

---

### Q12. What is the difference between Runnable and Callable?

| | Runnable | Callable<V> |
|---|---|---|
| Return type | void | Returns V |
| Exception | Cannot throw checked exceptions | Can throw checked exceptions |
| Used with | Thread, ExecutorService | ExecutorService (returns Future) |

```java
// Callable with return value
Callable<Integer> task = () -> 42;
Future<Integer> future = executor.submit(task);
int result = future.get(); // blocks until done
```

---

### Q13. What is Deadlock and how do you avoid it?

**Deadlock** occurs when two or more threads are waiting for each other's locks indefinitely.

```
Thread A holds Lock 1 → waits for Lock 2
Thread B holds Lock 2 → waits for Lock 1
```

**Prevention strategies:**
1. **Lock ordering** — always acquire locks in the same fixed order
2. **tryLock with timeout** — use `ReentrantLock.tryLock(timeout)` to back off
3. **Avoid nested locks** — minimize synchronized blocks
4. **Use concurrent utilities** — `ConcurrentHashMap`, `BlockingQueue` instead of manual locking

---

### Q14. What is Garbage Collection in Java? What are the types of GCs?

GC automatically reclaims heap memory from objects that are no longer reachable. JVM divides heap into:
- **Young Generation** (Eden + Survivor spaces): Minor GC runs here frequently
- **Old Generation (Tenured)**: Objects that survive many minor GCs move here
- **Metaspace** (Java 8+): Class metadata

**Types of GC:**

| GC | Description |
|---|---|
| Serial GC | Single-threaded, simple; for small apps |
| Parallel GC | Multi-threaded, throughput-focused (default Java 8) |
| G1 GC | Region-based, balanced latency/throughput (default Java 9+) |
| ZGC | Low-latency, concurrent, sub-millisecond pauses |
| Shenandoah | Similar to ZGC, concurrent compaction |

---

### Q15. What is Checked vs Unchecked Exception?

| | Checked | Unchecked |
|---|---|---|
| Extends | `Exception` | `RuntimeException` |
| Must handle | Yes (compile error if not) | No |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `ArrayIndexOutOfBoundsException` |

```java
// Checked — must try/catch or declare throws
public void readFile() throws IOException { ... }

// Unchecked — optional handling
public void divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("Division by zero");
}
```

---

### Q16. Coding: Count character frequency sorted by descending frequency

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class CharFrequency {
    public static void main(String[] args) {
        String s = "welcome to programming";

        s.chars()
            .filter(c -> c != ' ')
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
            .entrySet()
            .stream()
            .sorted(Map.Entry.<Character, Long>comparingByValue().reversed())
            .forEach(e -> System.out.println(e.getKey() + " -> " + e.getValue()));
    }
}
```

**Output:** Characters like `m`, `g`, `o` that appear most frequently will appear first.

---

---

## 🌱 2. Spring Boot

---

### Q17. What is `@SpringBootApplication`?

It is a convenience annotation that combines three annotations:

```java
@SpringBootConfiguration   // marks this as a configuration class
@EnableAutoConfiguration   // enables Spring Boot's auto-config
@ComponentScan             // scans the package for components
```

When you run the main class, Spring Boot starts an embedded server, loads auto-configurations from `spring.factories`, and initializes the application context.

---

### Q18. What is the difference between `@Component`, `@Service`, and `@Repository`?

All three are **stereotypes** that mark a class as a Spring bean — functionally they trigger the same component scan.

| Annotation | Semantic Role | Extra Behavior |
|---|---|---|
| `@Component` | Generic bean | None |
| `@Service` | Business logic layer | None (semantic clarity) |
| `@Repository` | Data access layer | Translates persistence exceptions to Spring `DataAccessException` |

Use the right annotation to communicate intent — `@Repository` is not just cosmetic because Spring wraps its exceptions.

---

### Q19. What is DispatcherServlet?

`DispatcherServlet` is the **front controller** in Spring MVC. All HTTP requests go through it first.

```
Client Request
    ↓
DispatcherServlet
    ↓
HandlerMapping → finds the right @Controller
    ↓
HandlerAdapter → invokes the method
    ↓
ViewResolver (for MVC) / MessageConverter (for REST)
    ↓
Response
```

In Spring Boot, it is auto-configured and mapped to `/` by default.

---

### Q20. What is the difference between `@Controller` and `@RestController`?

| | @Controller | @RestController |
|---|---|---|
| Returns | View name (template) | Object serialized to JSON/XML |
| Equivalent | @Controller | @Controller + @ResponseBody |
| Use case | MVC apps with Thymeleaf/JSP | REST APIs |

`@RestController` automatically applies `@ResponseBody` to every method, so the return value goes directly into the HTTP response body.

---

### Q21. How does Spring Boot Auto-Configuration work internally?

1. `@EnableAutoConfiguration` triggers the process.
2. Spring Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (or `spring.factories` in older versions).
3. Each auto-configuration class is annotated with `@ConditionalOn*` annotations:
   - `@ConditionalOnClass` — only loads if a specific class is on the classpath
   - `@ConditionalOnMissingBean` — only creates a bean if you haven't defined one yourself
4. This is why adding `spring-boot-starter-data-jpa` automatically configures Hibernate, DataSource, etc. — without any XML.

---

### Q22. What is the difference between `@Value` and `@ConfigurationProperties`?

| | @Value | @ConfigurationProperties |
|---|---|---|
| Usage | Single property | Group of related properties |
| Type safety | No | Yes |
| Relaxed binding | No | Yes (kebab-case, camelCase) |
| Validation | No | Yes (with `@Validated`) |

```java
// @Value
@Value("${app.timeout}")
private int timeout;

// @ConfigurationProperties
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private int timeout;
    private String name;
}
```

---

### Q23. What is Global Exception Handling in Spring Boot?

Use `@ControllerAdvice` + `@ExceptionHandler` to handle exceptions centrally:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("SERVER_ERROR", "Something went wrong"));
    }
}
```

This avoids try/catch duplication across controllers.

---

### Q24. What are Bean Scopes in Spring?

| Scope | Description |
|---|---|
| `singleton` | One instance per Spring context (default) |
| `prototype` | New instance every time it's requested |
| `request` | One per HTTP request (web apps) |
| `session` | One per HTTP session (web apps) |
| `application` | One per ServletContext lifecycle |

```java
@Bean
@Scope("prototype")
public MyService myService() { return new MyService(); }
```

---

### Q25. What are the types of Dependency Injection in Spring?

1. **Constructor Injection** (recommended — makes dependencies mandatory and final)
```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) { // Spring injects automatically
        this.paymentService = paymentService;
    }
}
```

2. **Setter Injection** (optional dependencies)
```java
@Autowired
public void setPaymentService(PaymentService paymentService) { ... }
```

3. **Field Injection** (not recommended — hard to test, hides dependencies)
```java
@Autowired
private PaymentService paymentService;
```

---

### Q26. How does `@Transactional` work internally?

Spring uses **AOP (Aspect-Oriented Programming)** and generates a proxy around your bean. When a transactional method is called:

1. Spring's proxy intercepts the call.
2. Opens a database transaction (or joins an existing one based on propagation).
3. Executes the method.
4. **Commits** on success, **rolls back** on `RuntimeException` (or as configured).

```java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRED)
public void transferMoney(Long from, Long to, double amount) {
    debit(from, amount);
    credit(to, amount);
}
```

**Important:** `@Transactional` does NOT work if you call the method from within the same class — Spring's proxy is bypassed.

---

### Q27. Is it possible to have more than one `main` method in a Spring Boot project?

**Yes.** Java allows any number of `public static void main(String[] args)` methods — each class can have one.

When you run the project via `mvn spring-boot:run` or the JAR, Spring Boot uses the class specified in the **`Main-Class`** attribute of `MANIFEST.MF`. Only that class's `main()` is executed as the entry point. Other `main()` methods in the project are simply ignored.

---

### Q28. What is Spring Boot Actuator?

Actuator exposes production-ready endpoints for monitoring and management:

```
/actuator/health     → app health status
/actuator/metrics    → JVM metrics, request counts, etc.
/actuator/info       → app version, custom info
/actuator/env        → environment properties
/actuator/loggers    → view/change log levels at runtime
```

Add it with: `spring-boot-starter-actuator`  
Secure sensitive endpoints using Spring Security — expose only what's necessary.

---

### Q29. How do you implement JWT Authentication in Spring Boot?

**Flow:**
```
User login → validate credentials → generate JWT → return to client
Client sends JWT in Authorization: Bearer <token> header
Filter validates token → sets SecurityContext → request proceeds
```

**Key components:**

```java
// 1. JwtUtil — generate and validate tokens
String token = Jwts.builder()
    .setSubject(username)
    .setExpiration(new Date(System.currentTimeMillis() + 86400000))
    .signWith(SignatureAlgorithm.HS256, secretKey)
    .compact();

// 2. JwtAuthFilter — intercepts every request
protected void doFilterInternal(HttpServletRequest req, ...) {
    String token = req.getHeader("Authorization").substring(7);
    String username = jwtUtil.extractUsername(token);
    // set authentication in SecurityContextHolder
}

// 3. SecurityConfig — register filter
http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

---

### Q30. What is the difference between JWT and Session?

| | JWT | Session |
|---|---|---|
| Storage | Client-side (token) | Server-side (session store) |
| Scalability | Stateless — scales easily | Requires sticky sessions or shared store |
| Expiry | Encoded in token | Managed by server |
| Revocation | Hard (needs blacklist) | Easy (delete session) |
| Use case | Microservices, mobile APIs | Traditional web apps |

---

### Q31. If adding OTP/CAPTCHA login without modifying the existing API — how?

Use the **Decorator** or **Chain of Responsibility** pattern with Spring's **Filter Chain** or **Strategy** pattern.

**Approach: Strategy Pattern + Factory**

```java
public interface LoginStrategy {
    boolean authenticate(LoginRequest request);
}

@Component("OTP")
public class OtpLoginStrategy implements LoginStrategy { ... }

@Component("PASSWORD")
public class PasswordLoginStrategy implements LoginStrategy { ... }

@Service
public class LoginService {
    @Autowired
    private Map<String, LoginStrategy> strategies; // Spring injects all beans

    public boolean login(String type, LoginRequest request) {
        return strategies.get(type).authenticate(request);
    }
}
```

The existing API endpoint remains unchanged. New strategies are added by implementing the interface and registering as a Spring bean.

---

---

## 🗄️ 3. Database & JPA / Hibernate

---

### Q32. How does Hibernate work internally?

When you call `entityManager.persist(entity)` or `save()`:

1. **Session** — Hibernate opens a session (wrapper around JDBC Connection)
2. **First-level Cache** — the entity is stored in the session cache
3. **Dirty Checking** — Hibernate tracks changes to managed entities
4. **Flush** — at transaction commit (or explicit flush), Hibernate generates SQL and sends it to DB
5. **JDBC** — the SQL is executed via the underlying connection pool (HikariCP by default)

```
Entity → Hibernate Session → SQL Generation → JDBC → Database
```

---

### Q33. What is the N+1 Problem in Hibernate?

When you fetch a list of entities and Hibernate fires **1 query for the list** and then **N additional queries for each entity's lazy association**.

```java
List<Order> orders = orderRepo.findAll(); // 1 query
for (Order o : orders) {
    o.getItems().size(); // N queries — one per order!
}
```

**Solutions:**
- Use `JOIN FETCH` in JPQL: `SELECT o FROM Order o JOIN FETCH o.items`
- Use `@EntityGraph` on repository methods
- Use `@BatchSize(size = 10)` on the collection

---

### Q34. What is the difference between Lazy and Eager Loading?

| | Lazy | Eager |
|---|---|---|
| When loaded | On access | Immediately with parent |
| Default for | `@OneToMany`, `@ManyToMany` | `@ManyToOne`, `@OneToOne` |
| Risk | LazyInitializationException outside session | Performance hit (always loads) |

```java
@OneToMany(fetch = FetchType.LAZY)   // default — loads items only when accessed
private List<OrderItem> items;

@ManyToOne(fetch = FetchType.EAGER)  // default — always loads with Order
private Customer customer;
```

---

### Q35. What is `@OneToMany` and where do you place it?

Place `@OneToMany` on the **parent/owner side** — the entity that "has many" of another.

```java
@Entity
public class Order {
    @Id
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderItem> items; // "order" refers to the field in OrderItem
}

@Entity
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id") // FK column in order_item table
    private Order order;
}
```

`mappedBy` tells Hibernate which side owns the FK — the `@ManyToOne` side always owns it.

---

### Q36. What is Transaction Management and when to use `@Transactional`?

Use `@Transactional` whenever you perform **multiple database operations that must succeed or fail together**.

```java
@Transactional
public void transfer(Long fromId, Long toId, double amount) {
    Account from = repo.findById(fromId).orElseThrow();
    Account to = repo.findById(toId).orElseThrow();
    from.debit(amount);  // if this fails...
    to.credit(amount);   // this should not commit either
}
```

**Without `@Transactional`:** each `save()` auto-commits. Use `EntityManager` with manual `begin/commit/rollback` if you want to avoid the annotation — but `@Transactional` is the clean Spring way.

---

### Q37. How do you optimize slow database queries?

1. **Indexing** — add indexes on columns used in WHERE, JOIN, ORDER BY
2. **Avoid SELECT \*** — fetch only needed columns with projections
3. **Fix N+1** — use JOIN FETCH or EntityGraph
4. **Pagination** — use `Pageable` — never load all records
5. **Query optimization** — analyze with `EXPLAIN PLAN`
6. **Caching** — Redis for frequently read, rarely changed data
7. **Connection pooling** — tune HikariCP pool size
8. **Database-level** — partitioning, read replicas for heavy read workloads

---

---

## ⚙️ 4. Microservices & Architecture

---

### Q38. What is Microservices Architecture and why over Monolith?

**Monolith:** Single deployable unit. All modules (auth, orders, payments) tightly coupled in one codebase.

**Microservices:** Each business capability is an independent service with its own DB, deployed separately.

**Why Microservices:**

| Benefit | Detail |
|---|---|
| Independent deployment | Deploy order-service without touching payment-service |
| Independent scaling | Scale only the high-load service |
| Technology freedom | Each service can use different language/DB |
| Fault isolation | One service failure doesn't bring down the system |

**Trade-offs:** Distributed system complexity, network latency, eventual consistency challenges.

---

### Q39. What are the different ways microservices communicate?

**Synchronous (Request/Response):**
- `RestTemplate` — blocking HTTP client (legacy)
- `WebClient` — reactive, non-blocking HTTP client (preferred)
- `Feign Client` — declarative REST client with Spring Cloud
- `gRPC` — high-performance binary protocol

**Asynchronous (Event-Driven):**
- `Apache Kafka` — distributed event streaming, high throughput
- `RabbitMQ` — AMQP-based message broker, routing flexibility
- `Redis Pub/Sub` — lightweight messaging

---

### Q40. What is an API Gateway and why is it required?

An API Gateway is the **single entry point** for all client requests. It handles:

- **Routing** — routes requests to the right microservice
- **Authentication** — validates JWT/OAuth tokens centrally
- **Rate Limiting** — prevents abuse
- **Load Balancing** — distributes traffic
- **SSL Termination** — handles HTTPS at the edge
- **Request/Response transformation** — adds/removes headers

Examples: **Spring Cloud Gateway**, Netflix Zuul, AWS API Gateway, Kong.

---

### Q41. What is the Circuit Breaker Pattern?

Circuit Breaker prevents **cascading failures** when a downstream service is slow or unavailable — similar to an electrical circuit breaker.

**States:**

```
CLOSED (normal) → OPEN (failing, reject calls) → HALF-OPEN (test recovery) → CLOSED
```

```java
// Using Resilience4j
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
public String callPayment() {
    return paymentClient.process();
}

public String fallback(Exception e) {
    return "Payment service unavailable. Try later.";
}
```

Configure thresholds: if 50% of calls fail in a window → circuit opens.

---

### Q42. What is Service Discovery?

In a microservices environment, service instances start/stop dynamically with changing IPs. **Service Discovery** lets services find each other without hardcoded URLs.

**Two types:**
1. **Client-side** (Eureka + Ribbon) — client queries Eureka registry, then does load balancing itself
2. **Server-side** (AWS ALB, Kubernetes) — load balancer handles it

```yaml
# Register with Eureka
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

Feign Client automatically resolves service name to an actual instance via Eureka.

---

### Q43. What is the difference between Synchronous and Asynchronous microservice communication?

| | Synchronous | Asynchronous |
|---|---|---|
| Pattern | Request/Response | Event/Message |
| Coupling | Tightly coupled | Loosely coupled |
| Example | REST, gRPC | Kafka, RabbitMQ |
| Failure impact | Caller blocked if service is down | Producer decoupled from consumer |
| Use case | Real-time data needed | Background processing, notifications |

If `order-service` calls `payment-service` synchronously and payment-service is slow — order-service is blocked. With Kafka, order-service publishes an event and moves on.

---

### Q44. What is the Bulkhead Pattern?

Isolates different services/resources into separate **pools** so a failure in one doesn't starve the others — like watertight compartments in a ship.

```java
// Using Resilience4j ThreadPoolBulkhead
@Bulkhead(name = "paymentBulkhead", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<String> callPayment() { ... }
```

If the payment service is slow and exhausts its thread pool, it doesn't affect threads serving the inventory service.

---

### Q45. What are design patterns used in Microservices?

| Pattern | Purpose |
|---|---|
| API Gateway | Single entry point |
| Circuit Breaker | Fault tolerance |
| Saga | Distributed transactions |
| Bulkhead | Resource isolation |
| CQRS | Separate read/write models |
| Event Sourcing | Audit trail via events |
| Sidecar | Cross-cutting concerns (logging, auth) alongside each service |
| Strangler Fig | Gradually migrate monolith to microservices |

---

### Q46. What is the Saga Pattern?

Saga manages **distributed transactions** across multiple services. Instead of a global ACID transaction (which doesn't work across services), Saga uses a sequence of **local transactions** with **compensating transactions** on failure.

**Two types:**
1. **Choreography** — services react to events (no central coordinator)
2. **Orchestration** — a central Saga orchestrator tells each service what to do

```
Order created →
  Payment charged →
    Inventory reserved →
      Order confirmed
        (if inventory fails → release payment → cancel order)
```

---

---

## 📨 5. Kafka & Messaging

---

### Q47. Why use Kafka instead of RabbitMQ?

| Feature | Kafka | RabbitMQ |
|---|---|---|
| Message model | Log-based (pull) | Queue-based (push) |
| Retention | Messages retained (configurable) | Messages deleted after consumption |
| Throughput | Very high (millions/sec) | High (thousands/sec) |
| Ordering | Per-partition guarantee | Per-queue |
| Use case | Event streaming, audit log, replay | Task queues, routing |

Use Kafka when you need **event streaming, replay, audit trail, or very high throughput**.  
Use RabbitMQ for **complex routing, task distribution, or lower volume**.

---

### Q48. How do Kafka partitions work?

A **topic** is split into **partitions** — each partition is an ordered, immutable log.

- **Producer** assigns messages to partitions via a key hash (or round-robin)
- **Consumer** reads from assigned partitions within a **consumer group**
- Each partition is consumed by exactly **one consumer** in a group — enabling parallel processing
- Message ordering is guaranteed **within a partition**, not across partitions

```java
// Messages with the same key go to the same partition (ordering preserved)
kafkaTemplate.send("orders", userId, orderEvent);
```

---

### Q49. What happens if a Kafka consumer crashes?

1. The Kafka **Group Coordinator** detects the consumer failure (via missed heartbeats).
2. A **rebalance** is triggered — remaining consumers in the group take over the failed consumer's partitions.
3. The new consumer continues from the **last committed offset** — no messages are lost.

To ensure no data loss: **commit offsets only after successful processing** (not auto-commit in critical flows).

---

### Q50. What is idempotency in distributed systems?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

In messaging, if a consumer crashes after processing but before committing the offset, it will reprocess the message. The consumer logic must handle this safely:

```java
// Idempotent: check before processing
if (!processedRepo.existsByMessageId(messageId)) {
    processOrder(order);
    processedRepo.save(messageId);
}
```

Kafka producers can also be made idempotent with `enable.idempotence=true`.

---

---

## 🎨 6. SOLID Principles

---

### Q51. What are SOLID principles? Explain with examples.

**S — Single Responsibility Principle**  
A class should have only one reason to change.

```java
// ❌ Wrong — one class handles too much
class Order { 
    void save() { ... }       // DB concern
    void sendEmail() { ... }  // notification concern
}

// ✅ Right — separate classes
class OrderRepository { void save() { ... } }
class OrderNotifier { void sendEmail() { ... } }
```

**O — Open/Closed Principle**  
Open for extension, closed for modification.

```java
// Add new payment types without changing existing code
interface PaymentStrategy { void pay(double amount); }
class CreditCardPayment implements PaymentStrategy { ... }
class UPIPayment implements PaymentStrategy { ... }
```

**L — Liskov Substitution Principle**  
Subclasses must be substitutable for their parent class.

```java
// Derived class must honor the contract of the base class
class Bird { void fly() { ... } }
class Penguin extends Bird { void fly() { throw new UnsupportedOperationException(); } } // ❌ Violates LSP
// Fix: separate Flying and Non-Flying birds via interfaces
```

**I — Interface Segregation Principle**  
Don't force clients to implement methods they don't use. Prefer small, specific interfaces.

```java
// ❌ Fat interface
interface Worker { void work(); void eat(); void sleep(); }

// ✅ Segregated
interface Workable { void work(); }
interface Eatable { void eat(); }
```

**D — Dependency Inversion Principle**  
Depend on abstractions, not concrete implementations.

```java
// ❌ High-level depends on low-level
class OrderService { MySQLOrderRepo repo = new MySQLOrderRepo(); }

// ✅ Depends on abstraction
class OrderService {
    private final OrderRepository repo; // interface
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
```

---

---

## 🔒 7. Spring Security

---

### Q52. What is the Spring Filter Chain?

Spring Security intercepts requests via a chain of **servlet filters** before they reach the controller.

Key filters in order:
1. `SecurityContextPersistenceFilter` — loads SecurityContext from session
2. `UsernamePasswordAuthenticationFilter` — handles form login
3. `JwtAuthenticationFilter` (custom) — validates JWT tokens
4. `ExceptionTranslationFilter` — handles `AccessDeniedException`
5. `FilterSecurityInterceptor` — final authorization check

Each filter can pass the request down the chain or short-circuit it with a response.

---

### Q53. What is the difference between Authentication and Authorization?

- **Authentication** — verifying **who you are** (login: username/password, JWT)
- **Authorization** — verifying **what you can do** (roles, permissions)

```java
// Authentication
http.formLogin().loginPage("/login");

// Authorization
http.authorizeHttpRequests()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated();
```

---

### Q54. What is the Singleton Design Pattern and where is it used in multithreading?

Singleton ensures only **one instance** of a class exists throughout the application.

```java
// Thread-safe Singleton using double-checked locking
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;

    private DatabaseConnection() {}

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}
```

**In Spring:** every `@Service`, `@Repository`, `@Component` bean is a singleton by default — Spring manages the single instance in its context.

**In multithreading:** `volatile` ensures visibility across threads; double-checked locking avoids synchronization overhead after initialization.

---

---

## ☁️ 8. System Design & Performance

---

### Q55. If an API request is slow end-to-end, how do you optimize it?

**Systematic approach:**

1. **Identify the bottleneck** — use APM tools (New Relic, Elastic APM) to trace which layer is slow
2. **Database layer:**
   - Add missing indexes
   - Fix N+1 queries (JOIN FETCH)
   - Add Redis caching for repeated queries
   - Paginate large result sets
3. **Application layer:**
   - Make independent service calls **parallel** using `CompletableFuture`
   - Use `WebClient` (non-blocking) instead of `RestTemplate`
4. **API Gateway:**
   - Enable response caching at gateway level
   - Add rate limiting to protect downstream services
5. **Infra layer:**
   - Scale the slow service horizontally
   - Use CDN for static assets

---

### Q56. What is distributed tracing?

Distributed tracing tracks a request as it flows through multiple microservices, assigning it a **unique trace ID**. Each service adds a **span** (start/end time + metadata).

Tools: **Zipkin**, **Jaeger**, **AWS X-Ray**  
Spring Cloud: **Micrometer Tracing** (formerly Sleuth) propagates trace IDs via HTTP headers automatically.

```
Request → API Gateway [traceId: abc123] 
              → Order Service [span: order-create]
                    → Payment Service [span: payment-process]
                          → DB [span: db-insert]
```

---

### Q57. What is Redis Caching and how do you use it in Spring Boot?

Redis is an in-memory key-value store used to cache frequently read data.

```java
// Enable caching
@SpringBootApplication
@EnableCaching
public class App { ... }

// Cache method result
@Cacheable(value = "products", key = "#id")
public Product getProductById(Long id) {
    return productRepo.findById(id).orElseThrow(); // DB hit only on cache miss
}

// Evict cache on update
@CacheEvict(value = "products", key = "#product.id")
public Product updateProduct(Product product) { ... }
```

In your Wipro project — you reduced API latency by caching with Redis, which is a strong real-world example to mention in interviews.

---

### Q58. What are the features introduced in Java 21?

1. **Virtual Threads (Project Loom)** — lightweight threads managed by JVM; massively improves throughput for I/O-bound tasks without changing code structure
2. **Sequenced Collections** — new interfaces `SequencedCollection`, `SequencedMap` for first/last element access
3. **Record Patterns** — pattern matching for records in `instanceof` and `switch`
4. **Pattern Matching for switch** — enhanced switch with type patterns
5. **String Templates** (Preview) — embedded expressions in strings
6. **Unnamed Classes and Instance Main Methods** (Preview) — simplifies small programs

Virtual Threads are the biggest highlight — they allow writing synchronous-looking code that performs like async, dramatically simplifying Spring Boot applications under load.

---

*— End of Q&A —*

> **Interview tip (especially for 3+ years exp):** Always back your answers with a real project example. Mention the Redis caching work, the MuleSoft-to-Spring Boot migration, or your API latency optimization. Interviewers at senior level value "how you applied it" over textbook definitions.
