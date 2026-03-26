# Release It! — Stability and Production Patterns

Based on *Release It! Design and Deploy Production-Ready Software* (2nd edition) by Michael T. Nygard.
Adapted for Java 25 + Spring Boot 4 + Virtual Threads.

## Table of Contents
1. Stability Patterns
2. Stability Anti-Patterns
3. Capacity Patterns
4. Operational Transparency
5. Application Guidelines

---

## 1. Stability Patterns

### Timeout

Every integration point must have a timeout. Systems that wait indefinitely will eventually exhaust their thread pool (or in Virtual Thread terms, pile up pending work until memory is exhausted).

**The rule**: No network call without a timeout. This includes:
- HTTP client calls (external APIs, webhooks)
- JDBC connections and queries
- Redis operations
- SMTP sending

```java
// HTTP client with timeouts (Spring WebClient)
@Bean
public WebClient externalWebClient() {
  var httpClient = HttpClient.create()
      .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3_000)
      .responseTimeout(Duration.ofSeconds(5))
      .doOnConnected(conn ->
          conn.addHandlerLast(new ReadTimeoutHandler(5))
              .addHandlerLast(new WriteTimeoutHandler(5)));
  return WebClient.builder()
      .clientConnector(new ReactorClientHttpConnector(httpClient))
      .build();
}

// JDBC timeout (application.yaml)
spring:
  datasource:
    hikari:
      connection-timeout: 3000      # ms to get a connection from pool
      validation-timeout: 1000
      idle-timeout: 600000
      max-lifetime: 1800000
  jpa:
    properties:
      jakarta.persistence.query.timeout: 5000   # query-level timeout
```

```yaml
# Redis timeout (application.yaml)
spring:
  data:
    redis:
      timeout: 2000ms          # command timeout
      connect-timeout: 1000ms
      lettuce:
        pool:
          max-active: 20
          max-wait: 500ms      # pool acquisition timeout
```

**Timeout values by operation type**:

| Operation | Connect Timeout | Read/Query Timeout |
|-----------|----------------|-------------------|
| HTTP external call | 3s | 5s |
| Database query | 3s | 5s |
| Redis command | 1s | 2s |
| SMTP send | 3s | 10s |

### Circuit Breaker

Prevent repeated calls to a failing service. After a threshold of failures, open the circuit and fail fast. After a recovery window, allow a probe request.

States: **Closed** (normal) → **Open** (failing fast) → **Half-Open** (probing)

```java
// Using Resilience4j (add to libs.versions.toml)
@Bean
public CircuitBreaker externalServiceCircuitBreaker(CircuitBreakerRegistry registry) {
  var config = CircuitBreakerConfig.custom()
      .failureRateThreshold(50)                    // open at 50% failure rate
      .waitDurationInOpenState(Duration.ofSeconds(30))
      .permittedNumberOfCallsInHalfOpenState(3)
      .slidingWindowSize(10)
      .build();
  return registry.circuitBreaker("external-service", config);
}

// Apply to external call
@Service
public class PushNotificationService {
  private final CircuitBreaker circuitBreaker;

  public void send(PushNotification notification) {
    circuitBreaker.executeRunnable(() -> pushClient.send(notification));
  }
}
```

**Where to apply Circuit Breakers**:
- Push notification service
- Email/SMTP sending
- External webhook delivery
- Third-party APIs (payment, lookup, etc.)

### Bulkhead

Isolate failures to a partition. If one part fails, it doesn't sink the whole ship.

**Thread pool bulkhead**: Give different work types their own bounded executor.

```java
// CPU-bound tasks (e.g., PDF generation, heavy transformations) get their own pool
@Bean("cpuBoundExecutor")
public Executor cpuBoundExecutor() {
  return new ThreadPoolExecutor(
      Runtime.getRuntime().availableProcessors(),
      Runtime.getRuntime().availableProcessors(),
      60L, TimeUnit.SECONDS,
      new LinkedBlockingQueue<>(100),              // bounded queue — reject, don't pile up
      new ThreadPoolExecutor.CallerRunsPolicy()    // backpressure
  );
}

// Virtual Thread executor for I/O-bound (default for Spring Boot 4)
// Don't create another Virtual Thread pool — the platform default is fine

// Usage
@Async("cpuBoundExecutor")
public CompletableFuture<ReportData> generateReport(UUID tenantId) { ... }
```

**Semaphore bulkhead**: Limit concurrent access to a shared resource.

```java
// Limit concurrent webhook deliveries
private final Semaphore webhookSemaphore = new Semaphore(50);

public void deliverWebhook(Webhook webhook) {
  if (!webhookSemaphore.tryAcquire(1, TimeUnit.SECONDS)) {
    throw new WebhookCapacityExceededException("Webhook delivery queue full");
  }
  try {
    doDeliver(webhook);
  } finally {
    webhookSemaphore.release();
  }
}
```

### Fail Fast

Validate and reject bad requests at the earliest possible point. Don't let invalid input propagate deep into the system.

