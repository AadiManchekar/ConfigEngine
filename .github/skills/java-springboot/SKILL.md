---
name: java-springboot
description: >
  Get best practices for developing applications with Spring Boot. Use this skill
  whenever the user is building, reviewing, or asking questions about a Spring Boot
  application. Trigger even if the user just mentions Spring,
  @RestController, application.yml, or any Spring-specific annotation or concept.
---

# Spring Boot Best Practices

## Project Setup & Structure

- **Build Tool:** Use Maven (`pom.xml`) or Gradle (`build.gradle`). Prefer Spring Boot Parent POM for dependency management — it provides tested, compatible versions. Override versions in the build file only when necessary, and document why.
- **Package Structure:** Organize by feature/domain (`com.example.app.order`, `com.example.app.user`), not by layer (`controller`, `service`, `repository`). This keeps related code co-located and makes modules easier to extract later.

---

## Dependency Injection

- **Constructor injection**: always. Avoids hidden dependencies, supports immutability, and makes unit testing straightforward (no Spring context needed).
- Declare injected fields as `private final`.
- Use stereotype annotations appropriately: `@Component`, `@Service`, `@Repository`, `@RestController`.

---

## Configuration

- **`application.yml`** for all config. YAML is preferred over `.properties` for its hierarchical clarity.
- **`@ConfigurationProperties`** for type-safe binding of related properties into a Java class.
- Use `${VAR:default}` syntax for environment variable fallbacks.
- **Profiles:** `application-dev.yml`, `application-prod.yml` for environment-specific overrides.
- **Never hardcode secrets.** Use environment variables, HashiCorp Vault, or AWS Secrets Manager.

---

## Validation & Error Handling

### Input Validation

Use **Java Bean Validation (JSR 380)** annotations on request DTOs:
- `@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Positive`, `@Email`
- Trigger validation with `@Valid` on controller method parameters.

Prefer **Java records for DTOs**: they are immutable and concise:

```java
public record CreateOrderRequest(
    @NotNull UUID customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @Positive BigDecimal total
) {}
```

### Custom Validators

For cross-field or complex business rules, implement `ConstraintValidator`:

```java
public class OrderValidator implements ConstraintValidator<ValidOrder, CreateOrderRequest> {
    @Override
    public boolean isValid(CreateOrderRequest req, ConstraintValidatorContext ctx) {
        if (req == null) return true; // @NotNull handles null
        boolean valid = true;
        if (req.total().compareTo(req.items().stream()
                .map(i -> i.price().multiply(BigDecimal.valueOf(i.quantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add)) != 0) {
            ctx.buildConstraintViolationWithTemplate("Total does not match item sum")
               .addPropertyNode("total").addConstraintViolation();
            valid = false;
        }
        return valid;
    }
}
```

### Enum Validation with Predicates

Use enums with built-in `Predicate`-based rules for domain-level validation:

```java
public enum OrderValidation {
    HAS_CUSTOMER(order -> order.customerId() != null),
    HAS_ITEMS(order -> order.items() != null && !order.items().isEmpty()),
    POSITIVE_TOTAL(order -> order.total().compareTo(BigDecimal.ZERO) > 0);

    private final Predicate<CreateOrderRequest> rule;

    OrderValidation(Predicate<CreateOrderRequest> rule) { this.rule = rule; }

    public boolean test(CreateOrderRequest order) { return rule.test(order); }

    public static boolean validateAll(CreateOrderRequest order) {
        for (OrderValidation v : values()) {
            if (!v.test(order)) return false;
        }
        return true;
    }
}
```

Use `@Enumerated(EnumType.STRING)` on JPA entity fields whose values come from a fixed enum.

### Error Responses (RFC 9457)

Enable standardized error responses:
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

Implement a global handler with `@RestControllerAdvice`:

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        log.warn("Resource not found: {}", ex.getMessage());
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setInstance(URI.create(req.getRequestURI()));
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problem);
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest req) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setType(URI.create("https://api.example.com/errors/validation"));
        problem.setProperty("errors", errors);
        return ResponseEntity.badRequest().body(problem);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleGeneric(Exception ex, HttpServletRequest req) {
        log.error("Unexpected error", ex);
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        problem.setType(URI.create("https://api.example.com/errors/internal"));
        return ResponseEntity.internalServerError().body(problem);
    }
}
```

---

## Web Layer (Controllers)

- Design clear, consistent RESTful endpoints.
- **Never expose JPA entities directly**: always use DTOs (prefer records).
- Return correct HTTP status codes:
  - `200 OK`: successful GET/PUT
  - `201 Created`: successful POST creating a resource
  - `204 No Content`: successful void operation (`@ResponseStatus(HttpStatus.NO_CONTENT)`)
  - `400` / `401` / `403` / `404` / `409` / `500` as appropriate
- Use `ResponseEntity<T>` when you need full control over status + headers + body.
- **Pagination:** Use `Pageable` and return `Page<T>` (when UI needs total count) or `Slice<T>` (skip expensive count query when not needed).

```java
@GetMapping
public ResponseEntity<Page<OrderSummary>> listOrders(
    @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
    return ResponseEntity.ok(orderService.findAll(pageable));
}
```

---

## Service Layer

- All business logic lives in `@Service` classes.
- Services must be **stateless**: no mutable instance fields shared across threads.
- Use `@Transactional` at the most granular level needed (method, not class, unless every method needs it).

---

## Data Layer (Repositories)

- Extend `JpaRepository<T, ID>` for standard CRUD.
- Use `@Query` or the Criteria API for complex queries.
- Use **DTO projections** (interface or class-based) to SELECT only what you need — avoids loading full entities unnecessarily.

---

## Declarative HTTP Clients

Prefer Spring's `@HttpExchange` over `RestTemplate`, `WebClient`, or OpenFeign:

```java
@HttpExchange("/api/configs")
public interface DeviceConfigClient {

