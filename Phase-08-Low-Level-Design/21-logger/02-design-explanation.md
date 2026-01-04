# 📝 Design a Logger - Design Explanation

## SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `LogEvent` | Store log event data | Event model changes |
| `LogFormatter` | Format log messages | Format requirements change |
| `LogAppender` | Write logs to destination | Output destination changes |
| `Logger` | Coordinate logging | Logging API changes |
| `LoggerFactory` | Manage logger instances | Logger creation logic changes |

**SRP in Action:**

```java
// LogEvent ONLY stores event data
public class LogEvent {
    private final LocalDateTime timestamp;
    private final LogLevel level;
    private final String message;
}

// LogFormatter ONLY formats messages
public class SimpleFormatter implements LogFormatter {
    public String format(LogEvent event) { }
}

// LogAppender ONLY writes to destination
public class FileAppender implements LogAppender {
    public void append(LogEvent event, String message) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Formatters:**

```java
// No changes to existing code
public class XMLFormatter implements LogFormatter {
    @Override
    public String format(LogEvent event) {
        return "<log><level>" + event.getLevel() + "</level>...</log>";
    }
}
```

**Adding New Appenders:**

```java
public class DatabaseAppender implements LogAppender {
    @Override
    public void append(LogEvent event, String message) {
        // Insert into database
    }
}

public class SlackAppender implements LogAppender {
    @Override
    public void append(LogEvent event, String message) {
        // Send to Slack webhook
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All appenders work interchangeably:**

```java
public class Logger {
    private final List<LogAppender> appenders;
    
    public void log(LogLevel level, String message) {
        String formatted = formatter.format(event);
        
        // Works with any LogAppender implementation
        for (LogAppender appender : appenders) {
            appender.append(event, formatted);
        }
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

```java
public interface LogAppender {
    void append(LogEvent event, String formattedMessage);
    void close();
}
```

**Could be improved:**

```java
public interface LogAppender {
    void append(LogEvent event, String formattedMessage);
}

public interface Closeable {
    void close();
}

public interface Flushable {
    void flush();
}

public class FileAppender implements LogAppender, Closeable, Flushable { }
public class ConsoleAppender implements LogAppender { }  // No close needed
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Design (Good):**

```java
public class Logger {
    private LogFormatter formatter;  // Interface
    private List<LogAppender> appenders;  // Interface
    
    // Depends on abstractions, not concrete classes
}
```

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. LogEvent stores event data, LogFormatter formats messages, LogAppender writes logs, Logger coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new formatters, appenders) without modifying existing code. Strategy pattern enables this. | N/A | - |
| **LSP** | PASS | All LogFormatter and LogAppender implementations properly implement their interface contracts. They are substitutable. | N/A | - |
| **ISP** | PASS | Interfaces are minimal and focused. LogFormatter and LogAppender are well-segregated. No unused methods. | N/A | - |
| **DIP** | PASS | Logger depends on LogFormatter and LogAppender interfaces (abstractions), not concrete implementations. DIP is well applied. | N/A | - |

---

## Design Patterns Used

### 1. Singleton Pattern

**Where:** LoggerFactory

```java
public class LoggerFactory {
    private static final LoggerFactory INSTANCE = new LoggerFactory();
    
    private LoggerFactory() { }
    
    public static LoggerFactory getInstance() {
        return INSTANCE;
    }
    
    public static Logger getLogger(String name) {
        return INSTANCE.getOrCreateLogger(name);
    }
}
```

---

### 2. Strategy Pattern

**Where:** Formatters and Appenders

```java
public interface LogFormatter {
    String format(LogEvent event);
}

// Different strategies
public class SimpleFormatter implements LogFormatter { }
public class JSONFormatter implements LogFormatter { }
public class XMLFormatter implements LogFormatter { }

// Logger uses strategy
public class Logger {
    private LogFormatter formatter;
    
    public void setFormatter(LogFormatter formatter) {
        this.formatter = formatter;
    }
}
```

---

### 3. Observer Pattern (Potential)

**Where:** Multiple appenders

```java
public class Logger {
    private final List<LogAppender> appenders;  // Observers
    
    public void log(LogLevel level, String message) {
        // Notify all observers
        for (LogAppender appender : appenders) {
            appender.append(event, formatted);
        }
    }
}
```

---

### 4. Factory Method Pattern

**Where:** Logger creation

```java
public class LoggerFactory {
    private Logger getOrCreateLogger(String name) {
        return loggers.computeIfAbsent(name, n -> {
            Logger logger = new Logger(n);
            // Configure with defaults
            return logger;
        });
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single Logger Class with Hardcoded Output

**What it is:**
A monolithic Logger class that directly writes to System.out and files without separation of concerns.

**Why rejected:**
- Violates Single Responsibility Principle - Logger would handle formatting, output, and coordination
- Not extensible - Adding new output destinations requires modifying core Logger class
- Hard to test - Cannot mock or replace output destinations
- Tight coupling - Changes to file handling affect console output logic

**What breaks:**
- Open/Closed Principle - System is not open for extension without modification
- Testability - Cannot easily test different output scenarios
- Maintainability - All logging logic in one place becomes complex

### Alternative 2: Global Static Logger Instance

**What it is:**
A single static Logger instance shared across the entire application, accessed via static methods.

**Why rejected:**
- No logger hierarchy or namespacing - Cannot configure different loggers for different components
- Difficult to test - Global state makes unit testing harder
- No flexibility - Cannot have different log levels or appenders for different parts of the application
- Thread safety concerns - Shared mutable state requires more synchronization

**What breaks:**
- Scalability - Cannot configure logging per module or component
- Testability - Global state complicates testing
- Flexibility - One-size-fits-all approach doesn't work for complex applications

---

## Thread Safety

### CopyOnWriteArrayList for Appenders

```java
public class Logger {
    private final List<LogAppender> appenders = new CopyOnWriteArrayList<>();
    
