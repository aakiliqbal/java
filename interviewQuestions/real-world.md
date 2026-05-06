---
layout: default
title: Real-World Questions
---

# Real-World Java Backend Interview Questions

<div class="page-intro">
  <p>This page contains scenario-based answers merged from both original files. The focus is production thinking: how to debug, what metrics to check, what trade-offs matter, and how to explain the answer in an interview.</p>
</div>

## Quick Navigation

- [Production Debugging](#production-debugging)
- [Concurrency Scenarios](#concurrency-scenarios)
- [Spring Boot Scenarios](#spring-boot-scenarios)
- [Database Scenarios](#database-scenarios)
- [Microservices Scenarios](#microservices-scenarios)
- [Kafka Scenarios](#kafka-scenarios)
- [System Design Scenarios](#system-design-scenarios)
- [DevOps And Project Questions](#devops-and-project-questions)

## Production Debugging

### 1. Application throws `OutOfMemoryError`. How will you debug?

First, identify the exact error type. `Java heap space`, `Metaspace`, `GC overhead limit exceeded`, and `Direct buffer memory` point to different root causes.

Steps:

1. Check logs around the failure time.
2. Capture heap dump using JVM options or tools.
3. Analyze heap dump with Eclipse MAT, VisualVM, JProfiler, or YourKit.
4. Check top memory consumers and retained object graph.
5. Review recent deployments and traffic changes.
6. Check GC logs to understand allocation rate and pause behavior.

Useful JVM options:

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdump.hprof
```

Common causes:

| Symptom | Likely cause |
|---|---|
| Large `List` or `Map` retained | Static collection, cache without eviction, loading all rows |
| Many class metadata objects | Classloader leak or excessive dynamic class generation |
| Many byte arrays | Large file processing, serialization, buffering |
| High old-gen usage after GC | Memory leak or genuine undersized heap |

Strong interview answer: do not immediately increase heap. First prove whether it is a leak, traffic growth, large query result, cache issue, or bad object lifecycle.

### 2. High CPU usage but low traffic. What could be the reason?

Possible reasons:

1. Infinite loop.
2. Heavy garbage collection.
3. Thread contention or lock spinning.
4. Bad regex causing catastrophic backtracking.
5. Background scheduler processing too much data.
6. Retry storm against a failing dependency.
7. Excessive logging.
8. Serialization/deserialization of large payloads.
9. Encryption/compression hotspot.

Debugging approach:

```bash
top
jcmd <pid> Thread.print
jstack <pid>
jcmd <pid> JFR.start
```

Find the process, identify hot threads, take multiple thread dumps 10-20 seconds apart, and check whether the same stack trace is repeatedly consuming CPU. If GC threads are hot, inspect GC logs and allocation rate.

### 3. Application slows down after running for a few hours. What will you check?

This usually indicates resource leakage or gradual saturation.

Check:

| Area | What to inspect |
|---|---|
| Heap | Memory leak, cache growth, old-gen trend |
| Threads | Thread leak, blocked threads, pool exhaustion |
| DB pool | Active connections, pending requests, leak detection |
| GC | Pause time, frequency, allocation rate |
| Logs | Error loops, retry loops, excessive logging |
| External APIs | Latency, timeout, failure rate |
| Kafka | Consumer lag, rebalance frequency |

A good answer separates mitigation from root cause. Mitigation may be restart, scale out, disable a feature, or rollback. Root cause requires dumps, metrics, traces, and code review.

### 4. Frequent GC pauses. How do you optimize?

Steps:

1. Enable and inspect GC logs.
2. Check allocation rate and object lifetime.
3. Analyze heap dump if old generation keeps growing.
4. Reduce temporary object creation.
5. Avoid loading large datasets into memory.
6. Add pagination or streaming.
7. Put size limits on caches.
8. Tune heap size after understanding usage.
9. Choose an appropriate collector such as G1 or ZGC.

Do not tune GC blindly. If the application creates too many temporary objects or retains unnecessary references, GC tuning only hides the issue.

### 5. Service becomes unresponsive randomly. What could be happening?

Likely causes:

1. Thread pool exhaustion.
2. DB connection pool exhaustion.
3. Deadlock.
4. Long GC pause.
5. External API calls without timeout.
6. Kafka consumer rebalance storm.
7. Load balancer health-check issue.
8. CPU throttling in container.
9. Memory leak leading to swap or OOM.

Debug with:

```text
Thread dump
Heap dump
GC logs
APM trace
Hikari metrics
CPU and memory metrics
Recent deployment diff
```

For a production interview, say you first reduce impact, then investigate. Impact reduction may be rollback, scaling, traffic shifting, disabling a bad job, or increasing timeout only as a temporary mitigation.

## Concurrency Scenarios

### 6. Thread stuck in `BLOCKED` state. How do you identify and fix it?

`BLOCKED` means the thread is waiting to acquire a monitor lock.

Steps:

1. Take a thread dump.
2. Find threads marked `BLOCKED`.
3. Identify the lock object and owner thread.
4. Check what the owner thread is doing.
5. Reduce synchronized block size or remove nested locks.
6. Replace manual locking with concurrent utilities where possible.

Fix options:

| Cause | Fix |
|---|---|
| Long work inside synchronized block | Move slow I/O outside lock |
| Nested locks | Enforce lock ordering |
| Shared mutable map | Use `ConcurrentHashMap` |
| Lock wait forever | Use `ReentrantLock.tryLock()` with timeout |

### 7. Multiple threads update shared data incorrectly. How will you fix it?

First identify whether the data needs atomicity, visibility, ordering, or all three.

Examples:

| Problem | Correct tool |
|---|---|
| Shared counter | `AtomicInteger` / `LongAdder` |
| Shared map | `ConcurrentHashMap` |
| Multiple fields must change together | `synchronized` / `ReentrantLock` |
| Stop/start flag | `volatile` |
| Producer-consumer workflow | `BlockingQueue` |

For database-backed shared state, JVM locks are not enough when multiple service instances are running. Use database transactions, optimistic locking, pessimistic locking, idempotency keys, or distributed locks depending on the case.

### 8. Thread pool gets exhausted under load. What is your approach?

Symptoms include high latency, queued requests, request timeouts, and many threads in `WAITING` or blocked on external calls.

Check:

1. Pool size and queue size.
2. Active thread count.
3. Queue depth.
4. Task execution time.
5. External dependency latency.
6. Missing timeouts.
7. Rejected task count.

Fixes:

1. Add timeouts to all external calls.
2. Use bulkheads for slow dependencies.
3. Reduce blocking work.
4. Increase pool size only after capacity analysis.
5. Apply backpressure or rate limiting.
6. Split pools by workload so one slow dependency does not consume all threads.

## Spring Boot Scenarios

### 9. Design an API with proper layering and global exception handling.

Recommended flow:

```text
Controller -> Service -> Repository -> Database
```

Controller handles HTTP concerns: request validation, status codes, and response DTOs. Service contains business logic and transaction boundaries. Repository handles persistence.

Example structure:

```java
@RestController
@RequestMapping("/orders")
class OrderController {
    private final OrderService orderService;

    @PostMapping
    ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(orderService.create(request));
    }
}
```

Use `@RestControllerAdvice` for centralized exception handling:

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    ResponseEntity<ErrorResponse> notFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }
}
```

Return consistent error responses with code, message, timestamp, path, and validation field errors.

### 10. How do you add OTP, CAPTCHA, and password login without modifying the existing API heavily?

Use Strategy pattern with Spring beans.

```java
interface LoginStrategy {
    LoginResult authenticate(LoginRequest request);
}

@Service("PASSWORD")
class PasswordLoginStrategy implements LoginStrategy {
    public LoginResult authenticate(LoginRequest request) {
        return null;
    }
}

@Service("OTP")
class OtpLoginStrategy implements LoginStrategy {
    public LoginResult authenticate(LoginRequest request) {
        return null;
    }
}
```

Then inject all strategies:

```java
@Service
class LoginService {
    private final Map<String, LoginStrategy> strategies;

    LoginService(Map<String, LoginStrategy> strategies) {
        this.strategies = strategies;
    }

    LoginResult login(String type, LoginRequest request) {
        LoginStrategy strategy = strategies.get(type);
        if (strategy == null) {
            throw new IllegalArgumentException("Unsupported login type");
        }
        return strategy.authenticate(request);
    }
}
```

This keeps the controller stable and adds new login mechanisms by adding new implementations. It follows Open/Closed Principle.

### 11. API calls another API sequentially. What is this model called and when is it a problem?

This is synchronous sequential composition.

```text
Request -> Service A -> Service B -> Service C -> Response
```

It is simple, but total latency becomes the sum of all downstream calls. If each call takes 300 ms, three sequential calls can take roughly 900 ms before application overhead.

Use it when each step depends on the previous result. If calls are independent, run them concurrently using `CompletableFuture`, WebClient, virtual threads, or async messaging depending on requirements.

### 12. Independent API calls can run concurrently. How would you improve latency?

If user profile, orders, and recommendations are independent, call them in parallel.

```java
CompletableFuture<User> user = supplyAsync(() -> userClient.get(id));
CompletableFuture<List<Order>> orders = supplyAsync(() -> orderClient.get(id));
CompletableFuture<List<Item>> recommendations = supplyAsync(() -> recommendationClient.get(id));

CompletableFuture.allOf(user, orders, recommendations).join();
```

Important production details:

1. Use a dedicated executor.
2. Set timeouts.
3. Handle partial failures.
4. Avoid overwhelming downstream services.
5. Add tracing so parallel calls are visible.

## Database Scenarios

### 13. How do you optimize slow database queries?

Start by proving where the time is spent.

Steps:

1. Check APM trace and slow query logs.
2. Run `EXPLAIN` or execution plan.
3. Verify indexes on filter, join, and sort columns.
4. Avoid `SELECT *`; fetch only required columns.
5. Fix N+1 queries.
6. Add pagination.
7. Review joins and cardinality.
8. Check lock waits.
9. Tune connection pool if threads wait for connections.
10. Use caching only for read-heavy stable data.

Example answer:

> I first identify the slow SQL through logs/APM, then check execution plan. If the query scans a large table, I add or adjust indexes. If Hibernate triggers N+1, I use join fetch, entity graph, or DTO projection. I also make sure the API is paginated and not loading unnecessary columns.

### 14. How do you handle concurrent updates?

Options:

| Technique | Use case |
|---|---|
| Optimistic locking | Most web updates, conflicts are rare |
| Pessimistic locking | High-conflict critical updates |
| Unique constraints | Prevent duplicate business keys |
| Idempotency key | Prevent duplicate request processing |
| Distributed lock | Cross-instance exclusive processing |

Optimistic locking example:

```java
@Version
private Long version;
```

If two users update the same row, one transaction succeeds and the other gets an optimistic lock exception. The API can return a conflict response and ask the client to retry with fresh data.

### 15. DB connection pool is exhausted. What will you check?

Check Hikari metrics:

1. Active connections.
2. Idle connections.
3. Pending threads.
4. Connection acquisition time.
5. Leak detection logs.

Common causes:

1. Slow queries holding connections.
2. Transactions doing external API calls before commit.
3. Connection leak.
4. Pool too small for legitimate load.
5. Database overloaded, making every query slow.

Fix the query/transaction behavior before increasing pool size. Bigger pools can make database saturation worse.

## Microservices Scenarios

### 16. API Gateway -> Controller -> Service -> DB is slow. How do you optimize end-to-end?

Use tracing to identify the bottleneck instead of guessing.

Layer-by-layer:

| Layer | What to check |
|---|---|
| Gateway | Rate limiting, routing latency, auth call latency |
| Controller | Request validation, payload size, serialization |
| Service | Sequential external calls, blocking calls, thread pool |
| Database | Slow query, missing index, N+1, connection wait |
| Network | DNS, TLS, retries, packet loss |

Fix examples:

1. Add DB indexes and pagination.
2. Replace sequential independent calls with parallel calls.
3. Add cache for stable read-heavy data.
4. Add timeout and circuit breaker for dependencies.
5. Scale only the bottleneck service.
6. Reduce payload size.

### 17. Downstream service is failing. How should caller service behave?

Caller service should fail fast and degrade safely.

Use:

1. Timeout to avoid hanging threads.
2. Retry with exponential backoff for transient errors.
3. Circuit breaker to stop repeated calls to unhealthy service.
4. Fallback for non-critical data.
5. Bulkhead to isolate dependency thread pool.
6. Alerts based on failure rate and latency.

Do not retry every failure. For example, retrying validation errors or non-idempotent payment calls can create duplicate side effects.

### 18. System processes duplicate requests. How will you handle it?

Use idempotency.

Flow:

1. Client sends `Idempotency-Key`.
2. Server stores the key with request hash and result.
3. If the same key arrives again, return the previous result.
4. Add a unique database constraint on the key.
5. Set TTL based on business requirement.

Database example:

```sql
CREATE UNIQUE INDEX ux_payment_idempotency_key
ON payments(idempotency_key);
```

This is critical for payments, order creation, coupon usage, and message consumers.

### 19. How do you scale microservices?

Scaling is not only adding instances.

Approach:

1. Make services stateless.
2. Add horizontal scaling behind a load balancer.
3. Use database indexes and read replicas for read-heavy load.
4. Cache frequently read data.
5. Move heavy background work to Kafka/queues.
6. Add rate limiting at gateway.
7. Use autoscaling based on CPU, memory, latency, queue lag, or custom metrics.
8. Split bottleneck services only when domain and load justify it.

Measure before scaling. Scaling a service will not help if the database is the bottleneck.

### 20. How do you handle distributed locking?

Distributed locking ensures only one instance processes a resource.

Use cases:

1. Scheduled job running on multiple pods.
2. Payment processing for same transaction.
3. Inventory reservation.
4. Coupon redemption.

Redis concept:

```text
SET lock:order:101 uniqueValue NX PX 30000
```

Rules:

1. Always set expiry.
2. Release only if the value matches.
3. Keep lock duration short.
4. Prefer proven libraries like Redisson.
5. Consider whether database constraints or optimistic locking are simpler.

## Kafka Scenarios

### 21. Kafka consumer crashes after processing but before committing offset. What happens?

Kafka will redeliver the message from the last committed offset. This can cause duplicate processing.

Correct design:

1. Process message.
2. Write result and processed message ID in the same database transaction if possible.
3. Commit offset only after successful processing.
4. Make the consumer idempotent.

```java
if (!processedMessageRepository.existsById(event.id())) {
    process(event);
    processedMessageRepository.save(new ProcessedMessage(event.id()));
}
```

Kafka generally provides at-least-once delivery in common setups, so consumer logic must tolerate duplicates.

### 22. Kafka lag is increasing. What will you check?

Lag means producers are writing faster than consumers are processing.

Check:

1. Consumer processing time.
2. Consumer errors and retries.
3. Number of partitions vs consumers.
4. Slow database or external API inside consumer.
5. Rebalance frequency.
6. Message size.
7. Broker health.

Fixes:

1. Optimize consumer processing.
2. Increase partitions if parallelism is limited.
3. Add consumers up to partition count.
4. Batch database writes.
5. Move slow external calls out of the critical consumer path.
6. Use DLQ for poison messages.

### 23. How do you maintain message ordering in Kafka?

Kafka guarantees order only within a partition. To maintain order for one business entity, use the same key for all related events.

```java
kafkaTemplate.send("orders", orderId, event);
```

Trade-off: all events for that key go to one partition, so a very hot key can limit parallelism.

### 24. How do you handle poison messages?

A poison message repeatedly fails and blocks progress.

Approach:

1. Retry a limited number of times.
2. Use backoff.
3. Send failed message to a dead-letter topic.
4. Include failure reason and original metadata.
5. Alert and inspect DLQ.
6. Build replay tooling after fixing the issue.

Never retry forever in the main consumer loop.

## System Design Scenarios

### 25. Design a payment processing system for 1M+ transactions daily.

High-level components:

```text
API Gateway
Payment Service
Order Service
User Service
Transaction Database
Kafka
Notification Service
Fraud Detection Service
Redis
Monitoring and Alerting
```

Flow:

1. Client sends payment request with idempotency key.
2. Gateway authenticates and rate-limits request.
3. Payment Service validates request.
4. Create payment row with `PENDING` status.
5. Call external payment gateway.
6. Update payment status.
7. Publish `PaymentSucceeded` or `PaymentFailed` event.
8. Order Service updates order status.
9. Notification Service sends email/SMS.

Important design points:

| Concern | Design |
|---|---|
| Duplicate payments | Idempotency key with unique constraint |
| Consistency | Saga with compensating actions |
| Reliability | Kafka events and retry policies |
| Security | JWT/OAuth2, TLS, audit logs, no raw card storage |
| Observability | Trace ID, metrics, payment success rate, gateway latency |
| Failure handling | Pending status reconciliation job |

Mention reconciliation because real payment systems can receive late callbacks or have uncertain gateway responses.

### 26. How would you implement rate limiting?

Rate limiting controls request volume per user, IP, API key, or tenant.

Algorithms:

1. Fixed window.
2. Sliding window.
3. Token bucket.
4. Leaky bucket.

For distributed systems, use Redis or gateway-native rate limiting.

Example policy:

```text
100 requests per minute per user
```

Return HTTP `429 Too Many Requests` and include headers such as remaining quota and retry-after when possible.

### 27. How do you use Redis caching correctly?

Use cache-aside:

```text
Read request -> Check Redis -> If miss, load DB -> Store in Redis -> Return
```

For updates:

1. Update database.
2. Evict or update cache.
3. Use TTL to limit stale data.

Common problems:

| Problem | Fix |
|---|---|
| Stale cache | TTL and eviction on write |
| Cache penetration | Cache nulls briefly or use Bloom filter |
| Cache stampede | Locking, early refresh, jittered TTL |
| Huge keys | Compress or redesign value shape |

Do not cache highly volatile data unless stale reads are acceptable.

## DevOps And Project Questions

### 28. API works locally but fails in production. What will you check?

Check differences between local and production:

1. Environment variables.
2. Active Spring profile.
3. Database URL and credentials.
4. Network/firewall/security group.
5. SSL certificates.
6. External API endpoint.
7. Secrets/config maps.
8. Java and dependency versions.
9. Timezone and locale.
10. File permissions.
11. Docker/Kubernetes resource limits.
12. Recent deployment changes.

Strong answer: local vs production failures usually come from config, credentials, network, dependency version, or infrastructure assumptions.

### 29. How do you monitor logs in production?

Use centralized logging.

Tools:

```text
ELK / EFK
Grafana Loki
Splunk
Datadog
New Relic
CloudWatch
```

Best practices:

1. Structured JSON logs.
2. Include trace ID and request ID.
3. Use correct log levels.
4. Avoid sensitive data.
5. Create alerts for error spikes.
6. Correlate logs with metrics and traces.

Example log fields:

```text
traceId, userId, endpoint, status, latencyMs, errorCode
```

### 30. How do you deploy a Spring Boot microservice with Docker?

Basic Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app
COPY target/app.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Production deployment should also include health checks, externalized config, secrets management, resource limits, logging, metrics, and rollback strategy.

### 31. Explain your project architecture clearly.

Use a layered explanation:

```text
Client -> API Gateway -> Controller -> Service -> Repository -> Database/External APIs
```

Sample answer:

> In my project, requests enter through the API Gateway where authentication and routing happen. The controller validates the request and passes it to the service layer. Business logic and transaction boundaries are in the service layer. Repositories handle database access using Spring Data JPA. For external integrations we use REST clients, and for asynchronous workflows we publish events. We monitor latency and errors using centralized logs, metrics, and tracing.

### 32. How do you handle production issues?

Use an incident structure:

1. Identify impact and severity.
2. Check alerts, logs, dashboards, and recent deployments.
3. Mitigate quickly through rollback, scaling, feature flag, or traffic shift.
4. Find root cause using evidence.
5. Apply permanent fix.
6. Add tests, alerts, or dashboards to prevent recurrence.
7. Document the incident.

Interviewers value calm, evidence-based debugging. Avoid saying you randomly restart services unless it is clearly a mitigation step.

### 33. How do you answer high-pressure situation questions?

Use a practical answer:

> I first focus on impact reduction, then root cause. I avoid random changes and use logs, metrics, traces, and recent deployment history to narrow the issue. I keep communication clear with the team and update stakeholders with what is known, what is unknown, and the next action. After the fix, I document the root cause and preventive actions.

### 34. How do you validate AI-generated code?

Treat AI as an assistant, not an authority.

Validation checklist:

1. Read the code line by line.
2. Run unit and integration tests.
3. Check edge cases.
4. Check security issues.
5. Verify dependency versions and APIs.
6. Compare with project style.
7. Benchmark if performance-sensitive.
8. Ensure no secrets, license issues, or copied proprietary code.

