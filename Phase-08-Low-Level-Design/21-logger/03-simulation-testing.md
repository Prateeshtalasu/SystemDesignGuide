# 📝 Design a Logger - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - Logging Flow

```mermaid
flowchart TD
    A["Step 1: Check Level<br/>Logger level: INFO<br/>Message level: INFO<br/>INFO.priority (1) >= INFO.priority (1) → ENABLED"]
    B["Step 2: Create LogEvent<br/>LogEvent { timestamp, level=INFO, loggerName=&quot;com.app.UserService&quot;, message=&quot;User logged in: john_doe&quot;, threadName=&quot;main&quot; }"]
    C["Step 3: Format Message<br/>SimpleFormatter.format(event) →<br/>\"2024-01-15 10:30:45.123 [INFO] [main] UserService - User...\""]
    D["Step 4: Send to Appenders<br/>ConsoleAppender.append() → System.out.println(... )<br/>FileAppender.append() → writer.write(... )"]

    A --> B --> C --> D
```

<details>
<summary>ASCII diagram (reference)</summary>

```text
logger.info("User logged in: %s", username)

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Check Level                                             │
├─────────────────────────────────────────────────────────────────┤
│ Logger level: INFO                                              │
│ Message level: INFO                                             │
│ INFO.priority (1) >= INFO.priority (1) → ENABLED               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Create LogEvent                                         │
├─────────────────────────────────────────────────────────────────┤
│ LogEvent {                                                      │
│   timestamp: 2024-01-15T10:30:45.123                           │
│   level: INFO                                                   │
│   loggerName: "com.app.UserService"                            │
│   message: "User logged in: john_doe"                          │
│   threadName: "main"                                           │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Format Message                                          │
├─────────────────────────────────────────────────────────────────┤
│ SimpleFormatter.format(event) →                                 │
│ "2024-01-15 10:30:45.123 [INFO] [main] UserService - User..."  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Send to Appenders                                       │
├─────────────────────────────────────────────────────────────────┤
│ ConsoleAppender.append() → System.out.println(...)             │
│ FileAppender.append() → writer.write(...)                      │
└─────────────────────────────────────────────────────────────────┘
```

</details>

**Final State:**
```
Log event created and formatted successfully
Message sent to all configured appenders (console, file)
Logging operation completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Log Below Threshold

**Initial State:**
```
Logger: level = WARN
Message level: INFO (below threshold)
```

**Step-by-step:**

1. `logger.info("User logged in")`
   - Check level: Logger level = WARN, message level = INFO
   - INFO priority (1) < WARN priority (2) → below threshold
   - Level check fails
   - Message not processed
   - No LogEvent created
   - No formatting, no appender calls
   - Method returns immediately (short-circuit)

2. `logger.debug("Debug message")` (also below threshold)
   - DEBUG priority (0) < WARN priority (2) → below threshold
   - Message not processed
   - No output

3. `logger.warn("Warning message")` (at threshold)
   - WARN priority (2) >= WARN priority (2) → enabled
   - Message processed successfully

**Final State:**
```
Logger: No log events created for INFO/DEBUG
Only WARN and ERROR messages processed
Level filtering working correctly
```

---

### Scenario 3: Concurrency/Race Condition - Concurrent Logging

**Initial State:**
```
Logger: Multiple appenders (Console, File)
Thread A: Log INFO message
Thread B: Log WARN message (concurrent)
Thread C: Log ERROR message (concurrent)
All threads log simultaneously
```

**Step-by-step (simulating concurrent logging):**

**Thread A:** `logger.info("Thread A message")` at time T0
**Thread B:** `logger.warn("Thread B message")` at time T0 (concurrent)
**Thread C:** `logger.error("Thread C message")` at time T0 (concurrent)

1. **Thread A:** Enters `logger.info()` method
   - Checks level: INFO >= Logger level → enabled
   - Creates LogEvent (timestamp captured)
   - Formats message: "2024-01-15 10:30:45.123 [INFO] Thread A message"
   - Iterates appenders: ConsoleAppender, FileAppender
   - ConsoleAppender: System.out.println() (thread-safe)
   - FileAppender: FileWriter.write() (synchronized internally)
   - Returns

2. **Thread B:** Enters `logger.warn()` method (concurrent)
   - Checks level: WARN >= Logger level → enabled
   - Creates LogEvent (different timestamp)
   - Formats message: "2024-01-15 10:30:45.124 [WARN] Thread B message"
   - Iterates appenders
   - ConsoleAppender: System.out.println() (thread-safe)
   - FileAppender: FileWriter.write() (synchronized)
   - Returns

3. **Thread C:** Enters `logger.error()` method (concurrent)
   - Checks level: ERROR >= Logger level → enabled
   - Creates LogEvent
   - Formats message
   - Sends to appenders
   - Returns

**Final State:**
```
Console output: All three messages printed (order may vary)
File output: All three messages written (order preserved by synchronization)
No message corruption or loss
Thread-safe logging operations
```

---

## STEP 6: Edge Cases & Testing Strategy

### Edge Cases

| Category | Edge Case | Expected Behavior |
|----------|-----------|-------------------|
| Level | Log below threshold | Message not processed |
| Level | Log at threshold | Message processed |
| Level | Change level at runtime | Immediately effective |
| Format | Null message | Handle gracefully |
| Format | Message with newlines | Preserve formatting |
| Format | Exception with cause chain | Include full stack |
| Appender | No appenders configured | No output (silent) |
| Appender | Appender throws exception | Continue to other appenders |
| File | File rotation during write | Complete write, then rotate |
| File | Disk full | Log error to stderr |
| Thread | Concurrent logging | Thread-safe output |
| Factory | Same logger name twice | Return cached instance |

### Unit Tests

```java
// LogLevelTest.java
public class LogLevelTest {
    