```java
// At filter/interceptor level: validate tenant/context before hitting service
@Component
public class TenantValidationFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest req, ...) {
    var tenantId = extractTenantId(req);
    if (!tenantRepository.existsById(tenantId)) {
      sendProblemDetail(response, HttpStatus.NOT_FOUND, "Tenant not found");
      return;
    }
    filterChain.doFilter(req, response);
  }
}

// At service level: validate required headers before doing expensive work
public OrderResponse placeOrder(PlaceOrderCommand command) {
  // Validate required fields present at boundary before loading aggregates
  Objects.requireNonNull(command.customerId(), "customerId required");

  // Validate resource accessible before loading data
  if (!orderRepository.existsByIdAndCustomerId(
        command.orderId(), command.customerId())) {
    throw new OrderNotFoundException(command.orderId());
  }
  // ... proceed with expensive operations
}
```

### Retry with Backoff

Retry transient failures, but only on idempotent operations. Use exponential backoff with jitter.

```java
// Retry configuration (Resilience4j)
@Bean
public Retry smtpRetry(RetryRegistry registry) {
  var config = RetryConfig.custom()
      .maxAttempts(3)
      .waitDuration(Duration.ofMillis(500))
      .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(500, 2.0, 0.5))
      .retryExceptions(MailException.class, ConnectException.class)
      .ignoreExceptions(InvalidAddressException.class)  // don't retry on bad input
      .build();
  return registry.retry("smtp", config);
}
```

**Retry decision table**:

| Operation | Retryable? | Reason |
|-----------|-----------|--------|
| GET requests | Yes | Idempotent |
| PUT requests | Yes | Idempotent by design |
| POST with idempotency key | Yes | Key ensures exactly-once |
| POST without idempotency | No | Risk of duplicate creation |
| DELETE | Yes (usually) | Idempotent |
| DB reads | Yes | No side effects |
| DB writes | With idempotency only | Risk of duplicate |

### Handshaking

Let callers and callees negotiate load. Use backpressure signals.

```java
// Return 429 when overloaded (not 500)
@ExceptionHandler(RateLimitExceededException.class)
public ResponseEntity<ProblemDetail> handleRateLimit(RateLimitExceededException ex) {
  var pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS);
  pd.setDetail("Rate limit exceeded");
  return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
      .header("Retry-After", "60")
      .body(pd);
}
```

---

## 2. Stability Anti-Patterns

### Cascading Failures

One component failure triggers failures in dependent components.

**Symptom**: A slow DB query causes the HTTP thread pool to exhaust, causing all requests to fail, even those that don't need the DB.

**Prevention**:
- Bulkhead: isolate DB-dependent and DB-independent request handlers
- Circuit Breaker: stop calling a failing dependency
- Timeouts: bounded waiting prevents indefinite blocking

### Blocked Threads

In Virtual Thread context this is less severe but still dangerous with CPU-bound work.

```java
// Bad: CPU-bound work on Virtual Thread pool
@GetMapping("/reports")
public ReportData generateReport() {
  return expensiveCpuWork();  // blocks a Virtual Thread for minutes
}

// Good: off-load to bounded executor
@GetMapping("/reports")
public CompletableFuture<ReportData> generateReport() {
  return CompletableFuture.supplyAsync(this::expensiveCpuWork, cpuBoundExecutor);
}
```

For Virtual Threads specifically: **never use `synchronized`** — it pins the carrier thread.

### Unbalanced Capacities

Upstream component produces faster than downstream can consume.

**Detection**: Message queues growing, `Retry-After` headers appearing, 503 responses from downstream.

**Prevention**:
- Queue with bounded capacity + `CallerRunsPolicy` (backpressure)
- Rate limiting at ingress (webhook receiver)
- Async processing with dead-letter queue for failures

```java
// Webhook receiver: accept fast, process asynchronously
@PostMapping("/webhooks/events")
@ResponseStatus(HttpStatus.ACCEPTED)  // 202, not 200
public void receiveWebhook(@RequestBody WebhookPayload payload) {
  webhookQueue.offer(payload);  // fast, non-blocking
}
```

### Slow Responses

Worse than fast failures because they hold resources for longer.

**Prevention**: Always set read timeouts. Return `503 Service Unavailable` with `Retry-After` instead of a slow 200.

### Unbounded Result Sets

DB queries without limits that return thousands of rows.

```java
// Bad: no limit
List<Order> allOrders = orderRepository.findByCustomerId(customerId);

// Good: cursor pagination with limit
List<Order> orders = orderRepository
    .findByCustomerIdAndIdLessThan(customerId, cursor, PageRequest.of(0, 50));
```

### Integration Point Failures

Every call to an external service (DB, cache, SMTP, push notification, webhooks) is a potential failure point.

**Mitigation checklist**:
- [ ] Timeout configured
- [ ] Circuit Breaker applied (for external HTTP)
- [ ] Retry with backoff (idempotent only)
- [ ] Fallback behavior defined (degrade gracefully)
- [ ] Error logged with context (tenantId, requestId)

