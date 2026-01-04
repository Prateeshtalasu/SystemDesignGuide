# ⚛️ Reactive Programming

---

## 0️⃣ Prerequisites

Before diving into Reactive Programming, you need to understand:

- **Java Concurrency**: Threads, ExecutorService (covered in `06-java-concurrency.md`)
- **Streams API**: map, filter, flatMap (covered in `04-streams-api.md`)
- **Functional Interfaces**: Consumer, Function, Supplier

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Problem with Blocking I/O

```java
// Traditional blocking code
public List<Order> getOrdersForUsers(List<String> userIds) {
    List<Order> allOrders = new ArrayList<>();
    
    for (String userId : userIds) {
        User user = userService.getUser(userId);     // Blocks ~50ms
        List<Order> orders = orderService.getOrders(user); // Blocks ~100ms
        allOrders.addAll(orders);
    }
    
    return allOrders;  // Total: 150ms × N users (sequential!)
}

// With 100 users: 15 seconds!
// Thread is BLOCKED for 15 seconds, doing nothing useful
```

### The Callback Hell Solution (and Its Problems)

```java
// Attempt to make it async with callbacks
public void getOrdersForUsers(List<String> userIds, Consumer<List<Order>> callback) {
    List<Order> allOrders = new CopyOnWriteArrayList<>();
    AtomicInteger remaining = new AtomicInteger(userIds.size());
    
    for (String userId : userIds) {
        userService.getUserAsync(userId, user -> {
            orderService.getOrdersAsync(user, orders -> {
                allOrders.addAll(orders);
                if (remaining.decrementAndGet() == 0) {
                    callback.accept(allOrders);
                }
            }, error -> {
                // Handle error
            });
        }, error -> {
            // Handle error
        });
    }
}

// Problems:
// 1. Callback hell - deeply nested, hard to read
// 2. Error handling scattered
// 3. Hard to compose operations
// 4. No backpressure handling
```

### What Reactive Programming Provides

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REACTIVE STREAMS                                      │
│                                                                          │
│   CORE CONCEPTS:                                                        │
│   ─────────────                                                         │
│                                                                          │
│   1. PUBLISHER: Produces data (0 to N elements)                        │
│      - Mono<T>: 0 or 1 element                                         │
│      - Flux<T>: 0 to N elements                                        │
│                                                                          │
│   2. SUBSCRIBER: Consumes data                                         │
│      - Requests data from publisher                                    │
│      - Receives onNext, onError, onComplete signals                   │
│                                                                          │
│   3. SUBSCRIPTION: Connection between publisher and subscriber         │
│      - Controls flow with request(n)                                   │
│      - Can cancel()                                                    │
│                                                                          │
│   4. BACKPRESSURE: Subscriber controls flow rate                       │
│      - "I can only handle 10 items at a time"                         │
│      - Prevents overwhelming slow consumers                            │
│                                                                          │
│   DATA FLOW:                                                            │
│   ┌──────────┐  subscribe   ┌────────────┐                             │
│   │ Publisher├─────────────►│ Subscriber │                             │
│   │ (Source) │              │ (Consumer) │                             │
│   └────┬─────┘              └─────┬──────┘                             │
│        │                          │                                     │
│        │◄────── request(n) ───────┤                                     │
│        │                          │                                     │
│        ├─────── onNext(item) ────►│                                     │
│        ├─────── onNext(item) ────►│                                     │
│        ├─────── onComplete() ────►│                                     │
│        │    or  onError(e) ──────►│                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2️⃣ Mono and Flux

### Mono: 0 or 1 Element