    @GetExchange("/{deviceId}")
    @Retryable(maxAttempts = 3, delay = 100, multiplier = 2, maxDelay = 1000)
    @ConcurrencyLimit(3)
    DeviceConfigResponse getConfig(@PathVariable String deviceId);
}
```

Register it via `HttpServiceProxyFactory` in a `@Configuration` class.

---

## Concurrency & Thread Safety

Spring beans are singletons by default — shared across all threads. Always design for concurrent access.

### Rules
- **No mutable instance state in beans.** If you need state, use local variables, method parameters, or thread-local storage.
- **`@Transactional` does not make code thread-safe.** It manages DB transactions, not concurrent access to shared state.
- **Stateless services are thread-safe by design**: all collaborators injected via constructor, no `static` mutable fields.

### When you need shared state

Use the right concurrency primitive:

| Scenario | Tool |
|---|---|
| Simple counter / flag | `AtomicInteger`, `AtomicBoolean` |
| Shared map/list | `ConcurrentHashMap`, `CopyOnWriteArrayList` |
| Complex invariants across fields | `synchronized` block or `ReentrantLock` |
| Cached computation | `ConcurrentHashMap.computeIfAbsent()` |

### `@Async` (background tasks)

```java
@Service
public class NotificationService {
    @Async
    public CompletableFuture<Void> sendEmail(String to, String subject) {
        // runs in Spring's task executor thread pool
        emailClient.send(to, subject);
        return CompletableFuture.completedFuture(null);
    }
}
```

Enable with `@EnableAsync` on a `@Configuration` class. Always configure a custom executor with bounded queue size and rejection policy — the default unbounded queue can cause memory exhaustion under load:

```java
@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("async-");
    executor.initialize();
    return executor;
}
```

### MDC in async/multi-threaded contexts

MDC is thread-local — it does **not** propagate across `@Async` or `CompletableFuture` threads automatically. Copy it explicitly:

```java
Map<String, String> mdcCopy = MDC.getCopyOfContextMap();
CompletableFuture.runAsync(() -> {
    if (mdcCopy != null) MDC.setContextMap(mdcCopy);
    try {
        // task logic
    } finally {
        MDC.clear();
    }
});
```

---

## Logging

- Use SLF4J: `private static final Logger log = LoggerFactory.getLogger(MyClass.class);`
- Use parameterized messages — never string concatenation: `log.info("Processing order {}", orderId);`
```yaml
logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

- Use **MDC** (`Mapped Diagnostic Context`) to attach `userId`, `correlationId`, `deviceId` to every log line in a request:

```java
MDC.put("correlationId", correlationId);
try {
    // all log calls here automatically include correlationId
} finally {
    MDC.clear(); // always clear — thread may be reused
}
```

Include MDC keys in the log pattern: `%X{correlationId}`.

---

## Testing

| What to test | Annotation |
|---|---|
| Service/component unit tests | JUnit 6 + Mockito (no Spring context) |
| Controller layer | `@WebMvcTest` |
| Repository layer | `@DataJpaTest` |
| Full integration | `@SpringBootTest` |
| External dependencies (DB, broker) | Testcontainers |

---

## Security

- Use **Spring Security** for authentication and authorization.
- Always hash passwords with **BCrypt** (`PasswordEncoder`).
- Prevent SQL injection: use Spring Data JPA or parameterized queries, never string-concatenated SQL.
- Encode output to prevent XSS.

---

## Performance

1. **Connection pooling**: Use a connection pool for databases (HikariCP), thread pools for async tasks, and HTTP client connection pools.
2. **Pagination**: Never return unbounded collections.
3. **Caching**: Use `@Cacheable` with Redis or Caffeine for read-heavy data.
4. **JPA optimization**: Use projections and `@EntityGraph` to avoid N+1 queries.
5. **Async I/O**: Use `@Async` or reactive patterns for I/O-bound work.
6. **Response compression**: Enable `server.compression.enabled=true` for large payloads.