---

## 3. Capacity Patterns

### Connection Pool Sizing

Formula: `pool size = (core count * 2) + effective_spindle_count`

For I/O-bound apps with Virtual Threads:

```yaml
# Recommended starting point for 4-core instance
spring:
  datasource:
    hikari:
      maximum-pool-size: 20     # adjust based on load testing
      minimum-idle: 5
```

Virtual Threads do NOT mean "unlimited connections". DB has finite connection slots. A large Virtual Thread count hitting a small connection pool creates contention on pool acquisition.

### Steady State

Systems must return to steady state after disturbances. Watch for:
- Memory that never releases → heap profiling, check for reference leaks
- Connection pools that drain → circuit break + timeout on acquisition
- Log files that grow without rotation → logback `RollingFileAppender`

---

## 4. Operational Transparency

### Health Checks

```java
// Spring Actuator — expose /actuator/health
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when-authorized

// Custom health indicator
@Component
public class RedisHealthIndicator extends AbstractHealthIndicator {
  private final RedisConnectionFactory connectionFactory;

  @Override
  protected void doHealthCheck(Health.Builder builder) {
    try (var conn = connectionFactory.getConnection()) {
      conn.ping();
      builder.up();
    } catch (Exception e) {
      builder.down().withException(e);
    }
  }
}
```

### Structured Logging for Observability

```java
// Log with correlation IDs and context
@Slf4j
@Service
public class OrderService {
  public void processOrder(UUID tenantId, UUID orderId, String requestId) {
    log.info("Processing order: tenantId={}, orderId={}, requestId={}",
        tenantId, orderId, requestId);

    // On failure
    log.error("Order processing failed: tenantId={}, orderId={}, requestId={}, error={}",
        tenantId, orderId, requestId, ex.getMessage(), ex);
  }
}
```

**Fields to always log**:
- `tenantId` / context identifier — for multi-tenant debugging
- `requestId` / `traceId` — for request tracing
- Operation outcome: `success`, `failure`, `partial`
- Duration for slow-path detection

### Metrics

Expose metrics via Micrometer (Spring Boot default):

```java
@Component
public class OrderMetrics {
  private final Counter ordersPlacedCounter;
  private final Timer orderProcessingTimer;

  public OrderMetrics(MeterRegistry registry) {
    this.ordersPlacedCounter = Counter.builder("app.orders.placed")
        .tag("channel", "web")
        .register(registry);
    this.orderProcessingTimer = Timer.builder("app.orders.processing.duration")
        .register(registry);
  }

  public void recordPlaced() { ordersPlacedCounter.increment(); }
  public <T> T recordProcessing(Supplier<T> operation) {
    return orderProcessingTimer.record(operation);
  }
}
```

---

## 5. Application Guidelines

### External Dependency Map

| Dependency | Failure Mode | Mitigation |
|-----------|-------------|------------|
| Database | Slow queries, connection exhaustion | Timeout + connection pool + query timeout |
| Cache (Redis, etc.) | Network partition, OOM eviction | Timeout + Circuit Breaker + graceful degradation |
| SMTP | Server down, rate limit | Circuit Breaker + Retry + async send |
| Push notification | Quota exceeded, auth failure | Circuit Breaker + dead-letter queue |
| External HTTP APIs | Timeout, 5xx, auth failure | Retry + idempotency key |

### Virtual Thread Specific Rules

```java
// FORBIDDEN: synchronized pins Virtual Thread carrier thread
public synchronized void criticalSection() { jdbcCall(); }

// REQUIRED: ReentrantLock allows Virtual Thread to park without pinning
private final ReentrantLock lock = new ReentrantLock();

public void criticalSection() {
  lock.lock();
  try {
    jdbcCall();  // Virtual Thread parks here, carrier thread is free
  } finally {
    lock.unlock();
  }
}

// FORBIDDEN: ThreadLocal state leaks across Virtual Thread reuse
private static final ThreadLocal<RequestContext> CTX = new ThreadLocal<>();

// REQUIRED: ScopedValue has bounded lifetime
private static final ScopedValue<RequestContext> CTX = ScopedValue.newInstance();
```

### Validate at the Boundary

Validate tenant context, required headers, and input constraints at the earliest entry point (filter, interceptor, or controller) — not deep in the service layer. Fail Fast prevents wasted work and makes error sources clear.

```java
// Fail Fast: reject early at controller boundary
@PostMapping("/orders")
public ResponseEntity<OrderResponse> placeOrder(
    @RequestHeader("X-Tenant-Id") UUID tenantId,   // 400 if missing
    @PathVariable UUID customerId,
    @RequestBody @Valid PlaceOrderRequest request) {
  // tenantId and input are validated before reaching the service
}
```

---

## Cross-References

- For Application Service where these patterns are implemented → `domain-driven-design.md`
- For Java concurrency idioms (ReentrantLock, ScopedValue) → `effective-java.md`
- For Spring Boot 4 health/actuator configuration → `spring-boot4-conventions.md`