```java
// Creating Mono
Mono<String> empty = Mono.empty();
Mono<String> just = Mono.just("Hello");
Mono<String> fromCallable = Mono.fromCallable(() -> expensiveOperation());
Mono<String> fromFuture = Mono.fromFuture(completableFuture);
Mono<String> defer = Mono.defer(() -> Mono.just(generateValue()));

// Error handling
Mono<String> error = Mono.error(new RuntimeException("Failed"));

// Transformations
Mono<Integer> length = Mono.just("Hello")
    .map(String::length);

Mono<User> user = Mono.just(userId)
    .flatMap(id -> userRepository.findById(id));

// Fallbacks
Mono<String> withDefault = mono
    .defaultIfEmpty("default")
    .switchIfEmpty(Mono.just("alternative"));

// Error handling
Mono<String> withErrorHandling = mono
    .onErrorReturn("fallback")
    .onErrorResume(e -> Mono.just("recovered"))
    .onErrorMap(e -> new CustomException(e));
```

### Flux: 0 to N Elements

```java
// Creating Flux
Flux<Integer> range = Flux.range(1, 10);
Flux<String> fromIterable = Flux.fromIterable(List.of("a", "b", "c"));
Flux<String> fromArray = Flux.fromArray(new String[]{"a", "b", "c"});
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));

// Transformations
Flux<Integer> doubled = Flux.range(1, 5)
    .map(n -> n * 2);

Flux<String> flatMapped = Flux.range(1, 3)
    .flatMap(n -> Flux.just(n + "a", n + "b"));

// Filtering
Flux<Integer> filtered = Flux.range(1, 10)
    .filter(n -> n % 2 == 0);

// Combining
Flux<String> merged = Flux.merge(flux1, flux2);
Flux<String> concatenated = Flux.concat(flux1, flux2);
Flux<Tuple2<String, Integer>> zipped = Flux.zip(stringFlux, intFlux);

// Aggregation
Mono<List<String>> collected = flux.collectList();
Mono<Long> count = flux.count();
Mono<Integer> reduced = Flux.range(1, 5).reduce(0, Integer::sum);
```

### Subscribing (Terminal Operations)

```java
// Basic subscribe
flux.subscribe();

// With consumer
flux.subscribe(item -> System.out.println("Received: " + item));

// With error handler
flux.subscribe(
    item -> System.out.println("Received: " + item),
    error -> System.err.println("Error: " + error)
);

// With completion handler
flux.subscribe(
    item -> System.out.println("Received: " + item),
    error -> System.err.println("Error: " + error),
    () -> System.out.println("Completed!")
);

// With subscription control
flux.subscribe(new BaseSubscriber<String>() {
    @Override
    protected void hookOnSubscribe(Subscription subscription) {
        request(1);  // Request one item at a time
    }
    
    @Override
    protected void hookOnNext(String value) {
        System.out.println("Received: " + value);
        request(1);  // Request next item
    }
});

// Block (for testing only!)
String result = mono.block();
List<String> results = flux.collectList().block();
```

---

## 3️⃣ Spring WebFlux

### Reactive REST Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserRepository userRepository;
    private final OrderService orderService;
    
    public UserController(UserRepository userRepository, OrderService orderService) {
        this.userRepository = userRepository;
        this.orderService = orderService;
    }
    
    // GET /api/users/{id}
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable String id) {
        return userRepository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    // GET /api/users
    @GetMapping
    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    // GET /api/users (streaming)
    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<User> streamUsers() {
        return userRepository.findAll()
            .delayElements(Duration.ofMillis(100));
    }
    
    // POST /api/users
    @PostMapping
    public Mono<ResponseEntity<User>> createUser(@RequestBody Mono<User> userMono) {
        return userMono
            .flatMap(userRepository::save)
            .map(user -> ResponseEntity
                .created(URI.create("/api/users/" + user.getId()))
                .body(user));
    }
    
    // GET /api/users/{id}/orders
    @GetMapping("/{id}/orders")
    public Flux<Order> getUserOrders(@PathVariable String id) {
        return userRepository.findById(id)
            .flatMapMany(orderService::getOrdersForUser);
    }
}
```

### Reactive Repository

```java
// Spring Data Reactive MongoDB
public interface UserRepository extends ReactiveMongoRepository<User, String> {
    