    @Test
    void testLevelPriority() {
        assertTrue(LogLevel.ERROR.isEnabled(LogLevel.DEBUG));
        assertTrue(LogLevel.WARN.isEnabled(LogLevel.WARN));
        assertFalse(LogLevel.DEBUG.isEnabled(LogLevel.INFO));
    }
}

// SimpleFormatterTest.java
public class SimpleFormatterTest {
    
    @Test
    void testFormat() {
        LogEvent event = new LogEvent(LogLevel.INFO, "TestLogger", 
            "Test message", null);
        
        SimpleFormatter formatter = new SimpleFormatter();
        String result = formatter.format(event);
        
        assertTrue(result.contains("[INFO]"));
        assertTrue(result.contains("TestLogger"));
        assertTrue(result.contains("Test message"));
    }
}
```

```java
// LoggerTest.java
public class LoggerTest {
    
    @Test
    void testLevelFiltering() {
        Logger logger = new Logger("Test");
        MockAppender appender = new MockAppender();
        logger.addAppender(appender);
        logger.setLevel(LogLevel.WARN);
        
        logger.debug("Debug message");
        logger.info("Info message");
        logger.warn("Warn message");
        logger.error("Error message");
        
        assertEquals(2, appender.getMessages().size());
    }
    
    @Test
    void testMultipleAppenders() {
        Logger logger = new Logger("Test");
        MockAppender appender1 = new MockAppender();
        MockAppender appender2 = new MockAppender();
        logger.addAppender(appender1);
        logger.addAppender(appender2);
        
        logger.info("Test message");
        
        assertEquals(1, appender1.getMessages().size());
        assertEquals(1, appender2.getMessages().size());
    }
}
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

```java
public class BufferedAppender implements LogAppender {
    private final LogAppender delegate;
    private final List<String> buffer;
    private final int bufferSize;
    
    @Override
    public void append(LogEvent event, String message) {
        synchronized (buffer) {
            buffer.add(message);
            if (buffer.size() >= bufferSize) {
                flush();
            }
        }
    }
    
    public void flush() {
        synchronized (buffer) {
            for (String msg : buffer) {
                delegate.append(null, msg);
            }
            buffer.clear();
        }
    }
}
```

### Q2: How would you implement structured logging?

```java
public class StructuredLogger {
    public LogBuilder info() {
        return new LogBuilder(LogLevel.INFO);
    }
    
    public class LogBuilder {
        private final Map<String, Object> fields = new LinkedHashMap<>();
        
        public LogBuilder field(String key, Object value) {
            fields.put(key, value);
            return this;
        }
        
        public void msg(String message) {
            // Log with all fields
        }
    }
}

// Usage
logger.info()
    .field("userId", 123)
    .field("action", "login")
    .field("duration", 45)
    .msg("User action completed");
```

### Q3: How would you implement log sampling?

```java
public class SamplingAppender implements LogAppender {
    private final LogAppender delegate;
    private final double sampleRate;
    private final Random random;
    
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
}
```

### Q4: What would you do differently with more time?

1. **Add async logging** - Non-blocking writes
2. **Add log correlation** - Trace IDs across services
3. **Add metrics** - Log counts, latencies
4. **Add compression** - For archived logs
5. **Add encryption** - For sensitive logs
6. **Add remote appender** - Send to log aggregator