    // Safe for concurrent iteration during logging
    // Safe for concurrent modification (add/remove appenders)
}
```

### Synchronized Appenders

```java
public class FileAppender implements LogAppender {
    @Override
    public synchronized void append(LogEvent event, String message) {
        // Only one thread writes at a time
        writer.write(message);
    }
}
```

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `log` | O(A) | A = number of appenders |
| `format` | O(M) | M = message length |
| `getLogger` | O(1) | ConcurrentHashMap lookup |

### Space Complexity

| Component | Space |
|-----------|-------|
| Logger cache | O(L) | L = unique loggers |
| LogEvent | O(M) | M = message length |
| File buffer | O(B) | B = buffer size |

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you implement async logging?

```java
public class AsyncAppender implements LogAppender {
    private final LogAppender delegate;
    private final BlockingQueue<LogTask> queue;
    private final Thread worker;
    
    public AsyncAppender(LogAppender delegate) {
        this.delegate = delegate;
        this.queue = new LinkedBlockingQueue<>(10000);
        this.worker = new Thread(this::processQueue);
        this.worker.setDaemon(true);
        this.worker.start();
    }
    
    @Override
    public void append(LogEvent event, String message) {
        queue.offer(new LogTask(event, message));
    }
    
    private void processQueue() {
        while (true) {
            try {
                LogTask task = queue.take();
                delegate.append(task.event, task.message);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```

### Q2: How would you implement log filtering?

```java
public interface LogFilter {
    boolean accept(LogEvent event);
}

public class LevelFilter implements LogFilter {
    private final LogLevel minLevel;
    
    @Override
    public boolean accept(LogEvent event) {
        return event.getLevel().isEnabled(minLevel);
    }
}

public class Logger {
    private final List<LogFilter> filters;
    
    public void log(LogLevel level, String message) {
        LogEvent event = new LogEvent(level, name, message, null);
        
        for (LogFilter filter : filters) {
            if (!filter.accept(event)) {
                return;  // Filtered out
            }
        }
        
        // Continue with logging
    }
}
```

### Q3: How would you implement MDC (Mapped Diagnostic Context)?

```java
public class MDC {
    private static final ThreadLocal<Map<String, String>> context = 
        ThreadLocal.withInitial(HashMap::new);
    
    public static void put(String key, String value) {
        context.get().put(key, value);
    }
    
    public static String get(String key) {
        return context.get().get(key);
    }
    
    public static void clear() {
        context.get().clear();
    }
    
    public static Map<String, String> getCopyOfContext() {
        return new HashMap<>(context.get());
    }
}

// Usage
MDC.put("userId", "12345");
MDC.put("requestId", "req-abc");
logger.info("Processing request");  // Includes MDC in output
```

### Q4: How would you implement log aggregation?

```java
public class LogAggregator {
    private final Map<String, AtomicInteger> errorCounts;
    private final ScheduledExecutorService scheduler;
    
    public void recordError(String errorType) {
        errorCounts.computeIfAbsent(errorType, k -> new AtomicInteger())
            .incrementAndGet();
    }
    
    private void reportAggregates() {
        for (Map.Entry<String, AtomicInteger> entry : errorCounts.entrySet()) {
            int count = entry.getValue().getAndSet(0);
            if (count > 0) {
                logger.info("Error {} occurred {} times in last minute",
                    entry.getKey(), count);
            }
        }
    }
}
```

### Q5: How would you implement structured logging?

```java
public class StructuredLogger {
    private final Logger logger;
    
    public LogBuilder info() {
        return new LogBuilder(LogLevel.INFO);
    }
    
    public class LogBuilder {
        private final LogLevel level;
        private final Map<String, Object> fields = new LinkedHashMap<>();
        
        public LogBuilder(LogLevel level) {
            this.level = level;
        }
        
        public LogBuilder field(String key, Object value) {
            fields.put(key, value);
            return this;
        }
        
        public void msg(String message) {
            String structuredMessage = message + " " + formatFields();
            logger.log(level, structuredMessage);
        }
        
        private String formatFields() {
            return fields.entrySet().stream()
                .map(e -> e.getKey() + "=" + e.getValue())
                .collect(Collectors.joining(", ", "[", "]"));
        }
    }
}

// Usage
structuredLogger.info()
    .field("userId", 123)
    .field("action", "login")
    .field("duration", 45)
    .msg("User action completed");
```

### Q6: How would you implement log sampling?

```java
public class SamplingAppender implements LogAppender {
    private final LogAppender delegate;
    private final double sampleRate;
    private final Random random;
    
    public SamplingAppender(LogAppender delegate, double sampleRate) {
        this.delegate = delegate;
        this.sampleRate = sampleRate;
        this.random = new Random();
    }
    
    @Override
    public void append(LogEvent event, String message) {
        // Always log errors
        if (event.getLevel() == LogLevel.ERROR) {
            delegate.append(event, message);
            return;
        }
        
        // Sample other levels
        if (random.nextDouble() < sampleRate) {
            delegate.append(event, message);
        }
    }
    
    @Override
    public void close() {
        delegate.close();
    }
}
```

### Q7: How would you handle log rotation in a distributed system?

```java
public class DistributedFileAppender implements LogAppender {
    private final String serviceName;
    private final String instanceId;
    private final FileAppender localAppender;
    private final RemoteLogService remoteService;
    
    public DistributedFileAppender(String serviceName, String instanceId) {
        this.serviceName = serviceName;
        this.instanceId = instanceId;
        this.localAppender = new FileAppender("logs/" + serviceName + "-" + instanceId + ".log");
        this.remoteService = new RemoteLogService();
    }
    
    @Override
    public void append(LogEvent event, String message) {
        // Write locally first
        localAppender.append(event, message);
        
        // Send critical logs to remote service
        if (event.getLevel() == LogLevel.ERROR) {
            remoteService.sendLog(serviceName, instanceId, event, message);
        }
    }
    
    @Override
    public void close() {
        localAppender.close();
        remoteService.flush();
    }
}
```

### Q8: How would you implement log buffering for performance?

```java
public class BufferedAppender implements LogAppender {
    private final LogAppender delegate;
    private final List<String> buffer;
    private final int bufferSize;
    private final long flushIntervalMs;
    private final ScheduledExecutorService scheduler;
    
    public BufferedAppender(LogAppender delegate, int bufferSize, long flushIntervalMs) {
        this.delegate = delegate;
        this.buffer = new ArrayList<>();
        this.bufferSize = bufferSize;
        this.flushIntervalMs = flushIntervalMs;
        this.scheduler = Executors.newScheduledThreadPool(1);
        
        scheduler.scheduleAtFixedRate(this::flush, flushIntervalMs, flushIntervalMs, TimeUnit.MILLISECONDS);
    }
    
    @Override
    public synchronized void append(LogEvent event, String message) {
        buffer.add(message);
        if (buffer.size() >= bufferSize) {
            flush();
        }
    }
    
    private synchronized void flush() {
        for (String msg : buffer) {
            delegate.append(null, msg);
        }
        buffer.clear();
    }
    
    @Override
    public void close() {
        flush();
        scheduler.shutdown();
        delegate.close();
    }
}
```

### Q9: How would you implement log correlation across services?

```java
public class CorrelationContext {
    private static final ThreadLocal<String> correlationId = new ThreadLocal<>();
    
    public static void setCorrelationId(String id) {
        correlationId.set(id);
    }
    
    public static String getCorrelationId() {
        return correlationId.get();
    }
    
    public static void clear() {
        correlationId.remove();
    }
}

public class CorrelationFormatter implements LogFormatter {
    private final LogFormatter delegate;
    
    public CorrelationFormatter(LogFormatter delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public String format(LogEvent event) {
        String baseMessage = delegate.format(event);
        String correlationId = CorrelationContext.getCorrelationId();
        
        if (correlationId != null) {
            return baseMessage + " [correlationId=" + correlationId + "]";
        }
        return baseMessage;
    }
}

// Usage
CorrelationContext.setCorrelationId("req-12345");
logger.info("Processing request");  // Includes correlation ID
```

### Q10: How would you handle log compression for archived logs?

```java
public class CompressedFileAppender implements LogAppender {
    private final String basePath;
    private final long maxFileSize;
    private BufferedWriter writer;
    private long currentSize;
    
    @Override
    public synchronized void append(LogEvent event, String message) {
        try {
            if (currentSize >= maxFileSize) {
                compressAndRotate();
            }
            
            writer.write(message);
            writer.newLine();
            writer.flush();
            currentSize += message.length() + 1;
            
        } catch (IOException e) {
            System.err.println("Failed to write log: " + e.getMessage());
        }
    }
    
    private void compressAndRotate() throws IOException {
        writer.close();
        
        String timestamp = String.valueOf(System.currentTimeMillis());
        String currentFile = basePath;
        String compressedFile = basePath + "." + timestamp + ".gz";
        
        // Compress file using GZIP
        try (FileInputStream fis = new FileInputStream(currentFile);
             FileOutputStream fos = new FileOutputStream(compressedFile);
             GZIPOutputStream gzos = new GZIPOutputStream(fos)) {
            
            byte[] buffer = new byte[8192];
            int len;
            while ((len = fis.read(buffer)) > 0) {
                gzos.write(buffer, 0, len);
            }
        }
        
        // Delete original and open new file
        Files.delete(Paths.get(currentFile));
        openNewFile();
    }
    
    private void openNewFile() throws IOException {
        writer = new BufferedWriter(new FileWriter(basePath, true));
        currentSize = 0;
    }
    
    @Override
    public void close() {
        try {
            if (writer != null) {
                writer.close();
            }
        } catch (IOException e) {
            System.err.println("Failed to close log file: " + e.getMessage());
        }
    }
}
```

### Q11: How would you implement log level inheritance in a logger hierarchy?

```java
public class HierarchicalLogger extends Logger {
    private final String name;
    private final HierarchicalLogger parent;
    private LogLevel effectiveLevel;
    
    public HierarchicalLogger(String name, HierarchicalLogger parent) {
        super(name);
        this.name = name;
        this.parent = parent;
    }
    
    @Override
    public LogLevel getLevel() {
        if (this.level != null) {
            return this.level;
        }
        // Inherit from parent
        return parent != null ? parent.getEffectiveLevel() : LogLevel.INFO;
    }
    
    public LogLevel getEffectiveLevel() {
        if (effectiveLevel != null) {
            return effectiveLevel;
        }
        effectiveLevel = getLevel();
        return effectiveLevel;
    }
    
    @Override
    public void log(LogLevel logLevel, String message, Throwable throwable) {
        if (!logLevel.isEnabled(getEffectiveLevel())) {
            return;
        }
        super.log(logLevel, message, throwable);
    }
}

// Usage
HierarchicalLogger rootLogger = new HierarchicalLogger("root", null);
rootLogger.setLevel(LogLevel.INFO);

HierarchicalLogger appLogger = new HierarchicalLogger("com.app", rootLogger);
// appLogger inherits INFO level from root

HierarchicalLogger serviceLogger = new HierarchicalLogger("com.app.Service", appLogger);
serviceLogger.setLevel(LogLevel.DEBUG);  // Override for this logger
```

### Q12: How would you implement log rate limiting to prevent log flooding?

```java
public class RateLimitedAppender implements LogAppender {
    private final LogAppender delegate;
    private final Map<String, RateLimiter> limiters;
    private final int maxLogsPerSecond;
    
    public RateLimitedAppender(LogAppender delegate, int maxLogsPerSecond) {
        this.delegate = delegate;
        this.limiters = new ConcurrentHashMap<>();
        this.maxLogsPerSecond = maxLogsPerSecond;
    }
    
    @Override
    public void append(LogEvent event, String message) {
        // Always allow errors through
        if (event.getLevel() == LogLevel.ERROR) {
            delegate.append(event, message);
            return;
        }
        
        String key = event.getLevel().toString();
        RateLimiter limiter = limiters.computeIfAbsent(key, 
            k -> RateLimiter.create(maxLogsPerSecond));
        
        if (limiter.tryAcquire()) {
            delegate.append(event, message);
        } else {
            // Log rate limit exceeded - could track this separately
        }
    }
    
    @Override
    public void close() {
        delegate.close();
    }
}

// Simple rate limiter implementation
class RateLimiter {
    private final long intervalNanos;
    private final AtomicLong nextFreeTicketNanos;
    
    public static RateLimiter create(double permitsPerSecond) {
        return new RateLimiter(permitsPerSecond);
    }
    
    private RateLimiter(double permitsPerSecond) {
        this.intervalNanos = (long)(1_000_000_000.0 / permitsPerSecond);
        this.nextFreeTicketNanos = new AtomicLong(System.nanoTime());
    }
    
    public boolean tryAcquire() {
        long now = System.nanoTime();
        long next = nextFreeTicketNanos.get();
        if (now >= next) {
            nextFreeTicketNanos.compareAndSet(next, now + intervalNanos);
            return true;
        }
        return false;
    }
}
```

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `log` | O(A × M) | A = appenders, M = message length |
| `format` | O(M) | M = message length |
| `getLogger` | O(1) | ConcurrentHashMap lookup |
| `addAppender` | O(1) | CopyOnWriteArrayList add |
| `setLevel` | O(1) | Simple assignment |

### Space Complexity

| Component | Space | Notes |
|-----------|-------|-------|
| Logger cache | O(L) | L = unique logger names |
| LogEvent | O(M) | M = message length + stack trace |
| File buffer | O(B) | B = buffer size (typically 8KB) |
| Appenders list | O(A) | A = number of appenders |

### Bottlenecks at Scale

**10x Usage (1K → 10K log messages/second):**
- Problem: File I/O becomes bottleneck, synchronous logging blocks threads, log file rotation overhead increases
- Solution: Implement async logging (background thread pool), use buffered file writes, optimize file rotation
- Tradeoff: Additional thread pool overhead, potential log loss on crash if not flushed

**100x Usage (1K → 100K log messages/second):**
- Problem: Single instance can't handle all logs, file I/O saturation, log storage exceeds disk capacity
- Solution: Shard logs by application/service, use distributed logging (Kafka, Fluentd), implement log aggregation service
- Tradeoff: Distributed system complexity, need log routing and aggregation infrastructure


### File Appender Specifics

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Write to file | O(M) | M = message length |
| Rotation check | O(1) | Size/date comparison |
| File rotation | O(1) | Rename + open new file |