    Flux<User> findByLastName(String lastName);
    
    Mono<User> findByEmail(String email);
    
    @Query("{ 'age': { $gt: ?0 } }")
    Flux<User> findUsersOlderThan(int age);
}

// Spring Data R2DBC (Reactive SQL)
public interface OrderRepository extends ReactiveCrudRepository<Order, Long> {
    
    Flux<Order> findByUserId(String userId);
    
    @Query("SELECT * FROM orders WHERE status = :status")
    Flux<Order> findByStatus(String status);
}
```

### WebClient (Reactive HTTP Client)

```java
@Service
public class ExternalApiService {
    
    private final WebClient webClient;
    
    public ExternalApiService(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
    
    public Mono<User> getUser(String id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(WebClientResponseException.NotFound.class, 
                e -> Mono.empty());
    }
    
    public Flux<Order> getOrders(String userId) {
        return webClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/orders")
                .queryParam("userId", userId)
                .build())
            .retrieve()
            .bodyToFlux(Order.class);
    }
    
    public Mono<User> createUser(User user) {
        return webClient.post()
            .uri("/users")
            .bodyValue(user)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

---

## 4️⃣ Backpressure Handling

### What is Backpressure?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BACKPRESSURE PROBLEM                                  │
│                                                                          │
│   Fast Producer                    Slow Consumer                        │
│   ┌──────────┐                    ┌──────────┐                         │
│   │ 1000/sec │────────────────────│ 100/sec  │                         │
│   └──────────┘                    └──────────┘                         │
│                                                                          │
│   Without backpressure:                                                 │
│   - Buffer grows indefinitely                                          │
│   - OutOfMemoryError                                                   │
│   - Or items are dropped                                               │
│                                                                          │
│   With backpressure:                                                    │
│   - Consumer tells producer: "I can handle 100 items"                  │
│   - Producer slows down or buffers appropriately                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Backpressure Strategies

```java
// 1. BUFFER - Store excess items (default, can OOM)
flux.onBackpressureBuffer(1000)  // Buffer up to 1000 items

// 2. DROP - Drop newest items when buffer full
flux.onBackpressureDrop(dropped -> log.warn("Dropped: {}", dropped))

// 3. LATEST - Keep only latest item
flux.onBackpressureLatest()

// 4. ERROR - Error when can't keep up
flux.onBackpressureError()

// Example: Rate limiting
Flux.interval(Duration.ofMillis(1))  // Fast producer
    .onBackpressureDrop()
    .publishOn(Schedulers.single())
    .doOnNext(n -> {
        try {
            Thread.sleep(100);  // Slow consumer
        } catch (InterruptedException e) {}
    })
    .subscribe();
```

### Controlling Request Rate

```java
// Limit rate of requests
flux.limitRate(100)  // Request 100 at a time

// Custom request control
flux.subscribe(new BaseSubscriber<Item>() {
    @Override
    protected void hookOnSubscribe(Subscription subscription) {
        request(10);  // Initial request
    }
    
    @Override
    protected void hookOnNext(Item item) {
        process(item);
        request(1);  // Request one more after processing
    }
});
```

---

## 5️⃣ Error Handling

### Error Handling Operators

```java
// Return fallback value on error
Mono<String> withFallback = mono
    .onErrorReturn("default");

// Return fallback value for specific exception
Mono<String> withSpecificFallback = mono
    .onErrorReturn(TimeoutException.class, "timeout occurred");

// Resume with another publisher
Mono<String> withResume = mono
    .onErrorResume(e -> {
        log.error("Error occurred", e);
        return Mono.just("recovered");
    });

// Resume with specific exception handling
Mono<String> withSpecificResume = mono
    .onErrorResume(NotFoundException.class, e -> Mono.empty())
    .onErrorResume(TimeoutException.class, e -> fetchFromCache());

// Transform error
Mono<String> withMappedError = mono
    .onErrorMap(e -> new CustomException("Wrapped", e));

// Retry on error
Mono<String> withRetry = mono
    .retry(3);  // Retry up to 3 times

// Retry with backoff
Mono<String> withRetryBackoff = mono
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
        .maxBackoff(Duration.ofSeconds(10))
        .filter(e -> e instanceof TransientException));

// Do something on error (side effect)
Mono<String> withErrorLog = mono
    .doOnError(e -> log.error("Error: {}", e.getMessage()));
```

### Global Error Handling in WebFlux

```java
@ControllerAdvice
public class GlobalErrorHandler {
    
    @ExceptionHandler(NotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleNotFound(NotFoundException e) {
        return Mono.just(ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(e.getMessage())));
    }
    
    @ExceptionHandler(Exception.class)
    public Mono<ResponseEntity<ErrorResponse>> handleGeneral(Exception e) {
        log.error("Unexpected error", e);
        return Mono.just(ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("Internal server error")));
    }
}
```

---

## 6️⃣ Schedulers and Threading

### Available Schedulers

```java
// Schedulers.immediate() - Current thread
// Schedulers.single() - Single reusable thread
// Schedulers.parallel() - Fixed pool for CPU-bound work
// Schedulers.boundedElastic() - Elastic pool for blocking I/O
// Schedulers.fromExecutor() - Custom executor

// publishOn - Switch thread for downstream operators
Flux.range(1, 10)
    .map(n -> n * 2)                    // Runs on calling thread
    .publishOn(Schedulers.parallel())
    .map(n -> n + 1)                    // Runs on parallel scheduler
    .subscribe();

// subscribeOn - Switch thread for entire chain
Flux.range(1, 10)
    .map(n -> n * 2)                    // Runs on boundedElastic
    .subscribeOn(Schedulers.boundedElastic())
    .map(n -> n + 1)                    // Also runs on boundedElastic
    .subscribe();
```

### When to Use Which Scheduler

```java
// CPU-bound work
Flux.range(1, 1000)
    .publishOn(Schedulers.parallel())
    .map(this::cpuIntensiveOperation)
    .subscribe();

// Blocking I/O (database, file, legacy API)
Mono.fromCallable(() -> blockingDatabaseCall())
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();

// Don't block on parallel scheduler!
// BAD:
Flux.range(1, 10)
    .publishOn(Schedulers.parallel())
    .map(n -> {
        Thread.sleep(1000);  // Blocks parallel thread!
        return n;
    });

// GOOD:
Flux.range(1, 10)
    .publishOn(Schedulers.boundedElastic())  // Use elastic for blocking
    .map(n -> {
        Thread.sleep(1000);
        return n;
    });
```

---

## 7️⃣ Reactive vs Imperative Trade-offs

### When to Use Reactive

```
✅ High concurrency with I/O-bound operations
✅ Streaming data (real-time updates, SSE)
✅ Microservices with many external calls
✅ Need backpressure handling
✅ Non-blocking requirement

❌ Simple CRUD applications
❌ CPU-bound operations
❌ Team unfamiliar with reactive
❌ Debugging is critical (stack traces are hard)
❌ Using blocking libraries
```

### Comparison

```java
// IMPERATIVE (Traditional)
public List<Order> getOrdersForUsers(List<String> userIds) {
    return userIds.stream()
        .map(userService::getUser)           // Blocking
        .flatMap(user -> orderService.getOrders(user).stream())  // Blocking
        .collect(Collectors.toList());
}

// REACTIVE
public Flux<Order> getOrdersForUsers(List<String> userIds) {
    return Flux.fromIterable(userIds)
        .flatMap(userService::getUser)       // Non-blocking
        .flatMap(orderService::getOrders);   // Non-blocking
}

// Pros of Reactive:
// - Non-blocking, efficient resource usage
// - Built-in backpressure
// - Composable operators

// Cons of Reactive:
// - Steeper learning curve
// - Harder to debug (no stack traces)
// - All libraries must be reactive
// - More complex error handling
```

---

## 8️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between Mono and Flux?**

A: Both are reactive publishers from Project Reactor:

- **Mono<T>**: Emits 0 or 1 element, then completes. Used for single values (like Optional).
- **Flux<T>**: Emits 0 to N elements, then completes. Used for collections/streams.

```java
Mono<User> user = userRepository.findById(id);  // 0 or 1 user
Flux<User> users = userRepository.findAll();     // 0 to N users
```

**Q: What is backpressure?**

A: Backpressure is a mechanism where a slow consumer can signal to a fast producer to slow down. Without it, a fast producer could overwhelm a slow consumer, causing buffer overflow or memory issues.

In reactive streams, the subscriber requests items with `request(n)`, and the publisher only sends that many items.

### L5 (Mid-Level) Questions

**Q: When would you use reactive programming vs traditional blocking code?**

A: Use reactive when:
- High concurrency with I/O-bound operations
- Streaming real-time data
- Need backpressure handling
- Building non-blocking microservices

Use traditional when:
- Simple CRUD applications
- CPU-bound operations
- Team is unfamiliar with reactive
- Using blocking libraries that can't be replaced

The main trade-off is complexity vs scalability. Reactive code is harder to write and debug but scales better under load.

**Q: How do you handle errors in reactive streams?**

A: Several operators:
- `onErrorReturn(value)`: Return fallback value
- `onErrorResume(e -> publisher)`: Switch to another publisher
- `onErrorMap(e -> newException)`: Transform the exception
- `retry(n)`: Retry N times
- `retryWhen(Retry.backoff(...))`: Retry with backoff

Always handle errors - unhandled errors terminate the stream.

### L6 (Senior) Questions

**Q: How would you migrate a blocking Spring MVC app to WebFlux?**

A: Step-by-step approach:

1. **Identify blocking calls**: Database, HTTP clients, file I/O
2. **Replace with reactive alternatives**:
   - JDBC → R2DBC
   - RestTemplate → WebClient
   - Blocking file I/O → wrap with `Schedulers.boundedElastic()`
3. **Convert controllers**: Return `Mono`/`Flux` instead of objects
4. **Update repositories**: Use `ReactiveCrudRepository`
5. **Handle blocking code**: Wrap unavoidable blocking calls with `subscribeOn(Schedulers.boundedElastic())`
6. **Test thoroughly**: Reactive bugs are subtle

Key consideration: ALL code in the chain must be non-blocking. One blocking call defeats the purpose.

**Q: With virtual threads (Project Loom), is reactive still relevant?**

A: Both have their place:

**Virtual threads** are simpler - write blocking code, get scalability. Good for:
- Teams unfamiliar with reactive
- Existing blocking codebases
- Simpler debugging

**Reactive** still valuable for:
- Streaming data (real-time updates)
- Backpressure requirements
- Functional composition of async operations
- Existing reactive ecosystems

Virtual threads solve the scalability problem but don't provide backpressure or streaming semantics that reactive offers.

---

## 9️⃣ One Clean Mental Summary

Reactive programming is about **non-blocking, asynchronous data streams** with **backpressure**. **Mono** represents 0-1 elements, **Flux** represents 0-N elements. Operators like `map`, `flatMap`, `filter` transform streams without blocking. **Backpressure** lets slow consumers tell fast producers to slow down. **WebFlux** is Spring's reactive web framework using WebClient for HTTP and R2DBC for databases. Use **Schedulers** appropriately: `parallel()` for CPU-bound, `boundedElastic()` for blocking I/O. Error handling with `onErrorResume`, `retry`, `retryWhen`. The trade-off: reactive is more scalable but harder to write and debug. With virtual threads (Java 21+), reactive is less necessary for pure scalability, but still valuable for streaming and backpressure. Choose reactive when you need its specific features, not just for scalability.

