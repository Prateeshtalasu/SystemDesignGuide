# 🌱 Spring Framework Fundamentals

---

## 0️⃣ Prerequisites

Before diving into Spring Framework, you need to understand:

- **OOP Fundamentals**: Classes, interfaces, inheritance (covered in `01-oop-fundamentals.md`)
- **Design Patterns**: Factory, Singleton, Proxy, Template Method (covered in `09-design-patterns.md`)
- **Java Annotations**: `@Override`, `@Deprecated`, custom annotations
- **Maven/Gradle**: Dependency management basics

Quick mental model:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING FRAMEWORK OVERVIEW                             │
│                                                                          │
│   Without Spring:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  public class OrderService {                                     │   │
│   │      private Database db = new MySQLDatabase();  // Hardcoded!  │   │
│   │      private EmailService email = new SmtpEmail(); // Hardcoded!│   │
│   │      private Logger log = new FileLogger();      // Hardcoded!  │   │
│   │  }                                                               │   │
│   │                                                                  │   │
│   │  Problems:                                                       │   │
│   │  - Hard to test (can't mock dependencies)                       │   │
│   │  - Hard to change implementations                               │   │
│   │  - Tight coupling everywhere                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   With Spring:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  @Service                                                        │   │
│   │  public class OrderService {                                     │   │
│   │      private final Database db;          // Injected!           │   │
│   │      private final EmailService email;   // Injected!           │   │
│   │      private final Logger log;           // Injected!           │   │
│   │                                                                  │   │
│   │      public OrderService(Database db, EmailService email,       │   │
│   │                          Logger log) {                          │   │
│   │          this.db = db;                                          │   │
│   │          this.email = email;                                    │   │
│   │          this.log = log;                                        │   │
│   │      }                                                          │   │
│   │  }                                                               │   │
│   │                                                                  │   │
│   │  Benefits:                                                       │   │
│   │  - Easy to test (inject mocks)                                  │   │
│   │  - Easy to swap implementations                                 │   │
│   │  - Loose coupling                                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Building enterprise applications involves:

```java
// Without a framework, you manage everything manually:

public class Application {
    public static void main(String[] args) {
        // 1. Create database connection
        DataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        
        // 2. Create repositories
        UserRepository userRepo = new UserRepositoryImpl(dataSource);
        OrderRepository orderRepo = new OrderRepositoryImpl(dataSource);
        
        // 3. Create services
        EmailService emailService = new SmtpEmailService();
        UserService userService = new UserService(userRepo, emailService);
        OrderService orderService = new OrderService(orderRepo, userService);
        
        // 4. Create controllers
        UserController userController = new UserController(userService);
        OrderController orderController = new OrderController(orderService);
        
        // 5. Set up HTTP server
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
        server.createContext("/users", userController);
        server.createContext("/orders", orderController);
        
        // 6. Handle transactions manually
        // 7. Handle security manually
        // 8. Handle configuration manually
        // ... hundreds of lines of boilerplate
    }
}
```

**Problems**:

1. **Boilerplate**: Tons of wiring code
2. **Coupling**: Components know how to create their dependencies
3. **Testing**: Can't easily swap implementations for testing
4. **Configuration**: Hardcoded values everywhere
5. **Cross-cutting concerns**: Logging, security, transactions scattered throughout

### What Spring Provides

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING ECOSYSTEM                                      │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     SPRING BOOT                                  │   │
│   │   Auto-configuration, embedded servers, production-ready        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│   ┌──────────┬──────────┬────┴─────┬──────────┬──────────┐             │
│   │ Spring   │ Spring   │ Spring   │ Spring   │ Spring   │             │
│   │ MVC      │ Data     │ Security │ Cloud    │ Batch    │             │
│   │          │          │          │          │          │             │
│   │ Web      │ Database │ Auth     │ Micro-   │ Batch    │             │
│   │ Layer    │ Access   │ & AuthZ  │ services │ Jobs     │             │
│   └──────────┴──────────┴──────────┴──────────┴──────────┘             │
│                              │                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    SPRING FRAMEWORK CORE                         │   │
│   │   IoC Container, DI, AOP, Events, Resources, i18n               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2️⃣ Inversion of Control (IoC) and Dependency Injection (DI)

### Core Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IoC vs DI                                             │
│                                                                          │
│   INVERSION OF CONTROL (IoC):                                           │
│   - A design principle                                                  │
│   - "Don't call us, we'll call you"                                    │
│   - Framework controls the flow, not your code                         │
│                                                                          │
│   DEPENDENCY INJECTION (DI):                                            │
│   - A pattern that implements IoC                                       │
│   - Dependencies are "injected" from outside                           │
│   - Object doesn't create its own dependencies                         │
│                                                                          │
│   Traditional:                      IoC/DI:                             │
│   ┌─────────────────┐              ┌─────────────────┐                 │
│   │   OrderService  │              │   OrderService  │                 │
│   │  ─────────────  │              │  ─────────────  │                 │
│   │  db = new DB()  │              │  db (injected)  │                 │
│   │  Creates its    │              │  Receives its   │                 │
│   │  dependencies   │              │  dependencies   │                 │
│   └─────────────────┘              └─────────────────┘                 │
│                                            ▲                            │
│                                            │ Injected by                │
│                                    ┌───────┴───────┐                   │
│                                    │ IoC Container │                   │
│                                    │ (Spring)      │                   │
│                                    └───────────────┘                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Types of Dependency Injection

```java
// 1. CONSTRUCTOR INJECTION (Recommended)
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    // Dependencies injected via constructor
    // @Autowired is optional for single constructor (Spring 4.3+)
    public OrderService(OrderRepository orderRepository, 
                        PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
}

// 2. SETTER INJECTION
@Service
public class OrderService {
    private OrderRepository orderRepository;
    
    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}

// 3. FIELD INJECTION (Not recommended)
@Service
public class OrderService {
    @Autowired  // Works but makes testing harder
    private OrderRepository orderRepository;
}
```

**Why Constructor Injection is Best**:

| Aspect           | Constructor          | Setter             | Field              |
| ---------------- | -------------------- | ------------------ | ------------------ |
| Immutability     | ✅ Can use `final`   | ❌ Mutable         | ❌ Mutable         |
| Required deps    | ✅ Enforced          | ❌ Optional        | ❌ Optional        |
| Testing          | ✅ Easy to mock      | ⚠️ Need setters    | ❌ Need reflection |
| Circular deps    | ✅ Fails fast        | ⚠️ Hidden          | ⚠️ Hidden          |

---

## 3️⃣ Spring Bean Lifecycle

### Complete Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING BEAN LIFECYCLE                                 │
│                                                                          │
│   1. INSTANTIATION                                                      │
│      └── Bean instance created                                          │
│                                                                          │
│   2. POPULATE PROPERTIES                                                │
│      └── Dependencies injected                                          │
│                                                                          │
│   3. BeanNameAware.setBeanName()                                        │
│      └── Bean receives its name                                         │
│                                                                          │
│   4. BeanFactoryAware.setBeanFactory()                                  │
│      └── Bean receives reference to factory                             │
│                                                                          │
│   5. ApplicationContextAware.setApplicationContext()                    │
│      └── Bean receives application context                              │
│                                                                          │
│   6. BeanPostProcessor.postProcessBeforeInitialization()                │
│      └── Pre-initialization processing                                  │
│                                                                          │
│   7. @PostConstruct                                                     │
│      └── Custom initialization method                                   │
│                                                                          │
│   8. InitializingBean.afterPropertiesSet()                              │
│      └── Interface-based initialization                                 │
│                                                                          │
│   9. Custom init-method                                                 │
│      └── XML/annotation configured init                                 │
│                                                                          │
│   10. BeanPostProcessor.postProcessAfterInitialization()                │
│       └── Post-initialization processing (AOP proxies created here)    │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│                        BEAN IS READY FOR USE                            │
│   ═══════════════════════════════════════════════════════════════════   │
│                                                                          │
│   11. @PreDestroy                                                       │
│       └── Custom cleanup method                                         │
│                                                                          │
│   12. DisposableBean.destroy()                                          │
│       └── Interface-based cleanup                                       │
│                                                                          │
│   13. Custom destroy-method                                             │
│       └── XML/annotation configured cleanup                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Lifecycle Example

```java
@Component
public class DatabaseConnection implements InitializingBean, DisposableBean {
    
    private Connection connection;
    
    // Constructor - Step 1
    public DatabaseConnection() {
        System.out.println("1. Constructor called");
    }
    
    // Setter injection - Step 2
    @Autowired
    public void setDataSource(DataSource dataSource) {
        System.out.println("2. Dependencies injected");
    }
    
    // @PostConstruct - Step 7
    @PostConstruct
    public void postConstruct() {
        System.out.println("7. @PostConstruct - Opening connection");
        // Initialize resources
    }
    
    // InitializingBean - Step 8
    @Override
    public void afterPropertiesSet() {
        System.out.println("8. afterPropertiesSet - Additional setup");
    }
    
    // @PreDestroy - Step 11
    @PreDestroy
    public void preDestroy() {
        System.out.println("11. @PreDestroy - Closing connection");
        // Cleanup resources
    }
    
    // DisposableBean - Step 12
    @Override
    public void destroy() {
        System.out.println("12. destroy - Final cleanup");
    }
}
```

---

## 4️⃣ Bean Scopes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BEAN SCOPES                                           │
│                                                                          │
│   SINGLETON (Default)                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  One instance per Spring container                              │   │
│   │                                                                  │   │
│   │  Request 1 ──┐                                                  │   │
│   │  Request 2 ──┼──► Same Bean Instance                           │   │
│   │  Request 3 ──┘                                                  │   │
│   │                                                                  │   │
│   │  Use for: Stateless services, repositories                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   PROTOTYPE                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  New instance every time bean is requested                      │   │
│   │                                                                  │   │
│   │  Request 1 ──► Instance 1                                       │   │
│   │  Request 2 ──► Instance 2                                       │   │
│   │  Request 3 ──► Instance 3                                       │   │
│   │                                                                  │   │
│   │  Use for: Stateful beans, builders                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   REQUEST (Web only)                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  One instance per HTTP request                                  │   │
│   │                                                                  │   │
│   │  HTTP Request 1 ──► Instance 1                                  │   │
│   │  HTTP Request 2 ──► Instance 2                                  │   │
│   │                                                                  │   │
│   │  Use for: Request-scoped data (user context)                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   SESSION (Web only)                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  One instance per HTTP session                                  │   │
│   │                                                                  │   │
│   │  Session A (multiple requests) ──► Instance 1                   │   │
│   │  Session B (multiple requests) ──► Instance 2                   │   │
│   │                                                                  │   │
│   │  Use for: Shopping cart, user preferences                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scope Examples

```java
// Singleton (default) - one instance shared
@Service
@Scope("singleton")  // Optional, this is default
public class OrderService {
    // Shared by all requests
}

// Prototype - new instance each time
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    // Each injection gets a new cart
}

// Request scope - one per HTTP request
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String userId;
    private Instant requestTime;
    // Fresh for each HTTP request
}

// Session scope - one per HTTP session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {
    private User currentUser;
    private List<String> recentlyViewed;
    // Persists across requests in same session
}
```

### Scope Gotcha: Injecting Prototype into Singleton

```java
// PROBLEM: Prototype bean injected into singleton
@Service
public class OrderService {
    
    @Autowired
    private ShoppingCart cart;  // Injected ONCE at startup!
    
    public void addToCart(Item item) {
        cart.add(item);  // Same cart for all users! BUG!
    }
}

// SOLUTION 1: Provider
@Service
public class OrderService {
    
    @Autowired
    private Provider<ShoppingCart> cartProvider;
    
    public void addToCart(Item item) {
        ShoppingCart cart = cartProvider.get();  // New cart each time
        cart.add(item);
    }
}

// SOLUTION 2: ObjectFactory
@Service
public class OrderService {
    
    @Autowired
    private ObjectFactory<ShoppingCart> cartFactory;
    
    public void addToCart(Item item) {
        ShoppingCart cart = cartFactory.getObject();
        cart.add(item);
    }
}

// SOLUTION 3: Scoped proxy
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    // Proxy handles creating new instances
}
```

---

## 5️⃣ Aspect-Oriented Programming (AOP)

### The Problem AOP Solves

```java
// WITHOUT AOP: Cross-cutting concerns scattered everywhere
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Logging
        log.info("Creating order: {}", request);
        
        // Security check
        if (!securityContext.hasPermission("CREATE_ORDER")) {
            throw new AccessDeniedException();
        }
        
        // Start transaction
        Transaction tx = transactionManager.begin();
        
        try {
            // Actual business logic (what we care about)
            Order order = new Order(request);
            orderRepository.save(order);
            
            // Commit transaction
            tx.commit();
            
            // Logging
            log.info("Order created: {}", order.getId());
            
            return order;
        } catch (Exception e) {
            // Rollback transaction
            tx.rollback();
            
            // Logging
            log.error("Failed to create order", e);
            
            throw e;
        }
    }
}

// WITH AOP: Clean business logic
@Service
public class OrderService {
    
    @Transactional
    @PreAuthorize("hasPermission('CREATE_ORDER')")
    @Logged
    public Order createOrder(OrderRequest request) {
        // Just business logic!
        Order order = new Order(request);
        orderRepository.save(order);
        return order;
    }
}
```

### AOP Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AOP TERMINOLOGY                                       │
│                                                                          │
│   ASPECT: A module that encapsulates cross-cutting concerns             │
│           (logging, security, transactions)                             │
│                                                                          │
│   JOIN POINT: A point in execution (method call, exception)             │
│               where aspect can be applied                               │
│                                                                          │
│   ADVICE: Action taken at a join point                                  │
│           - Before: Run before method                                   │
│           - After: Run after method (regardless of outcome)             │
│           - AfterReturning: Run after successful return                 │
│           - AfterThrowing: Run after exception                          │
│           - Around: Wrap method execution                               │
│                                                                          │
│   POINTCUT: Expression that selects join points                         │
│             "Apply this advice to these methods"                        │
│                                                                          │
│   TARGET: The object being advised                                      │
│                                                                          │
│   PROXY: Object created by AOP to implement advice                      │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   Client ──► Proxy ──► Target Object                            │   │
│   │                │                                                 │   │
│   │                ▼                                                 │   │
│   │           ┌─────────┐                                           │   │
│   │           │ Advice  │ (Before, After, Around)                   │   │
│   │           └─────────┘                                           │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### AOP Implementation

```java
// Custom annotation for logging
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Logged {
    String value() default "";
}

// Aspect implementation
@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    // Pointcut: All methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // Pointcut: Methods annotated with @Logged
    @Pointcut("@annotation(logged)")
    public void loggedMethods(Logged logged) {}
    
    // Before advice
    @Before("serviceMethods()")
    public void logBefore(JoinPoint joinPoint) {
        log.info("Entering: {}.{}()", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName());
    }
    
    // After returning advice
    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("Exiting: {}.{}() with result: {}", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            result);
    }
    
    // After throwing advice
    @AfterThrowing(pointcut = "serviceMethods()", throwing = "ex")
    public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
        log.error("Exception in {}.{}(): {}", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            ex.getMessage());
    }
    
    // Around advice (most powerful)
    @Around("loggedMethods(logged)")
    public Object logAround(ProceedingJoinPoint joinPoint, Logged logged) throws Throwable {
        long start = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().getName();
        
        log.info("[{}] Starting: {}", logged.value(), methodName);
        
        try {
            Object result = joinPoint.proceed();  // Execute target method
            
            long duration = System.currentTimeMillis() - start;
            log.info("[{}] Completed: {} in {}ms", logged.value(), methodName, duration);
            
            return result;
        } catch (Exception e) {
            log.error("[{}] Failed: {} - {}", logged.value(), methodName, e.getMessage());
            throw e;
        }
    }
}

// Usage
@Service
public class OrderService {
    
    @Logged("ORDER")
    public Order createOrder(OrderRequest request) {
        // Business logic
        return new Order(request);
    }
}
```

### Common Pointcut Expressions

```java
// All methods in a package
@Pointcut("execution(* com.example.service.*.*(..))")

// All public methods
@Pointcut("execution(public * *(..))")

// Methods returning specific type
@Pointcut("execution(Order com.example..*.*(..))")

// Methods with specific parameter
@Pointcut("execution(* *..*(String, ..))")

// Methods annotated with @Transactional
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")

// All methods in classes annotated with @Service
@Pointcut("@within(org.springframework.stereotype.Service)")

// Combine pointcuts
@Pointcut("serviceMethods() && loggedMethods()")
```

---

## 6️⃣ Spring Boot Auto-Configuration

### How Auto-Configuration Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING BOOT AUTO-CONFIGURATION                        │
│                                                                          │
│   @SpringBootApplication                                                │
│        │                                                                 │
│        ├── @SpringBootConfiguration (same as @Configuration)            │
│        │                                                                 │
│        ├── @EnableAutoConfiguration                                     │
│        │        │                                                        │
│        │        └── Loads META-INF/spring.factories                     │
│        │            (or META-INF/spring/org.springframework.boot.       │
│        │             autoconfigure.AutoConfiguration.imports)           │
│        │                                                                 │
│        └── @ComponentScan                                               │
│                 │                                                        │
│                 └── Scans for @Component, @Service, @Repository, etc.  │
│                                                                          │
│   AUTO-CONFIGURATION PROCESS:                                           │
│   ═══════════════════════════                                           │
│                                                                          │
│   1. Spring Boot starts                                                 │
│   2. Reads auto-configuration classes from spring.factories            │
│   3. For each class, checks @Conditional annotations                   │
│   4. If conditions met, configuration is applied                       │
│                                                                          │
│   Example: DataSourceAutoConfiguration                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  @ConditionalOnClass(DataSource.class)                          │   │
│   │  // Only if DataSource is on classpath                          │   │
│   │                                                                  │   │
│   │  @ConditionalOnMissingBean(DataSource.class)                    │   │
│   │  // Only if user hasn't defined their own DataSource            │   │
│   │                                                                  │   │
│   │  @EnableConfigurationProperties(DataSourceProperties.class)     │   │
│   │  // Bind application.properties to DataSourceProperties         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Common Conditional Annotations

```java
// Bean created only if class is on classpath
@ConditionalOnClass(DataSource.class)

// Bean created only if class is NOT on classpath
@ConditionalOnMissingClass("com.example.SomeClass")

// Bean created only if another bean exists
@ConditionalOnBean(DataSource.class)

// Bean created only if another bean does NOT exist
@ConditionalOnMissingBean(DataSource.class)

// Bean created only if property has specific value
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")

// Bean created only in web application
@ConditionalOnWebApplication

// Bean created only if expression is true
@ConditionalOnExpression("${feature.enabled:false} and ${another.flag:true}")
```

### Custom Auto-Configuration

```java
// Custom auto-configuration class
@Configuration
@ConditionalOnClass(NotificationService.class)
@EnableConfigurationProperties(NotificationProperties.class)
public class NotificationAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "notification.enabled", havingValue = "true", matchIfMissing = true)
    public NotificationService notificationService(NotificationProperties properties) {
        return new DefaultNotificationService(properties);
    }
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "notification.type", havingValue = "email")
    public NotificationSender emailSender() {
        return new EmailNotificationSender();
    }
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "notification.type", havingValue = "sms")
    public NotificationSender smsSender() {
        return new SmsNotificationSender();
    }
}

// Configuration properties
@ConfigurationProperties(prefix = "notification")
public class NotificationProperties {
    private boolean enabled = true;
    private String type = "email";
    private String from;
    
    // Getters and setters
}

// application.yml
// notification:
//   enabled: true
//   type: email
//   from: noreply@example.com
```

---

## 7️⃣ Spring MVC Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING MVC REQUEST FLOW                               │
│                                                                          │
│   HTTP Request                                                          │
│        │                                                                 │
│        ▼                                                                 │
│   ┌─────────────────┐                                                   │
│   │DispatcherServlet│  Front Controller - receives all requests        │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │ HandlerMapping  │  Finds which controller handles the request      │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │ HandlerAdapter  │  Invokes the controller method                   │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │   Controller    │  Processes request, returns ModelAndView         │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │  ViewResolver   │  Resolves view name to actual view               │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │      View       │  Renders response (JSON, HTML, etc.)             │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   HTTP Response                                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### REST Controller Example

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    
    private final OrderService orderService;
    
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    // GET /api/v1/orders
    @GetMapping
    public ResponseEntity<List<OrderDTO>> getAllOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        List<OrderDTO> orders = orderService.findAll(page, size);
        return ResponseEntity.ok(orders);
    }
    
    // GET /api/v1/orders/{id}
    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // POST /api/v1/orders
    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        OrderDTO created = orderService.create(request);
        URI location = URI.create("/api/v1/orders/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }
    
    // PUT /api/v1/orders/{id}
    @PutMapping("/{id}")
    public ResponseEntity<OrderDTO> updateOrder(
            @PathVariable Long id,
            @Valid @RequestBody UpdateOrderRequest request) {
        
        return orderService.update(id, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // DELETE /api/v1/orders/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 8️⃣ @Component vs @Bean

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    @Component vs @Bean                                   │
│                                                                          │
│   @Component (and @Service, @Repository, @Controller)                   │
│   ─────────────────────────────────────────────────────                 │
│   - Used on CLASSES you write                                           │
│   - Detected via component scanning                                     │
│   - Automatic bean registration                                         │
│                                                                          │
│   @Service                                                              │
│   public class OrderService {                                           │
│       // Spring creates and manages this bean                           │
│   }                                                                      │
│                                                                          │
│   @Bean                                                                 │
│   ─────                                                                 │
│   - Used on METHODS in @Configuration classes                          │
│   - For third-party classes you can't annotate                         │
│   - More control over instantiation                                    │
│                                                                          │
│   @Configuration                                                        │
│   public class AppConfig {                                              │
│                                                                          │
│       @Bean                                                             │
│       public RestTemplate restTemplate() {                              │
│           // Can't put @Component on RestTemplate (third-party)        │
│           return new RestTemplate();                                    │
│       }                                                                  │
│                                                                          │
│       @Bean                                                             │
│       public ObjectMapper objectMapper() {                              │
│           ObjectMapper mapper = new ObjectMapper();                     │
│           mapper.configure(...);  // Custom configuration              │
│           return mapper;                                                │
│       }                                                                  │
│   }                                                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### When to Use Which

| Scenario                                | Use          |
| --------------------------------------- | ------------ |
| Your own service/repository class       | @Component   |
| Third-party library class               | @Bean        |
| Need custom instantiation logic         | @Bean        |
| Simple POJO with no dependencies        | @Component   |
| Multiple beans of same type             | @Bean        |
| Conditional bean creation               | @Bean        |

---

## 9️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is Dependency Injection and why is it useful?**

A: Dependency Injection is a design pattern where an object's dependencies are provided ("injected") from outside rather than created internally.

Benefits:
1. **Loose coupling**: Classes don't know how to create their dependencies
2. **Testability**: Easy to inject mocks for testing
3. **Flexibility**: Can swap implementations without changing code
4. **Maintainability**: Dependencies are explicit in constructor

Example: Instead of `new MySQLDatabase()` inside a service, the database is passed in via constructor.

**Q: What is the difference between @Component and @Service?**

A: Functionally, they're identical - both register a bean with Spring. The difference is semantic:

- `@Component`: Generic stereotype for any Spring-managed component
- `@Service`: Indicates a service layer class (business logic)
- `@Repository`: Indicates a data access layer class (adds exception translation)
- `@Controller`: Indicates a web controller

Using the right annotation makes code more readable and enables layer-specific features (like `@Repository`'s exception translation).

### L5 (Mid-Level) Questions

**Q: Explain the Spring Bean lifecycle.**

A: The lifecycle has several phases:

1. **Instantiation**: Bean instance created
2. **Property population**: Dependencies injected
3. **Aware interfaces**: `BeanNameAware`, `ApplicationContextAware` called
4. **BeanPostProcessor.postProcessBeforeInitialization**: Pre-init processing
5. **@PostConstruct**: Custom initialization
6. **InitializingBean.afterPropertiesSet**: Interface-based init
7. **Custom init-method**: Configured init method
8. **BeanPostProcessor.postProcessAfterInitialization**: AOP proxies created

For destruction:
9. **@PreDestroy**: Custom cleanup
10. **DisposableBean.destroy**: Interface-based cleanup
11. **Custom destroy-method**: Configured cleanup

**Q: What is AOP and when would you use it?**

A: AOP (Aspect-Oriented Programming) separates cross-cutting concerns from business logic.

Cross-cutting concerns appear across multiple classes:
- Logging
- Security
- Transactions
- Caching
- Error handling

Instead of duplicating this code everywhere, you define it once in an Aspect and apply it declaratively. Spring uses proxies to intercept method calls and apply advice.

Example: `@Transactional` is implemented via AOP - Spring wraps your method with transaction begin/commit/rollback logic.

### L6 (Senior) Questions

**Q: How does Spring Boot auto-configuration work?**

A: Auto-configuration uses several mechanisms:

1. **@EnableAutoConfiguration** triggers the process
2. Spring Boot reads `META-INF/spring.factories` or `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. Each auto-configuration class has `@Conditional` annotations
4. Conditions are evaluated: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
5. If conditions pass, the configuration is applied

This is why adding `spring-boot-starter-data-jpa` to your dependencies automatically configures a DataSource, EntityManagerFactory, and transaction manager - the auto-configuration detects JPA classes on the classpath.

You can override auto-configuration by defining your own beans (due to `@ConditionalOnMissingBean`).

**Q: How would you handle circular dependencies in Spring?**

A: Circular dependencies occur when Bean A depends on Bean B, and Bean B depends on Bean A.

Solutions:
1. **Redesign**: Usually indicates a design problem. Extract common functionality into a third bean.

2. **Setter injection**: Instead of constructor injection, use setter injection for one dependency. Spring can create both beans, then inject.

3. **@Lazy**: Mark one dependency as lazy-loaded:
   ```java
   public ServiceA(@Lazy ServiceB serviceB) { }
   ```

4. **ObjectProvider**: Inject a provider instead of the bean directly:
   ```java
   public ServiceA(ObjectProvider<ServiceB> serviceBProvider) { }
   ```

Best practice: Circular dependencies are a code smell. Redesign to eliminate them.

---

## 🔟 One Clean Mental Summary

Spring Framework's core value is **Inversion of Control** - the framework manages object creation and lifecycle, not your code. **Dependency Injection** implements IoC by providing dependencies from outside (prefer constructor injection for immutability and testability). Beans are Spring-managed objects with configurable **scopes** (singleton, prototype, request, session). The **bean lifecycle** includes initialization callbacks (`@PostConstruct`) and destruction callbacks (`@PreDestroy`). **AOP** handles cross-cutting concerns (logging, transactions, security) by intercepting method calls via proxies. **Spring Boot** adds auto-configuration that detects your classpath and configures beans automatically (DataSource if JPA is present, etc.). Use `@Component`/`@Service`/`@Repository` for your classes, `@Bean` for third-party classes. The key insight: Spring manages complexity so you can focus on business logic. Don't fight the framework - embrace dependency injection, use the right annotations, and let Spring wire everything together.

